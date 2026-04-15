# Розділ 16. Struct: моделювання стану агента

## Анотація

До цього моменту ми описували агента через окремі змінні: `pos_x`, `pos_y`, `battery`, `heading`, `airborne`. У Розділі 12 функція `cmd_forward` приймала п'ять параметрів, і кожна нова змінна стану додавала параметр до десятка функцій. Другий агент потребував дублювання всіх змінних із суфіксом `_2`. Структури (struct) вирішують цю проблему: пов'язані дані об'єднуються в єдиний тип, а функції стають методами, прив'язаними до цього типу. Замість `print_status(id, x, y, battery, heading)` — `agent.print_status()`. Замість п'яти окремих змінних — один об'єкт `Agent`. Цей розділ — поворотний момент: агент перестає бути "купою змінних" і стає справжнім об'єктом.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Створити структуру з іменованими полями та ініціалізувати її.
2. Пояснити різницю між named struct, tuple struct та unit struct.
3. Реалізувати методи через блок `impl` з `&self`, `&mut self` та `self`.
4. Створити конструктор `new()` як асоційовану функцію.
5. Застосувати ownership та borrowing до структур.
6. Використати struct update syntax `..other`.

---

## Ключові терміни

**Struct (структура)** — користувацький тип, що групує пов'язані дані в іменовані поля.

**Field (поле)** — іменований елемент структури з визначеним типом.

**Method (метод)** — функція, прив'язана до типу через `impl`, що приймає `self`.

**Associated function (асоційована функція)** — функція в `impl`, що не приймає `self` (наприклад, конструктор).

**Self** — псевдонім для типу, всередині якого знаходиться `impl`.

**impl block** — блок, де визначаються методи та асоційовані функції для типу.

**Tuple struct** — структура з неіменованими полями, доступ через індекс.

**Unit struct** — структура без полів.

---

## Мотиваційний кейс

У великих проєктах розрізнені змінні стають некерованими. Уявіть систему з 50 агентами, кожен з 10 параметрами — це 500 окремих змінних. Функція, що оновлює позицію одного агента, приймає десяток параметрів, і легко переплутати, які з них належать якому агенту. У 2014 році баг у системі керування дронами Amazon Prime Air стався саме через плутанину з параметрами: координати одного дрона потрапили у функцію іншого. Структури усувають цю проблему: кожен агент — окремий об'єкт, і його дані неможливо переплутати з даними іншого.

---

## 16.1. Від окремих змінних до структури

### Проблема: розкидані дані

Ось як виглядав стан агента в Розділі 12:

```rust
let mut pos_x: f64 = 0.0;
let mut pos_y: f64 = 0.0;
let mut battery: u8 = 100;
let mut heading: u16 = 0;
let mut airborne: bool = false;
```

П'ять змінних, ніяк не зв'язаних між собою. Компілятор не знає, що `pos_x` та `pos_y` — це координати одного агента. Ви можете випадково передати `pos_x` одного агента разом із `battery` іншого — і компілятор не попередить.

### Рішення: структура

Структура групує пов'язані дані в єдиний тип:

```rust
struct Agent {
    pos_x: f64,
    pos_y: f64,
    battery: u8,
    heading: u16,
    airborne: bool,
}
```

Тепер усі дані агента — в одному місці. Створення агента:

```rust
fn main() {
    let agent = Agent {
        pos_x: 0.0,
        pos_y: 0.0,
        battery: 100,
        heading: 0,
        airborne: false,
    };
    
    println!("Позиція: ({}, {})", agent.pos_x, agent.pos_y);
    println!("Батарея: {}%", agent.battery);
}
```

Доступ до полів — через крапку: `agent.pos_x`, `agent.battery`. Кожне поле має тип, і компілятор перевіряє типи при кожному зверненні.

### Синтаксис оголошення

Структура оголошується ключовим словом `struct`, за яким ім'я (CamelCase, з великої літери) та блок з полями:

```rust
struct Point {
    x: f64,
    y: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}
```

