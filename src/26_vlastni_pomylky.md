# Розділ 26. Власні типи помилок та thiserror

## Анотація

У Розділах 24–25 ви навчились обробляти та пропагувати помилки через Option, Result та оператор `?`. Але типом помилки був або `String` (зрозуміло для людини, неструктуровано для коду), або `Box<dyn Error>` (приймає все, але caller не може зробити match на конкретний варіант). Для реального БПЛА-агента цього недостатньо: "GPS offline" потребує повторної спроби через 5 секунд, "батарея критична" — негайного повернення на базу, "невалідна конфігурація" — зупинки запуску. Caller повинен розрізняти ці ситуації програмно, а не парсити текст повідомлення.

Рішення — створити власний enum помилок, де кожен варіант описує конкретну помилку зі структурованими даними. Крейт `thiserror` автоматизує реалізацію стандартних traits (Display, Error, From), а `anyhow` надає зручну альтернативу для програм, де конкретний тип помилки менш важливий.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Створити enum помилок з варіантами різних типів (unit, tuple, struct).
2. Реалізувати traits `Display` та `Error` вручну для розуміння механізму.
3. Використати `thiserror` для автоматичної реалізації через derive-макроси.
4. Реалізувати `From` для автоматичної конвертації помилок при використанні `?`.
5. Побудувати ієрархію помилок (модульний рівень → рівень агента → рівень місії).
6. Обрати між `thiserror` та `anyhow` для конкретної ситуації.

---

## Ключові терміни

**Custom error type (власний тип помилки)** — enum, що описує всі можливі помилки модуля чи бібліотеки. Кожен варіант — окрема категорія помилок.

**thiserror** — крейт, що автоматично реалізує traits Display, Error та From через derive-макроси. Призначений для бібліотек.

**anyhow** — крейт, що надає тип `anyhow::Error` — обгортку для будь-якої помилки з підтримкою ланцюжків контексту. Призначений для програм (бінарних крейтів).

**Error hierarchy (ієрархія помилок)** — структура, де помилки нижнього рівня (SensorError) обгортаються помилками вищого рівня (AgentError), які, в свою чергу — помилками ще вищого (MissionError).

---

## Мотиваційний кейс

Система керування рухом автомобілів Tesla обробляє десятки тисяч помилок на секунду — від незначних (одноразове спрацювання ультразвукового сенсора) до критичних (відмова гальм). Кожна помилка класифікована за типом, джерелом та серйозністю. Система не просто виводить текстове повідомлення — вона програмно вирішує: ігнорувати, зафіксувати в лозі, попередити водія, або негайно передати керування людині. Ця класифікація можлива лише тому, що помилки представлені як структуровані типи, а не рядки.

Наш БПЛА-агент потребує того самого: програмне розрізнення категорій помилок для автоматичного прийняття рішень.

---

## 26.1. Проблема: String та Box\<dyn Error\> — недостатньо

У Розділах 24–25 ми використовували два підходи до типу помилки. Розглянемо обмеження кожного.

`String` як тип помилки — зрозумілий для людини, але код не може його аналізувати:

```rust
fn start_mission(battery: u8) -> Result<(), String> {
    if battery < 20 {
        return Err(format!("Батарея критична: {}%", battery));
    }
    Ok(())
}

fn main() {
    match start_mission(10) {
        Ok(()) => println!("Місія розпочата"),
        Err(msg) => {
            // Як визначити ТИП помилки?
            // Парсити рядок? if msg.contains("Батарея") ???
            // Це крихке та ненадійне
            println!("Помилка: {}", msg);
        }
    }
}
```

Caller бачить лише текст. Щоб з'ясувати, чи це проблема батареї, GPS чи маршруту — потрібно парсити рядок. Це крихко: зміна тексту повідомлення зламає логіку обробки.

`Box<dyn Error>` — приймає будь-яку помилку, але caller теж не може зробити match:

