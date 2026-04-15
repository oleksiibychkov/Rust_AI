# Розділ 35. Серіалізація та serde

## Анотація

Агент БПЛА створює дані: стан місії, координати маршруту, показання сенсорів, виявлені об'єкти. Ці дані живуть у пам'яті як Rust-структури — Position, AgentState, MissionConfig. Але пам'ять — це лише один з контекстів. Конфігурація агента зберігається у файлі TOML на диску. Стан місії передається координатору по мережі як JSON. Snapshot стану зберігається у компактному бінарному форматі для відтворення після перезапуску. Кожен із цих сценаріїв потребує серіалізації (перетворення структури в послідовність байтів) та десеріалізації (зворотне перетворення).

Крейт `serde` — де-факто стандарт серіалізації у Rust. Він не прив'язаний до конкретного формату: одна анотація `#[derive(Serialize, Deserialize)]` дає вашому типу можливість перетворюватись на JSON, TOML, YAML, MessagePack, bincode та десятки інших форматів — без жодних змін у коді структури. Це можливо завдяки trait-based архітектурі: serde визначає traits Serialize та Deserialize, а кожен формат реалізує свій Serializer та Deserializer.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Додати `serde`, `serde_json`, `serde_toml` до проєкту через Cargo.toml.
2. Серіалізувати та десеріалізувати struct та enum через derive-макроси.
3. Налаштувати серіалізацію через атрибути: `rename`, `default`, `skip`, `tag`.
4. Серіалізувати enum з даними у різних режимах (tagged, untagged).
5. Зберігати та завантажувати стан агента з файлу (JSON, TOML).
6. Обрати формат серіалізації для конкретної задачі.

---

## Ключові терміни

**Серіалізація (serialization)** — перетворення структури даних у послідовність байтів для збереження або передачі.

**Десеріалізація (deserialization)** — зворотний процес: відновлення структури з послідовності байтів.

**serde** — фреймворк серіалізації для Rust. Визначає traits Serialize та Deserialize, незалежні від конкретного формату.

**serde_json** — реалізація serde для формату JSON.

