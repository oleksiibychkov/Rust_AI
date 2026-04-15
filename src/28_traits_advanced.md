# Розділ 28. Traits: власні абстракції та trait objects

## Анотація

У Розділі 27 ви навчились використовувати стандартні traits: Display для виводу, Debug для дебагу, Clone для копіювання. Але справжня сила traits — у створенні власних абстракцій. Trait `Agent` з методами `perceive()`, `decide()`, `act()` дозволяє написати код, що працює з будь-яким типом агента — Drone, GroundBot, UnderwaterVehicle — не знаючи конкретного типу. Це поліморфізм: один інтерфейс, різні реалізації. Rust пропонує два варіанти поліморфізму — статичний (generics, compile-time, zero-cost) та динамічний (trait objects, runtime, гнучкий). Цей розділ пояснює обидва, і головне — коли який обирати.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Створити власний trait з обов'язковими та default методами.
2. Реалізувати trait для кількох різних типів.
3. Використати `&dyn Trait` та `Box<dyn Trait>` для динамічного поліморфізму.
4. Пояснити різницю між static dispatch (generics) та dynamic dispatch (trait objects).
5. Пояснити object safety та які traits можуть бути trait objects.
6. Обрати між trait objects, generics та enum для конкретної задачі.

---

## Ключові терміни

**Custom trait (власний трейт)** — trait, визначений у вашому коді для моделювання доменної поведінки.

**Default method (метод за замовчуванням)** — метод trait з реалізацією у самому trait. Типи можуть використовувати її як є або перевизначити.