```rust
use std::error::Error;

fn load_mission(path: &str) -> Result<(), Box<dyn Error>> {
    let _content = std::fs::read_to_string(path)?;
    Ok(())
}

fn main() {
    match load_mission("mission.txt") {
        Ok(()) => println!("Завантажено"),
        Err(e) => {
            // e — Box<dyn Error>. Який конкретний тип всередині?
            // Потрібен downcast — незручний та може провалитись
            if let Some(io_err) = e.downcast_ref::<std::io::Error>() {
                println!("IO помилка: {}", io_err);
            } else {
                println!("Інша помилка: {}", e);
            }
        }
    }
}
```

downcast працює, але це рантайм-перевірка типу, яка може провалитись. Rust пропонує краще рішення — типу, де кожен варіант відомий на етапі компіляції.

---

## 26.2. Enum помилок: структурований підхід

Ідея проста: створити enum, де кожен варіант — окрема категорія помилок. Caller робить match на варіанти та обробляє кожен по-своєму. Це не нова концепція — ви вже працювали з enum у Розділі 17. Але тепер enum застосовується для моделювання помилок, а не станів агента.

Ключове рішення при проєктуванні enum помилок — які дані включити в кожен варіант. Правило: варіант повинен містити достатньо інформації, щоб caller міг прийняти рішення без додаткових запитів. "Сенсор offline" — недостатньо: який сенсор? Без цієї інформації caller не знає, чи це GPS (критично) чи термометр (не критично). "Сенсор 'GPS' offline" — краще. "Сенсор 'GPS' offline, остання відповідь 5с тому" — ще краще: можна вирішити, чи варто чекати (5с — можливо тимчасовий збій) чи переключитись на альтернативу.

Варіанти enum можуть мати різну структуру, як і звичайні enum з Розділу 17: unit (без даних), tuple (позиційні поля), struct (іменовані поля).

```rust
#[derive(Debug)]
enum SensorError {
    Offline(String),           // tuple: назва сенсора
    InvalidReading(f64),       // tuple: невалідне значення
    Timeout { sensor: String, ms: u64 }, // struct: деталі таймауту
}

fn read_sensor(id: &str) -> Result<f64, SensorError> {
    match id {
        "gps" => Err(SensorError::Offline(String::from("GPS"))),
        "temp" => Err(SensorError::InvalidReading(-999.0)),
        "baro" => Err(SensorError::Timeout { 
            sensor: String::from("Барометр"), 
            ms: 5000 
        }),
        _ => Ok(42.0),
    }
}

fn main() {
    match read_sensor("gps") {
        Ok(val) => println!("Показання: {:.1}", val),
        Err(SensorError::Offline(name)) => {
            println!("{} offline — спробувати через 5с", name);
        }
        Err(SensorError::InvalidReading(val)) => {
            println!("Невалідне показання {:.1} — ігнорувати", val);
        }
        Err(SensorError::Timeout { sensor, ms }) => {
            println!("{} не відповів за {} мс — перезавантажити", sensor, ms);
        }
    }
}
```

Кожна гілка match — конкретна стратегія обробки. Offline — повторна спроба. InvalidReading — ігнорувати. Timeout — перезавантажити сенсор. Це неможливо зі String чи Box<dyn Error> без крихкого парсингу рядків.

Крім того, match є exhaustive — компілятор перевіряє, що всі варіанти покриті. Якщо ви додасте новий варіант `SensorError::Calibrating` — кожне місце з match одразу покаже помилку компіляції "non-exhaustive patterns". Це означає, що додавання нового типу помилки автоматично змушує оновити всю обробку — жодна помилка не буде проігнорована випадково. Порівняйте з String: додавання нового тексту повідомлення не спричиняє жодних попереджень у коді обробки.

---

## 26.3. Реалізація Display та Error вручну

Щоб enum помилок повноцінно працював в екосистемі Rust, потрібно реалізувати два traits. Розглянемо їх окремо, бо розуміння ручної реалізації допомагає зрозуміти, що робить thiserror під капотом.

`std::fmt::Display` — trait для текстового представлення. Коли ви пишете `println!("Помилка: {}", err)` — викликається Display. Коли `?` виводить помилку при завершенні main — теж Display. Без Display ваш enum не можна вивести для людини.

