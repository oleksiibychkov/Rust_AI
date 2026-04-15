# Розділ 18. Модулі та видимість

## Анотація

Код агента з Розділів 16–17 вже має структури, enum, методи — і все це в одному файлі main.rs. При 300–500 рядках це ще терпимо, але реальний проєкт з кількома тисячами рядків в одному файлі — некерований. Модулі вирішують цю проблему: код розбивається на логічні одиниці з чіткою відповідальністю. Модуль `agent` містить структуру та методи агента. Модуль `navigation` — функції маршрутизації. Модуль `sensors` — обробку сенсорних даних. Кожен модуль контролює, що видимо зовні (`pub`), а що приховано (приватне за замовчуванням). Це інкапсуляція — захист внутрішніх деталей від зовнішнього коду.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Створити inline-модуль через `mod name { }` та винести його в окремий файл.
2. Пояснити правила видимості: що за замовчуванням приватне, як зробити публічним.
3. Використати `use` для скорочення шляхів та `pub use` для re-exports.
4. Організувати проєкт агента як набір модулів у файловій системі.
5. Пояснити різницю між `crate`, `super`, `self` у шляхах.

---

## Ключові терміни

**Module (модуль)** — логічна одиниця організації коду з власним простором імен та правилами видимості.

**pub** — ключове слово, що робить елемент публічним (видимим ззовні модуля).

**use** — оператор імпорту, що додає шлях у поточний простір імен.

**crate** — кореневий модуль проєкту (main.rs або lib.rs).

**super** — батьківський модуль.

**Re-export (pub use)** — публікація елемента з внутрішнього модуля через зовнішній.

---

## Мотиваційний кейс

Linux kernel — понад 30 мільйонів рядків коду. Уявіть це в одному файлі. Неможливо: два розробники не можуть одночасно редагувати один файл без конфліктів; пошук функції в 30-мільйонному файлі — годинник; одна помилка ламає все. Тому ядро розбите на тисячі модулів: `fs/` (файлові системи), `net/` (мережа), `drivers/` (драйвери). Кожен модуль — автономна одиниця з чітким інтерфейсом. Модулі у Rust працюють за тим самим принципом, але з перевагою: компілятор перевіряє видимість на етапі компіляції.

---

## 18.1. Inline-модулі: перший крок

### Модуль як блок коду

Найпростіший спосіб створити модуль — inline, прямо у файлі:

```rust
mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }
    
    pub fn distance(x1: f64, y1: f64, x2: f64, y2: f64) -> f64 {
        let dx = x2 - x1;
        let dy = y2 - y1;
        (dx * dx + dy * dy).sqrt()
    }
    
    fn internal_helper() {
        // ця функція приватна — не видима ззовні модуля
    }
}

fn main() {
    let sum = math::add(3, 5);
    let dist = math::distance(0.0, 0.0, 3.0, 4.0);
    println!("Сума: {}, відстань: {:.1}", sum, dist);
    
    // math::internal_helper();  // ПОМИЛКА: функція приватна
}
```

`mod math { ... }` створює модуль з ім'ям `math`. Всередині — функції, структури, enum, інші модулі. `pub fn add` — функція публічна, доступна ззовні через `math::add`. `fn internal_helper` — без `pub`, тому приватна: доступна лише всередині модуля `math`.

Виклик через повний шлях: `math::add(3, 5)`. Подвійна двокрапка `::` розділяє модуль та елемент.

### Правило видимості за замовчуванням

В Rust все приватне за замовчуванням. Це фундаментальна відмінність від мов, де все публічне за замовчуванням (як у Python). Щоб зробити щось видимим ззовні модуля, потрібно явно написати `pub`. Це стосується функцій, структур, enum, констант, типів.

Для структур — окреме правило: навіть якщо структура `pub`, її поля приватні за замовчуванням:

```rust
mod agent {
    pub struct Agent {
        pub id: String,       // публічне поле
        battery: u8,          // ПРИВАТНЕ поле
    }
    
    impl Agent {
        pub fn new(id: &str) -> Self {
            Self {
                id: String::from(id),
                battery: 100,
            }
        }
        
        pub fn battery(&self) -> u8 {
            self.battery  // доступ до приватного поля через публічний метод
        }
    }
}

fn main() {
    let a = agent::Agent::new("SCOUT-01");
    println!("ID: {}", a.id);           // OK — pub поле
    // println!("{}", a.battery);        // ПОМИЛКА — приватне поле
    println!("Батарея: {}%", a.battery());  // OK — pub метод
}
```

Приватні поля з публічними методами (getter) — це інкапсуляція. Зовнішній код не може встановити `battery = 200` напряму — лише через методи, що валідують значення. Це захист від невалідного стану.

Для enum — якщо enum публічний, всі його варіанти автоматично публічні. Це відрізняється від struct, де кожне поле потребує окремого `pub`.

### Інкапсуляція через приватність

Приватні поля з публічними методами — це інкапсуляція, один з найважливіших принципів проєктування ПЗ. Інкапсуляція означає: зовнішній код взаємодіє з об'єктом лише через визначений інтерфейс (публічні методи), а внутрішня реалізація прихована.

Чому це важливо для БПЛА? Уявіть, що поле `battery: u8` публічне. Будь-який модуль може написати `agent.battery = 200` — батарея більша за максимум. Або `agent.battery = 0` — агент раптово "розрядився" без причини. З приватним полем та getter/setter методами:

```rust
mod agent {
    pub struct Agent {
        battery: u8,  // приватне!
    }
    
    impl Agent {
        pub fn new() -> Self {
            Self { battery: 100 }
        }
        
        // Getter — лише читання
        pub fn battery(&self) -> u8 {
            self.battery
        }
        
        // Setter з валідацією
        pub fn consume_battery(&mut self, amount: u8) {
            self.battery = self.battery.saturating_sub(amount);
        }
        
        pub fn recharge(&mut self) {
            self.battery = 100;  // гарантовано валідне значення
        }
    }
}

fn main() {
    let mut a = agent::Agent::new();
    a.consume_battery(30);
    println!("Батарея: {}%", a.battery());  // 70
    
    // a.battery = 200;  // ПОМИЛКА: поле приватне!
    // Неможливо створити невалідний стан
}
```

Інваріант "батарея від 0 до 100" підтримується методами. Зовнішній код фізично не може його порушити, бо не має прямого доступу до поля. Це набагато надійніше, ніж сподіватись, що "ніхто не запише погане значення".

### Типові рівні видимості

| Рівень | Синтаксис | Хто бачить |
|--------|-----------|-----------|
| Приватний | (без ключового слова) | Тільки поточний модуль |
| pub(crate) | `pub(crate)` | Всі модулі у поточному crate |
| pub(super) | `pub(super)` | Батьківський модуль |
| Публічний | `pub` | Всі, включаючи зовнішні crate |

Починайте з приватного і розширюйте видимість лише за потреби — так само, як з ownership (починайте з &self, розширюйте до &mut self).

---

## 18.2. Модулі в окремих файлах

### Файлова структура проєкту

Inline-модулі зручні для маленьких блоків, але для реального проєкту кожен модуль — окремий файл. Rust має чітке правило: ім'я модуля = ім'я файлу.

Структура проєкту:

```
my_drone/
├── Cargo.toml
└── src/
    ├── main.rs         // кореневий модуль
    ├── position.rs     // модуль position
    ├── agent.rs        // модуль agent
    └── navigation.rs   // модуль navigation
```

У `main.rs` оголошуємо модулі:

```rust
// main.rs
mod position;      // шукає файл src/position.rs
mod agent;         // шукає файл src/agent.rs
mod navigation;    // шукає файл src/navigation.rs

fn main() {
    let mut scout = agent::Agent::new("SCOUT-01");
    println!("{}", scout.status());
}
```