**Trait object (трейт-об'єкт)** — `&dyn Trait` або `Box<dyn Trait>` — посилання на значення невідомого типу, що реалізує trait. Конкретний тип визначається у runtime.

**Static dispatch (статична диспетчеризація)** — виклик методу trait розв'язується компілятором. Generics: для кожного типу генерується окрема версія функції (monomorphization).

**Dynamic dispatch (динамічна диспетчеризація)** — виклик методу trait розв'язується у runtime через таблицю віртуальних функцій (vtable).

**Object safety (об'єктна безпека)** — набір правил, що визначають, чи можна trait використати як trait object.

---

## Мотиваційний кейс

Рій БПЛА складається не з ідентичних агентів. Drone-розвідник несе камеру та летить високо. Drone-вантажний несе вантаж та летить низько. GroundBot — наземний робот, що рухається по дорогах. Але координатор рою повинен працювати з усіма однаково: видати завдання, отримати статус, перевірити готовність. Координатор не повинен знати внутрішню будову кожного агента — лише те, що він вміє perceive (сприймати середовище), decide (приймати рішення) та act (діяти).

Це класична задача поліморфізму: різні типи, спільний інтерфейс. В ООП-мовах це вирішується через наслідування та абстрактні класи. Rust не має наслідування — замість нього traits.

---

## 28.1. Створення власного trait

Trait оголошується ключовим словом `trait`, за яким йдуть сигнатури методів. Тіла методів немає — лише контракт:

```rust
trait Agent {
    fn id(&self) -> &str;
    fn position(&self) -> (f64, f64);
    fn battery(&self) -> u8;
    fn execute_step(&mut self);
}
```

Trait Agent вимагає чотири методи. Будь-який тип, що реалізує Agent, зобов'язаний мати всі чотири. Якщо забути один — помилка компіляції.

Реалізація для двох різних типів:

```rust
struct Drone {
    name: String,
    x: f64,
    y: f64,
    charge: u8,
    altitude: f64,
}

struct GroundBot {
    serial: String,
    x: f64,
    y: f64,
    fuel: u8,
    speed: f64,
}

impl Agent for Drone {
    fn id(&self) -> &str { &self.name }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn battery(&self) -> u8 { self.charge }
    fn execute_step(&mut self) {
        self.x += 1.0;
        self.charge = self.charge.saturating_sub(2);
        println!("[{}] Летить на висоті {:.0}м", self.name, self.altitude);
    }
}

impl Agent for GroundBot {
    fn id(&self) -> &str { &self.serial }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn battery(&self) -> u8 { self.fuel }
    fn execute_step(&mut self) {
        self.x += self.speed * 0.1;
        self.fuel = self.fuel.saturating_sub(1);
        println!("[{}] Їде зі швидкістю {:.0}", self.serial, self.speed);
    }
}
```

Drone та GroundBot — абсолютно різні типи з різними полями (altitude vs speed, charge vs fuel). Але обидва реалізують Agent, і зовнішній код може працювати з ними однаково: викликати `id()`, `position()`, `battery()`, `execute_step()` — не знаючи, Drone це чи GroundBot.

---

## 28.2. Default methods: поведінка за замовчуванням

Часто деякі методи trait мають "типову" реалізацію, яка підходить більшості типів. Замість змушувати кожен тип повторювати один і той самий код — trait може надати default method:

```rust
trait Agent {
    fn id(&self) -> &str;
    fn position(&self) -> (f64, f64);
    fn battery(&self) -> u8;
    fn execute_step(&mut self);
    
    // Default method — реалізація "за замовчуванням"
    fn is_operational(&self) -> bool {
        self.battery() > 10
    }
    
    fn status_report(&self) -> String {
        format!("[{}] на {:?}, батарея {}%, operational: {}", 
                self.id(), self.position(), self.battery(), self.is_operational())
    }
    
    fn needs_recharge(&self) -> bool {
        self.battery() < 20
    }
}
```

`is_operational`, `status_report` та `needs_recharge` мають тіло прямо у trait. Типи, що реалізують Agent, отримують ці методи безкоштовно — не потрібно їх писати. Але можуть перевизначити, якщо потрібна кастомна логіка:

```rust
impl Agent for Drone {
    fn id(&self) -> &str { &self.name }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn battery(&self) -> u8 { self.charge }
    fn execute_step(&mut self) {
        self.x += 1.0;
        self.charge = self.charge.saturating_sub(2);
    }
    
    // Перевизначення: дрон operational лише якщо батарея > 10 І висота > 0
    fn is_operational(&self) -> bool {
        self.battery() > 10 && self.altitude > 0.0
    }
    // status_report та needs_recharge — використовуємо default
}

impl Agent for GroundBot {
    fn id(&self) -> &str { &self.serial }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn battery(&self) -> u8 { self.fuel }
    fn execute_step(&mut self) {
        self.x += self.speed * 0.1;
        self.fuel = self.fuel.saturating_sub(1);
    }
    // Всі default methods працюють як є
}
```

Default methods — це не "стандартна реалізація, яку ніхто не змінює". Це розумні замовчування: працюють для більшості типів, але кожен тип може адаптувати їх під свою специфіку. Drone перевизначив is_operational, бо для нього operational включає ще й перевірку висоти. GroundBot задоволений замовчуванням.

Важлива деталь: default method може викликати інші методи trait (включаючи обов'язкові). `status_report` викликає `id()`, `position()`, `battery()`, `is_operational()` — усі визначені у тому ж trait. Це дозволяє побудувати складну поведінку поверх простого контракту: тип реалізує 4 базових методи, а отримує 7 (3 default зверху).

---

## 28.3. Trait objects: динамічний поліморфізм

До цього моменту ми знали конкретний тип при компіляції: `let d: Drone = ...` або `let g: GroundBot = ...`. Але що, якщо потрібен вектор агентів різних типів?

```rust
// Не компілюється: Vec<???> — який тип?
// let agents = vec![drone, ground_bot]; // різні типи!
```

Vec зберігає елементи одного типу. Drone та GroundBot — різні типи, навіть якщо обидва реалізують Agent. Рішення — trait object: `&dyn Agent` або `Box<dyn Agent>`. Це посилання на "будь-що, що реалізує Agent", де конкретний тип визначається у runtime.

```rust
fn print_status(agent: &dyn Agent) {
    println!("{}", agent.status_report());
}

fn main() {
    let drone = Drone {
        name: String::from("SCOUT-01"),
        x: 10.0, y: 20.0, charge: 85, altitude: 150.0,
    };
    
    let bot = GroundBot {
        serial: String::from("GND-03"),
        x: 5.0, y: 5.0, fuel: 90, speed: 30.0,
    };
    
    // Одна функція, різні типи
    print_status(&drone);
    print_status(&bot);
}
```

`&dyn Agent` — посилання на trait object. `dyn` означає "dynamic dispatch" — метод буде визначатись у runtime. Функція `print_status` не знає, Drone це чи GroundBot — вона знає лише, що об'єкт реалізує Agent.

Для збереження trait objects у колекціях потрібен `Box<dyn Agent>`, бо `dyn Agent` не має фіксованого розміру (Drone та GroundBot мають різний розмір):

```rust
fn main() {
    let agents: Vec<Box<dyn Agent>> = vec![
        Box::new(Drone {
            name: String::from("SCOUT-01"),
            x: 0.0, y: 0.0, charge: 100, altitude: 200.0,
        }),
        Box::new(GroundBot {
            serial: String::from("GND-03"),
            x: 0.0, y: 0.0, fuel: 100, speed: 25.0,
        }),
    ];
    
    println!("Рій з {} агентів:", agents.len());
    for agent in &agents {
        println!("  {}", agent.status_report());
    }
    
    // Знайти агентів, що потребують зарядки
    let need_charge: Vec<&Box<dyn Agent>> = agents.iter()
        .filter(|a| a.needs_recharge())
        .collect();
    
    println!("Потребують зарядки: {}", need_charge.len());
}
```

`Vec<Box<dyn Agent>>` — вектор, де кожен елемент — Box з довільним типом, що реалізує Agent. Box зберігає об'єкт на купі та має фіксований розмір (один вказівник + vtable pointer). Це дозволяє змішувати Drone та GroundBot в одному векторі.

---

## 28.4. Як працює dynamic dispatch: vtable

Коли компілятор бачить `&dyn Agent`, він створює "fat pointer" — два вказівники замість одного. Перший вказує на дані об'єкта (поля Drone або GroundBot). Другий — на vtable (virtual table, таблицю віртуальних функцій) — масив вказівників на конкретні реалізації методів.

```
&dyn Agent для Drone:
┌──────────────┐
│ data_ptr ────┼──→ [Drone { name: "SCOUT-01", x: 10.0, ... }]
│ vtable_ptr ──┼──→ ┌──────────────────────────┐
└──────────────┘    │ id:           Drone::id   │
                    │ position:     Drone::pos  │
                    │ battery:      Drone::bat  │
                    │ execute_step: Drone::step │
                    │ is_operational: Drone::op │
                    │ ...                       │
                    └──────────────────────────┘

&dyn Agent для GroundBot:
┌──────────────┐
│ data_ptr ────┼──→ [GroundBot { serial: "GND-03", ... }]
│ vtable_ptr ──┼──→ ┌──────────────────────────────┐
└──────────────┘    │ id:           GroundBot::id   │
                    │ position:     GroundBot::pos  │
                    │ battery:      GroundBot::bat  │
                    │ execute_step: GroundBot::step │
                    │ is_operational: default impl  │
                    │ ...                           │
                    └──────────────────────────────┘
```

Коли код викликає `agent.status_report()`, процесор: 1) читає vtable_ptr з fat pointer, 2) знаходить у vtable адресу status_report, 3) стрибає за цією адресою. Це один додатковий рівень непрямості (indirection) порівняно з прямим викликом — зазвичай кілька наносекунд.