`std::error::Error` — маркерний trait, що позначає тип як "помилку". Він потребує Display + Debug (Debug зазвичай через derive). Error дає два методи: `source()` (попередня помилка в ланцюжку) та `description()` (застарілий, не використовуйте). Головне: Error потрібен, щоб ваш enum був сумісний з `Box<dyn Error>`, з anyhow, та з будь-яким кодом, що працює з trait Error. Без реалізації Error ваш enum — просто enum, а не "помилка" з точки зору екосистеми.

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum NavigationError {
    NoGpsSignal,
    TargetOutOfRange { distance: f64, max: f64 },
    RouteBlocked(String),
}

impl fmt::Display for NavigationError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            NavigationError::NoGpsSignal => {
                write!(f, "GPS-сигнал відсутній")
            }
            NavigationError::TargetOutOfRange { distance, max } => {
                write!(f, "Ціль на відстані {:.1}, максимум {:.1}", distance, max)
            }
            NavigationError::RouteBlocked(reason) => {
                write!(f, "Маршрут заблоковано: {}", reason)
            }
        }
    }
}

impl Error for NavigationError {}

fn navigate(target: (f64, f64)) -> Result<Vec<(f64, f64)>, NavigationError> {
    let distance = (target.0 * target.0 + target.1 * target.1).sqrt();
    if distance > 100.0 {
        Err(NavigationError::TargetOutOfRange { distance, max: 100.0 })
    } else {
        Ok(vec![(0.0, 0.0), target])
    }
}

fn main() {
    match navigate((200.0, 300.0)) {
        Ok(route) => println!("Маршрут: {} точок", route.len()),
        Err(e) => println!("Помилка навігації: {}", e),
        // "Помилка навігації: Ціль на відстані 360.6, максимум 100.0"
    }
}
```

Метод `fmt` у Display приймає форматер `f` та записує у нього текст через макрос `write!`. Кожен варіант потребує окремої гілки match. Тип повернення `fmt::Result` — це `Result<(), fmt::Error>`, де помилка означає, що запис у форматер провалився (надзвичайно рідко на практиці).

Ручна реалізація — це 15+ рядків шаблонного коду для кожного enum. Display потребує match з write! для кожного варіанту. Error — порожній impl (default implementation достатня для більшості випадків, бо source() повертає None, що означає "ця помилка не має попередника"). Для трьох варіантів це терпимо. Для десяти — стомлює і створює ризик помилки при копіюванні шаблону. Саме тому існує thiserror.

---

## 26.4. thiserror: автоматизація через derive

Крейт `thiserror` робить те саме, що ручна реалізація, але через derive-макроси. Ви пишете enum з анотаціями, а thiserror генерує impl Display, impl Error та (опціонально) impl From. Додайте у Cargo.toml:

```toml
[dependencies]
thiserror = "1"
```

Тепер замість ручного impl:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum NavigationError {
    #[error("GPS-сигнал відсутній")]
    NoGpsSignal,
    
    #[error("Ціль на відстані {distance:.1}, максимум {max:.1}")]
    TargetOutOfRange { distance: f64, max: f64 },
    
    #[error("Маршрут заблоковано: {0}")]
    RouteBlocked(String),
}
```

Три рядки замість п'ятнадцяти. Макрос `#[derive(Error)]` автоматично реалізує Display (через шаблони `#[error("...")]`) та Error. Синтаксис шаблонів у `#[error("...")]` працює як `format!`: `{field_name}` для іменованих полів, `{0}` для полів за позицією, `{field:.1}` для форматування. Зверніть увагу: `use thiserror::Error` — це import макросу derive з крейту thiserror, а не стандартний `std::error::Error`. Вони не конфліктують, бо thiserror::Error — це derive-макрос, а std::error::Error — trait.