Імена структур — CamelCase: `Point`, `Agent`, `BatteryStatus`, `SensorReading`. Імена полів — snake_case: `pos_x`, `battery_level`, `is_active`. Кожне поле має явний тип. Порядок полів при оголошенні не має значення для функціональності, але впливає на розташування у пам'яті.

### Простий приклад: Point

Почнемо з найпростішої структури — точки у двовимірному просторі:

```rust
struct Point {
    x: f64,
    y: f64,
}

fn distance(p1: &Point, p2: &Point) -> f64 {
    let dx = p2.x - p1.x;
    let dy = p2.y - p1.y;
    (dx * dx + dy * dy).sqrt()
}

fn main() {
    let base = Point { x: 0.0, y: 0.0 };
    let target = Point { x: 3.0, y: 4.0 };
    
    let dist = distance(&base, &target);
    println!("Відстань: {:.1}", dist);  // 5.0
}
```

Функція `distance` приймає два посилання на Point (`&Point`) — позичає, не забирає. Після виклику обидві точки доступні. Порівняйте з `distance(x1, y1, x2, y2)` — чотири параметри проти двох. З Point код зрозуміліший: ви передаєте "точку", а не "чотири числа, що мають бути координатами".

---

## 16.2. Мутабельність структури

Щоб змінити поле структури, вся структура повинна бути `mut`:

```rust
fn main() {
    let mut point = Point { x: 0.0, y: 0.0 };
    point.x = 10.0;
    point.y = 20.0;
    println!("({}, {})", point.x, point.y);  // (10, 20)
}
```

Rust не дозволяє зробити окремі поля мутабельними — або вся структура `mut`, або вся іммутабельна. Це спрощує міркування про код: якщо ви бачите `let agent = Agent { ... }` (без mut) — жодне поле не може змінитись.

Через `&mut` у функціях:

```rust
fn move_point(p: &mut Point, dx: f64, dy: f64) {
    p.x += dx;
    p.y += dy;
}

fn main() {
    let mut pos = Point { x: 0.0, y: 0.0 };
    move_point(&mut pos, 5.0, 3.0);
    println!("({}, {})", pos.x, pos.y);  // (5, 3)
}
```

Замість `fn move_point(x: &mut f64, y: &mut f64, ...)` — один параметр `p: &mut Point`. Набагато чистіше.

---

## 16.3. Методи через impl

### Від функцій до методів

До цього ми писали функції, що приймають структуру як параметр: `distance(&p1, &p2)`, `move_point(&mut pos, 5.0, 3.0)`. Rust дозволяє прив'язати функції до типу через блок `impl` — вони стають методами:

```rust
struct Point {
    x: f64,
    y: f64,
}

impl Point {
    fn distance_to(&self, other: &Point) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }
    
    fn translate(&mut self, dx: f64, dy: f64) {
        self.x += dx;
        self.y += dy;
    }
}

fn main() {
    let mut pos = Point { x: 0.0, y: 0.0 };
    let target = Point { x: 3.0, y: 4.0 };
    
    println!("Відстань: {:.1}", pos.distance_to(&target));  // 5.0
    
    pos.translate(1.0, 2.0);
    println!("Нова позиція: ({}, {})", pos.x, pos.y);  // (1, 2)
}
```

`distance_to` та `translate` — методи Point. Виклик через крапку: `pos.distance_to(&target)` замість `distance_to(&pos, &target)`. Перший параметр `&self` або `&mut self` — це посилання на екземпляр, для якого викликається метод.

### Три варіанти self

Перший параметр методу визначає, як метод взаємодіє зі структурою. Це прямо пов'язано з ownership та borrowing з Розділів 13–15:

`&self` — іммутабельне позичання. Метод читає поля, але не може їх змінювати. Це аналог `&T` з Розділу 15 для звичайних функцій. Використовуйте для методів, що обчислюють, перевіряють або виводять інформацію.

`&mut self` — мутабельне позичання. Метод може читати та змінювати поля. Це аналог `&mut T`. Використовуйте для методів, що модифікують стан об'єкта.

`self` — передача за значенням (move). Метод забирає володіння структурою. Після виклику оригінал недоступний. Рідко використовується — зазвичай для методів-трансформаторів, що "споживають" об'єкт і повертають щось нове.

