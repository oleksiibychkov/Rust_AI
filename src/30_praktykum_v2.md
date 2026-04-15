# Розділ 30. Практикум: Робастний агент v2.0

## Анотація

Це фінальний розділ Частини III. У Розділах 21–29 ви вивчили динамічні колекції (Vec, HashMap, HashSet), ітератори, обробку помилок (Option, Result, ?, thiserror), traits та generics — кожну тему окремо. Тепер час об'єднати все у рефакторингу агента з v1.0 (Розділ 20) до v2.0. Агент стане робастною системою: динамічні дані замість фіксованих полів, структурована обробка помилок замість паніки, абстрактний інтерфейс Agent замість конкретного типу. Цей практикум побудований як покроковий рефакторинг — не переписування з нуля, а еволюція існуючого коду.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Провести рефакторинг існуючого проєкту з інтеграцією нових концепцій.
2. Замінити фіксовані поля на динамічні колекції.
3. Замінити паніку на структуровану обробку помилок.
4. Витягти абстрактний інтерфейс через trait.
5. Застосувати ітератори для обробки колекцій.
6. Провести AI-assisted code review рефакторингу.

---

## 30.1. Що змінюється: v1.0 → v2.0

Порівняємо обмеження v1.0 та можливості v2.0:

| Аспект | v1.0 (Розділ 20) | v2.0 (цей розділ) |
|--------|------------------|-------------------|
| Позиції | Одна поточна | Історія у Vec |
| Завдання | Одна поточна команда | Черга у Vec |
| Виявлені об'єкти | Не зберігаються | HashMap за id |
| Відвідані зони | Не відстежуються | HashSet |
| Помилки | unwrap / паніка | Result + AgentError |
| Інтерфейс | Конкретний struct Agent | trait Agent |
| Обробка даних | Цикли for | Ітератори |

Рефакторинг — це не "викинути та написати заново". Це покрокова зміна з перевіркою на кожному кроці. Кожен крок — компілюється, тести проходять, функціональність зберігається або розширюється.

---

## 30.2. Файлова структура v2.0

```
drone_agent_v2/
├── Cargo.toml
└── src/
    ├── main.rs           // точка входу
    ├── error.rs          // AgentError, SensorError, NavigationError
    ├── position.rs       // Position + HasPosition trait
    ├── state.rs          // AgentState enum
    ├── agent.rs          // trait Agent + struct DroneAgent
    ├── mission.rs        // MissionLog, TaskQueue
    └── sensor.rs         // SensorBuffer
```

Порівняно з v1.0 додались три модулі: error.rs (ієрархія помилок), mission.rs (динамічні дані місії), sensor.rs (буфер показань). Trait Agent виділений у agent.rs.

---

## 30.3. Крок 1: ієрархія помилок (error.rs)

Замінюємо всі String та unwrap на структуровані типи:

```rust
// src/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum SensorError {
    #[error("Сенсор '{name}' не відповідає")]
    Offline { name: String },
    #[error("Невалідне показання: {value}")]
    InvalidReading { value: f64 },
}

#[derive(Debug, Error)]
pub enum NavigationError {
    #[error("GPS недоступний")]
    NoGps,
    #[error("Ціль за межами досяжності: {distance:.1} > {max:.1}")]
    OutOfRange { distance: f64, max: f64 },
}

#[derive(Debug, Error)]
pub enum AgentError {
    #[error(transparent)]
    Sensor(#[from] SensorError),
    #[error(transparent)]
    Navigation(#[from] NavigationError),
    #[error("Батарея критична: {level}%")]
    LowBattery { level: u8 },
    #[error("Невалідна команда: {0}")]
    InvalidCommand(String),
    #[error("Агент не operational")]
    NotOperational,
}
```

Cargo.toml:

```toml
[dependencies]
thiserror = "1"
```

---

## 30.4. Крок 2: Position з trait HasPosition (position.rs)

```rust
// src/position.rs
use std::fmt;

#[derive(Debug, Clone, PartialEq, Default)]
pub struct Position {
    pub x: f64,
    pub y: f64,
}

impl Position {
    pub fn new(x: f64, y: f64) -> Self {
        Position { x, y }
    }
    
    pub fn distance_to(&self, other: &Position) -> f64 {
        ((other.x - self.x).powi(2) + (other.y - self.y).powi(2)).sqrt()
    }
}

impl fmt::Display for Position {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.1}, {:.1})", self.x, self.y)
    }
}

// Generic trait для будь-чого з координатами
pub trait HasPosition {
    fn pos(&self) -> &Position;
    
    fn distance_to_point(&self, target: &Position) -> f64 {
        self.pos().distance_to(target)
    }
}

impl HasPosition for Position {
    fn pos(&self) -> &Position { self }
}
```