**serde_toml** (toml) — реалізація serde для формату TOML (Tom's Obvious, Minimal Language), популярного для конфігураційних файлів.

---

## Мотиваційний кейс

Рій БПЛА виконує багатогодинну місію. Що станеться при збої живлення або аварійному перезапуску? Без збереження стану — агент "забуває" все: позицію, історію, виявлені об'єкти. Місія починається з нуля. Якщо перед збоєм стан серіалізувався у файл кожні 30 секунд — після перезапуску агент десеріалізує останній snapshot і продовжує з того ж місця. 

Інший сценарій: координатор рою надсилає команди агентам і отримує звіти. Команда — це Rust-структура `Command { target: Position, priority: u8 }`. Але мережа передає байти, не Rust-структури. Потрібен спільний формат: JSON для читабельності (дебаг, логи), або bincode для швидкості (мілісекундні команди в реальному часі). serde робить обидва варіанти тривіальними.

---

## 35.1. Підключення serde

serde складається з ядра (traits та derive-макроси) та формат-специфічних крейтів. Типова конфігурація Cargo.toml:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"
```

`features = ["derive"]` вмикає процедурні макроси `#[derive(Serialize, Deserialize)]`. Без цієї фічі довелось би реалізовувати traits вручну — десятки рядків шаблонного коду для кожного типу.

`serde_json` — для JSON. `toml` — для TOML (конфігурації). Для інших форматів: `serde_yaml`, `bincode` (компактний бінарний), `serde_cbor`, `rmp-serde` (MessagePack).

---

## 35.2. Базова серіалізація та десеріалізація

Найпростіший випадок — derive для структури з примітивними полями:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Position {
    x: f64,
    y: f64,
}

#[derive(Debug, Serialize, Deserialize)]
struct AgentSnapshot {
    id: String,
    position: Position,
    battery: u8,
    altitude: f64,
    active: bool,
}

fn main() {
    let snapshot = AgentSnapshot {
        id: String::from("SCOUT-01"),
        position: Position { x: 47.5, y: 35.2 },
        battery: 85,
        altitude: 150.0,
        active: true,
    };
    
    // Серіалізація: Rust struct → JSON string
    let json = serde_json::to_string_pretty(&snapshot).unwrap();
    println!("JSON:\n{}", json);
    
    // Десеріалізація: JSON string → Rust struct
    let restored: AgentSnapshot = serde_json::from_str(&json).unwrap();
    println!("Відновлено: {:?}", restored);
}
```

Вивід:

```json
{
  "id": "SCOUT-01",
  "position": {
    "x": 47.5,
    "y": 35.2
  },
  "battery": 85,
  "altitude": 150.0,
  "active": true
}
```

`serde_json::to_string_pretty` — серіалізація з форматуванням (відступи, переноси). `to_string` — компактний JSON без пробілів. `from_str` — десеріалізація з рядка. Обидва повертають `Result` — десеріалізація може провалитись, якщо JSON невалідний або не відповідає структурі.

Derive Serialize та Deserialize працює рекурсивно: AgentSnapshot містить Position, і Position теж повинен мати derive. Якщо будь-яке вкладене поле не реалізує Serialize — помилка компіляції.

Стандартні типи (String, Vec, HashMap, Option, числа, bool) вже реалізують Serialize та Deserialize — додаткових зусиль не потрібно. `Vec<Position>` серіалізується як JSON-масив, `HashMap<String, f64>` — як JSON-об'єкт, `Option<f64>` — як значення або null.

---

## 35.3. Серіалізація enum

Enum — типовий спосіб моделювання станів та повідомлень у Rust. serde підтримує кілька режимів серіалізації enum:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
enum AgentState {
    Idle,
    Patrolling,
    Moving { target_x: f64, target_y: f64 },
    Emergency(String),
}

fn main() {
    let states = vec![
        AgentState::Idle,
        AgentState::Moving { target_x: 10.0, target_y: 20.0 },
        AgentState::Emergency(String::from("GPS втрачено")),
    ];
    
    let json = serde_json::to_string_pretty(&states).unwrap();
    println!("{}", json);
}
```

За замовчуванням serde серіалізує enum як externally tagged:

```json
[
  "Idle",
  { "Moving": { "target_x": 10.0, "target_y": 20.0 } },
  { "Emergency": "GPS втрачено" }
]
```

Unit-варіант (Idle) — рядок. Struct-варіант (Moving) — об'єкт із ключем-назвою варіанту. Tuple-варіант (Emergency) — ключ із значенням.

Для повідомлень між агентами зручніший internally tagged режим:

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]
enum Command {
    MoveTo { x: f64, y: f64 },
    SetAltitude { meters: f64 },
    ReturnToBase,
    Emergency { reason: String },
}
```

```json
{ "type": "MoveTo", "x": 10.0, "y": 20.0 }
{ "type": "ReturnToBase" }
{ "type": "Emergency", "reason": "Низька батарея" }
```

`#[serde(tag = "type")]` додає поле "type" із назвою варіанту. Це зручніше для споживачів (JavaScript, Python), бо кожне повідомлення — плаский JSON-об'єкт із полем-дискримінатором.

Є також `#[serde(tag = "type", content = "data")]` (adjacently tagged) та `#[serde(untagged)]` (без тегу — serde пробує кожен варіант по черзі).

---

## 35.4. Атрибути serde: тонке налаштування

serde має десятки атрибутів для налаштування серіалізації. Найуживаніші:

`#[serde(rename = "name")]` — змінити назву поля у серіалізованому форматі:

```rust
#[derive(Serialize, Deserialize)]
struct Agent {
    #[serde(rename = "agentId")]
    id: String,                      // JSON: "agentId", не "id"
    #[serde(rename = "batteryLevel")]
    battery: u8,                     // JSON: "batteryLevel"
}
```

Корисно коли Rust-конвенція (snake_case) відрізняється від JSON-конвенції (camelCase). Є також `#[serde(rename_all = "camelCase")]` на рівні struct — перейменовує всі поля одразу.

`#[serde(default)]` — використати Default::default() якщо поле відсутнє у JSON:

```rust
#[derive(Serialize, Deserialize)]
struct Config {
    target: (f64, f64),
    #[serde(default)]
    max_altitude: f64,   // якщо відсутнє — 0.0 (Default для f64)
    #[serde(default = "default_battery_threshold")]
    battery_threshold: u8, // якщо відсутнє — значення з функції
}

fn default_battery_threshold() -> u8 { 20 }
```

`#[serde(skip)]` — не серіалізувати поле (пропустити):

```rust
#[derive(Serialize, Deserialize)]
struct Agent {
    id: String,
    battery: u8,
    #[serde(skip)]
    internal_counter: u32,  // не потрапить у JSON
}
```