Вибір між ними — той самий принцип, що і для функцій: починайте з `&self` (мінімальні права) і розширюйте лише за потреби.

```rust
impl Point {
    // &self — лише читання
    fn is_origin(&self) -> bool {
        self.x == 0.0 && self.y == 0.0
    }
    
    // &mut self — модифікація
    fn reset(&mut self) {
        self.x = 0.0;
        self.y = 0.0;
    }
    
    // self — споживання (рідко)
    fn into_tuple(self) -> (f64, f64) {
        (self.x, self.y)  // Point знищується, кортеж повертається
    }
}

fn main() {
    let mut p = Point::new(3.0, 4.0);
    
    println!("На початку? {}", p.is_origin());  // false — &self, p доступна
    
    p.reset();                                    // &mut self, p змінена
    println!("Після reset? {}", p.is_origin());  // true
    
    let coords = p.into_tuple();                 // self — p переміщена!
    // println!("{}", p.x);                       // ПОМИЛКА: p переміщена
    println!("Координати: {:?}", coords);        // OK: (0.0, 0.0)
}
```

Зверніть увагу на `into_tuple(self)`: після виклику `p` більше не існує — метод "спожив" структуру. Ім'я `into_` — конвенція Rust для методів, що споживають об'єкт: `into_string()`, `into_iter()`, `into_inner()`.
```

### Асоційовані функції та конструктор new()

Функція в `impl`, що не приймає `self` — це асоційована функція. Найчастіший випадок — конструктор:

```rust
impl Point {
    fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }
    
    fn origin() -> Self {
        Self { x: 0.0, y: 0.0 }
    }
}

fn main() {
    let a = Point::new(3.0, 4.0);    // виклик через ::
    let b = Point::origin();
    println!("a: ({}, {}), b: ({}, {})", a.x, a.y, b.x, b.y);
}
```

`Self` — псевдонім для типу структури (тут `Point`). `Self { x, y }` — скорочений синтаксис (field init shorthand): якщо ім'я змінної збігається з ім'ям поля, можна не писати `x: x`.

Асоційовані функції викликаються через `::` (а не крапку): `Point::new(3.0, 4.0)`, `Point::origin()`.

---

## 16.4. Ownership у структурах

Структура володіє своїми полями. Це фундаментальний принцип Rust, що випливає з правил ownership (Розділ 13). Коли структура знищується (drop) — всі її поля теж знищуються. Це рекурсивний процес: Agent Drop → String `id` Drop → дані купи звільнені, u8 `battery` — просто зникає зі стеку.

```rust
struct Agent {
    id: String,       // String — на купі, структура володіє
    battery: u8,      // u8 — на стеку
}

fn main() {
    let agent = Agent {
        id: String::from("БПЛА-07"),
        battery: 85,
    };
    println!("{}: {}%", agent.id, agent.battery);
}  // agent Drop → id (String) Drop → пам'ять купи звільнена
```

Коли `agent` виходить із scope, Rust автоматично знищує структуру. Кожне поле знищується окремо: String звільняє свої дані на купі, u8 просто зникає зі стеку. Це все відбувається без участі програміста — компілятор вставляє виклики drop автоматично.

### Move всієї структури

Якщо структура має хоча б одне не-Copy поле (наприклад, String), вся структура стає не-Copy. Це логічно: побітова копія Agent скопіювала б вказівник String, створивши два "власники" одних даних на купі — double free при drop. Тому Rust робить move:

```rust
fn main() {
    let agent = Agent {
        id: String::from("БПЛА-07"),
        battery: 85,
    };
    
    let backup = agent;  // Move! agent недійсна
    // println!("{}", agent.id);  // ПОМИЛКА
    println!("{}", backup.id);    // OK
}
```

Після `let backup = agent` усі поля переміщені. `agent` — "порожня оболонка", будь-яке звернення до неї — помилка компіляції.

Якщо структура містить лише Copy-поля, її можна зробити Copy через derive:

```rust
#[derive(Debug, Clone, Copy)]
struct Position {
    x: f64,  // Copy
    y: f64,  // Copy
}

