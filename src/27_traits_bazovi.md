# Розділ 27. Traits: базові концепції

## Анотація

До цього моменту кожен тип у нашому проєкті — самостійний острів. `DroneState` має свої методи, `Position` — свої, `SensorError` — свої. Але багато типів потребують спільної поведінки: будь-який тип повинен вміти виводитись для дебагу, порівнюватись на рівність, клонуватись. Без спільного інтерфейсу кожен тип реалізує `fn display()` по-своєму — з різними іменами, різними сигнатурами, без гарантій. Traits вирішують цю проблему: trait визначає контракт ("що тип повинен вміти"), а кожен тип реалізує цей контракт по-своєму ("як саме він це робить"). Це ключовий механізм абстракції в Rust — аналог інтерфейсів у Java, протоколів у Swift, але зі своїми особливостями.

Цей розділ зосереджений на розумінні самої концепції trait та на стандартних traits, які кожен Rust-програміст використовує щодня: Display, Debug, Clone, Default, PartialEq, PartialOrd. Розділ 28 розширить тему: власні traits, trait objects, динамічний поліморфізм. Розділ 29 додасть generics та trait bounds.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити, що таке trait як контракт поведінки.
2. Оголосити trait та реалізувати його для власного типу.
3. Реалізувати `Display` для форматованого виводу через `{}`.
4. Використовувати `Debug` через `#[derive(Debug)]` та реалізовувати вручну.
5. Застосовувати `Clone`, `Default`, `PartialEq`, `PartialOrd` через derive та ручну реалізацію.
6. Пояснити orphan rule та чому вона існує.
7. Обирати між derive та ручною реалізацією для конкретного trait.

---

## Ключові терміни

**Trait (трейт)** — іменований набір методів, що визначає контракт поведінки. Тип, що реалізує trait, гарантує наявність усіх методів trait.

**impl Trait for Type** — реалізація trait для конкретного типу. Визначає, як саме тип виконує контракт.

**derive** — атрибут `#[derive(TraitName)]`, що автоматично генерує реалізацію trait на основі полів структури.

**Orphan rule (правило сироти)** — обмеження: реалізувати trait для типу можна лише якщо trait або тип визначені у вашому крейті.

**Marker trait (маркерний трейт)** — trait без методів, що позначає властивість типу. Наприклад, `Copy` означає "тип можна копіювати побітово".

---

## Мотиваційний кейс

У 2014 році розробники Servo (браузерного движка від Mozilla, написаного на Rust) зіткнулись з класичною проблемою: різні компоненти (парсер HTML, движок рендерингу, мережевий стек) працювали з різними типами даних, але потребували спільних операцій — серіалізація, порівняння, клонування, форматований вивід для логів. У C++ для цього використовували б абстрактні базові класи з віртуальними методами, що створює проблеми з множинним наслідуванням та vtable overhead. Rust traits дали елегантне рішення: кожен тип реалізує потрібні traits, і компілятор гарантує, що контракт виконано — без наслідування, без runtime overhead.

---

## 27.1. Що таке trait: контракт поведінки

Trait — це обіцянка. Коли тип реалізує trait, він обіцяє: "я вмію робити те, що цей trait вимагає". Trait Display обіцяє: "мене можна вивести через `println!("{}", x)`". Trait Clone — "мене можна глибоко скопіювати". Trait PartialEq — "мене можна порівняти на рівність через `==`".

Аналогія: сертифікат пілота. Людина, що має сертифікат, обіцяє: "я вмію керувати літаком". Різні пілоти роблять це по-різному (різний стиль, різний досвід), але кожен гарантує базовий набір навичок. Сертифікат — це trait. Конкретні навички кожного пілота — це impl.

Синтаксис оголошення trait:

```rust
trait Describable {
    fn describe(&self) -> String;
}
```