`#[serde(skip_serializing_if = "Option::is_none")]` — пропустити поле якщо воно None:

```rust
#[derive(Serialize, Deserialize)]
struct Report {
    agent_id: String,
    status: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<String>,  // є у JSON лише якщо Some
}
```

---

## 35.5. TOML: конфігурація агента

TOML — формат конфігураційних файлів, який Cargo.toml використовує. Більш читабельний за JSON для людського редагування:

```rust
use serde::{Serialize, Deserialize};
use std::fs;

#[derive(Debug, Serialize, Deserialize)]
struct MissionConfig {
    name: String,
    max_agents: u32,
    area: Area,
    waypoints: Vec<Waypoint>,
}

#[derive(Debug, Serialize, Deserialize)]
struct Area {
    min_x: f64,
    min_y: f64,
    max_x: f64,
    max_y: f64,
}

#[derive(Debug, Serialize, Deserialize)]
struct Waypoint {
    name: String,
    x: f64,
    y: f64,
    priority: u8,
}

fn load_config(path: &str) -> Result<MissionConfig, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;
    let config: MissionConfig = toml::from_str(&content)?;
    Ok(config)
}

fn save_config(path: &str, config: &MissionConfig) -> Result<(), Box<dyn std::error::Error>> {
    let toml_string = toml::to_string_pretty(config)?;
    fs::write(path, toml_string)?;
    Ok(())
}
```

Відповідний TOML-файл `mission.toml`:

```toml
name = "Патрулювання зони A"
max_agents = 5

[area]
min_x = 0.0
min_y = 0.0
max_x = 100.0
max_y = 100.0

[[waypoints]]
name = "Alpha"
x = 10.0
y = 20.0
priority = 1

[[waypoints]]
name = "Bravo"
x = 50.0
y = 50.0
priority = 2
```

`[area]` — вкладена структура. `[[waypoints]]` — масив структур (кожен `[[waypoints]]` — елемент Vec<Waypoint>). TOML інтуїтивно зрозумілий для оператора, що редагує конфігурацію місії вручну.

---

## 35.6. Збереження та відновлення стану агента

Практичний патерн — periodic checkpoint: зберігати стан кожні N секунд, відновлювати при запуску:

```rust
use serde::{Serialize, Deserialize};
use std::fs;
use std::collections::HashMap;

#[derive(Debug, Serialize, Deserialize)]
struct AgentCheckpoint {
    id: String,
    position: (f64, f64),
    battery: u8,
    altitude: f64,
    history: Vec<(f64, f64)>,
    detected_objects: HashMap<String, (f64, f64)>,
    mission_time_sec: u64,
}

impl AgentCheckpoint {
    fn save(&self, path: &str) -> Result<(), Box<dyn std::error::Error>> {
        let json = serde_json::to_string_pretty(self)?;
        fs::write(path, json)?;
        Ok(())
    }
    
    fn load(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let content = fs::read_to_string(path)?;
        let checkpoint: Self = serde_json::from_str(&content)?;
        Ok(checkpoint)
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Створюємо checkpoint
    let mut objects = HashMap::new();
    objects.insert(String::from("OBJ-01"), (15.0, 25.0));
    objects.insert(String::from("OBJ-02"), (40.0, 60.0));
    
    let checkpoint = AgentCheckpoint {
        id: String::from("SCOUT-01"),
        position: (47.5, 35.2),
        battery: 72,
        altitude: 150.0,
        history: vec![(0.0, 0.0), (10.0, 5.0), (25.0, 18.0), (47.5, 35.2)],
        detected_objects: objects,
        mission_time_sec: 3600,
    };
    
    // Зберігаємо
    checkpoint.save("agent_state.json")?;
    println!("Стан збережено");
    
    // Відновлюємо (імітація перезапуску)
    let restored = AgentCheckpoint::load("agent_state.json")?;
    println!("Відновлено: {} на {:?}, батарея {}%, {} об'єктів",
             restored.id, restored.position, restored.battery,
             restored.detected_objects.len());
    
    Ok(())
}
```

Vec, HashMap, String — все серіалізується автоматично. serde рекурсивно обходить структуру, кожен тип "знає", як себе серіалізувати. Після десеріалізації ви отримуєте повноцінну Rust-структуру з усіма полями — ownership, методи, traits — все працює.

---

## 35.7. Вибір формату серіалізації