fn main() {
    let p1 = Position { x: 1.0, y: 2.0 };
    let p2 = p1;  // Copy! p1 та p2 обидві дійсні
    println!("{:?} {:?}", p1, p2);
}
```

`Position` може бути Copy, бо всі поля (f64, f64) — Copy. `Agent` не може бути Copy, бо поле `id: String` — не Copy.

### Передача структури у функцію

```rust
fn print_agent(agent: &Agent) {
    println!("Агент {}: батарея {}%", agent.id, agent.battery);
}

fn recharge(agent: &mut Agent) {
    agent.battery = 100;
    println!("Агент {} заряджено", agent.id);
}

fn main() {
    let mut agent = Agent {
        id: String::from("БПЛА-07"),
        battery: 45,
    };
    
    print_agent(&agent);     // &Agent — позичаємо для читання
    recharge(&mut agent);    // &mut Agent — позичаємо для зміни
    print_agent(&agent);     // знову позичаємо — agent все ще наша
}
```

Один параметр `&Agent` замість п'яти окремих. Функція отримує доступ до всіх полів через одне посилання. Ownership залишається у `main`.

---

## 16.5. Struct update syntax

Коли потрібно створити нову структуру, що відрізняється від існуючої лише кількома полями:

```rust
fn main() {
    let default_agent = Agent {
        id: String::from("DEFAULT"),
        battery: 100,
    };
    
    let scout = Agent {
        id: String::from("SCOUT-01"),
        ..default_agent  // решта полів з default_agent
    };
    
    // default_agent.id переміщено у scout (String — move)
    // default_agent.battery скопійовано (u8 — Copy)
    println!("{}: {}%", scout.id, scout.battery);
}
```

`..default_agent` означає: "всі поля, які я не вказав явно, візьми з `default_agent`". Для Copy-полів — копіювання. Для не-Copy полів — move. Тому `default_agent` може стати частково або повністю недійсною після цього.

---

## 16.6. Tuple structs та unit structs

### Tuple struct

Структура з неіменованими полями — доступ через індекс:

```rust
struct Coordinates(f64, f64, f64);

fn main() {
    let pos = Coordinates(10.0, 20.0, 50.0);
    println!("x={}, y={}, z={}", pos.0, pos.1, pos.2);
}
```

Tuple struct корисна для обгортки одного типу (newtype pattern):

```rust
struct Meters(f64);
struct Degrees(f64);

fn set_altitude(alt: Meters) { /* ... */ }
fn set_heading(hdg: Degrees) { /* ... */ }

fn main() {
    let alt = Meters(100.0);
    let hdg = Degrees(90.0);
    
    set_altitude(alt);    // OK
    // set_altitude(hdg); // ПОМИЛКА: Degrees != Meters
}
```

Компілятор не дозволить передати `Degrees` замість `Meters`, хоча обидва — обгортки навколо f64. Це type safety через newtype.

### Unit struct

Структура без полів. Використовується як маркер:

```rust
struct EmergencyMode;

fn handle_emergency(_mode: EmergencyMode) {
    println!("Аварійний режим активовано!");
}
```

---

## 16.7. Практика: структура Agent для БПЛА

Тепер об'єднаємо все в повноцінну структуру агента:

```rust
struct Position {
    x: f64,
    y: f64,
}

struct Agent {
    id: String,
    position: Position,
    battery: u8,
    heading: u16,
    airborne: bool,
}

impl Position {
    fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }
    
    fn distance_to(&self, other: &Position) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }
}

impl Agent {
    fn new(id: &str) -> Self {
        Self {
            id: String::from(id),
            position: Position::new(0.0, 0.0),
            battery: 100,
            heading: 0,
            airborne: false,
        }
    }
    
    fn status(&self) -> String {
        let state = if self.airborne { "у повітрі" } else { "на землі" };
        format!("{}: ({:.0},{:.0}) {}% [{}]",
            self.id, self.position.x, self.position.y,
            self.battery, state)
    }
    
    fn takeoff(&mut self) {
        if !self.airborne && self.battery >= 10 {
            self.airborne = true;
            self.battery -= 10;
            println!("  {} — зліт!", self.id);
        }
    }
    