Зверніть увагу: GroundBot не перевизначив `is_operational`, тому у його vtable — адреса default implementation з trait Agent. Drone перевизначив — у його vtable адреса Drone::is_operational. Vtable створюється компілятором для кожної комбінації (тип, trait) і зберігається у бінарному файлі програми.

---

## 28.5. Object safety: не кожен trait може бути trait object

Не всі traits можна використовувати як `dyn Trait`. Trait повинен бути object safe — задовольняти набір правил. Головні:

Методи не повинні повертати `Self`. Чому? `dyn Trait` не знає конкретний тип — тому не може повернути "значення мого типу":

```rust
trait Clonable {
    fn clone_self(&self) -> Self; // НЕ object safe
}
// let x: &dyn Clonable = ...; // ПОМИЛКА
```

Методи не повинні мати generic параметри. Vtable має фіксований набір вказівників, а generics потребують окремої версії для кожного типу:

```rust
trait Processor {
    fn process<T>(&self, data: T); // НЕ object safe
}
```

Trait не повинен вимагати `Sized`. За замовчуванням trait objects використовують `?Sized`, що означає "розмір невідомий при компіляції".

Clone — не object safe (clone повертає Self). Display та Debug — object safe. Ваш Agent — object safe (методи повертають конкретні типи, не Self).

