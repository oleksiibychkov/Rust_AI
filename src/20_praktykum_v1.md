# Розділ 20. Практикум: Автономний агент v1.0

## Анотація

Це фінальний розділ Частини II. Ви вивчили ownership (13–14), borrowing (15), structs (16), enums (17), модулі (18) та тестування (19). Кожна тема розглядалась окремо. Тепер час об'єднати все в одному модульному проєкті — повноцінному автономному агенті з чіткою архітектурою, машиною станів, інкапсуляцією та тестами. Цей агент стане фундаментом для Частини III, де з'являться колекції, обробка помилок та traits.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Спроєктувати модульну архітектуру Rust-проєкту з кількох файлів.
2. Реалізувати структуру з приватними полями, публічними методами та конструктором.
3. Реалізувати машину станів через enum з переходами у match.
4. Написати unit-тести для кожного модуля.
5. Провести AI-assisted code review та зафіксувати рішення у промпт-журналі.

---

## 20.1. Архітектура проєкту

### Файлова структура

```
drone_agent_v1/
├── Cargo.toml
└── src/
    ├── main.rs          // точка входу, оголошення модулів
    ├── position.rs      // координати та геометрія
    ├── state.rs         // enum AgentState + переходи
    ├── agent.rs         // struct Agent + методи
    └── command.rs       // enum Command + обробка
```

П'ять файлів — п'ять відповідальностей. `position.rs` не знає про Agent. `state.rs` не знає про Position. `agent.rs` залежить від обох. `command.rs` залежить від agent. `main.rs` об'єднує все.

### Залежності між модулями

```
main.rs
  ├── command.rs → agent.rs → position.rs
  │                         → state.rs
  └── (використовує agent та command)
```

Стрілки вказують напрямок залежності (хто від кого залежить). Циклів немає — це обов'язкова вимога Rust.

---

## 20.2. Модуль position: координати

```rust
// src/position.rs

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Position {
    x: f64,
    y: f64,
}

impl Position {
    pub fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }

    pub fn origin() -> Self {
        Self { x: 0.0, y: 0.0 }
    }

    pub fn x(&self) -> f64 { self.x }
    pub fn y(&self) -> f64 { self.y }

    pub fn distance_to(&self, other: &Position) -> f64 {
        let dx = other.x - self.x;
        let dy = other.y - self.y;
        (dx * dx + dy * dy).sqrt()
    }

    pub fn translate(&mut self, dx: f64, dy: f64) {
        self.x += dx;
        self.y += dy;
    }

    pub fn is_origin(&self) -> bool {
        self.x == 0.0 && self.y == 0.0
    }
}

impl std::fmt::Display for Position {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "({:.1}, {:.1})", self.x, self.y)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_origin() {
        let p = Position::origin();
        assert!(p.is_origin());
        assert_eq!(p.x(), 0.0);
        assert_eq!(p.y(), 0.0);
    }

    #[test]
    fn test_distance_345() {
        let a = Position::origin();
        let b = Position::new(3.0, 4.0);
        assert!((a.distance_to(&b) - 5.0).abs() < 1e-10);
    }

    #[test]
    fn test_distance_symmetric() {
        let a = Position::new(1.0, 2.0);
        let b = Position::new(4.0, 6.0);
        assert_eq!(a.distance_to(&b), b.distance_to(&a));
    }

    #[test]
    fn test_translate() {
        let mut p = Position::origin();
        p.translate(5.0, 3.0);
        assert_eq!(p.x(), 5.0);
        assert_eq!(p.y(), 3.0);
    }

    #[test]
    fn test_copy_semantics() {
        let p1 = Position::new(1.0, 2.0);
        let p2 = p1;  // Copy — p1 залишається дійсною
        assert_eq!(p1, p2);
    }
}
```

Position — `Copy` (всі поля `f64` — Copy). Це означає: `let p2 = p1` копіює, а не переміщує. Поля приватні, доступ через getter-и `x()`, `y()`. `Display` дозволяє `println!("{}", pos)`.