    fn move_forward(&mut self) {
        if self.airborne && self.battery >= 5 {
            match self.heading {
                0 => self.position.y += 1.0,
                90 => self.position.x += 1.0,
                180 => self.position.y -= 1.0,
                270 => self.position.x -= 1.0,
                _ => {}
            }
            self.battery -= 5;
        }
    }
    
    fn distance_to_base(&self) -> f64 {
        let base = Position::new(0.0, 0.0);
        self.position.distance_to(&base)
    }
}

fn main() {
    let mut scout = Agent::new("SCOUT-01");
    println!("{}", scout.status());
    
    scout.takeoff();
    for _ in 0..3 {
        scout.move_forward();
    }
    
    println!("{}", scout.status());
    println!("До бази: {:.1} м", scout.distance_to_base());
}
```

Вивід:

```
SCOUT-01: (0,0) 100% [на землі]
  SCOUT-01 — зліт!
SCOUT-01: (0,3) 75% [у повітрі]
До бази: 3.0 м
```

Порівняйте з кодом Розділу 12: замість десятка окремих змінних та функцій з п'ятьма параметрами — один об'єкт `Agent` з методами. `scout.takeoff()` замість `cmd_takeoff(&mut airborne, &mut battery)`. `scout.status()` замість `print_status(id, x, y, battery, heading)`.

### Аналіз рішень

Розглянемо, як ownership та borrowing працюють у цьому коді:

`Agent::new(id: &str)` — приймає `&str` (позичаємо рядок), створює `String::from(id)` всередині. Agent стає власником нового String. Caller зберігає свій літерал.

`status(&self)` — іммутабельне позичання. Метод читає всі поля, формує String і повертає його (move назовні). Agent залишається незмінною.

`takeoff(&mut self)` — мутабельне позичання. Змінює `self.airborne` та `self.battery`. Потребує `let mut scout`.

`move_forward(&mut self)` — теж мутабельне. Змінює `self.position` та `self.battery`. Компілятор перевіряє: під час виклику `move_forward` ніхто інший не має посилання на `scout`.

`distance_to_base(&self)` — іммутабельне. Створює тимчасову `Position::new(0.0, 0.0)` (на стеку), обчислює відстань, тимчасова точка знищується при виході.

Вкладена структура `Position` у полі `position` — це композиція. Agent володіє Position, яка в свою чергу володіє двома f64. При Drop Agent → Drop Position → дані звільнені. Методи Position доступні через `self.position.distance_to(...)`.

### Другий агент — одна структура, кілька екземплярів

Створення другого агента — один рядок, замість дублювання п'яти змінних:

```rust
fn main() {
    let mut scout = Agent::new("SCOUT-01");
    let mut transport = Agent::new("TRANS-02");
    
    scout.takeoff();
    transport.takeoff();
    
    for _ in 0..3 {
        scout.move_forward();
    }
    
    println!("{}", scout.status());
    println!("{}", transport.status());
    
    let dist = scout.position.distance_to(&transport.position);
    println!("Відстань між агентами: {:.1}", dist);
}
```

Тепер додавання 100 агентів — це масив або вектор (Частина III): `let agents: Vec<Agent> = ...` Кожен агент — незалежний об'єкт з власними даними. Жодної плутанини з індексами змінних.

---

## Prompt Engineering: code review структури

```
Ось моя структура Agent та її методи:
[код]
Зроби code review:
1. Чи правильно обрано &self / &mut self для кожного методу?
2. Які методи варто додати?
3. Чи є антипатерни?
```

---

## Лабораторна робота No16

### Мета

Створити структуру для агента з методами.

### Завдання

**Частина 1 (3 бали):** Створіть структуру `Position` з методами `new`, `distance_to`, `translate`, `is_origin`.

Приклад рішення:

```rust
#[derive(Debug)]
struct Position {
    x: f64,
    y: f64,
}