Якщо потрібен "клонований trait object" — є патерн з допоміжним trait:

```rust
trait AgentClone: Agent {
    fn clone_box(&self) -> Box<dyn Agent>;
}

impl<T: Agent + Clone + 'static> AgentClone for T {
    fn clone_box(&self) -> Box<dyn Agent> {
        Box::new(self.clone())
    }
}
```

Це просунута техніка, яку рідко потрібно писати вручну — зазвичай бібліотеки (наприклад, `dyn-clone`) надають її як derive.

---

## 28.6. Static vs dynamic dispatch: коли який

Rust пропонує два підходи до поліморфізму, кожен з яких має чіткі trade-offs.

Static dispatch через generics (Розділ 29 розкриє тему глибше):

```rust
fn print_status_static<A: Agent>(agent: &A) {
    println!("{}", agent.status_report());
}
```

Компілятор створює окрему версію функції для кожного типу: `print_status_static::<Drone>` та `print_status_static::<GroundBot>`. Виклик методу — прямий, без vtable. Це zero-cost: такий самий код, як якби ви написали окрему функцію для кожного типу. Але: не можна зберігати різні типи в одному Vec, і бінарний файл зростає (окрема копія функції для кожного типу).

Dynamic dispatch через trait objects:

```rust
fn print_status_dynamic(agent: &dyn Agent) {
    println!("{}", agent.status_report());
}
```

Одна версія функції для всіх типів. Виклик через vtable — мінімальний overhead. Можна зберігати різні типи у `Vec<Box<dyn Agent>>`. Але: компілятор не може інлайнити виклики через vtable, що може вплинути на продуктивність у критичних циклах.

Правило вибору:

| Критерій | Static (generics) | Dynamic (dyn Trait) |
|----------|-------------------|---------------------|
| Різні типи в одній колекції | Ні | Так |
| Тип відомий при компіляції | Так | Не обов'язково |
| Продуктивність | Максимальна | Мінімальний overhead |
| Розмір бінарника | Більший (monomorphization) | Менший |
| Гнучкість (плагіни, runtime вибір) | Обмежена | Повна |

Для БПЛА-агента: якщо рій складається з агентів одного типу (лише Drone) — generics. Якщо рій гетерогенний (Drone + GroundBot + UnderwaterVehicle) — trait objects.

---

## 28.7. Enum vs trait object: третій варіант

Часто забувають, що в Rust є третій спосіб поліморфізму — enum. Якщо набір варіантів відомий при компіляції та фіксований:

```rust
enum AnyAgent {
    Drone(Drone),
    Ground(GroundBot),
}

impl AnyAgent {
    fn status_report(&self) -> String {
        match self {
            AnyAgent::Drone(d) => d.status_report(),
            AnyAgent::Ground(g) => g.status_report(),
        }
    }
}

fn main() {
    let agents: Vec<AnyAgent> = vec![
        AnyAgent::Drone(Drone { /* ... */ }),
        AnyAgent::Ground(GroundBot { /* ... */ }),
    ];
    
    for agent in &agents {
        println!("{}", agent.status_report());
    }
}
```

Enum — compile-time, без vtable overhead, exhaustive match. Але додавання нового типу агента потребує зміни enum та кожного match. Trait object — відкритий для розширення: новий тип просто реалізує trait, жоден існуючий код не змінюється.

Порівняння трьох підходів:

| Підхід | Відкритий для нових типів | Продуктивність | Різні типи в Vec | Exhaustive match |
|--------|--------------------------|----------------|------------------|-----------------|
| Generics | Так (compile-time) | Найвища | Ні | — |
| Trait object | Так (runtime) | Мінімальний overhead | Так | Ні |
| Enum | Ні (потребує зміни) | Висока | Так | Так |

Enum — коли варіантів мало та вони стабільні (AgentState: Idle/Patrol/Return). Trait object — коли варіанти додаватимуться (нові типи агентів, плагіни). Generics — коли тип відомий при компіляції і потрібна максимальна продуктивність.

## 28.8. Практика: trait Agent для рою БПЛА

Побудуємо повну систему з trait Agent, двома типами агентів та координатором рою:

```rust
use std::fmt;

// --- Trait Agent ---

trait Agent: fmt::Display {
    fn id(&self) -> &str;
    fn position(&self) -> (f64, f64);
    fn battery(&self) -> u8;
    fn execute_step(&mut self);
    
    fn is_operational(&self) -> bool {
        self.battery() > 10
    }
    
    fn distance_to(&self, target: (f64, f64)) -> f64 {
        let (x, y) = self.position();
        ((target.0 - x).powi(2) + (target.1 - y).powi(2)).sqrt()
    }
}

// --- Drone ---

struct Drone {
    name: String,
    x: f64,
    y: f64,
    charge: u8,
    altitude: f64,
}

impl Agent for Drone {
    fn id(&self) -> &str { &self.name }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn battery(&self) -> u8 { self.charge }
    fn execute_step(&mut self) {
        self.y += 2.0;
        self.charge = self.charge.saturating_sub(3);
    }
    fn is_operational(&self) -> bool {
        self.battery() > 10 && self.altitude > 0.0
    }
}

impl fmt::Display for Drone {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Drone '{}' alt={:.0}m bat={}%", self.name, self.altitude, self.charge)
    }
}

// --- GroundBot ---

struct GroundBot {
    serial: String,
    x: f64,
    y: f64,
    fuel: u8,
}

impl Agent for GroundBot {
    fn id(&self) -> &str { &self.serial }
    fn position(&self) -> (f64, f64) { (self.x, self.y) }
    fn battery(&self) -> u8 { self.fuel }
    fn execute_step(&mut self) {
        self.x += 1.5;
        self.fuel = self.fuel.saturating_sub(1);
    }
}

impl fmt::Display for GroundBot {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "GroundBot '{}' fuel={}%", self.serial, self.fuel)
    }
}

// --- Координатор рою ---

fn swarm_status(agents: &[Box<dyn Agent>]) {
    println!("=== Статус рою ({} агентів) ===", agents.len());
    for agent in agents {
        let status = if agent.is_operational() { "OK" } else { "УВАГА" };
        println!("  [{}] {} — {}", status, agent, agent.battery());
    }
}

fn find_nearest(agents: &[Box<dyn Agent>], target: (f64, f64)) -> Option<&str> {
    agents.iter()
        .filter(|a| a.is_operational())
        .min_by(|a, b| {
            a.distance_to(target)
                .partial_cmp(&b.distance_to(target))
                .unwrap()
        })
        .map(|a| a.id())
}

fn main() {
    let mut swarm: Vec<Box<dyn Agent>> = vec![
        Box::new(Drone { name: String::from("SCOUT-01"), x: 0.0, y: 0.0, charge: 90, altitude: 200.0 }),
        Box::new(Drone { name: String::from("SCOUT-02"), x: 10.0, y: 5.0, charge: 45, altitude: 150.0 }),
        Box::new(GroundBot { serial: String::from("GND-01"), x: 3.0, y: 0.0, fuel: 80 }),
        Box::new(GroundBot { serial: String::from("GND-02"), x: 20.0, y: 10.0, fuel: 15 }),
    ];
    
    swarm_status(&swarm);
    
    // Знайти найближчого до цілі
    let target = (15.0, 10.0);
    if let Some(nearest) = find_nearest(&swarm, target) {
        println!("Найближчий до {:?}: {}", target, nearest);
    }
    
    // Виконати крок для всіх
    println!("\n--- Крок симуляції ---");
    for agent in &mut swarm {
        agent.execute_step();
    }
    
    swarm_status(&swarm);
}
```

Зверніть увагу на `trait Agent: fmt::Display` — це supertrait bound, що означає: "кожен тип, що реалізує Agent, повинен також реалізувати Display". Це дозволяє використовувати `{}` для будь-якого `&dyn Agent`.

Функції `swarm_status` та `find_nearest` приймають `&[Box<dyn Agent>]` — слайс trait objects. Вони не знають конкретних типів. Вони працюють з абстракцією Agent і використовують лише методи trait. Додавання третього типу агента (UnderwaterVehicle) не потребує жодних змін у цих функціях — лише `impl Agent for UnderwaterVehicle`.

---

## 28.9. Prompt Engineering: вибір підходу до поліморфізму

### Промпт-шаблон

```
Я проєктую систему з [опис].
Є типи: [список типів з відмінностями].
Операції: [які спільні операції потрібні].

Порівняй три підходи:
1. Enum з match
2. Trait object (dyn Trait)
3. Generics

Для кожного: плюси, мінуси, приклад коду.
Який рекомендуєш для мого випадку і чому?
```