| Формат | Крейт | Читабельний | Розмір | Швидкість | Коли використовувати |
|--------|-------|-------------|--------|-----------|---------------------|
| JSON | serde_json | Так | Великий | Середня | API, логи, дебаг, обмін з іншими мовами |
| TOML | toml | Так | Середній | Середня | Конфігурації |
| YAML | serde_yaml | Так | Середній | Середня | Конфігурації з ієрархією |
| bincode | bincode | Ні | Мінімальний | Висока | Внутрішня комунікація, кеш, snapshots |
| MessagePack | rmp-serde | Ні | Малий | Висока | Мережева комунікація |

Для БПЛА: конфігурація місії — TOML (оператор редагує вручну). Повідомлення між агентами — bincode (мінімальний розмір, максимальна швидкість, обидва боки — Rust). API для зовнішніх систем — JSON (універсальний, читабельний). Збереження стану — JSON при дебагу (можна прочитати), bincode у продакшені (менший файл, швидше).

Ключова перевага serde: одна анотація `#[derive(Serialize, Deserialize)]` — і тип працює з усіма форматами. Зміна формату — одна зміна у коді (serde_json → bincode), без змін у структурах.

---

## 35.8. Prompt Engineering: проєктування формату повідомлень

### Промпт-шаблон

```
Я проєктую систему повідомлень між агентами БПЛА.
Типи повідомлень: [список: команди, звіти, статуси...]

Спроєктуй Rust-структури з serde для цих повідомлень.
Для enum — обери оптимальний режим tagged/untagged.
Покажи приклад JSON для кожного типу повідомлення.
```

### Вправа з PE

Опишіть AI протокол комунікації вашого рою (які повідомлення координатор → агент, агент → координатор). Попросіть згенерувати структури з serde. Перевірте: чи всі поля серіалізуються? Чи є skip для внутрішніх полів? Чи правильний формат enum?

---

## Лабораторна робота No35

### Мета

Реалізувати збереження/завантаження стану агента та формат повідомлень.

### Завдання базового рівня

1. `#[derive(Serialize, Deserialize)]` для AgentState, Position, Agent з Розділу 30.
2. Методи `save_json(&self, path)` та `load_json(path) -> Result<Self>`.
3. Конфігурація місії у TOML: завантажити з файлу, використати значення.
4. Enum Command з `#[serde(tag = "type")]` для повідомлень координатору.
5. Тести: серіалізувати → десеріалізувати → перевірити рівність.

### Варіанти

**A.** Порівняння форматів: серіалізувати AgentCheckpoint у JSON, TOML та bincode. Порівняти розмір файлу та час серіалізації/десеріалізації (1000 повторень).

**B.** Міграція формату: версія 1 AgentState має 5 полів, версія 2 — 7. Використати `#[serde(default)]` для зворотної сумісності: десеріалізація JSON v1 у struct v2.

**C.** Протокол рою: enum Message з 5+ варіантами (Command, Report, Heartbeat, Alert, Config). Серіалізація/десеріалізація з тестами для кожного варіанту.

### Критерії

| Критерій | Бал |
|----------|-----|
| Derive Serialize/Deserialize для 3+ типів | 20 |
| Збереження/завантаження файлу (JSON або TOML) | 20 |
| Атрибути serde (rename, default, skip, tag) | 15 |
| Roundtrip тести (serialize → deserialize → assert_eq) | 25 |
| Обробка помилок (невалідний файл, відсутні поля) | 10 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `the trait Serialize is not implemented`

Забули derive або не додали serde у Cargo.toml:

```rust
struct Pos { x: f64, y: f64 }
serde_json::to_string(&Pos { x: 1.0, y: 2.0 }); // ПОМИЛКА
```

Виправлення: `#[derive(Serialize)]` та `serde = { version = "1", features = ["derive"] }`.

### Помилка 2: `missing field` при десеріалізації

JSON не містить поле, яке очікує struct:

```json
{ "id": "SCOUT-01" }
```

```rust
#[derive(Deserialize)]
struct Agent { id: String, battery: u8 } // battery відсутній у JSON
```

Виправлення: `#[serde(default)]` на полі battery, або зробити поле `Option<u8>`.

### Помилка 3: `invalid type` для enum

Неправильний формат enum у JSON. Externally tagged очікує `{"Moving": {...}}`, а internally tagged — `{"type": "Moving", ...}`.