Зверніть увагу на тести: кожен перевіряє одну властивість. `test_distance_symmetric` — математичний інваріант (відстань симетрична). `test_copy_semantics` — перевірка, що Copy працює.

---

## 20.3. Модуль state: машина станів

```rust
// src/state.rs

#[derive(Debug, Clone)]
pub enum AgentState {
    Grounded,
    Airborne { altitude: f64 },
    Charging { target_percent: u8 },
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

    pub fn is_airborne(&self) -> bool {
        matches!(self, AgentState::Airborne { .. })
    }

    pub fn is_grounded(&self) -> bool {
        matches!(self, AgentState::Grounded)
    }

    pub fn can_accept_commands(&self) -> bool {
        match self {
            AgentState::Emergency { .. } => false,
            _ => true,
        }
    }
}

impl std::fmt::Display for AgentState {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AgentState::Grounded => write!(f, "На землі"),
            AgentState::Airborne { altitude } => write!(f, "У повітрі ({:.0} м)", altitude),
            AgentState::Charging { target_percent } => write!(f, "Зарядка (ціль {}%)", target_percent),
            AgentState::Emergency { reason } => write!(f, "АВАРІЯ: {}", reason),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_grounded_is_operational() {
        let s = AgentState::Grounded;
        assert!(s.is_operational());
    }

    #[test]
    fn test_emergency_not_operational() {
        let s = AgentState::Emergency { reason: String::from("тест") };
        assert!(!s.is_operational());
    }

    #[test]
    fn test_airborne_can_accept_commands() {
        let s = AgentState::Airborne { altitude: 100.0 };
        assert!(s.can_accept_commands());
    }

    #[test]
    fn test_emergency_cannot_accept_commands() {
        let s = AgentState::Emergency { reason: String::from("тест") };
        assert!(!s.can_accept_commands());
    }
}
```

AgentState — не Copy (варіант Emergency містить String). Тому `#[derive(Clone)]` — для явного копіювання через `.clone()`. Кожен метод — `&self`, бо лише читає стан.

---

## 20.4. Модуль agent: структура агента

