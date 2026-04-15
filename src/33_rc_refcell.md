# Розділ 33. Rc\<T\> та RefCell\<T\>: спільне володіння

## Анотація

Ownership у Rust — один власник на одне значення. Box підкреслює це: одне значення на купі, один Box, один власник. Але деякі структури даних не вписуються у цю модель. Граф: кілька вузлів посилаються на один спільний вузол. Кілька компонентів агента (навігація, сенсори, логер) діляться однією конфігурацією. Два віджети UI відображають один і той самий об'єкт. Для таких випадків Rust пропонує Rc\<T\> — reference-counted pointer, що дозволяє кільком "власникам" ділити одне значення. А RefCell\<T\> додає внутрішню мутабельність: можливість змінювати дані, навіть коли існують лише іммутабельні посилання — з перевіркою правил у runtime замість compile-time. Разом Rc\<RefCell\<T\>\> створюють потужний патерн для спільних мутабельних даних в однопотоковому коді.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити, коли ownership "один власник" недостатній.
2. Використати Rc\<T\> для спільного володіння з підрахунком посилань.
3. Пояснити різницю між Rc::clone та T::clone.
4. Використати RefCell\<T\> для внутрішньої мутабельності.
5. Пояснити, чому RefCell перевіряє правила borrowing у runtime.
6. Застосувати патерн Rc\<RefCell\<T\>\> для спільних мутабельних даних.
7. Використати Weak\<T\> для уникнення циклічних посилань.

---

## Ключові терміни

**Rc\<T\> (Reference Counting)** — smart pointer із підрахунком посилань. Кілька Rc можуть вказувати на одне значення. Значення звільняється, коли останній Rc знищується.

**RefCell\<T\>** — контейнер з внутрішньою мутабельністю. Дозволяє отримати &mut T через &self, перевіряючи правила borrowing у runtime.

**Interior mutability (внутрішня мутабельність)** — патерн, де зовнішній інтерфейс іммутабельний (&self), але внутрішній стан може змінюватись.

**Weak\<T\>** — "слабке" посилання, що не впливає на підрахунок і не запобігає звільненню. Для розриву циклічних посилань.

**Cell\<T\>** — як RefCell, але для Copy-типів. Замість borrow/borrow_mut — get/set. Без runtime panic.

---

## Мотиваційний кейс