Виправлення: перевірте який режим тегування використовується (`#[serde(tag = "type")]` чи ні) і відповідно сформуйте JSON.

### Помилка 4: `key must be a string` для HashMap

HashMap з нерядковим ключем (наприклад, `HashMap<(i32, i32), String>`) не серіалізується у JSON, бо JSON-ключі — завжди рядки.

Виправлення: конвертувати ключ у String (`HashMap<String, String>`) або використати бінарний формат (bincode, MessagePack).

---

## Додатково

### Custom Serialize/Deserialize

Для більшості типів derive достатній. Але іноді потрібна кастомна логіка: серіалізувати Position як масив `[x, y]` замість об'єкта `{"x": ..., "y": ...}`, або серіалізувати DateTime у конкретному форматі. serde надає API для ручної реалізації, але це просунута тема. Для початку — derive покриває 95% потреб.

### serde та продуктивність

serde_json — не найшвидший JSON-парсер (simd-json швидший). Але для більшості задач — достатньо швидкий: сотні мегабайт на секунду. bincode значно швидший (гігабайти/с) і створює менші файли, але нечитабельний для людини. Вибір формату — баланс між читабельністю, розміром та швидкістю.

---

## Контрольні запитання

### Рівень 1

1. Що таке серіалізація та десеріалізація?

Відповідь: серіалізація — перетворення Rust-структури у формат для збереження/передачі (JSON, TOML, байти). Десеріалізація — зворотне: з формату назад у Rust-структуру.

2. Як додати підтримку serde до своєї структури?

Відповідь: `#[derive(Serialize, Deserialize)]` та залежності у Cargo.toml: `serde = { version = "1", features = ["derive"] }` плюс формат-крейт (serde_json, toml тощо).

### Рівень 2

3. Чим `#[serde(tag = "type")]` відрізняється від серіалізації enum за замовчуванням?

Відповідь: за замовчуванням — externally tagged: `{"MoveTo": {"x": 10}}`. З `tag = "type"` — internally tagged: `{"type": "MoveTo", "x": 10}`. Internally tagged зручніший для споживачів (плаский об'єкт із дискримінатором).

4. Навіщо `#[serde(default)]`?

Відповідь: якщо поле відсутнє у вхідних даних — використати Default::default() замість помилки. Забезпечує зворотну сумісність: старий JSON (без нового поля) десеріалізується у нову struct.

### Рівень 3

5. Як серіалізувати HashMap<(i32, i32), String> у JSON?

Відповідь: напряму не можна — JSON-ключі мають бути рядками. Рішення: конвертувати ключ у рядок при серіалізації (`HashMap<String, String>` де ключ — "x,y"), або використати Vec<((i32, i32), String)> замість HashMap, або використати бінарний формат (bincode).

### Рівень 4

6. Чому serde використовує traits (Serialize, Deserialize), а не конкретний формат?

Відповідь: trait-based архітектура розділяє "що серіалізувати" (Serialize на типі) від "як серіалізувати" (Serializer у формат-крейті). Одна анотація derive — і тип працює з будь-яким форматом. Додавання нового формату не потребує змін у типах. Додавання нового типу не потребує змін у форматах. Це принцип відкритості/закритості (Open/Closed Principle) через traits.

---

## Резюме

serde — фреймворк серіалізації на основі traits Serialize та Deserialize. `#[derive(Serialize, Deserialize)]` дає типу можливість перетворюватись у будь-який підтримуваний формат без зміни коду.

Формати: JSON (читабельний, універсальний), TOML (конфігурації), bincode (компактний, швидкий). Один derive — всі формати.

Атрибути: `rename` (перейменування полів), `default` (значення за замовчуванням), `skip` (пропуск полів), `tag` (режим серіалізації enum).

Типовий патерн: serialize → write to file/network → read → deserialize. Roundtrip тест: serialize → deserialize → assert_eq.

---

## Що далі

Стан агента можна зберегти та відновити. Повідомлення можна серіалізувати для передачі. Але передавати їх поки нікуди — агент працює в одному потоці.

У Розділі 36 ви нарешті запустите кілька агентів паралельно через `std::thread::spawn`. Кожен агент — окремий потік зі своїм замиканням (`move || { ... }`), своїми даними, своїм циклом виконання. Traits Send та Sync визначатимуть, які дані можна передати в інший потік, а які — ні. Компілятор Rust гарантуватиме відсутність data races на етапі компіляції — те, що жодна інша мова не забезпечує без runtime overhead.