HasPosition — generic trait (Розділ 29), що дозволить find_nearest працювати з будь-яким типом, не лише Position.

---

## 30.5. Крок 3: trait Agent та DroneAgent (agent.rs)

```rust
// src/agent.rs
use std::fmt;
use std::collections::{HashMap, HashSet};
use crate::position::{Position, HasPosition};
use crate::state::AgentState;
use crate::error::AgentError;

pub trait Agent: fmt::Display {
    fn id(&self) -> &str;
    fn position(&self) -> &Position;
    fn battery(&self) -> u8;
    fn state(&self) -> &AgentState;
    fn execute_step(&mut self) -> Result<(), AgentError>;
    
    fn is_operational(&self) -> bool {
        self.battery() > 10 && *self.state() != AgentState::Emergency
    }
    
    fn needs_recharge(&self) -> bool {
        self.battery() < 20
    }
}

pub struct DroneAgent {
    id: String,
    position: Position,
    battery: u8,
    state: AgentState,
    altitude: f64,
    // Нові поля v2.0
    history: Vec<Position>,
    visited_zones: HashSet<(i32, i32)>,
    detected_objects: HashMap<String, (Position, String)>,
}

impl DroneAgent {
    pub fn new(id: &str) -> Self {
        DroneAgent {
            id: String::from(id),
            position: Position::default(),
            battery: 100,
            state: AgentState::Idle,
            altitude: 0.0,
            history: Vec::new(),
            visited_zones: HashSet::new(),
            detected_objects: HashMap::new(),
        }
    }
    
    pub fn move_to(&mut self, target: &Position) -> Result<(), AgentError> {
        if !self.is_operational() {
            return Err(AgentError::NotOperational);
        }
        if self.battery < 5 {
            return Err(AgentError::LowBattery { level: self.battery });
        }
        
        // Зберігаємо поточну позицію в історію
        self.history.push(self.position.clone());
        
        // Позначаємо зону як відвідану (цілочисельна сітка)
        let zone = (self.position.x as i32, self.position.y as i32);
        self.visited_zones.insert(zone);
        
        // Переміщення
        self.position = target.clone();
        self.battery = self.battery.saturating_sub(3);
        
        Ok(())
    }
    
    pub fn detect_object(&mut self, id: &str, pos: Position, kind: &str) {
        self.detected_objects.insert(
            String::from(id),
            (pos, String::from(kind)),
        );
    }
    
    pub fn path_length(&self) -> f64 {
        self.history.windows(2)
            .map(|w| w[0].distance_to(&w[1]))
            .sum()
    }
    
    pub fn coverage(&self, grid_size: i32) -> f64 {
        let total = (grid_size * grid_size) as f64;
        self.visited_zones.len() as f64 / total * 100.0
    }
    
    pub fn objects_by_type(&self, kind: &str) -> Vec<(&String, &Position)> {
        self.detected_objects.iter()
            .filter(|(_, (_, k))| k == kind)
            .map(|(id, (pos, _))| (id, pos))
            .collect()
    }
}

impl Agent for DroneAgent {
    fn id(&self) -> &str { &self.id }
    fn position(&self) -> &Position { &self.position }
    fn battery(&self) -> u8 { self.battery }
    fn state(&self) -> &AgentState { &self.state }
    
    fn execute_step(&mut self) -> Result<(), AgentError> {
        if !self.is_operational() {
            return Err(AgentError::NotOperational);
        }
        self.battery = self.battery.saturating_sub(2);
        if self.battery < 10 {
            self.state = AgentState::Emergency;
        }
        Ok(())
    }
}

impl HasPosition for DroneAgent {
    fn pos(&self) -> &Position { &self.position }
}

impl fmt::Display for DroneAgent {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[{}] {} | {} | bat={}% | alt={:.0}m | history={} | objects={}",
               self.id, self.state, self.position, self.battery,
               self.altitude, self.history.len(), self.detected_objects.len())
    }
}
```

Ключові зміни порівняно з v1.0: execute_step повертає `Result<(), AgentError>` замість `()` — помилки не ігноруються. Історія позицій у `Vec<Position>`. Відвідані зони у `HashSet`. Виявлені об'єкти у `HashMap`. path_length через `windows(2).map().sum()` — ітераторний стиль.

---

## 30.6. Крок 4: AgentState (state.rs)