`mod position;` (з крапкою з комою, без фігурних дужок) каже компілятору: "знайди файл `src/position.rs` і включи його як модуль `position`". Це відрізняється від inline-модуля `mod position { ... }`, де код прямо у main.rs.

Компілятор шукає файл за двома правилами. Перше — `src/position.rs` (модуль-файл). Друге — `src/position/mod.rs` (модуль-директорія). Якщо знайдено обидва — помилка компіляції. Використовуйте один підхід: для простих модулів — файл, для модулів з підмодулями — директорія.

Важливо: `mod name;` — це оголошення, не імпорт. Оголошення каже "цей модуль існує і є частиною проєкту". Імпорт (`use`) — "я хочу використовувати конкретний елемент з модуля". Без `mod` компілятор не знає про файл. Без `use` — доведеться писати повний шлях.

### Що відбувається при компіляції

Коли ви пишете `mod position;` у main.rs, компілятор:

Знаходить файл `src/position.rs`. Компілює його як модуль з ім'ям `position`. Робить доступним через шлях `crate::position`. Перевіряє видимість: що `pub`, а що приватне.

Кожен файл `.rs` у проєкті — це потенційний модуль. Але він стає частиною проєкту лише після оголошення через `mod`. Файл без `mod` — невидимий для компілятора.

### Вміст модульних файлів

```rust
// src/position.rs
pub struct Position {
    pub x: f64,
    pub y: f64,
}

impl Position {
    pub fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }
    
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }
    
    pub fn origin() -> Self {
        Self { x: 0.0, y: 0.0 }
    }
}
```

```rust
// src/agent.rs
use crate::position::Position;   // імпорт з іншого модуля

pub struct Agent {
    id: String,
    pub position: Position,
    battery: u8,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: String::from(id),
            position: Position::origin(),
            battery: 100,
        }
    }
    
    pub fn status(&self) -> String {
        format!("{}: ({:.0},{:.0}) {}%",
            self.id, self.position.x, self.position.y, self.battery)
    }
    
    pub fn id(&self) -> &str {
        &self.id
    }
    
    pub fn battery(&self) -> u8 {
        self.battery
    }
}
```

```rust
// src/navigation.rs
use crate::position::Position;

pub fn calculate_route(from: &Position, to: &Position) -> f64 {
    from.distance_to(to)
}

pub fn is_in_bounds(pos: &Position, max: f64) -> bool {
    pos.x >= 0.0 && pos.x <= max && pos.y >= 0.0 && pos.y <= max
}
```

### Шляхи: crate, super, self

Шляхи в Rust-модулях працюють подібно до файлових шляхів: `crate::` — абсолютний (від кореня), `super::` — відносний (від батька), `self::` — поточний каталог.

`crate` — кореневий модуль (main.rs або lib.rs). `use crate::position::Position` — від кореня до модуля position, звідти — тип Position. Це абсолютний шлях — він працює однаково незалежно від того, з якого модуля ви його пишете.

`super` — батьківський модуль. Корисно для вкладених модулів, що хочуть звернутись до сусіднього елемента або елемента рівнем вище:

```rust
mod outer {
    pub fn greet() { println!("hello from outer"); }
    
    pub mod inner {
        pub fn call_parent() {
            super::greet();  // батьківський модуль outer
        }
        
        pub mod deep {
            pub fn call_grandparent() {
                super::super::greet();  // два рівні вгору
            }
        }
    }
}

fn main() {
    outer::inner::call_parent();
    outer::inner::deep::call_grandparent();
}
```

`self` — поточний модуль. Рідко потрібен явно, але іноді для ясності при імпорті з поточного модуля: `use self::helper::compute;`. Також використовується в `use self::AgentState::*;` для імпорту всіх варіантів enum.

### Коли використовувати абсолютні vs відносні шляхи