```rust
// src/agent.rs

use crate::position::Position;
use crate::state::AgentState;

const BATTERY_MAX: u8 = 100;
const COST_TAKEOFF: u8 = 10;
const COST_MOVE: u8 = 5;
const COST_TURN: u8 = 2;
const MIN_TAKEOFF_BATTERY: u8 = 10;
const DEFAULT_ALTITUDE: f64 = 50.0;

pub struct Agent {
    id: String,
    position: Position,
    state: AgentState,
    battery: u8,
    heading: u16,
}

impl Agent {
    pub fn new(id: &str) -> Self {
        Self {
            id: String::from(id),
            position: Position::origin(),
            state: AgentState::Grounded,
            battery: BATTERY_MAX,
            heading: 0,
        }
    }

    // --- Getters ---
    pub fn id(&self) -> &str { &self.id }
    pub fn position(&self) -> &Position { &self.position }
    pub fn battery(&self) -> u8 { self.battery }
    pub fn heading(&self) -> u16 { self.heading }
    pub fn state(&self) -> &AgentState { &self.state }

    pub fn status(&self) -> String {
        format!("{}: {} | {}° | {}%",
            self.id, self.state, self.heading, self.battery)
    }

    // --- Переходи станів ---
    pub fn takeoff(&mut self) -> bool {
        if !self.state.is_grounded() {
            return false;
        }
        if self.battery < MIN_TAKEOFF_BATTERY {
            return false;
        }
        self.battery -= COST_TAKEOFF;
        self.state = AgentState::Airborne { altitude: DEFAULT_ALTITUDE };
        true
    }

    pub fn land(&mut self) -> bool {
        if !self.state.is_airborne() {
            return false;
        }
        self.state = AgentState::Grounded;
        true
    }

    pub fn move_forward(&mut self) -> bool {
        if !self.state.is_airborne() || self.battery < COST_MOVE {
            return false;
        }
        let (dx, dy) = match self.heading {
            0 => (0.0, 1.0),
            90 => (1.0, 0.0),
            180 => (0.0, -1.0),
            270 => (-1.0, 0.0),
            _ => (0.0, 0.0),
        };
        self.position.translate(dx, dy);
        self.battery -= COST_MOVE;
        true
    }

    pub fn turn_right(&mut self) -> bool {
        if !self.state.is_airborne() || self.battery < COST_TURN {
            return false;
        }
        self.heading = (self.heading + 90) % 360;
        self.battery -= COST_TURN;
        true
    }

    pub fn start_charging(&mut self) -> bool {
        if !self.state.is_grounded() {
            return false;
        }
        self.state = AgentState::Charging { target_percent: BATTERY_MAX };
        true
    }

    pub fn emergency(&mut self, reason: &str) {
        self.state = AgentState::Emergency {
            reason: String::from(reason),
        };
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_new_agent() {
        let a = Agent::new("T-01");
        assert_eq!(a.id(), "T-01");
        assert_eq!(a.battery(), 100);
        assert!(a.state().is_grounded());
        assert!(a.position().is_origin());
    }

    #[test]
    fn test_takeoff_success() {
        let mut a = Agent::new("T");
        assert!(a.takeoff());
        assert!(a.state().is_airborne());
        assert_eq!(a.battery(), 90);
    }

    #[test]
    fn test_takeoff_from_airborne_fails() {
        let mut a = Agent::new("T");
        a.takeoff();
        assert!(!a.takeoff());  // вже у повітрі
    }

    #[test]
    fn test_land_from_grounded_fails() {
        let mut a = Agent::new("T");
        assert!(!a.land());
    }

    #[test]
    fn test_move_consumes_battery() {
        let mut a = Agent::new("T");
        a.takeoff();
        let before = a.battery();
        a.move_forward();
        assert_eq!(a.battery(), before - COST_MOVE);
    }

    #[test]
    fn test_move_changes_position() {
        let mut a = Agent::new("T");
        a.takeoff();
        a.move_forward();
        assert!(!a.position().is_origin());
    }

    #[test]
    fn test_emergency_from_any_state() {
        let mut a = Agent::new("T");
        a.takeoff();
        a.emergency("тест");
        assert!(!a.state().is_operational());
    }

    #[test]
    fn test_turn_changes_heading() {
        let mut a = Agent::new("T");
        a.takeoff();
        assert_eq!(a.heading(), 0);
        a.turn_right();
        assert_eq!(a.heading(), 90);
    }
}
```

Ключові рішення:

Поля приватні — зовнішній код не може встановити `battery = 200`. Доступ через getter-и. Модифікація — через методи з валідацією.

Методи повертають `bool` — true при успіху, false при відмові. Це дозволяє caller вирішити, як реагувати на відмову. Виняток — `emergency`, яка завжди успішна (аварійний режим доступний з будь-якого стану).

Константи винесені на рівень модуля — `COST_TAKEOFF`, `MIN_TAKEOFF_BATTERY`. Зміна вартості зльоту — одна зміна в одному місці.

---

## 20.5. Модуль command: обробка команд