impl Position {
    fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }
    
    fn distance_to(&self, other: &Position) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }
    
    fn translate(&mut self, dx: f64, dy: f64) {
        self.x += dx;
        self.y += dy;
    }
    
    fn is_origin(&self) -> bool {
        self.x == 0.0 && self.y == 0.0
    }
}
```

Зверніть увагу на вибір self: `distance_to(&self)` — лише читає координати. `translate(&mut self)` — змінює координати. `is_origin(&self)` — лише перевіряє. `new()` — асоційована функція без self.

**Частина 2 (4 бали):** Створіть структуру `Agent` з полями id, position, battery, heading. Методи: `new`, `status`, `takeoff`, `land`, `move_forward`, `turn`, `is_battery_critical`.

**Частина 3 (3 бали):** Попросіть AI зробити code review вашої структури. Запишіть у промпт-журнал.

### Критерії

| Критерій | Бал |
|----------|-----|
| Position з 4 методами | 30 |
| Agent з 7 методами, правильні &self/&mut self | 40 |
| AI code review + промпт-журнал | 30 |

---

## Troubleshooting

### `field is private`

За замовчуванням поля структури приватні для зовнішніх модулів. Поки ми працюємо в одному файлі — це не проблема. У Розділі 18 (Модулі) розберемо `pub`.

### `cannot use struct update syntax with non-Copy fields`

Struct update syntax `..other` переміщує не-Copy поля:

```rust
let a = Agent::new("ALPHA");
let b = Agent { id: String::from("BETA"), ..a };
// a.id переміщено (String), a.battery скопійовано (u8)
// println!("{}", a.id);  // ПОМИЛКА — move
println!("{}", a.battery);  // OK — Copy
```

Виправлення: clone не-Copy полів або вказати всі поля явно.

### `method not found for struct`

Метод визначено в `impl`, але для іншого типу, або з помилкою у назві. Перевірте `impl Point` та `struct Point`.

### `cannot borrow immutable field as mutable`

Спроба змінити поле через іммутабельне посилання:

```rust
fn bad(agent: &Agent) {
    agent.battery = 50;  // ПОМИЛКА: &Agent — тільки читання
}
```

Виправлення: `agent: &mut Agent`.

### `missing field in initializer`

Не всі поля вказані:

```rust
let p = Point { x: 1.0 };  // ПОМИЛКА: y missing
```

Rust вимагає ініціалізації всіх полів. Вказуйте кожне поле або використовуйте `..other`.

### `partial move` — часткове переміщення

```rust
let agent = Agent::new("SCOUT");
let name = agent.id;          // Move поля id
println!("{}", agent.battery); // OK — u8 Copy
// println!("{}", agent.id);   // ПОМИЛКА — id переміщено
```

Виправлення: `let name = agent.id.clone();` або `let name = &agent.id;`

---

## Контрольні запитання

### Рівень 1

1. Чим struct відрізняється від кортежу?

Відповідь: struct має іменовані поля (доступ через `point.x`), кортеж — індексовані (доступ через `tuple.0`). Struct створює новий тип із змістовним ім'ям.

2. Що таке `&self` у методі?

Відповідь: іммутабельне посилання на екземпляр структури. Метод може читати поля, але не змінювати.

### Рівень 2

3. Чому `let agent2 = agent1` переміщує, а не копіює Agent?

Відповідь: Agent містить String (не-Copy тип). Тому вся структура не-Copy, і `let agent2 = agent1` — move.

4. Чим `Point::new(1.0, 2.0)` відрізняється від `point.distance_to(&other)`?

Відповідь: `Point::new` — асоційована функція (не приймає self, викликається через `::`). `distance_to` — метод (приймає &self, викликається через `.`).

### Рівень 3

5. Напишіть метод `fn recharge(&mut self)` для Agent.

### Рівень 4

6. Чому Rust не дозволяє зробити окремі поля mut? Які переваги це дає?

Відповідь: спрощує міркування про код та роботу borrow checker. Якщо структура `let` (не mut) — гарантовано жодне поле не зміниться. Interior mutability (RefCell) для спеціальних випадків — тема Частини IV.

---

## Додатково

### Композиція: структури у структурах

Реальні об'єкти складаються з частин. Agent має Position, Position має координати. Це називається композиція — побудова складного типу з простіших:

```rust
struct Position {
    x: f64,
    y: f64,
}