Абсолютні (`crate::`) — у більшості випадків. Вони зрозуміліші та не ламаються при переміщенні модуля. Відносні (`super::`) — коли модулі тісно пов'язані та часто переміщуються разом (наприклад, підмодулі одного модуля).

Аналогія з файловою системою: `crate::position::Position` — це як `/home/project/position/Position.rs`. `super::position::Position` — це як `../position/Position.rs`. Абсолютний шлях надійніший.

---

## 18.3. use: скорочення шляхів

Повні шляхи (`crate::position::Position`) — довгі. `use` створює короткий псевдонім:

```rust
// Без use — довгі шляхи
fn main() {
    let p = crate::position::Position::new(1.0, 2.0);
    let a = crate::agent::Agent::new("X");
}

// З use — коротко
use crate::position::Position;
use crate::agent::Agent;

fn main() {
    let p = Position::new(1.0, 2.0);
    let a = Agent::new("X");
}
```

### Конвенції use

Для типів (struct, enum) — імпортуйте сам тип: `use crate::position::Position;` Потім використовуйте: `Position::new(...)`. Це зручно, бо ви працюєте з типом часто.

Для функцій — імпортуйте батьківський модуль: `use crate::navigation;` Потім використовуйте: `navigation::calculate_route(...)`. Це зберігає контекст: видно, що функція з модуля navigation, а не "звідкись".

Для enum-варіантів — зазвичай імпортуйте сам enum: `use crate::state::AgentState;` Потім: `AgentState::Grounded`. Іноді для часто використовуваних варіантів можна імпортувати їх напряму: `use crate::state::AgentState::*;` Тоді: просто `Grounded`. Але це може створити конфлікти імен, тому використовуйте обережно.

### Перейменування через as

Якщо два типи мають однакове ім'я (з різних модулів), використовуйте `as`:

```rust
use crate::navigation::Position as NavPosition;
use crate::rendering::Position as RenderPosition;

fn main() {
    let nav = NavPosition::new(1.0, 2.0);
    let render = RenderPosition::new(100, 200);
}
```

### Групований use

```rust
// Окремі use
use crate::position::Position;
use crate::agent::Agent;
use crate::state::AgentState;

// Групований — компактніше
use crate::{
    position::Position,
    agent::Agent,
    state::AgentState,
};
```

Обидва варіанти еквівалентні. Групований зручніший для багатьох імпортів з одного crate.

---

## 18.4. pub use: re-exports

Іноді внутрішня структура модулів — деталь реалізації. Зовнішній код не повинен знати, що `Position` лежить у `position.rs` — він хоче просто `use my_drone::Position`.

Re-export вирішує це:

```rust
// main.rs або lib.rs
mod position;
mod agent;
mod navigation;

// Re-export ключових типів на верхньому рівні
pub use position::Position;
pub use agent::Agent;
```

Тепер зовнішній код може писати `use my_drone::Position` замість `use my_drone::position::Position`. Внутрішня організація (окремий файл position.rs) прихована.

---

## 18.5. Практика: проєкт агента як набір модулів

Розберемо, як організувати проєкт БПЛА з Розділів 16–17 у модулі:

```
drone_agent/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── position.rs
    ├── state.rs
    └── agent.rs
```

```rust
// src/state.rs — enum станів
pub enum AgentState {
    Grounded,
    Airborne { altitude: f64 },
    Charging { percent: u8 },
    Emergency { reason: String },
}

impl AgentState {
    pub fn name(&self) -> &str {
        match self {
            AgentState::Grounded => "Grounded",
            AgentState::Airborne { .. } => "Airborne",
            AgentState::Charging { .. } => "Charging",
            AgentState::Emergency { .. } => "Emergency",
        }
    }
    
    pub fn is_operational(&self) -> bool {
        !matches!(self, AgentState::Emergency { .. })
    }
}
```