Атрибут `#[from]` — друга ключова можливість thiserror. Він автоматично реалізує `From` для конвертації при `?`:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AgentError {
    #[error("Помилка навігації: {0}")]
    Navigation(#[from] NavigationError),
    
    #[error("Помилка сенсора: {0}")]
    Sensor(#[from] SensorError),
    
    #[error("Помилка IO: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Батарея критична: {level}%")]
    LowBattery { level: u8 },
    
    #[error("Невалідна конфігурація: {0}")]
    InvalidConfig(String),
}
```

`#[from]` генерує:

```rust
impl From<NavigationError> for AgentError {
    fn from(e: NavigationError) -> Self {
        AgentError::Navigation(e)
    }
}
```

Тепер `?` автоматично обгортає NavigationError у AgentError::Navigation:

```rust
fn prepare_and_navigate(target: (f64, f64), battery: u8) -> Result<Vec<(f64, f64)>, AgentError> {
    if battery < 20 {
        return Err(AgentError::LowBattery { level: battery });
    }
    
    let route = navigate(target)?;  // NavigationError → AgentError::Navigation автоматично
    
    Ok(route)
}
```

Без `#[from]` довелося б писати `navigate(target).map_err(AgentError::Navigation)?` — явна конвертація. З `#[from]` — просто `?`, і From робить конвертацію автоматично.

---

## 26.5. Ієрархія помилок: від модуля до місії

У реальному проєкті помилки мають природну ієрархію, що відповідає архітектурі модулів. Модуль sensor знає про SensorError — таймаути, невалідні показання, відсутність калібрування. Модуль navigation — про NavigationError — втрату GPS, перешкоди, вихід за межі. Модуль agent об'єднує обидва в AgentError. Модуль mission — у MissionError, що включає AgentError плюс помилки конфігурації та комунікації.

Ця ієрархія — не довільне рішення, а відображення залежностей між модулями. Agent залежить від sensor та navigation, тому AgentError містить SensorError та NavigationError. Mission залежить від agent, тому MissionError містить AgentError. Ієрархія помилок дзеркалює ієрархію модулів.

Кожен рівень обгортає помилки нижнього рівня, додаючи контекст. Caller на верхньому рівні може обробити помилку з потрібною деталізацією: або за категорією верхнього рівня ("проблема з агентом" — загальна стратегія), або "розгорнути" до конкретної помилки нижнього ("NoGps у навігації конкретного агента" — специфічна стратегія).

```rust
use thiserror::Error;

// Рівень 1: модулі
#[derive(Debug, Error)]
enum SensorError {
    #[error("Сенсор '{name}' offline")]
    Offline { name: String },
    #[error("Таймаут сенсора '{sensor}': {ms} мс")]
    Timeout { sensor: String, ms: u64 },
}

#[derive(Debug, Error)]
enum NavigationError {
    #[error("GPS-сигнал відсутній")]
    NoGps,
    #[error("Перешкода на маршруті: {0}")]
    Obstacle(String),
}

// Рівень 2: агент
#[derive(Debug, Error)]
enum AgentError {
    #[error("Помилка сенсора: {0}")]
    Sensor(#[from] SensorError),
    #[error("Помилка навігації: {0}")]
    Navigation(#[from] NavigationError),
    #[error("Батарея критична: {0}%")]
    LowBattery(u8),
}

// Рівень 3: місія
#[derive(Debug, Error)]
enum MissionError {
    #[error("Помилка агента '{agent_id}': {source}")]
    Agent {
        agent_id: String,
        #[source] source: AgentError,
    },
    #[error("Невалідна конфігурація місії: {0}")]
    Config(String),
    #[error("Помилка комунікації: {0}")]
    Communication(String),
}
```

Обробка на верхньому рівні:

```rust
fn handle_mission_error(err: &MissionError) {
    match err {
        MissionError::Agent { agent_id, source } => {
            match source {
                AgentError::LowBattery(level) => {
                    println!("Агент {}: повернути на базу (батарея {}%)", agent_id, level);
                }
                AgentError::Navigation(NavigationError::NoGps) => {
                    println!("Агент {}: перейти на інерціальну навігацію", agent_id);
                }
                AgentError::Sensor(SensorError::Timeout { sensor, ms }) => {
                    println!("Агент {}: перезапустити {} (таймаут {}мс)", agent_id, sensor, ms);
                }
                other => {
                    println!("Агент {}: загальна помилка — {}", agent_id, other);
                }
            }
        }
        MissionError::Config(msg) => {
            println!("Конфігурація: зупинити запуск — {}", msg);
        }
        MissionError::Communication(msg) => {
            println!("Зв'язок: спробувати альтернативний канал — {}", msg);
        }
    }
}
```

Caller розгортає помилку до потрібного рівня деталізації. Для одних ситуацій достатньо знати "проблема з агентом". Для інших — "конкретно NoGps у навігації конкретного агента". Ієрархія дає обидва рівні.

Атрибут `#[source]` (на відміну від `#[from]`) позначає поле як "джерело" помилки без автоматичної реалізації From. Це для випадків, де потрібні додаткові поля (agent_id), і автоматична конвертація неможлива — потрібно створювати MissionError::Agent вручну з передачею id.

---

## 26.6. anyhow: альтернатива для програм

`thiserror` — для бібліотек, де caller потребує match на конкретні помилки. `anyhow` — для програм (main.rs, CLI-утиліти, сервіси), де caller — це ви самі, і зазвичай достатньо вивести повідомлення та зупинити виконання або записати в лог.

Різниця між бібліотекою та програмою — фундаментальна для розуміння вибору. Бібліотека (crate, що експортує функції для інших) не знає, хто її викликає. Може програма з GUI, може сервер, може embedded-система. Бібліотека повинна дати caller-у максимум інформації для прийняття рішення — тому конкретний enum через thiserror. Програма — кінцевий споживач. Вона знає контекст. "Не вдалось підключитись до бази" — програма може вивести повідомлення, записати в лог, спробувати альтернативне з'єднання. Для цього достатньо текстового опису помилки з ланцюжком контексту.

Додайте у Cargo.toml:

```toml
[dependencies]
anyhow = "1"
```

anyhow надає тип `anyhow::Result<T>` (аліас для `Result<T, anyhow::Error>`) та макрос `anyhow!` для створення помилок:

```rust
use anyhow::{Context, Result, anyhow};

fn load_config(path: &str) -> Result<String> {
    let content = std::fs::read_to_string(path)
        .context(format!("Не вдалось прочитати конфігурацію '{}'", path))?;
    
    if content.is_empty() {
        return Err(anyhow!("Файл конфігурації порожній"));
    }
    
    Ok(content)
}

fn prepare_mission(config_path: &str) -> Result<()> {
    let config = load_config(config_path)
        .context("Підготовка місії")?;
    
    println!("Конфігурація: {} байтів", config.len());
    Ok(())
}

fn main() -> Result<()> {
    prepare_mission("mission.toml")?;
    println!("Місія підготовлена");
    Ok(())
}
```

`.context("опис")` — головна фішка anyhow. Він додає прошарок контексту до помилки, створюючи ланцюжок. При виведенні помилки (наприклад, при аварійному завершенні main) весь ланцюжок видимий:

```
Error: Підготовка місії

Caused by:
    0: Не вдалось прочитати конфігурацію 'mission.toml'
    1: No such file or directory (os error 2)
```

Це як stack trace, але для бізнес-логіки: верхній рівень — "що намагались зробити", нижній — "що саме пішло не так на рівні ОС". Без anyhow довелося б будувати цей ланцюжок вручну через map_err на кожному рівні.

На практиці thiserror та anyhow часто використовуються разом у одному проєкті: бібліотечні модулі (sensor.rs, navigation.rs) визначають помилки через thiserror, а main.rs обробляє їх через anyhow:

```rust
// sensor.rs — бібліотечний код
use thiserror::Error;

#[derive(Debug, Error)]
pub enum SensorError {
    #[error("Сенсор offline")]
    Offline,
}

// main.rs — програмний код
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let reading = read_sensor("gps")
        .context("Ініціалізація місії")?;
    // SensorError автоматично конвертується в anyhow::Error
    Ok(())
}
```

Це розділення відповідальностей: модулі описують що може піти не так (thiserror), а точка входу вирішує як це показати (anyhow).

---

## 26.7. Практика: ієрархія помилок БПЛА-агента

Побудуємо повну систему помилок для агента з Розділу 20, замінивши всі String та unwrap на структуровані типи:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum SensorError {
    #[error("Сенсор '{name}' не відповідає (таймаут {timeout_ms}мс)")]
    Timeout { name: String, timeout_ms: u64 },
    
    #[error("Невалідне показання сенсора '{name}': {value}")]
    InvalidReading { name: String, value: f64 },
    
    #[error("Сенсор '{0}' не ініціалізований")]
    NotInitialized(String),
}

#[derive(Debug, Error)]
pub enum NavigationError {
    #[error("GPS-сигнал втрачено")]
    GpsLost,
    
    #[error("Ціль ({x:.1}, {y:.1}) за межами досяжності ({distance:.1} > {max:.1})")]
    OutOfRange { x: f64, y: f64, distance: f64, max: f64 },
    
    #[error("Перешкода виявлена на ({x:.1}, {y:.1})")]
    Obstacle { x: f64, y: f64 },
}

#[derive(Debug, Error)]
pub enum AgentError {
    #[error(transparent)]
    Sensor(#[from] SensorError),
    
    #[error(transparent)]
    Navigation(#[from] NavigationError),
    
    #[error("Батарея критична: {level}% (мінімум {min}%)")]
    LowBattery { level: u8, min: u8 },
    
    #[error("Некоректний стан: {0}")]
    InvalidState(String),
}

fn check_battery(level: u8, min: u8) -> Result<(), AgentError> {
    if level < min {
        Err(AgentError::LowBattery { level, min })
    } else {
        Ok(())
    }
}

fn get_position() -> Result<(f64, f64), NavigationError> {
    // Симуляція: GPS працює
    Ok((47.12, 35.45))
}

fn read_altitude() -> Result<f64, SensorError> {
    Ok(150.0) // симуляція
}

fn execute_step(battery: u8) -> Result<(f64, f64, f64), AgentError> {
    check_battery(battery, 20)?;
    let pos = get_position()?;     // NavigationError → AgentError::Navigation через #[from]
    let alt = read_altitude()?;    // SensorError → AgentError::Sensor через #[from]
    Ok((pos.0, pos.1, alt))
}

fn main() {
    match execute_step(15) {
        Ok((x, y, alt)) => println!("Позиція: ({:.1}, {:.1}), висота: {:.0}", x, y, alt),
        Err(AgentError::LowBattery { level, min }) => {
            println!("Повернення на базу: батарея {}% < {}%", level, min);
        }
        Err(AgentError::Navigation(nav_err)) => {
            println!("Навігація: {}", nav_err);
        }
        Err(AgentError::Sensor(sen_err)) => {
            println!("Сенсор: {}", sen_err);
        }
        Err(other) => {
            println!("Інша помилка: {}", other);
        }
    }
}
```

Атрибут `#[error(transparent)]` означає: "для Display використовувати Display внутрішньої помилки". Тобто `AgentError::Sensor(SensorError::Timeout { ... })` виведеться як "Сенсор 'GPS' не відповідає (таймаут 5000мс)" — без зайвого обгортання "Помилка сенсора: Сенсор 'GPS'...".

---

## 26.8. Prompt Engineering: проєктування ієрархії помилок

### Промпт-шаблон

```
Я пишу систему керування БПЛА з модулями: 
sensor, navigation, communication, mission.
Кожен модуль може мати 3-5 типів помилок.

Спроєктуй ієрархію помилок з thiserror:
1. Enum для кожного модуля
2. Загальний AgentError з #[from] для кожного
3. MissionError верхнього рівня
4. Для кожного варіанту — які дані зберігати

Поясни вибір: чому ці дані, а не інші.
```

### Вправа з PE

Опишіть AI можливі помилки вашого агента з Розділу 20. Попросіть спроєктувати ієрархію. Оцініть: чи всі помилки покриті? Чи достатньо даних у кожному варіанті для прийняття рішення (не просто для виведення)? Чи не занадто глибока ієрархія?

---

## Лабораторна робота No26

### Мета

Замінити всі String та Box\<dyn Error\> у коді агента на структуровані типи помилок.

### Завдання базового рівня

1. Створити `SensorError` з 3+ варіантами через thiserror.
2. Створити `NavigationError` з 3+ варіантами.
3. Створити `AgentError` з #[from] для обох.
4. Переписати функції агента з Розділів 20–25, використовуючи нові типи.
5. У main — match на конкретні варіанти з різною обробкою.

### Варіанти

**A.** Повна ієрархія: додати CommunicationError, ConfigError, MissionError. Продемонструвати три рівні вкладеності match.

**B.** Graceful degradation: для кожного варіанту AgentError реалізувати функцію `fn severity(&self) -> Severity` (Critical, Warning, Info). Обробка залежить від severity.

**C.** anyhow + thiserror разом: модулі визначають помилки через thiserror, main.rs використовує anyhow з .context(). Показати ланцюжок контексту при вкладеній помилці.

### Критерії

| Критерій | Бал |
|----------|-----|
| Жоден String як тип помилки | 15 |
| thiserror derive з інформативними повідомленнями | 20 |
| #[from] для автоматичної конвертації | 15 |
| match на конкретні варіанти з різною обробкою | 20 |
| Тести помилкових сценаріїв | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `conflicting implementations of From`

Два поля з `#[from]` мають однаковий тип:

```rust
#[derive(Debug, Error)]
enum MyError {
    #[error("A: {0}")]
    A(#[from] std::io::Error),
    #[error("B: {0}")]
    B(#[from] std::io::Error), // ПОМИЛКА: конфлікт From<io::Error>
}
```

Виправлення: кожен тип може мати лише один `#[from]`. Якщо потрібні два варіанти для io::Error — один з #[from], другий створювати вручну через map_err.

### Помилка 2: `thiserror::Error not found`

Забули додати залежність:

```toml
[dependencies]
thiserror = "1"
```

Або неправильний import: потрібно `use thiserror::Error;`, а не `use std::error::Error;` (для derive).

### Помилка 3: `{field}` у #[error] не знаходить поле

Для tuple-варіантів використовуйте `{0}`, `{1}`, а не іменовані поля.

### Помилка 4: anyhow::Error не можна match

`anyhow::Error` — це opaque тип. Для match на конкретні помилки використовуйте `downcast_ref`:

```rust
if let Some(sensor_err) = err.downcast_ref::<SensorError>() {
    // обробка SensorError
}
```

Це незручно — саме тому для бібліотек використовують thiserror, а не anyhow.

---

## Додатково

### Коли один enum, коли кілька

Правило: один модуль — один enum помилок. Якщо модуль sensor має функції read, calibrate, initialize — усі повертають SensorError. Якщо модуль зростає і помилки стають занадто різноманітними (20+ варіантів) — це сигнал розбити модуль на кілька.

Уникайте "God Error" — єдиного enum на весь проєкт із 50 варіантами. Це еквівалент "God Object" в ООП: все залежить від усього, зміна одного варіанту може зачепити далекий код.

### source() та ланцюжки помилок

Trait Error має метод `source()`, що повертає "попередню" помилку. thiserror автоматично реалізує source() для полів з `#[from]` та `#[source]`. Це дозволяє побудувати ланцюжок: MissionError → AgentError → SensorError → io::Error. Логери та діагностичні інструменти можуть пройти весь ланцюжок і показати повну "стек-трасу" помилки.

### Порівняння підходів

| Підхід | Плюси | Мінуси | Коли використовувати |
|--------|-------|--------|---------------------|
| `String` | Простий, гнучкий | Неструктурований, match неможливий | Прототипи, скрипти |
| `Box<dyn Error>` | Приймає все | match через downcast, runtime | ? у main, швидкий код |
| `thiserror` enum | Структурований match, compile-time | Більше коду, потрібен derive | Бібліотеки, серйозний код |
| `anyhow` | Зручний context(), без enum | match незручний | Програми (main.rs) |

---

## Контрольні запитання

### Рівень 1

1. Чим enum помилок кращий за String?

Відповідь: enum дає структурований match на конкретні варіанти (LowBattery, GpsLost тощо). String потребує парсингу тексту, що крихко та ненадійно. Додавання нового варіанту до enum — помилка компіляції у всіх match; зміна тексту String — мовчазна поломка.

2. Що робить `#[from]` у thiserror?

Відповідь: автоматично реалізує `From<InnerError> for OuterError`, що дозволяє оператору `?` конвертувати InnerError у OuterError без map_err.

### Рівень 2

3. Чим thiserror відрізняється від anyhow?

Відповідь: thiserror генерує impl Display/Error/From для ваших enum — створює типи помилок. anyhow надає тип-обгортку для будь-якої помилки з ланцюжками .context(). thiserror — для бібліотек (caller потребує match). anyhow — для програм (помилки виводяться або логуються).

4. Що означає `#[error(transparent)]`?

Відповідь: Display зовнішнього варіанту делегується Display внутрішньої помилки. Замість "Помилка навігації: GPS відсутній" виведеться просто "GPS відсутній" — без зайвого обгортання.

### Рівень 3

5. Створіть ієрархію помилок для системи з модулями config, network, storage. Кожен — 2 варіанти. Об'єднайте в AppError з #[from].

Відповідь:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("Файл не знайдено: {0}")]
    NotFound(String),
    #[error("Невалідний формат: {0}")]
    InvalidFormat(String),
}

#[derive(Debug, Error)]
enum NetworkError {
    #[error("Таймаут з'єднання: {0}мс")]
    Timeout(u64),
    #[error("Хост недоступний: {0}")]
    Unreachable(String),
}

#[derive(Debug, Error)]
enum StorageError {
    #[error("Диск повний")]
    DiskFull,
    #[error("Файл пошкоджений: {0}")]
    Corrupted(String),
}

#[derive(Debug, Error)]
enum AppError {
    #[error(transparent)]
    Config(#[from] ConfigError),
    #[error(transparent)]
    Network(#[from] NetworkError),
    #[error(transparent)]
    Storage(#[from] StorageError),
}
```

### Рівень 4

6. Чому "God Error" (один enum на весь проєкт) — антипатерн?

Відповідь: єдиний enum з 30+ варіантами створює залежність усього коду від одного типу. Зміна, додавання або видалення будь-якого варіанту змушує перекомпілювати весь проєкт і оновити кожен match. Модуль sensor не повинен "знати" про помилки модуля network. Ієрархія enum (кожен модуль — свій enum, верхній рівень об'єднує через #[from]) зберігає ізоляцію модулів і мінімізує каскадні зміни.

---

## Резюме

Власний enum помилок — структурований підхід до обробки помилок, де кожен варіант описує конкретну категорію з відповідними даними. Caller робить match на варіанти і обробляє кожну помилку по-різному — програмно, а не через парсинг тексту.

`thiserror` автоматизує реалізацію Display, Error та From через derive-макроси: `#[derive(Error)]`, `#[error("шаблон")]`, `#[from]`. Мінімум boilerplate.

`anyhow` — альтернатива для програм: тип-обгортка з ланцюжками .context(). Зручний для main.rs та CLI, де match на помилки не потрібен.

Ієрархія: один модуль — один enum. Верхній рівень об'єднує нижні через #[from]. Уникайте "God Error".

`thiserror` для бібліотек, `anyhow` для програм, разом — для проєктів із обома частинами.

---

## Що далі

Помилки структуровані, пропагація працює. Але агент поки що — це одна struct з фіксованим набором полів. Якщо з'явиться GroundBot замість Drone — доведеться копіювати код із змінами. Якщо потрібно обробляти агентів різних типів однаково (відправити команду будь-якому агенту, отримати статус від будь-якого) — потрібна абстракція.

У Розділі 27 ви вивчите traits — механізм Rust для визначення спільної поведінки. Trait `Agent` з методами `perceive()`, `decide()`, `act()` дозволить створювати різні типи агентів, що реалізують спільний інтерфейс. Стандартні traits (`Display`, `Debug`, `Clone`, `Default`) зроблять ваші типи "першокласними громадянами" екосистеми Rust.