### Вправа з PE

Опишіть AI ваш рій БПЛА: які типи агентів, які спільні операції, чи будуть додаватись нові типи. Попросіть порівняти enum, trait object та generics. Оцініть рекомендацію: чи врахував AI можливість розширення? Чи врахував вимоги до продуктивності?

---

## Лабораторна робота No28

### Мета

Реалізувати поліморфний рій агентів через trait objects.

### Завдання базового рівня

1. Trait `Agent` з методами: id, position, battery, execute_step + 2 default methods.
2. Два типи агентів з різними полями та різною логікою execute_step.
3. `Vec<Box<dyn Agent>>` — рій із агентів обох типів.
4. Функції: swarm_status, find_nearest, count_operational — приймають `&[Box<dyn Agent>]`.
5. Тести для кожного типу та для гетерогенного рою.

### Варіанти

**A.** Три типи агентів: Drone (літає, витрачає 3% батареї за крок), GroundBot (їде, 1% за крок), StationaryCamera (нерухома, 0.5% за крок). Координатор обирає найближчий operational агент для кожного завдання.

**B.** Порівняння enum vs trait object: реалізуйте ту саму задачу обома способами. Виміряйте різницю в рядках коду. Опишіть, що зміниться при додаванні четвертого типу агента.

**C.** Trait з associated type: `trait Sensor { type Reading; fn read(&self) -> Self::Reading; }`. Реалізуйте для GPS (повертає (f64, f64)), Altimeter (f64), Camera (String опис). Поясніть, чому цей trait НЕ може бути trait object.

### Критерії

| Критерій | Бал |
|----------|-----|
| Trait з обов'язковими та default методами | 20 |
| Два+ типи з різною реалізацією | 20 |
| Vec\<Box\<dyn Agent\>\> працює коректно | 20 |
| Функції приймають &[Box\<dyn Agent\>] | 10 |
| Тести | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `the trait Agent cannot be made into an object`

Trait не є object safe:

```rust
trait BadAgent {
    fn clone_self(&self) -> Self; // повертає Self — не object safe
}
// let x: &dyn BadAgent = ...; // ПОМИЛКА
```

Виправлення: замінити `Self` на конкретний тип або Box<dyn Trait>. Або винести метод з Self у окремий trait.

### Помилка 2: `the size for values of type dyn Agent cannot be known at compilation time`

Спроба зберегти dyn Trait без Box:

```rust
let agents: Vec<dyn Agent> = vec![]; // ПОМИЛКА: розмір невідомий
```

Виправлення: `Vec<Box<dyn Agent>>` — Box має фіксований розмір (один вказівник + vtable pointer).

### Помилка 3: `method not found for &dyn Agent`

Метод не визначений у trait, лише в impl конкретного типу:

```rust
trait Agent { fn id(&self) -> &str; }
impl Drone { fn altitude(&self) -> f64 { self.altitude } } // лише для Drone

fn check(agent: &dyn Agent) {
    println!("{}", agent.altitude()); // ПОМИЛКА: altitude не в trait Agent
}
```

Виправлення: або додати метод у trait, або робити downcast (якщо потрібен доступ до специфіки конкретного типу — це сигнал, що абстракція неправильна).

### Помилка 4: `cannot borrow as mutable` при ітерації з execute_step

```rust
for agent in &swarm { // & — іммутабельне позичання
    agent.execute_step(); // потребує &mut self — ПОМИЛКА
}
```

Виправлення: `for agent in &mut swarm` або `for agent in swarm.iter_mut()`.

---

## Додатково

### Supertrait bounds

Trait може вимагати, щоб тип реалізував інші traits:

```rust
trait Agent: fmt::Display + fmt::Debug + Clone {
    fn id(&self) -> &str;
}
```

Тепер кожен `impl Agent for T` вимагає, щоб T також реалізував Display, Debug та Clone. Це корисно для trait objects: `&dyn Agent` автоматично підтримує `{}` та `{:?}`.

Але обережно: `Clone` робить trait не object safe (Clone::clone повертає Self). Якщо потрібен суперtraiт Clone і trait objects одночасно — використовуйте patерн clone_box з секції 28.5.

### Trait як тип параметра: impl Trait