```rust
// src/agent.rs — структура агента
use crate::position::Position;
use crate::state::AgentState;

pub struct Agent {
    id: String,
    position: Position,
    state: AgentState,
    battery: u8,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: String::from(id),
            position: Position::origin(),
            state: AgentState::Grounded,
            battery: 100,
        }
    }
    
    pub fn status(&self) -> String {
        format!("{}: [{}] батарея {}%", self.id, self.state.name(), self.battery)
    }
    
    pub fn id(&self) -> &str { &self.id }
    pub fn battery(&self) -> u8 { self.battery }
    pub fn position(&self) -> &Position { &self.position }
    pub fn state(&self) -> &AgentState { &self.state }
    
    pub fn takeoff(&mut self) {
        if let AgentState::Grounded = self.state {
            if self.battery >= 10 {
                self.battery -= 10;
                self.state = AgentState::Airborne { altitude: 50.0 };
            }
        }
    }
}
```

```rust
// src/main.rs — точка входу
mod position;
mod state;
mod agent;

use agent::Agent;

fn main() {
    let mut scout = Agent::new("SCOUT-01");
    println!("{}", scout.status());
    
    scout.takeoff();
    println!("{}", scout.status());
}
```

Кожен файл — окрема відповідальність: `position.rs` — геометрія, `state.rs` — стани, `agent.rs` — агент з логікою. `main.rs` — лише точка входу та оголошення модулів. Зміна в одному модулі не вимагає зміни в інших, якщо публічний інтерфейс залишився тим самим.

### Аналіз модульної структури

Чому саме такий розподіл? `position.rs` не залежить від жодного іншого модуля — це базовий тип. `state.rs` теж незалежний — enum станів самодостатній. `agent.rs` залежить від обох: `use crate::position::Position` та `use crate::state::AgentState`. `main.rs` залежить від `agent`.

Залежності утворюють дерево без циклів:

```
main.rs
  └── agent.rs
        ├── position.rs
        └── state.rs
```

Циклічні залежності (модуль A залежить від B, а B від A) — помилка компіляції у Rust. Це добре: циклічні залежності ускладнюють розуміння коду та створюють проблеми при компіляції.

### Що публічне, а що приватне

У `agent.rs`: структура `Agent` — `pub` (зовнішній код створює агентів). Поля `id`, `battery`, `state` — приватні (зовнішній код не може напряму змінити батарею на 200%). Поле `position` — `pub` (для зручності доступу до координат). Методи `new`, `status`, `takeoff` — `pub` (публічний API). Внутрішні helper-функції — без `pub`.

Принцип мінімальної видимості: робіть `pub` лише те, що справді потрібно ззовні. Менше `pub` — менше точок, де зовнішній код може зламати інваріанти.

---

## Prompt Engineering: модульна декомпозиція

```
Ось мій код з одного файлу (400 рядків):
[код]
Запропонуй модульну структуру:
1. Які модулі створити?
2. Що має бути pub, що приватне?
3. Де використати pub use для re-exports?
```

### Приклад діалогу

```
Студент: У мене Agent, Position, AgentState, Command — все в main.rs.
AI: Рекомендую 4 модулі:
  position.rs — Position (pub struct, pub поля)
  state.rs — AgentState (pub enum)
  command.rs — Command enum + execute logic
  agent.rs — Agent struct (приватні поля, pub методи)
В main.rs: mod оголошення + pub use для Position, Agent.
```

---

## Лабораторна робота No18

### Мета

Перетворити монолітний код на модульний проєкт.

### Завдання

**Частина 1 (4 бали):** Візьміть код Agent з Розділів 16–17 (Position, AgentState, Agent) і розділіть на три файли-модулі. Переконайтесь, що компілюється.

**Частина 2 (3 бали):** Додайте модуль `navigation` з функціями `calculate_route` та `is_in_bounds`. Використайте `use crate::position::Position`.

**Частина 3 (3 бали):** Попросіть AI запропонувати модульну структуру для вашого коду. Порівняйте з вашим рішенням. Промпт-журнал.

### Критерії