Trait `Describable` вимагає один метод: `describe`, що приймає `&self` (посилання на об'єкт) і повертає String. Будь-який тип, що реалізує Describable, повинен мати цей метод.

Реалізація для конкретного типу:

```rust
struct Position {
    x: f64,
    y: f64,
}

struct Drone {
    id: String,
    position: Position,
    battery: u8,
}

impl Describable for Position {
    fn describe(&self) -> String {
        format!("({:.1}, {:.1})", self.x, self.y)
    }
}

impl Describable for Drone {
    fn describe(&self) -> String {
        format!("Дрон '{}' на {} з батареєю {}%", 
                self.id, self.position.describe(), self.battery)
    }
}

fn main() {
    let pos = Position { x: 47.5, y: 35.2 };
    let drone = Drone {
        id: String::from("SCOUT-01"),
        position: Position { x: 10.0, y: 20.0 },
        battery: 85,
    };
    
    println!("{}", pos.describe());
    // (47.5, 35.2)
    println!("{}", drone.describe());
    // Дрон 'SCOUT-01' на (10.0, 20.0) з батареєю 85%
}
```

Position та Drone — різні типи з різними полями. Але обидва реалізують Describable, і обидва гарантують наявність методу `describe()`. Реалізація різна: Position виводить координати, Drone — повну інформацію. Контракт — спільний.

Це відрізняє trait від простого методу. Метод `fn describe(&self)` в impl-блоці типу — це "цей тип має метод describe". Trait Describable — це "будь-який тип, що реалізує Describable, гарантовано має метод describe". Різниця — в узагальненні: з trait ви можете написати функцію, що приймає "будь-що, що вміє себе описати" (тема Розділу 28).

---

## 27.2. Display: вивід для людини

`std::fmt::Display` — trait, що визначає, як тип виводиться через `println!("{}", value)`, `format!("{}", value)` та інші макроси форматування з плейсхолдером `{}`. Це "обличчя" типу для кінцевого користувача — зрозуміле, чисте, без технічних деталей.

Display не можна derive — його завжди реалізовують вручну, бо компілятор не може вгадати, як саме ви хочете представити тип для людини. Чи Position виводити як "(47.5, 35.2)", чи "lat=47.5 lon=35.2", чи "47°30'N 35°12'E" — це рішення програміста.

```rust
use std::fmt;

struct Position {
    x: f64,
    y: f64,
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1})", self.x, self.y)
    }
}

struct Drone {
    id: String,
    position: Position,
    battery: u8,
}

impl fmt::Display for Drone {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[{}] {} батарея: {}%", self.id, self.position, self.battery)
    }
}

fn main() {
    let drone = Drone {
        id: String::from("SCOUT-01"),
        position: Position { x: 47.5, y: 35.2 },
        battery: 85,
    };
    
    println!("{}", drone);
    // [SCOUT-01] (47.5, 35.2) батарея: 85%
    
    // Display використовується автоматично в format!, to_string()
    let status = format!("Статус: {}", drone);
    println!("{}", status);
    
    // to_string() працює для будь-якого типу з Display
    let text: String = drone.to_string();
    println!("Як рядок: {}", text);
}
```

Зверніть увагу: у Display для Drone ми пишемо `self.position` у `write!` — і це автоматично викликає Display для Position. Тобто Display композабельний: реалізувавши його для частин, ви використовуєте його при реалізації для цілого.

Метод `to_string()` доступний автоматично для кожного типу, що реалізує Display. Не потрібно реалізовувати його окремо — стандартна бібліотека має бланкетну реалізацію `impl<T: Display> ToString for T`. Це приклад сили trait system: реалізувавши один trait, ви безкоштовно отримуєте функціональність, побудовану поверх нього.

Коли реалізовувати Display? Для будь-якого типу, який може бути показаний кінцевому користувачу: структури агента, стани місії, координати, повідомлення про помилки. Для внутрішніх типів, що не потрібні користувачу — достатньо Debug.

---

## 27.3. Debug: вивід для програміста

`std::fmt::Debug` — trait для технічного виводу через `println!("{:?}", value)` та `{:#?}` (pretty-print). На відміну від Display, Debug показує повну структуру даних — назви полів, вкладені структури, технічні деталі. Це інструмент для дебагу, не для користувача.

Debug можна і зазвичай потрібно derive — автоматична реалізація виводить усі поля:

```rust
#[derive(Debug)]
struct SensorReading {
    sensor_id: String,
    value: f64,
    timestamp: u64,
    valid: bool,
}

fn main() {
    let reading = SensorReading {
        sensor_id: String::from("TEMP-01"),
        value: 23.5,
        timestamp: 1700000000,
        valid: true,
    };
    
    // Компактний формат
    println!("{:?}", reading);
    // SensorReading { sensor_id: "TEMP-01", value: 23.5, timestamp: 1700000000, valid: true }
    
    // Pretty-print (з відступами)
    println!("{:#?}", reading);
    // SensorReading {
    //     sensor_id: "TEMP-01",
    //     value: 23.5,
    //     timestamp: 1700000000,
    //     valid: true,
    // }
}
```

`#[derive(Debug)]` працює лише якщо всі поля структури теж реалізують Debug. Усі стандартні типи (числа, String, bool, Vec, HashMap тощо) реалізують Debug, тому для більшості структур derive працює без проблем.

Коли реалізовувати Debug вручну? Коли автоматичний вивід занадто "шумний": структура з 20 полями, де для дебагу достатньо 3. Або коли потрібно приховати чутливі дані (пароль, токен):

```rust
use std::fmt;

struct AgentCredentials {
    id: String,
    auth_token: String,
}

impl fmt::Debug for AgentCredentials {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("AgentCredentials")
            .field("id", &self.id)
            .field("auth_token", &"***") // приховати токен
            .finish()
    }
}

fn main() {
    let cred = AgentCredentials {
        id: String::from("SCOUT-01"),
        auth_token: String::from("secret_token_12345"),
    };
    
    println!("{:?}", cred);
    // AgentCredentials { id: "SCOUT-01", auth_token: "***" }
}
```

`f.debug_struct("Name").field("name", &value).finish()` — builder API для ручного Debug. Зручніше, ніж write! з ручним форматуванням.

Display vs Debug — принципова різниця. Display — для кінцевого користувача: чистий, зрозумілий текст. Debug — для програміста: повна технічна інформація. println!("{}", x) викликає Display. println!("{:?}", x) викликає Debug. Обидва корисні, але для різних ситуацій. В логах місії БПЛА — Display. При дебагу у консолі — Debug.

---

## 27.4. Clone та Copy: два способи копіювання

Clone та Copy — traits, що визначають, як тип копіюється. Різниця між ними фундаментальна для Rust, і ви вже стикались з нею у Розділах 13–14 (ownership).

`Copy` — побітове копіювання, дешеве та неявне. Коли ви пишете `let y = x` для Copy-типу, x не переміщується — створюється побітова копія, і обидві змінні доступні. Copy може мати лише тип, чиє побітове копіювання семантично коректне: числа, bool, char, кортежі та масиви з Copy-елементів. Copy не можна реалізувати для типів з heap-даними (String, Vec), бо побітова копія створила б два вказівники на один блок пам'яті — double free при знищенні.

`Clone` — глибоке копіювання, явне та потенційно дороге. Виклик `.clone()` створює повну незалежну копію, включаючи heap-дані. `String::clone()` виділяє новий блок на купі та копіює вміст. `Vec::clone()` виділяє новий блок та клонує кожен елемент.

```rust
#[derive(Debug, Clone)]
struct MissionPlan {
    waypoints: Vec<(f64, f64)>,
    priority: u8,
    description: String,
}

fn main() {
    let original = MissionPlan {
        waypoints: vec![(0.0, 0.0), (10.0, 20.0), (30.0, 40.0)],
        priority: 1,
        description: String::from("Патрулювання периметру"),
    };
    
    // clone() створює повну незалежну копію
    let mut modified = original.clone();
    modified.priority = 3;
    modified.waypoints.push((50.0, 60.0));
    
    // Оригінал не змінився
    println!("Оригінал: {} точок, пріоритет {}", 
             original.waypoints.len(), original.priority);
    // Оригінал: 3 точок, пріоритет 1
    
    println!("Змінена: {} точок, пріоритет {}", 
             modified.waypoints.len(), modified.priority);
    // Змінена: 4 точок, пріоритет 3
}
```

`#[derive(Clone)]` працює, якщо всі поля реалізують Clone. Для MissionPlan це Vec (Clone) та String (Clone) та u8 (Clone + Copy).

Коли реалізовувати Clone вручну? Коли потрібна нестандартна логіка копіювання. Наприклад, при клонуванні агента можна скинути лічильник кроків чи згенерувати новий id:

```rust
impl Clone for Drone {
    fn clone(&self) -> Self {
        Drone {
            id: format!("{}-clone", self.id), // новий id
            position: self.position.clone(),
            battery: 100, // повна батарея для клона
        }
    }
}
```

Правило: якщо тип може мати Copy — derive Copy (та Clone, бо Copy потребує Clone). Якщо ні — derive Clone. Ручна реалізація — лише коли потрібна кастомна логіка.

---

## 27.5. Default: значення за замовчуванням

Trait `Default` визначає "типове" значення для типу. Для чисел — 0, для String — порожній рядок, для Vec — порожній вектор, для bool — false.

```rust
#[derive(Debug, Default)]
struct AgentConfig {
    max_altitude: f64,    // 0.0
    max_speed: f64,       // 0.0
    battery_threshold: u8, // 0
    autonomous: bool,     // false
    name: String,         // ""
}

fn main() {
    let config = AgentConfig::default();
    println!("{:#?}", config);
    // AgentConfig {
    //     max_altitude: 0.0,
    //     max_speed: 0.0,
    //     battery_threshold: 0,
    //     autonomous: false,
    //     name: "",
    // }
}
```

Нулі та порожні рядки — не завжди розумні замовчування. Для AgentConfig краще мати осмислені значення: висота 500м, швидкість 50, поріг батареї 20%. Тоді потрібна ручна реалізація:

```rust
impl Default for AgentConfig {
    fn default() -> Self {
        AgentConfig {
            max_altitude: 500.0,
            max_speed: 50.0,
            battery_threshold: 20,
            autonomous: false,
            name: String::from("unnamed"),
        }
    }
}
```

Default особливо зручний зі struct update syntax:

```rust
fn main() {
    // Створити конфіг з кастомною висотою, решта — за замовчуванням
    let config = AgentConfig {
        max_altitude: 1000.0,
        name: String::from("RECON-01"),
        ..Default::default()
    };
    println!("Висота: {}, швидкість: {}", config.max_altitude, config.max_speed);
    // Висота: 1000, швидкість: 50 (з Default)
}
```

`..Default::default()` заповнює всі невказані поля значеннями з Default. Це стандартний патерн для конфігурацій з багатьма параметрами: вказати лише ті, що відрізняються від типових.

---

## 27.6. PartialEq та PartialOrd: порівняння

`PartialEq` дозволяє порівняння через `==` та `!=`. `PartialOrd` додає `<`, `>`, `<=`, `>=`.

```rust
#[derive(Debug, PartialEq)]
struct Position {
    x: f64,
    y: f64,
}

fn main() {
    let a = Position { x: 10.0, y: 20.0 };
    let b = Position { x: 10.0, y: 20.0 };
    let c = Position { x: 30.0, y: 40.0 };
    
    println!("a == b: {}", a == b); // true
    println!("a == c: {}", a == c); // false
    println!("a != c: {}", a != c); // true
}
```

`#[derive(PartialEq)]` порівнює всі поля: `a == b` тоді й тільки тоді, коли `a.x == b.x` та `a.y == b.y`.

Чому "Partial", а не просто "Eq"? Тому що `f64` не має повної рівності: `NaN != NaN`. `PartialEq` дозволяє це — деякі значення можуть бути нерівні самі собі. `Eq` (без Partial) — суворіший trait, що гарантує рефлексивність (`a == a` для будь-якого a). Якщо ваша структура не містить f64 — реалізуйте і PartialEq, і Eq через derive.

Ручна реалізація PartialEq — коли потрібна кастомна логіка порівняння. Наприклад, порівнювати дронів лише за id, ігноруючи позицію та батарею:

```rust
impl PartialEq for Drone {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id // два дрони "рівні", якщо однаковий id
    }
}
```

PartialOrd дає впорядкування — корисне для сортування:

```rust
#[derive(Debug, PartialEq, PartialOrd)]
struct Priority(u8);

fn main() {
    let low = Priority(1);
    let high = Priority(5);
    
    println!("low < high: {}", low < high); // true
    
    let mut tasks = vec![Priority(3), Priority(1), Priority(5), Priority(2)];
    tasks.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!("{:?}", tasks); // [Priority(1), Priority(2), Priority(3), Priority(5)]
}
```

---

## 27.7. derive: автоматична реалізація

`#[derive(...)]` — атрибут, що наказує компілятору автоматично згенерувати реалізацію trait на основі полів. Derive працює механічно: для кожного поля викликає відповідний метод trait і комбінує результати.

Які стандартні traits можна derive:

| Trait | Що робить | Коли derive |
|-------|----------|-------------|
| `Debug` | Технічний вивід `{:?}` | Майже завжди |
| `Clone` | Глибоке копіювання `.clone()` | Коли потрібні копії |
| `Copy` | Побітове копіювання (неявне) | Лише для "простих" типів без heap |
| `PartialEq` | Порівняння `==` | Коли потрібне порівняння |
| `Eq` | Повна рівність (маркер) | Коли немає f64 |
| `PartialOrd` | Порівняння `<, >, <=, >=` | Коли потрібне впорядкування |
| `Ord` | Повне впорядкування | Коли немає f64 + потрібен sort |
| `Hash` | Хеш-значення | Коли тип — ключ HashMap |
| `Default` | Значення за замовчуванням | Коли є осмислене замовчування |

Типовий derive для структури агента:

```rust
#[derive(Debug, Clone, PartialEq)]
struct AgentState {
    id: String,
    position: (f64, f64),
    battery: u8,
    active: bool,
}
```

Обмеження derive: всі поля повинні реалізувати відповідний trait. Якщо хоча б одне поле не реалізує Clone — derive Clone не спрацює. Це каскадне обмеження: щоб derive Clone для AgentState, потрібно Clone для String, (f64, f64), u8, bool — всі вони мають Clone, тому працює.

Не можна derive: Display (завжди вручну), Error (через thiserror або вручну).

---

## 27.8. Orphan rule: обмеження реалізації

Rust має правило: ви можете реалізувати trait для типу лише якщо trait або тип (або обидва) визначені у вашому крейті. Не можна реалізувати чужий trait для чужого типу.

```rust
// Правильно: ваш trait для чужого типу
trait Describable {
    fn describe(&self) -> String;
}
impl Describable for String { // ваш trait для std::String
    fn describe(&self) -> String {
        format!("Рядок: '{}'", self)
    }
}

// Правильно: чужий trait для вашого типу
use std::fmt;
struct Drone { id: String }
impl fmt::Display for Drone { // std::Display для вашого Drone
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Drone({})", self.id)
    }
}

// ЗАБОРОНЕНО: чужий trait для чужого типу
// impl fmt::Display for Vec<i32> { ... } // ПОМИЛКА: ні Display, ні Vec не ваші
```

Навіщо це правило? Уявіть: крейт A реалізує Display для Vec<i32> — виводить елементи через кому. Крейт B теж реалізує Display для Vec<i32> — виводить по рядку. Ваша програма залежить від A та B — який Display використати? Orphan rule запобігає цьому конфлікту: лише автор Vec (стандартна бібліотека) або автор trait може додати реалізацію.

Обхід: якщо потрібно реалізувати чужий trait для чужого типу — створіть newtype wrapper:

```rust
struct FormattedVec(Vec<i32>);

impl fmt::Display for FormattedVec {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let items: Vec<String> = self.0.iter().map(|x| x.to_string()).collect();
        write!(f, "[{}]", items.join(", "))
    }
}
```

FormattedVec — ваш тип, тому реалізація Display дозволена.

## 27.9. Практика: traits для БПЛА-агента

Зберемо все разом і реалізуємо повний набір traits для типів агента з Розділу 20.

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
struct Position {
    x: f64,
    y: f64,
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1})", self.x, self.y)
    }
}