```rust
// src/state.rs
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
pub enum AgentState {
    Idle,
    Patrolling,
    Returning,
    Charging,
    Emergency,
}

impl fmt::Display for AgentState {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AgentState::Idle => write!(f, "Idle"),
            AgentState::Patrolling => write!(f, "Patrol"),
            AgentState::Returning => write!(f, "Return"),
            AgentState::Charging => write!(f, "Charge"),
            AgentState::Emergency => write!(f, "EMERGENCY"),
        }
    }
}

impl Default for AgentState {
    fn default() -> Self { AgentState::Idle }
}
```

---

## 30.7. Крок 5: main.rs — інтеграція

```rust
// src/main.rs
mod error;
mod position;
mod state;
mod agent;

use position::Position;
use agent::{Agent, DroneAgent};

fn swarm_status(agents: &[Box<dyn Agent>]) {
    println!("=== Рій: {} агентів ===", agents.len());
    for agent in agents {
        let marker = if agent.is_operational() { " " } else { "!" };
        println!(" [{}] {}", marker, agent);
    }
    
    let operational = agents.iter().filter(|a| a.is_operational()).count();
    let need_charge = agents.iter().filter(|a| a.needs_recharge()).count();
    println!("Operational: {}/{}, need recharge: {}", operational, agents.len(), need_charge);
}

fn main() {
    let mut scout = DroneAgent::new("SCOUT-01");
    let mut recon = DroneAgent::new("RECON-02");
    
    // Симуляція місії
    let waypoints = vec![
        Position::new(10.0, 0.0),
        Position::new(10.0, 10.0),
        Position::new(0.0, 10.0),
        Position::new(0.0, 0.0),
    ];
    
    println!("--- Патрулювання ---");
    for wp in &waypoints {
        match scout.move_to(wp) {
            Ok(()) => println!("SCOUT-01 → {}", wp),
            Err(e) => println!("SCOUT-01 помилка: {}", e),
        }
    }
    
    // Виявлення об'єктів
    scout.detect_object("OBJ-01", Position::new(5.0, 3.0), "будівля");
    scout.detect_object("OBJ-02", Position::new(8.0, 7.0), "транспорт");
    scout.detect_object("OBJ-03", Position::new(2.0, 9.0), "будівля");
    
    // Статистика через ітератори
    println!("\n--- Статистика SCOUT-01 ---");
    println!("Пройдено: {:.1} одиниць", scout.path_length());
    println!("Покриття (10x10): {:.0}%", scout.coverage(10));
    
    let buildings = scout.objects_by_type("будівля");
    println!("Будівлі ({}):", buildings.len());
    for (id, pos) in &buildings {
        println!("  {} на {}", id, pos);
    }
    
    // Гетерогенний рій через trait objects
    println!("\n--- Рій ---");
    let swarm: Vec<Box<dyn Agent>> = vec![
        Box::new(scout),
        Box::new(recon),
    ];
    swarm_status(&swarm);
}
```

---

## 30.8. Що де використано: мапа концепцій

| Концепція | Де у v2.0 | Розділ |
|-----------|-----------|--------|
| Vec | history, waypoints | 21 |
| HashMap | detected_objects | 22 |
| HashSet | visited_zones | 22 |
| Ітератори | path_length (windows), objects_by_type (filter+map), swarm_status (filter+count) | 23 |
| Option | HasPosition::find_nearest повертає Option | 24 |
| Result | execute_step, move_to повертають Result | 24-25 |
| Оператор ? | всередині move_to | 25 |
| thiserror | AgentError, SensorError, NavigationError | 26 |
| Traits | Agent, HasPosition, Display, Default | 27-28 |
| Trait objects | Vec\<Box\<dyn Agent\>\> | 28 |
| Generics | HasPosition як generic bound | 29 |

---

## 30.9. Prompt Engineering: AI-assisted рефакторинг

PE-фокус цього практикуму — повний цикл AI-assisted рефакторингу. Не генерація з нуля, а покрокова модернізація.

### Промпт-шаблон: план рефакторингу

```
Ось мій код агента v1.0: [вставити]

Вимоги до v2.0:
- Історія позицій у Vec
- Виявлені об'єкти у HashMap
- Result замість unwrap для execute_step та move_to
- Trait Agent з default methods

Запропонуй покроковий план рефакторингу.
Для кожного кроку: що змінити, які файли зачіпає, 
які тести додати після кроку.
```

### Промпт-шаблон: review v2.0

```
Ось мій рефакторинг v1.0 → v2.0: [вставити diff або новий код]

Перевір:
1. Чи всі unwrap замінені на Result?
2. Чи правильна ієрархія помилок?
3. Чи мінімальні trait bounds?
4. Чи ітератори використані де доречно?
5. Які тести відсутні?
```

### Промпт-журнал