Система керування БПЛА має три підсистеми: навігація, сенсори та комунікація. Усі три потребують доступу до конфігурації місії (цільові координати, дозволена зона, параметри зв'язку). Конфігурація одна, а підсистем три. З ownership "один власник" — лише одна підсистема може володіти конфігурацією. Інші дві отримують лише посилання &Config, що прив'язує їх lifetime до власника. Це ускладнює архітектуру: хто створює Config, хто зберігає, як гарантувати, що посилання валідне?

Rc\<Config\> вирішує це: кожна підсистема має свою "копію" Rc (насправді — свій лічильник), і Config живе, поки хоча б одна підсистема жива. А якщо конфігурація може змінюватись під час місії (оновлення цілі від координатора) — Rc\<RefCell\<Config\>\> дає спільний мутабельний доступ.

---

## 33.1. Rc\<T\>: кілька власників одного значення

Rc (Reference Counting) дозволяє кільком змінним "володіти" одним значенням. Кожен `Rc::clone` збільшує лічильник посилань. Кожне знищення Rc — зменшує. Коли лічильник досягає нуля — значення звільняється.

```rust
use std::rc::Rc;

fn main() {
    let config = Rc::new(String::from("Місія: патрулювання зони A"));
    println!("Лічильник після створення: {}", Rc::strong_count(&config)); // 1
    
    let nav_config = Rc::clone(&config);    // лічильник: 2
    let sensor_config = Rc::clone(&config); // лічильник: 3
    
    println!("Лічильник: {}", Rc::strong_count(&config)); // 3
    
    println!("Навігація бачить: {}", nav_config);
    println!("Сенсори бачать: {}", sensor_config);
    
    drop(sensor_config); // лічильник: 2
    println!("Після drop sensor: {}", Rc::strong_count(&config)); // 2
    
    drop(nav_config);    // лічильник: 1
    println!("Після drop nav: {}", Rc::strong_count(&config)); // 1
    
} // config знищується, лічильник: 0 → String звільняється
```

`Rc::clone(&config)` — не глибоке копіювання! Це дешева операція: створюється новий Rc, що вказує на ті самі дані, і лічильник збільшується на 1. Самі дані (String "Місія: патрулювання...") існують в одному екземплярі на купі. Усі три Rc бачать один і той самий рядок.

Чому `Rc::clone`, а не `config.clone()`? Технічно обидва працюють однаково. Але конвенція Rust — писати `Rc::clone(&x)` замість `x.clone()`, щоб підкреслити: це не глибоке копіювання даних (яке може бути дорогим), а лише збільшення лічильника (дешева операція). При читанні коду `Rc::clone` одразу каже "спільне володіння", а `x.clone()` може бути помилково інтерпретоване як "копіювання всіх даних".

Модель пам'яті:

```
Стек:                            Купа:
┌───────────────┐               ┌───────────────────────┐
│ config: ptr ──┼───┐           │ strong_count: 3       │
├───────────────┤   ├──────────→│ data: "Місія: ..."    │
│ nav_config: ptr──┘           │                       │
├───────────────┤   ┌──────────→│                       │
│ sensor_config: ptr┘           └───────────────────────┘
└───────────────┘
```

Три Rc на стеку, кожен — вказівник на один блок на купі. Блок містить лічильник (strong_count = 3) та самі дані.

Обмеження Rc: надає лише іммутабельний доступ. `Rc<T>` дає `&T`, а не `&mut T`. Кілька Rc вказують на одні дані — мутація через будь-який з них порушила б правило "або кілька &T, або один &mut T". Для мутації потрібен RefCell.

Друге обмеження: Rc не є Send — не можна передати в інший потік. Для багатопотоковості — Arc (Розділ 37).

---

## 33.2. RefCell\<T\>: внутрішня мутабельність

Нормальні правила Rust: щоб змінити значення, потрібно &mut T, що потребує mut змінної та ексклюзивного доступу. RefCell порушує це правило контрольовано: дозволяє отримати &mut T через &self, перевіряючи правила borrowing у runtime.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    
    // Іммутабельне позичання через borrow()
    {
        let borrowed = data.borrow(); // повертає Ref<Vec<i32>> — як &Vec
        println!("Читання: {:?}", *borrowed);
    } // borrowed знищується — позичання завершується
    
    // Мутабельне позичання через borrow_mut()
    {
        let mut borrowed = data.borrow_mut(); // повертає RefMut<Vec<i32>> — як &mut Vec
        borrowed.push(4);
        println!("Після зміни: {:?}", *borrowed);
    } // borrowed знищується — позичання завершується
    
    println!("Фінал: {:?}", data.borrow());
}
```

`borrow()` повертає `Ref<T>` — guard, що поводиться як &T. `borrow_mut()` повертає `RefMut<T>` — guard, що поводиться як &mut T. Правила ті самі, що й для звичайних посилань: кілька borrow() одночасно — OK. Один borrow_mut() без borrow() — OK. borrow() + borrow_mut() одночасно — ПАНІКА у runtime.

Чому runtime, а не compile-time? Тому що компілятор не завжди може відстежити, коли позичання починається і закінчується — особливо коли RefCell "захований" у Rc, переданий у кілька місць, і позичання залежить від runtime логіки. RefCell переносить перевірку з компіляції у runtime, даючи гнучкість ціною можливої паніки.

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(42);
    
    let a = data.borrow();     // іммутабельне позичання
    let b = data.borrow();     // ще одне — OK
    println!("{}, {}", *a, *b);
    
    // let c = data.borrow_mut(); // ПАНІКА! вже є іммутабельні позичання
    
    drop(a);
    drop(b);
    
    let mut c = data.borrow_mut(); // тепер OK — попередні позичання завершились
    *c = 100;
}
```

Правило: мінімізуйте scope позичань RefCell. Тримайте Ref/RefMut якомога коротше — це зменшує ризик паніки. Ідеально: borrow → використати → drop в тому ж блоці.

---

## 33.3. Rc\<RefCell\<T\>\>: спільні мутабельні дані