Окрім generics та dyn Trait, є третій синтаксис — `impl Trait` у позиції параметра:

```rust
fn process(agent: &impl Agent) {
    println!("{}", agent.status_report());
}
```

Це синтаксичний цукор для generics: `fn process<A: Agent>(agent: &A)`. Компілятор використовує static dispatch. Різниця лише у читабельності: `impl Trait` простіший для одного параметра, generics — для складних bounds та множинних параметрів.

`impl Trait` у позиції повернення — окрема тема: `fn create_agent() -> impl Agent` означає "повертаю конкретний тип, що реалізує Agent, але caller не знає який". Це opaque type — корисний для приховування реалізації.

---

## Контрольні запитання

### Рівень 1

1. Що таке default method у trait?

Відповідь: метод з реалізацією прямо у trait. Типи отримують його безкоштовно, але можуть перевизначити.

2. Чим `&dyn Agent` відрізняється від `&Drone`?

Відповідь: `&Drone` — посилання на конкретний тип, відомий при компіляції. `&dyn Agent` — посилання на будь-який тип, що реалізує Agent; конкретний тип визначається у runtime через vtable.

### Рівень 2

3. Чому `Vec<dyn Agent>` не компілюється, а `Vec<Box<dyn Agent>>` — так?

Відповідь: Vec потребує елементи однакового розміру. `dyn Agent` — тип без відомого розміру (Drone та GroundBot мають різний розмір). `Box<dyn Agent>` — фіксований розмір (два вказівники: data + vtable), незалежно від конкретного типу всередині.

4. Чим dynamic dispatch відрізняється від static dispatch за продуктивністю?

Відповідь: static dispatch (generics) — прямий виклик, компілятор може інлайнити. dynamic dispatch (dyn Trait) — виклик через vtable, один додатковий рівень непрямості, інлайнінг неможливий. На практиці різниця — наносекунди на виклик, критична лише в tight loops із мільйонами ітерацій.

### Рівень 3

5. Коли обирати enum, коли trait object, коли generics?

Відповідь: enum — коли варіантів мало, вони стабільні, і потрібен exhaustive match (AgentState). Trait object — коли типи додаватимуться без зміни існуючого коду (нові типи агентів, плагіни). Generics — коли тип відомий при компіляції і потрібна максимальна продуктивність.

### Рівень 4

6. Чому Clone робить trait не object safe? Як обійти?

Відповідь: Clone::clone(&self) -> Self — повертає Self, а для dyn Trait конкретний тип невідомий при компіляції, тому компілятор не може визначити розмір повертаного значення. Обхід: створити допоміжний trait з методом `fn clone_box(&self) -> Box<dyn Agent>`, де повертається Box фіксованого розміру замість Self невідомого розміру.

---

## Резюме

Власні traits визначають доменну поведінку: `trait Agent` з методами perceive/decide/act. Default methods надають реалізацію за замовчуванням поверх обов'язкових методів — типи отримують функціональність безкоштовно, але можуть перевизначити.

Trait objects (`&dyn Trait`, `Box<dyn Trait>`) — динамічний поліморфізм: один інтерфейс для різних типів у runtime. Виклик через vtable з мінімальним overhead. Дозволяють `Vec<Box<dyn Agent>>` з агентами різних типів.

Object safety: trait можна використати як trait object, лише якщо методи не повертають Self і не мають generic параметрів.

Три підходи до поліморфізму: generics (compile-time, zero-cost, один тип), trait objects (runtime, гнучкий, різні типи в колекції), enum (compile-time, exhaustive match, закритий набір).

Supertrait bounds (`trait Agent: Display + Debug`) вимагають додаткові traits від реалізації. `impl Trait` — синтаксичний цукор для generics у параметрах.

---

## Що далі

Trait objects дали динамічний поліморфізм — різні типи в одній колекції. Але для багатьох задач потрібен статичний поліморфізм: "функція, що працює з будь-яким типом, що реалізує Agent, але конкретний тип відомий при компіляції". Це generics та trait bounds — тема Розділу 29. Ви навчитесь писати `fn process<A: Agent>(agent: &A)`, де компілятор створює оптимізовану версію для кожного типу. Monomorphization забезпечує zero-cost abstractions: generic-код такий самий швидкий, як код для конкретного типу.