Для цього практикуму обов'язковий промпт-журнал з мінімум 3 записами: план рефакторингу, review коду, генерація тестів. Для кожного запису: промпт, відповідь AI (ключові моменти), що прийнято / відхилено / модифіковано, обґрунтування.

---

## Лабораторна робота No30

### Мета

Провести рефакторинг агента v1.0 → v2.0 з інтеграцією концепцій Частини III.

### Завдання

1. Створити ієрархію помилок через thiserror (мінімум 3 enum).
2. Додати Vec для історії позицій, HashMap для об'єктів, HashSet для зон.
3. Замінити всі unwrap на Result з пропагацією через `?`.
4. Витягти trait Agent з мінімум 3 default methods.
5. Реалізувати swarm_status через `&[Box<dyn Agent>]`.
6. Написати мінімум 10 тестів (включаючи помилкові сценарії).
7. Вести промпт-журнал з 3+ записами.

### Критерії

| Критерій | Бал |
|----------|-----|
| Ієрархія помилок (thiserror, #[from]) | 15 |
| Динамічні колекції (Vec, HashMap, HashSet) | 15 |
| Жоден unwrap у продакшен-коді | 10 |
| Trait Agent з default methods | 15 |
| Ітератори замість циклів де доречно | 10 |
| Тести (10+, включаючи error cases) | 15 |
| Промпт-журнал (3+ записи з аналізом) | 10 |
| Модульна архітектура (6+ файлів) | 10 |

---

## Troubleshooting

### `move_to` не компілюється після додавання history

Якщо history зберігає Position, а Position не Clone — `self.history.push(self.position)` переміщує position. Рішення: derive Clone для Position та використовувати `self.position.clone()`.

### `cannot borrow self as mutable more than once`

Типова проблема при одночасній модифікації кількох полів struct через методи, що приймають `&mut self`. Рішення: працювати з полями напряму (`self.history.push(...)`, `self.battery -= 3`) замість виклику інших `&mut self` методів.

### Trait object не підтримує метод конкретного типу

`agent.altitude()` не працює для `&dyn Agent` — altitude не у trait. Рішення: або додати метод у trait (якщо він потрібен для всіх агентів), або downcast (якщо потрібен лише для Drone — але це сигнал переглянути дизайн).

---

## Контрольні запитання

### Рівень 2

1. Порівняйте v1.0 та v2.0: що покращилось у кожному аспекті (дані, помилки, абстракція)?

Відповідь: дані — фіксовані поля → динамічні колекції (агент пам'ятає історію, відстежує зони, зберігає об'єкти). Помилки — unwrap/паніка → Result/AgentError (graceful degradation замість краху). Абстракція — конкретний struct → trait Agent (гетерогенний рій, легке додавання нових типів).

### Рівень 4

2. Яке архітектурне рішення v2.0 ви б змінили для рою з 1000 агентів? Чому?

Відповідь: HashSet<(i32, i32)> для зон — з 1000 агентів потрібна спільна карта зон, а не окрема для кожного. HashMap об'єктів — теж спільний для рою, щоб уникнути дублікатів. Trait objects Vec<Box<dyn Agent>> — може бути повільним при частій ітерації; якщо всі агенти одного типу — generics. Це підводить до Частини IV (Arc, Mutex, shared state).

---

## Резюме

Рефакторинг v1.0 → v2.0 інтегрує всі концепції Частини III: Vec для динамічних даних (історія позицій), HashMap та HashSet для структурованого зберігання (об'єкти, зони), ітератори для обробки (path_length через windows, фільтрація об'єктів), Result та thiserror для безпечної обробки помилок (execute_step, move_to повертають Result), trait Agent з default methods для абстрактного інтерфейсу, trait objects для гетерогенного рою.

Покроковий рефакторинг — правильний підхід: кожна зміна ізольована, тести перевіряють коректність після кожного кроку. AI-assisted workflow: план рефакторингу → реалізація → review → тести.

---

## Що далі

Частина III завершена. Ваш агент — робастна система з динамічними даними, структурованими помилками та абстрактним інтерфейсом. Але він однопотоковий: один агент виконує одну дію за раз. Рій з 10 агентів працює послідовно, не паралельно.

Частина IV ("Smart Pointers та спільна пам'ять") змінює це. Розділ 31 — замикання (closures), фундамент для паралельних обчислень. Розділ 32 — Box<T> та розміщення на heap. Розділи 33–34 — Rc, Arc, RefCell, Mutex — механізми спільного володіння та синхронізації. Розділ 35 — потоки (threads). Розділ 36 — канали (channels). Агент стане елементом паралельного рою, де кілька агентів працюють одночасно, обмінюючись повідомленнями.