Комбінація Rc (кілька власників) + RefCell (мутабельність) дає патерн "кілька компонентів можуть читати та змінювати одні дані":

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct MissionConfig {
    target: (f64, f64),
    max_altitude: f64,
    active: bool,
}

struct Navigator {
    config: Rc<RefCell<MissionConfig>>,
}

struct SensorManager {
    config: Rc<RefCell<MissionConfig>>,
}

impl Navigator {
    fn check_target(&self) {
        let config = self.config.borrow(); // іммутабельне позичання
        println!("Навігація: ціль ({:.1}, {:.1})", config.target.0, config.target.1);
    }
}

impl SensorManager {
    fn update_target(&self, new_target: (f64, f64)) {
        let mut config = self.config.borrow_mut(); // мутабельне позичання
        config.target = new_target;
        println!("Сенсори: ціль оновлена на ({:.1}, {:.1})", new_target.0, new_target.1);
    }
}

fn main() {
    let config = Rc::new(RefCell::new(MissionConfig {
        target: (100.0, 200.0),
        max_altitude: 500.0,
        active: true,
    }));
    
    let nav = Navigator { config: Rc::clone(&config) };
    let sensors = SensorManager { config: Rc::clone(&config) };
    
    nav.check_target();           // Навігація: ціль (100.0, 200.0)
    sensors.update_target((150.0, 250.0));  // оновлює config
    nav.check_target();           // Навігація: ціль (150.0, 250.0) — бачить зміну!
    
    println!("Лічильник: {}", Rc::strong_count(&config)); // 3
}
```

Navigator та SensorManager "ділять" одну MissionConfig. SensorManager змінює target — Navigator одразу бачить зміну, бо обидва вказують на одні дані. Це неможливо зі звичайним ownership (один власник) чи &/&mut (статичні правила borrowing).

Ключове зауваження: Rc\<RefCell\<T\>\> — патерн для однопотокового коду. У багатопотоковому — Arc\<Mutex\<T\>\> (Розділ 37). Rc не Send, RefCell не Sync — компілятор заборонить передачу в інший потік.

---

## 33.4. Weak\<T\>: розрив циклічних посилань

Rc підраховує посилання. Якщо два Rc вказують одне на одного — циклічне посилання. Лічильник кожного ніколи не досягне нуля, бо інший тримає його "живим". Результат — memory leak, що Rust зазвичай запобігає.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Agent {
    id: String,
    // Командир знає про підлеглих, підлеглі — про командира
    subordinates: Vec<Rc<RefCell<Agent>>>,
    commander: Option<Weak<RefCell<Agent>>>, // Weak — не збільшує лічильник
}

fn main() {
    let commander = Rc::new(RefCell::new(Agent {
        id: String::from("CMD-01"),
        subordinates: vec![],
        commander: None,
    }));
    
    let scout = Rc::new(RefCell::new(Agent {
        id: String::from("SCOUT-01"),
        subordinates: vec![],
        commander: Some(Rc::downgrade(&commander)), // Weak посилання
    }));
    
    // Командир знає про підлеглого (сильне посилання)
    commander.borrow_mut().subordinates.push(Rc::clone(&scout));
    
    // Підлеглий знає про командира (слабке посилання)
    if let Some(cmd_ref) = &scout.borrow().commander {
        if let Some(cmd) = cmd_ref.upgrade() { // Weak → Option<Rc>
            println!("Командир: {}", cmd.borrow().id);
        }
    }
    
    println!("Commander strong_count: {}", Rc::strong_count(&commander)); // 1 (не 2!)
    println!("Scout strong_count: {}", Rc::strong_count(&scout)); // 2 (commander + main)
}
```

`Rc::downgrade(&rc)` створює Weak — "слабке" посилання, що не збільшує strong_count. Коли всі сильні Rc знищуються — значення звільняється, навіть якщо Weak ще існують. `weak.upgrade()` повертає `Option<Rc<T>>`: Some, якщо значення ще живе, None — якщо вже звільнене.

Правило для графів: "батько → дитина" — Rc (сильне, тримає живим). "Дитина → батько" — Weak (слабке, не створює цикл). Це гарантує, що видалення батька звільняє всіх дітей (якщо жодних інших Rc на них немає).

---

## 33.5. Cell\<T\>: простіша альтернатива для Copy-типів