```rust
// src/command.rs

use crate::agent::Agent;

pub enum Command {
    Takeoff,
    Land,
    Forward,
    TurnRight,
    Charge,
    Emergency(String),
    Status,
}

impl Command {
    pub fn from_input(input: &str) -> Option<Command> {
        match input.trim().to_lowercase().as_str() {
            "зліт" => Some(Command::Takeoff),
            "посадка" => Some(Command::Land),
            "вперед" => Some(Command::Forward),
            "право" => Some(Command::TurnRight),
            "заряд" => Some(Command::Charge),
            "статус" => Some(Command::Status),
            s if s.starts_with("аварія ") => {
                let reason = s.trim_start_matches("аварія ").to_string();
                Some(Command::Emergency(reason))
            }
            _ => None,
        }
    }
}

pub fn execute(agent: &mut Agent, cmd: &Command) -> String {
    match cmd {
        Command::Takeoff => {
            if agent.takeoff() { format!("Зліт виконано") }
            else { format!("Зліт неможливий") }
        }
        Command::Land => {
            if agent.land() { format!("Посадка виконана") }
            else { format!("Посадка неможлива") }
        }
        Command::Forward => {
            if agent.move_forward() { format!("Рух вперед") }
            else { format!("Рух неможливий") }
        }
        Command::TurnRight => {
            if agent.turn_right() { format!("Поворот праворуч") }
            else { format!("Поворот неможливий") }
        }
        Command::Charge => {
            if agent.start_charging() { format!("Зарядка почалась") }
            else { format!("Зарядка неможлива") }
        }
        Command::Emergency(reason) => {
            agent.emergency(reason);
            format!("АВАРІЯ: {}", reason)
        }
        Command::Status => agent.status(),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_takeoff() {
        assert!(matches!(Command::from_input("зліт"), Some(Command::Takeoff)));
    }

    #[test]
    fn test_parse_unknown() {
        assert!(Command::from_input("невідома").is_none());
    }

    #[test]
    fn test_execute_takeoff() {
        let mut agent = Agent::new("T");
        let result = execute(&mut agent, &Command::Takeoff);
        assert_eq!(result, "Зліт виконано");
        assert!(agent.state().is_airborne());
    }
}
```

`Command::from_input` повертає `Option<Command>` — None для невідомих команд. Функція `execute` приймає `&mut Agent` (позичає для модифікації) та `&Command` (позичає для читання).

---

## 20.6. Головний файл: main.rs

```rust
// src/main.rs

mod position;
mod state;
mod agent;
mod command;

use agent::Agent;
use command::Command;
use std::io;

fn read_input() -> String {
    use std::io::Write;
    print!("> ");
    io::stdout().flush().unwrap();
    let mut input = String::new();
    io::stdin().read_line(&mut input).unwrap();
    input.trim().to_string()
}

fn main() {
    println!("=== Автономний агент БПЛА v1.0 ===");
    println!("Команди: зліт, посадка, вперед, право, заряд, статус, вихід\n");

    let mut scout = Agent::new("SCOUT-01");
    println!("{}\n", scout.status());

    loop {
        let input = read_input();
        if input == "вихід" {
            println!("\nЗавершення. {}", scout.status());
            break;
        }

        match Command::from_input(&input) {
            Some(cmd) => {
                let result = command::execute(&mut scout, &cmd);
                println!("  {}", result);
            }
            None => println!("  Невідома команда: '{}'", input),
        }
    }
}
```

`main.rs` — лише 30 рядків: оголошення модулів, цикл команд, вивід. Жодної бізнес-логіки.

---

## 20.7. Аналіз: що ми використали з Частини II

| Концепція | Де застосовано |
|-----------|---------------|
| Ownership (13–14) | Agent володіє String id; AgentState::Emergency володіє reason |
| Borrowing (15) | `&self` у getter-ах, `&mut Agent` у execute, `&str` у параметрах |
| Struct (16) | Position, Agent — дані з методами |
| Enum (17) | AgentState (машина станів), Command (перелік команд) |
| Модулі (18) | 5 файлів з чіткою відповідальністю |
| Тести (19) | Unit-тести в кожному модулі |

---

## 20.8. Обмеження та мотивація для Частини III

Агент v1.0 працює, але має обмеження:

Немає історії позицій — агент не пам'ятає, де він був. Рішення: `Vec<Position>` (Частина III, колекції).

Фіксований набір waypoints — патрульний маршрут "зашитий" у код. Рішення: `Vec` та `HashMap` для динамічних маршрутів.

Примітивна обробка помилок — `bool` для успіху/невдачі замість детальної інформації. Рішення: `Result<T, E>` з власними типами помилок (Частина III).

Немає серіалізації — стан агента не можна зберегти чи передати. Рішення: serde (Частина III).