| Критерій | Бал |
|----------|-----|
| Три модулі компілюються | 40 |
| Модуль navigation з use | 30 |
| AI review + промпт-журнал | 30 |

---

## Troubleshooting

### `unresolved import`

Шлях у `use` не знайдено. Це одна з найчастіших помилок при роботі з модулями:

```
error[E0432]: unresolved import `crate::sensors`
```

Причини та виправлення:

Файл модуля не створено — перевірте, що `src/sensors.rs` існує. `mod name;` не оголошено в main.rs — додайте `mod sensors;` у main.rs. Елемент не `pub` — перевірте, що функція або тип має `pub`. Помилка у назві — Rust чутливий до регістру: `Sensors` та `sensors` — різні.

### `function is private`

Функція існує, але не позначена `pub`:

```
error[E0603]: function `helper` is private
```

Виправлення: додайте `pub` перед `fn helper`. Або перегляньте дизайн — можливо, ця функція повинна залишатись приватною, а зовнішній код має використовувати інший публічний метод.

### `field is private`

Поле структури приватне. Навіть якщо struct — `pub`, поля за замовчуванням приватні:

```rust
// В модулі agent:
pub struct Agent { battery: u8 }  // battery приватне

// В main:
let a = Agent::new("X");
println!("{}", a.battery);  // ПОМИЛКА
```

Рішення А: `pub battery: u8` — зробити поле публічним (якщо прямий доступ безпечний). Рішення Б (краще): getter-метод `pub fn battery(&self) -> u8 { self.battery }`.

### `cannot find type in this scope`

Тип існує в модулі, але не імпортований:

```rust
// Забули use
fn main() {
    let p = Position::new(1.0, 2.0);  // ПОМИЛКА: Position not found
}
```

Виправлення: додайте `use crate::position::Position;` або використайте повний шлях `position::Position::new(1.0, 2.0)`.

### Порядок `mod` оголошень

Порядок `mod` у main.rs не має значення для компіляції — компілятор аналізує всі модулі перед початком перевірки типів. Але для читабельності — оголошуйте базові модулі першими: `position` перед `agent` (бо agent залежить від position).

### Циклічні залежності

```
error[E0391]: cycle detected when processing `agent::Agent`
```

Модуль A залежить від B, а B від A — це заборонено. Рішення: винести спільні типи в третій модуль `common`, від якого залежать обидва.

---

## Додатково

### Модулі-директорії

Для великих модулів з підмодулями — використовуйте директорії:

```
src/
├── main.rs
└── agent/
    ├── mod.rs         // головний файл модуля agent
    ├── state.rs       // підмодуль state
    └── battery.rs     // підмодуль battery
```

У `src/agent/mod.rs`:

```rust
mod state;
mod battery;

pub use state::AgentState;
pub use battery::BatterySystem;

pub struct Agent { /* ... */ }
```

Зверніть увагу на `pub use` — підмодулі `state` та `battery` приховані (не `pub mod`), але їхні ключові типи доступні через `agent::AgentState` та `agent::BatterySystem`. Зовнішній код не знає про існування підмодулів — це деталь реалізації.

### pub(crate) — видимість лише в межах crate

`pub` робить елемент видимим для всіх (включаючи зовнішні crates). `pub(crate)` — видимий лише всередині поточного проєкту:

```rust
pub(crate) fn internal_calculation() -> f64 {
    // доступна у всіх модулях цього проєкту, але не ззовні
    42.0
}
```

Це зручно для helper-функцій, що потрібні кільком модулям всередині проєкту, але не є частиною публічного API.

### lib.rs vs main.rs

`main.rs` — бінарний crate (програма з fn main). `lib.rs` — бібліотечний crate (колекція публічних типів та функцій для використання іншими проєктами). Проєкт може мати обидва: `src/lib.rs` (бібліотека) та `src/main.rs` (програма, що використовує бібліотеку).