RefCell потрібен для типів із ownership (String, Vec). Для Copy-типів (i32, f64, bool) є простіший Cell — без borrow/borrow_mut, без runtime panic:

```rust
use std::cell::Cell;

struct AgentStats {
    steps: Cell<u32>,
    distance: Cell<f64>,
}

impl AgentStats {
    fn new() -> Self {
        AgentStats {
            steps: Cell::new(0),
            distance: Cell::new(0.0),
        }
    }
    
    fn record_step(&self, dist: f64) {
        // &self — іммутабельне! Але Cell дозволяє змінювати
        self.steps.set(self.steps.get() + 1);
        self.distance.set(self.distance.get() + dist);
    }
    
    fn report(&self) {
        println!("Кроків: {}, відстань: {:.1}", 
                 self.steps.get(), self.distance.get());
    }
}

fn main() {
    let stats = AgentStats::new(); // не mut!
    stats.record_step(5.0);
    stats.record_step(3.2);
    stats.record_step(7.1);
    stats.report(); // Кроків: 3, відстань: 15.3
}
```

Cell працює через get/set замість borrow/borrow_mut. Get копіює значення (тому T: Copy), set замінює. Жодних guard-ів, жодних runtime panic. Обмеження: лише для Copy-типів (числа, bool, char). Для String, Vec — використовуйте RefCell.

Cell ідеальний для лічильників, прапорців та метрик — даних, що часто змінюються через &self.

---

## 33.6. Практика: компоненти агента зі спільними даними

```rust
use std::rc::Rc;
use std::cell::RefCell;
use std::collections::HashMap;

type SharedMap = Rc<RefCell<HashMap<String, (f64, f64)>>>;

struct Navigator {
    world_map: SharedMap,
}

struct SensorModule {
    world_map: SharedMap,
}

struct Logger {
    world_map: SharedMap,
}

impl SensorModule {
    fn detect_object(&self, id: &str, pos: (f64, f64)) {
        self.world_map.borrow_mut().insert(String::from(id), pos);
        println!("Sensor: виявлено {} на ({:.0}, {:.0})", id, pos.0, pos.1);
    }
}

impl Navigator {
    fn nearest_object(&self, from: (f64, f64)) -> Option<String> {
        let map = self.world_map.borrow();
        map.iter()
            .min_by(|(_, a), (_, b)| {
                let da = ((a.0 - from.0).powi(2) + (a.1 - from.1).powi(2)).sqrt();
                let db = ((b.0 - from.0).powi(2) + (b.1 - from.1).powi(2)).sqrt();
                da.partial_cmp(&db).unwrap()
            })
            .map(|(id, _)| id.clone())
    }
}

impl Logger {
    fn dump(&self) {
        let map = self.world_map.borrow();
        println!("Карта ({} об'єктів):", map.len());
        for (id, (x, y)) in map.iter() {
            println!("  {} → ({:.0}, {:.0})", id, x, y);
        }
    }
}

fn main() {
    let map: SharedMap = Rc::new(RefCell::new(HashMap::new()));
    
    let sensor = SensorModule { world_map: Rc::clone(&map) };
    let nav = Navigator { world_map: Rc::clone(&map) };
    let log = Logger { world_map: Rc::clone(&map) };
    
    sensor.detect_object("OBJ-01", (10.0, 20.0));
    sensor.detect_object("OBJ-02", (50.0, 30.0));
    sensor.detect_object("OBJ-03", (15.0, 25.0));
    
    if let Some(nearest) = nav.nearest_object((12.0, 22.0)) {
        println!("Найближчий: {}", nearest);
    }
    
    log.dump();
}
```

Три компоненти ділять одну HashMap через Rc\<RefCell\<...\>\>. SensorModule додає об'єкти (borrow_mut). Navigator шукає найближчий (borrow). Logger виводить все (borrow). Кожен компонент працює через &self — жодного &mut self, що спрощує архітектуру.

---

## 33.7. Prompt Engineering: вибір smart pointer

### Промпт-шаблон

```
Моя архітектура:
- Компонент A читає дані X
- Компонент B читає та змінює X
- Компонент C читає X
- Все в одному потоці

Який smart pointer обрати: Box, Rc, Rc<RefCell>? 
Покажи код та поясни trade-offs.
```