impl Default for Position {
    fn default() -> Self {
        Position { x: 0.0, y: 0.0 } // база
    }
}

impl Position {
    fn distance_to(&self, other: &Position) -> f64 {
        ((other.x - self.x).powi(2) + (other.y - self.y).powi(2)).sqrt()
    }
}

#[derive(Debug, Clone, PartialEq)]
enum AgentState {
    Idle,
    Patrolling,
    Returning,
    Charging,
    Emergency(String),
}

impl fmt::Display for AgentState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AgentState::Idle => write!(f, "Очікування"),
            AgentState::Patrolling => write!(f, "Патрулювання"),
            AgentState::Returning => write!(f, "Повернення на базу"),
            AgentState::Charging => write!(f, "Зарядка"),
            AgentState::Emergency(reason) => write!(f, "Аварія: {}", reason),
        }
    }
}

impl Default for AgentState {
    fn default() -> Self {
        AgentState::Idle
    }
}

#[derive(Debug, Clone)]
struct Agent {
    id: String,
    position: Position,
    battery: u8,
    state: AgentState,
    altitude: f64,
}

impl fmt::Display for Agent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[{}] {} | {} | батарея: {}% | висота: {:.0}м",
               self.id, self.state, self.position, self.battery, self.altitude)
    }
}