Немає абстракцій — кожен тип агента (розвідник, транспорт, бойовий) потребує окремих struct. Рішення: traits та generics (Частина III).

---

## Prompt Engineering: повний цикл AI-assisted розробки

Цей розділ — ваш перший досвід повного циклу:

Крок 1: Спроєктуйте архітектуру самостійно. Намалюйте модулі та залежності на папері.

Крок 2: Реалізуйте код. Почніть з Position (найпростіший), потім State, Agent, Command, main.

Крок 3: Попросіть AI зробити code review:

```
Ось мій проєкт агента БПЛА (4 модулі):
[код кожного файлу]
Code review:
1. Чи правильно обрано pub / private?
2. Чи правильно &self / &mut self у методах?
3. Які тести додати?
4. Яке рефакторинг запропонуєш?
```

Крок 4: Оцініть кожну пораду AI — прийняти / відхилити / модифікувати. Запишіть у промпт-журнал з обгрунтуванням.

---

## Лабораторна робота No20

### Мета

Створити модульний проєкт автономного агента v1.0.

### Завдання

**Частина 1 (4 бали):** Реалізуйте проєкт з 4 модулів (position, state, agent, command). Мінімум 5 методів Agent, 4 варіанти AgentState, 6 варіантів Command.

**Частина 2 (3 бали):** Написати мінімум 15 unit-тестів (5 для Position, 5 для Agent, 5 для Command). Всі повинні проходити.

**Частина 3 (3 бали):** AI code review + промпт-журнал з мінімум 5 записами.

### Критерії

| Критерій | Бал |
|----------|-----|
| 4 модулі, компілюється, працює | 40 |
| 15+ тестів, всі зелені | 30 |
| AI review + промпт-журнал (5 записів) | 30 |

---

## Troubleshooting

### `unresolved import crate::position`

Забули `mod position;` у main.rs. Кожен модуль повинен бути оголошений.

### Методи Agent не бачать Position

Додайте `use crate::position::Position;` на початку agent.rs.

### `cannot borrow agent as mutable` в циклі main

Перевірте, що `Command::from_input` не позичає agent. Парсинг команди не залежить від агента.

---

## Контрольні запитання

### Рівень 1

1. Скільки модулів у проєкті v1.0? Назвіть відповідальність кожного.

### Рівень 2

2. Чому поля Agent приватні, а методи публічні?

Відповідь: інкапсуляція — зовнішній код не може встановити невалідне значення батареї. Модифікація — тільки через методи з валідацією.

### Рівень 3

3. Додайте стан Patrolling з вектором waypoints. Як це змінить архітектуру?

### Рівень 4

4. Порівняйте v1.0 (Частина II) з Розділом 12 (Частина I). Що покращилось?

Відповідь: замість 7 окремих змінних — структура Agent. Замість else-if — enum Command + match. Замість одного файлу — 5 модулів. Замість ручної перевірки — 15+ автоматичних тестів. Інваріанти захищені приватними полями.

---

## Резюме

Автономний агент v1.0 об'єднує всі концепції Частини II: ownership (Agent володіє String id), borrowing (getter-и через &self, модифікація через &mut self), struct (Position, Agent), enum (AgentState, Command), модулі (5 файлів з чіткою відповідальністю), тести (unit-тести в кожному модулі).

Архітектура: position (геометрія) → state (машина станів) → agent (логіка) → command (обробка команд) → main (точка входу).

Приватні поля + публічні методи = інкапсуляція. Bool-повернення для операцій = явна обробка відмов. Константи для параметрів = легка зміна конфігурації.

---

## Що далі

Частина II завершена. Ваш агент — модульний об'єкт з машиною станів та тестами. Але він обмежений: фіксовані дані, примітивна обробка помилок, один тип для всіх агентів.

У Частині III ("Робастність, дані та інструментарій") агент отримає динамічні колекції (Vec, HashMap), повноцінну обробку помилок (Result, оператор ?), абстракції (traits, generics), серіалізацію (serde) та перетвориться на робастну систему, готову до багатопотоковості у Частині IV.