### Вправа з PE

Опишіть AI архітектуру свого агента: які компоненти, хто що читає/змінює, один потік чи кілька. Попросіть обрати комбінацію smart pointers. Оцініть: чи не запропонував AI зайву складність (Rc де достатньо &)?

---

## Лабораторна робота No33

### Мета

Реалізувати систему компонентів агента зі спільними даними.

### Завдання базового рівня

1. `Rc<RefCell<HashMap<...>>>` — спільна карта для 2+ компонентів.
2. Один компонент додає дані, інший — читає та аналізує.
3. Демонстрація: зміна через один компонент видима іншому.
4. Тести з перевіркою strong_count та коректності даних.

### Варіанти

**A.** Граф із Weak: struct Node з children (Vec<Rc<...>>) та parent (Weak<...>). Побудувати дерево з 3 рівнів. Перевірити, що drop кореня звільняє всіх нащадків.

**B.** Кеш із Cell: struct Cache з `hits: Cell<u32>`, `misses: Cell<u32>`, `data: RefCell<HashMap<...>>`. Метод `get(&self)` збільшує hits/misses через Cell, читає/записує дані через RefCell.

**C.** Observer pattern: Rc<RefCell<Vec<Box<dyn Fn(&Event)>>>> — список обробників подій. Метод subscribe додає обробник, emit викликає всі.

### Критерії

| Критерій | Бал |
|----------|-----|
| Rc для спільного володіння | 20 |
| RefCell з коректними borrow/borrow_mut | 20 |
| Мінімальний scope позичань | 15 |
| Тести (включаючи strong_count) | 20 |
| Обґрунтований вибір Rc vs Box vs & | 15 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `already borrowed: BorrowMutError`

Runtime паніка — одночасне borrow та borrow_mut:

```rust
let data = RefCell::new(42);
let a = data.borrow();
let b = data.borrow_mut(); // ПАНІКА!
```

Виправлення: drop(a) перед borrow_mut, або реструктурувати код, щоб позичання не перекривались.

### Помилка 2: `Rc<T> cannot be sent between threads safely`

```rust
let rc = Rc::new(42);
std::thread::spawn(move || println!("{}", rc)); // ПОМИЛКА: Rc не Send
```

Виправлення: використати Arc замість Rc для багатопотоковості (Розділ 37).

### Помилка 3: memory leak через циклічні Rc

Два Rc вказують одне на одного — strong_count ніколи не стає 0. Rust не має garbage collector — це справжній memory leak.

Виправлення: замінити одне з посилань на Weak через Rc::downgrade.

### Помилка 4: `cannot borrow as mutable` для Rc

```rust
let rc = Rc::new(vec![1, 2, 3]);
rc.push(4); // ПОМИЛКА: Rc дає лише &T, не &mut T
```

Виправлення: `Rc::new(RefCell::new(vec![...]))` та `rc.borrow_mut().push(4)`.

---

## Додатково

### Таблиця вибору smart pointer

| Ситуація | Smart pointer |
|----------|--------------|
| Один власник, дані на купі | Box\<T\> |
| Кілька власників, лише читання, один потік | Rc\<T\> |
| Кілька власників, читання + запис, один потік | Rc\<RefCell\<T\>\> |
| Лічильник/прапорець через &self | Cell\<T\> (для Copy) |
| Кілька власників, кілька потоків | Arc\<T\> (Розділ 37) |
| Кілька власників + запис, кілька потоків | Arc\<Mutex\<T\>\> (Розділ 37) |
| Зворотне посилання (уникнення циклів) | Weak\<T\> |

### Compile-time vs runtime borrowing

Звичайні посилання (&T, &mut T): правила перевіряються компілятором. Якщо код компілюється — він безпечний. Помилки — при компіляції.

RefCell: ті самі правила, але перевіряються у runtime. Код компілюється, але може панікувати, якщо правила порушені. Це свідомий trade-off: гнучкість архітектури ціною можливої runtime panic.

Правило: використовуйте RefCell лише коли звичайні посилання неможливі (наприклад, interior mutability через &self, або Rc\<RefCell\<T\>\> для спільних даних). Не використовуйте RefCell "для зручності" — якщо можна зробити через &mut, це завжди краще.