Для навчального проєкту БПЛА використовуйте `main.rs`. Для бібліотеки (якщо хочете, щоб інші проєкти могли використовувати ваш Agent) — `lib.rs`.

### Модулі та тестування (preview Розділу 19)

Модулі спрощують тестування. Кожен модуль може мати вбудовані тести:

```rust
// src/position.rs
pub struct Position { pub x: f64, pub y: f64 }

impl Position {
    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_distance() {
        let a = Position { x: 0.0, y: 0.0 };
        let b = Position { x: 3.0, y: 4.0 };
        assert!((a.distance_to(&b) - 5.0).abs() < 0.001);
    }
}
```

`#[cfg(test)]` — блок компілюється лише при тестуванні (`cargo test`). `use super::*` — імпортує все з батьківського модуля (тобто з position). Тести — поруч з кодом, але не включаються у фінальний бінарник. Детально — у Розділі 19.

### cargo doc — автоматична документація

Модулі полегшують генерацію документації:

```bash
cargo doc --open
```

Ця команда генерує HTML-документацію для всього проєкту. Кожен `pub` елемент потрапляє у документацію. Документаційні коментарі `///` (з Розділу 11) відображаються як описи. Модульна структура стає навігацією документації: модуль agent → struct Agent → методи new, status, takeoff.

---

## Контрольні запитання

### Рівень 1

1. Що означає, що елемент "приватний за замовчуванням"?

Відповідь: без `pub` функція/структура/поле видиме лише всередині свого модуля. Зовнішній код не може до нього звернутись.

2. Як оголосити модуль в окремому файлі?

Відповідь: у main.rs написати `mod name;` (з крапкою з комою). Створити файл `src/name.rs`.

### Рівень 2

3. Чому поля struct приватні, а варіанти enum публічні за замовчуванням?

Відповідь: struct — це дані, що потребують захисту (інкапсуляція). Enum — це набір варіантів типу: щоб використати enum, потрібно бачити його варіанти. Приховані варіанти зробили б match неможливим.

4. Чим `use crate::position::Position` відрізняється від `use super::position::Position`?

Відповідь: `crate::` — абсолютний шлях від кореня проєкту. `super::` — відносний від батьківського модуля. Результат може бути однаковим, але `crate::` зрозуміліший.

### Рівень 3

5. Навіщо потрібен `pub use` (re-export)?

Відповідь: щоб спростити публічний інтерфейс. Замість `use my_crate::internal::deep::module::Type` — `use my_crate::Type`. Внутрішня структура прихована.

### Рівень 4

6. Як би ви організували проєкт з 20 модулями? Чи потрібні піддиректорії?

Відповідь: групувати пов'язані модулі у директорії-модулі. Наприклад: `agent/` (state, battery, sensors), `navigation/` (route, pathfinding), `communication/` (protocol, serialization). Кожна директорія має `mod.rs` з re-exports ключових типів.

---

## Резюме

Модулі організовують код у логічні одиниці. Inline-модулі — `mod name { }`. Файлові — `mod name;` + файл `src/name.rs`.

Все приватне за замовчуванням. `pub` робить елемент видимим ззовні. Для struct — `pub` потрібне для кожного поля окремо. Для enum — варіанти публічні автоматично.

`use` скорочує шляхи. `crate::` — від кореня, `super::` — від батьківського модуля. `pub use` — re-export для спрощення інтерфейсу.

Інкапсуляція: приватні поля + публічні методи (getter/setter). Зовнішній код не може створити невалідний стан.

Модулі-директорії для великих проєктів. `pub(crate)` — видимість лише в межах проєкту.

---

## Що далі

Код розділений на модулі. Але як перевірити, що кожен модуль працює правильно? У Розділі 19 (Модульне тестування) ви дізнаєтесь про `#[test]`, `assert!`, `cargo test` та напишете тести для кожного модуля агента. Тести — це специфікація поведінки: якщо тест проходить, модуль працює згідно специфікації. Це критично перед тим, як ускладнювати систему у Частині III.