struct Velocity {
    dx: f64,
    dy: f64,
}

struct Agent {
    id: String,
    position: Position,     // вкладена структура
    velocity: Velocity,     // ще одна
    battery: u8,
}
```

Agent володіє Position та Velocity. Доступ до вкладених полів — через ланцюжок крапок: `agent.position.x`, `agent.velocity.dx`. Кожен рівень вкладеності — це поле структури, яке, у свою чергу, є структурою з власними полями.

Переваги композиції: кожна "частина" може мати свої методи (`Position::distance_to`, `Velocity::speed`), і ці методи працюють незалежно від Agent. Ви можете використати Position окремо — для waypoints, для бази, для цілей — без Agent.

### Debug та Display

Щоб виводити структуру через `println!("{:?}", agent)`, додайте `#[derive(Debug)]`:

```rust
#[derive(Debug)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("{:?}", p);      // Point { x: 3.0, y: 4.0 }
    println!("{:#?}", p);     // красивий формат з відступами
}
```

`#[derive(Debug)]` — атрибут, що автоматично генерує реалізацію трейту Debug для структури. Без нього `println!("{:?}", p)` не скомпілюється — компілятор не знає, як форматувати вашу структуру. Ми детально розглянемо derive та трейти у Частині III, а поки — додавайте `#[derive(Debug)]` до кожної структури для зручного дебагу.

Також корисні derive-атрибути для простих структур:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Position {
    x: f64,
    y: f64,
}
```

`Clone` — дозволяє `.clone()`. `PartialEq` — дозволяє порівняння `==`. Ми розберемо кожен детально у відповідних розділах.

### Кілька impl блоків

Для одного типу можна мати кілька `impl` блоків — вони об'єднуються:

```rust
impl Point {
    fn new(x: f64, y: f64) -> Self { Self { x, y } }
    fn origin() -> Self { Self { x: 0.0, y: 0.0 } }
}

impl Point {
    fn distance_to(&self, other: &Point) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

Це корисно для організації коду: конструктори в одному блоці, обчислювальні методи — в іншому, методи модифікації — в третьому. Компілятор об'єднує всі блоки в один тип.

### Приватність полів (preview Розділу 18)

За замовчуванням поля структури приватні для зовнішніх модулів. Поки весь код в одному файлі — це не має значення. Але коли ви розділите код на модулі (Розділ 18), публічний доступ до полів потребуватиме `pub`:

```rust
pub struct Agent {
    id: String,          // приватне — доступ лише всередині модуля
    pub battery: u8,     // публічне — доступ ззовні
}
```

Типова практика: поля приватні, доступ — через методи (getter/setter). Це інкапсуляція — контроль доступу до даних.

---

## Резюме

Struct групує пов'язані дані у єдиний тип з іменованими полями. Це замінює набір окремих змінних одним об'єктом.

impl додає методи до структури. `&self` — читання, `&mut self` — зміна, `self` — споживання. Асоційовані функції (без self) — конструктори.

Структура володіє своїми полями (ownership). Коли структура Drop — всі поля теж Drop. Якщо хоча б одне поле не-Copy — вся структура не-Copy.

Struct update syntax `..other` копіює або переміщує поля з іншого екземпляра.

Tuple struct — для обгорток (newtype). Unit struct — для маркерів.

У параметрах: `&Agent` для читання, `&mut Agent` для зміни — один параметр замість десятка.

---

## Що далі

Структура описує дані агента. Але як описати його стани? "На землі", "у повітрі", "на зарядці", "аварійний режим" — це не просто рядки, а обмежений набір варіантів. Якщо стан — рядок, нічого не заважає написати `"на зимлі"` (з помилкою) — і компілятор не помітить.

У Розділі 17 (Enum та Pattern Matching) ви створите enum `AgentState` з варіантами `Grounded`, `Airborne`, `Charging`, `Emergency`. Match гарантуватиме обробку кожного стану. Забути один варіант — помилка компіляції. Агент отримає повноцінну машину станів.