---

## Контрольні запитання

### Рівень 1

1. Чим Rc::clone відрізняється від звичайного clone?

Відповідь: Rc::clone лише збільшує лічильник посилань (дешева операція, наносекунди). Звичайний clone копіює всі дані (може бути дорогим). Обидва технічно ідентичні для Rc, але конвенція — писати Rc::clone для ясності.

2. Що станеться при одночасних borrow() та borrow_mut() на RefCell?

Відповідь: runtime паніка з повідомленням "already borrowed: BorrowMutError". Це runtime-аналог помилки компіляції "cannot borrow as mutable because also borrowed as immutable".

### Рівень 2

3. Чому Rc не є Send (не можна передати в інший потік)?

Відповідь: лічильник посилань Rc не атомарний — збільшення/зменшення не thread-safe. Два потоки, що одночасно клонують/знищують Rc, можуть пошкодити лічильник (data race). Arc використовує атомарні операції — повільніше, але thread-safe.

4. Навіщо Weak, якщо є Rc?

Відповідь: Rc створює "сильне" посилання, що тримає значення живим. Якщо два Rc вказують одне на одного (цикл) — лічильники ніколи не стануть 0, memory leak. Weak — "слабке" посилання, не впливає на strong_count. Значення звільняється, коли всі сильні Rc знищені, навіть якщо Weak ще існують.

### Рівень 3

5. Реалізуйте shared counter через Rc\<Cell\<u32\>\> з методами increment та get через &self.

Відповідь:

```rust
use std::rc::Rc;
use std::cell::Cell;

struct SharedCounter {
    count: Rc<Cell<u32>>,
}

impl SharedCounter {
    fn new() -> Self {
        SharedCounter { count: Rc::new(Cell::new(0)) }
    }
    fn clone_ref(&self) -> Self {
        SharedCounter { count: Rc::clone(&self.count) }
    }
    fn increment(&self) {
        self.count.set(self.count.get() + 1);
    }
    fn get(&self) -> u32 {
        self.count.get()
    }
}
```

### Рівень 4

6. Порівняйте архітектури: (a) один власник + &/&mut, (b) Rc\<RefCell\<T\>\>, (c) клонування даних. Коли що обирати?

Відповідь: (a) найшвидший, compile-time перевірки, але обмежує архітектуру (lifetime залежності). (b) гнучкий, runtime перевірки, ризик паніки, трохи повільніший (підрахунок + runtime borrow check). (c) найпростіший, кожен компонент має свою копію, але зміни не видимі іншим і витрачає пам'ять. Вибір: (a) за замовчуванням. (b) коли кілька компонентів мають ділити мутабельний стан і lifetime-архітектура занадто складна. (c) коли дані малі і кожен компонент може мати свою незалежну копію.

---

## Резюме

Rc\<T\> — спільне володіння через підрахунок посилань. Кілька Rc вказують на одне значення; звільняється при strong_count = 0. Rc::clone — дешева (лічильник), не глибока копія. Лише однопотоковий; лише іммутабельний доступ.

RefCell\<T\> — внутрішня мутабельність. borrow() → Ref (як &T), borrow_mut() → RefMut (як &mut T). Правила ті самі, але перевіряються у runtime — паніка при порушенні. Cell\<T\> — простіша альтернатива для Copy-типів (get/set, без паніки).

Rc\<RefCell\<T\>\> — спільні мутабельні дані в однопотоковому коді. Кілька компонентів читають та змінюють одне значення. Arc\<Mutex\<T\>\> — багатопотоковий аналог (Розділ 37).

Weak\<T\> — слабке посилання для розриву циклів. Не збільшує strong_count, upgrade() повертає Option.

---

## Що далі

Rc та RefCell працюють з ownership: хто скільки разів володіє і хто може змінювати. Але є ще одне "ownership-питання", яке ми торкались, але не розглядали глибоко: lifetime — як довго живуть посилання. У Розділі 34 ви вивчите анотації lifetime (`'a`), правила elision, та `'static`. Це дозволить створювати структури з посиланнями (`AgentView<'a>` — "вікно" в стан агента без копіювання) та розуміти, чому компілятор іноді вимагає явних lifetime-анотацій.