impl Default for Agent {
    fn default() -> Self {
        Agent {
            id: String::from("UNNAMED"),
            position: Position::default(),
            battery: 100,
            state: AgentState::default(),
            altitude: 0.0,
        }
    }
}

// Порівняння агентів за id (два агенти з однаковим id — "один і той самий")
impl PartialEq for Agent {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

fn main() {
    // Default + struct update syntax
    let scout = Agent {
        id: String::from("SCOUT-01"),
        battery: 85,
        altitude: 150.0,
        state: AgentState::Patrolling,
        ..Default::default() // position — за замовчуванням (0,0)
    };
    
    // Display — для логів та користувача
    println!("{}", scout);
    // [SCOUT-01] Патрулювання | (0.0, 0.0) | батарея: 85% | висота: 150м
    
    // Debug — для дебагу
    println!("{:#?}", scout);
    
    // Clone — для створення копій (наприклад, snapshot стану)
    let snapshot = scout.clone();
    println!("Snapshot: {}", snapshot);
    
    // PartialEq — порівняння за id
    let same_id = Agent {
        id: String::from("SCOUT-01"),
        battery: 50, // інша батарея
        ..Default::default()
    };
    println!("Той самий агент? {}", scout == same_id); // true (за id)
    
    // Default — базовий агент
    let default_agent = Agent::default();
    println!("За замовчуванням: {}", default_agent);
}
```

Кожен trait виконує свою роль: Display — для логів місії та статусних повідомлень оператору. Debug — для діагностики проблем. Clone — для створення знімків стану (наприклад, перед ризикованою операцією). Default — для ініціалізації з мінімальними параметрами. PartialEq — для пошуку агента у колекції за id.

---

## 27.10. Prompt Engineering: вибір traits

### Промпт-шаблон

```
Ось моя структура:
[вставити struct]

Які стандартні traits варто реалізувати/derive? 
Для кожного поясни: навіщо саме цей trait потрібен 
для цього типу, і derive чи вручну.
```

### Промпт-шаблон: рев'ю Display

```
Переглянь мою реалізацію Display:
[вставити impl Display]

Чи інформативний вивід? Чи не занадто технічний 
для кінцевого користувача? Чи не пропущена важлива 
інформація? Запропонуй альтернативний формат.
```

### Вправа з PE

Реалізуйте Display для вашого AgentError з Розділу 26 (якщо використовували thiserror — тепер зробіть вручну для розуміння). Попросіть AI оцінити повідомлення: чи достатньо інформації для дебагу? Чи зрозумілі повідомлення для оператора? Порівняйте свою реалізацію з тією, що згенерував thiserror.

---

## Лабораторна робота No27

### Мета

Реалізувати повний набір traits для типів агента.

### Завдання базового рівня

1. Реалізувати Display для Position, AgentState, Agent — інформативний вивід для оператора.
2. Derive Debug, Clone, PartialEq для всіх трьох.
3. Реалізувати Default для Agent з осмисленими значеннями (не нулі).
4. Написати тести: перевірити Display output, PartialEq логіку, Default значення.

### Варіанти

**A.** Кастомний PartialOrd для Agent: порівняння за пріоритетом (Emergency > Returning > Patrolling > Idle). Сортування вектора агентів за пріоритетом.

**B.** Кастомний Debug для Agent з фільтрацією полів: приховати auth_token, скоротити великий Vec waypoints до "N waypoints".

**C.** Реалізувати Display та Debug для enum Mission з 5+ варіантами. Display — для оператора (українською, зрозуміло). Debug — для логу (технічні деталі).

### Критерії

| Критерій | Бал |
|----------|-----|
| Display реалізовано вручну для 3+ типів | 25 |
| Debug, Clone, PartialEq через derive | 15 |
| Default з осмисленими значеннями | 15 |
| Обґрунтований вибір derive vs ручна реалізація | 15 |
| Тести | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `the trait Display is not implemented`

Спроба вивести тип через `{}` без Display:

```rust
#[derive(Debug)]
struct Pos { x: f64, y: f64 }
fn main() {
    let p = Pos { x: 1.0, y: 2.0 };
    println!("{}", p); // ПОМИЛКА: Display not implemented
}
```

Виправлення: реалізувати Display вручну, або використати `{:?}` (Debug).

### Помилка 2: `the trait Clone is not implemented` при derive

Поле структури не реалізує потрібний trait:

```rust
struct NoClone;

#[derive(Clone)]
struct MyStruct {
    field: NoClone, // ПОМИЛКА: NoClone не реалізує Clone
}
```

Виправлення: додати derive Clone до NoClone, або реалізувати Clone вручну для MyStruct.

### Помилка 3: `only traits defined in the current crate can be implemented for types defined outside`

Порушення orphan rule:

```rust
impl std::fmt::Display for Vec<i32> { ... } // ПОМИЛКА
```

Виправлення: створити newtype wrapper: `struct MyVec(Vec<i32>)` та реалізувати Display для MyVec.

### Помилка 4: `Copy cannot be implemented for types with destructors`

Спроба derive Copy для типу з heap-даними:

```rust
#[derive(Clone, Copy)]
struct Data {
    name: String, // ПОМИЛКА: String має Drop, тому не може бути Copy
}
```

Виправлення: видалити Copy. Використовувати Clone для явного копіювання через .clone().

---

## Додатково

### Marker traits: Copy, Eq, Send, Sync

Деякі traits не мають методів — вони лише "позначають" тип як такий, що має певну властивість. `Copy` означає "побітове копіювання безпечне". `Eq` означає "рівність рефлексивна" (на відміну від PartialEq). `Send` означає "тип можна передати в інший потік". `Sync` означає "посилання на тип можна ділити між потоками". Send та Sync — тема Частини IV (багатопотоковість), але важливо знати, що вони існують і реалізуються автоматично для більшості типів.

### Trait coherence та майбутнє

Orphan rule — частина ширшого поняття trait coherence: для кожної комбінації (тип, trait) повинна існувати не більше однієї реалізації у всій програмі. Це гарантує, що поведінка типу однозначна — немає "конфліктуючих реалізацій". Зворотний бік: обмеження гнучкості. Specialization (часткова спеціалізація trait для конкретних типів) — нічна фіча, що дозволяє послабити це обмеження у контрольованих випадках.

### Стандартні traits як екосистема

Traits у Rust — не ізольовані одиниці, а взаємопов'язана екосистема. Display дає to_string() безкоштовно. PartialOrd дає sort для Vec. Hash + Eq дає можливість бути ключем HashMap. Default дає struct update syntax. Реалізуючи стандартні traits, ваш тип стає "першокласним громадянином" Rust: працює з println!, з колекціями, з серіалізацією, з тестовим фреймворком (assert_eq потребує PartialEq + Debug).

Мінімальний набір traits для будь-якого "серйозного" типу: Debug (завжди), Clone (майже завжди), PartialEq (для тестів). Display — якщо тип показується користувачу. Default — якщо є осмислене значення за замовчуванням.

---

## Контрольні запитання

### Рівень 1

1. Що таке trait у Rust?

Відповідь: іменований набір методів, що визначає контракт поведінки. Тип, що реалізує trait через `impl Trait for Type`, гарантує наявність усіх методів trait.

2. Чим Display відрізняється від Debug?

Відповідь: Display (`{}`) — для кінцевого користувача: чистий, зрозумілий текст. Debug (`{:?}`) — для програміста: повна технічна інформація з назвами полів. Display реалізується вручну, Debug зазвичай derive.

### Рівень 2

3. Чому Display не можна derive?

Відповідь: компілятор не може вгадати, як саме програміст хоче представити тип для людини. Position можна вивести як "(47.5, 35.2)", "lat=47.5 lon=35.2", або "47°30'N" — це рішення програміста, не машини.

4. Чим Clone відрізняється від Copy?

Відповідь: Copy — неявне побітове копіювання при `let y = x` (дешеве, лише для типів без heap-даних). Clone — явне глибоке копіювання через `.clone()` (може бути дорогим, виділяє нову пам'ять на купі для String, Vec тощо).

### Рівень 3

5. Реалізуйте Display для enum:

```rust
enum Direction { North, South, East, West }
```

Відповідь:

```rust
impl fmt::Display for Direction {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Direction::North => write!(f, "Північ"),
            Direction::South => write!(f, "Південь"),
            Direction::East => write!(f, "Схід"),
            Direction::West => write!(f, "Захід"),
        }
    }
}
```

### Рівень 4

6. Що таке orphan rule і як її обійти?

Відповідь: orphan rule забороняє реалізувати чужий trait для чужого типу (наприклад, Display для Vec<i32>). Це запобігає конфліктуючим реалізаціям від різних крейтів. Обхід: newtype wrapper — створити свій тип-обгортку `struct MyVec(Vec<i32>)` та реалізувати trait для нього.

7. Чому `PartialEq`, а не просто `Eq`?

Відповідь: PartialEq дозволяє типам, де деякі значення нерівні самі собі (f64: NaN != NaN). Eq — суворіший маркерний trait, що гарантує рефлексивність (a == a для всіх a). Для типів без f64 — derive обидва. Для типів з f64 — лише PartialEq.

---

## Резюме

Trait — контракт поведінки: набір методів, які тип зобов'язується мати. `impl Trait for Type` — реалізація контракту. Стандартні traits роблять тип "першокласним громадянином" Rust.

Display (`{}`) — для кінцевого користувача, завжди вручну. Debug (`{:?}`) — для програміста, зазвичай derive. Clone — глибоке копіювання, явне через `.clone()`. Copy — побітове, неявне, лише для "простих" типів. Default — значення за замовчуванням, зручне зі struct update syntax. PartialEq — порівняння `==`, derive або вручну з кастомною логікою.

Orphan rule: реалізувати trait для типу можна лише якщо trait або тип — ваш. Обхід — newtype wrapper.

`#[derive(...)]` — автоматична реалізація, працює якщо всі поля реалізують відповідний trait. Мінімальний набір: Debug + Clone + PartialEq.

---

## Що далі

Стандартні traits дали типам спільну поведінку: Display, Clone, Default. Але поки що trait — це "бірка" на конкретному типі. Справжня сила traits розкривається, коли вони стають інструментом абстракції: "функція, що приймає будь-який тип, що реалізує trait Agent". Різні типи агентів (Drone, GroundBot, UnderwaterVehicle) з різними полями та різною логікою — але спільним інтерфейсом perceive/decide/act.

У Розділі 28 ви навчитесь створювати власні traits, визначати default methods, і працювати з trait objects (`&dyn Trait`, `Box<dyn Trait>`) — механізмом динамічного поліморфізму в Rust. Це дозволить створити рій з різних типів агентів, що працюють через єдиний інтерфейс.
