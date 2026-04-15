# Розділ 17. Enum та Pattern Matching: машина станів агента

## Анотація

У Розділі 16 ви створили структуру Agent з полем `airborne: bool`. Це працює для двох станів, але реальний агент має більше: на землі, у повітрі, на зарядці, аварійний режим, патрулювання. Кодувати це через кілька bool (`airborne`, `charging`, `emergency`) — небезпечно: нічого не заважає встановити `airborne = true` та `charging = true` одночасно, що фізично неможливо. Enum вирішує цю проблему: кожен варіант стану — окреме значення, і агент може бути лише в одному стані одночасно. Pattern matching через `match` гарантує, що ви обробили кожен можливий стан — забути один варіант неможливо, бо компілятор не дозволить. Це робить enum ідеальним інструментом для моделювання машини станів.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Створити enum з різними видами варіантів (unit, tuple, struct).
2. Використати `match` для обробки всіх варіантів enum з гарантією повноти.
3. Застосувати `if let` та `while let` для обробки одного варіанту.
4. Пояснити, що таке `Option<T>` та використати `Some`/`None` замість null.
5. Пояснити `Result<T, E>` як засіб обробки помилок (вступ).
6. Спроєктувати машину станів агента через enum з переходами.

---

## Ключові терміни

**Enum (перелічення)** — тип, що може приймати одне значення з фіксованого набору варіантів.

**Variant (варіант)** — одне з можливих значень enum.

**Pattern matching** — зіставлення значення з набором шаблонів для вибору гілки виконання.

**Exhaustive matching** — вимога `match` обробити всі можливі варіанти.

**Option** — стандартний enum для значення, що може бути відсутнім: `Some(T)` або `None`.

**Result** — стандартний enum для операції, що може завершитись помилкою: `Ok(T)` або `Err(E)`.

**Algebraic data type (ADT)** — тип, варіанти якого можуть містити різні дані.

---

## Мотиваційний кейс

У 2018 році безпілотний автомобіль Uber збив пішохода. Розслідування виявило, що система класифікації об'єктів "вагалась" між станами "пішохід", "велосипед" та "невідомий об'єкт", і замість обробки кожного стану з відповідною реакцією — перемикалась між ними, кожного разу "забуваючи" попередній контекст. Якби система мала чітку машину станів із enum та exhaustive matching — компілятор вимагав би обробити кожен стан, включаючи "невідомий об'єкт". Пропустити варіант було б неможливо.

---

## 17.1. Що таке enum і навіщо він потрібен

### Проблема: набір bool-прапорців

У Розділі 16 стан агента визначався полем `airborne: bool`. Але реальний агент має більше двох станів. Спроба кодувати їх через bool:

```rust
struct Agent {
    is_grounded: bool,
    is_airborne: bool,
    is_charging: bool,
    is_emergency: bool,
}
```

Ця модель дозволяє невалідні комбінації: `is_airborne = true, is_charging = true` — агент одночасно летить і заряджається? `is_grounded = false, is_airborne = false` — агент ніде? Компілятор не бачить проблеми, бо кожен bool незалежний.

### Рішення: enum

Enum визначає тип, що може мати рівно одне значення з фіксованого набору:

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

fn main() {
    let heading = Direction::North;
    
    match heading {
        Direction::North => println!("На північ"),
        Direction::South => println!("На південь"),
        Direction::East  => println!("На схід"),
        Direction::West  => println!("На захід"),
    }
}
```

`Direction` може бути лише одним із чотирьох варіантів. Не може бути "між" North та South. Не може бути невалідним. `match` вимагає обробити кожен варіант — якщо забути `West`, компілятор видасть помилку.

Порівняйте з підходом через рядки: `let heading = "north"`. Нічого не заважає написати `"nrth"` (з помилкою) — і компілятор не помітить. З enum `Direction::Nrth` — це помилка компіляції, бо такого варіанту не існує.

Порівняйте з підходом через числа: `let heading: u8 = 0` (0=північ, 1=південь...). Що означає `heading = 5`? Нічого — але компілятор дозволить. З enum п'ятого варіанту просто не існує.

Enum дає type safety — гарантію, що значення завжди валідне.

### Синтаксис оголошення

Enum оголошується ключовим словом `enum`, ім'я — CamelCase, варіанти — теж CamelCase:

```rust
enum AgentState {
    Grounded,      // на землі
    Airborne,      // у повітрі
    Charging,      // на зарядці
    Emergency,     // аварійний режим
}
```

Варіанти доступні через `::`: `AgentState::Grounded`, `AgentState::Airborne`. Кожен варіант — це значення типу `AgentState`. Змінна типу `AgentState` в кожен момент часу містить рівно один варіант.

---

## 17.2. Варіанти з даними: algebraic data types

Enum у Rust — це не просто перелік констант (як у C або Java). Кожен варіант може містити різні дані. Це робить Rust-enum повноцінним algebraic data type (ADT).

### Unit variants — без даних

```rust
enum Color {
    Red,
    Green,
    Blue,
}
```

Найпростіший випадок — варіанти без додаткових даних. Аналог констант.

### Tuple variants — з позиційними даними

```rust
enum Shape {
    Circle(f64),            // радіус
    Rectangle(f64, f64),    // ширина, висота
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rectangle(w, h) => w * h,
    }
}

fn main() {
    let c = Shape::Circle(5.0);
    let r = Shape::Rectangle(3.0, 4.0);
    println!("Коло: {:.2}", area(&c));        // 78.54
    println!("Прямокутник: {:.2}", area(&r)); // 12.00
}
```

`Circle` містить одне f64 (радіус), `Rectangle` — два (ширина, висота). Кожен варіант — як окремий кортеж. У `match` дані витягуються через деструктуризацію: `Shape::Circle(r)` зв'язує значення радіуса зі змінною `r`.

### Struct variants — з іменованими полями

```rust
enum AgentState {
    Grounded,
    Airborne { altitude: f64 },
    Moving { target_x: f64, target_y: f64, speed: f64 },
    Charging { percent: u8 },
    Emergency { reason: String },
}
```

Варіант `Moving` має три іменованих поля — як вбудована структура. Різні варіанти містять різні дані: `Grounded` — нічого, `Airborne` — висоту, `Emergency` — текст причини. Це неможливо моделювати через struct з bool-прапорцями — де б ви зберігали `altitude`, якщо агент не у повітрі?

---

## 17.3. Pattern matching через match

`match` — це конструкція, що порівнює значення з набором шаблонів і виконує код відповідної гілки. Головна особливість: match — exhaustive, тобто вимагає обробити ВСІ можливі варіанти.

### Базовий match

```rust
fn describe_state(state: &AgentState) -> String {
    match state {
        AgentState::Grounded => String::from("На землі"),
        AgentState::Airborne { altitude } =>
            format!("У повітрі, висота {:.0} м", altitude),
        AgentState::Moving { target_x, target_y, speed } =>
            format!("Рух до ({:.0},{:.0}), швидкість {:.1}", target_x, target_y, speed),
        AgentState::Charging { percent } =>
            format!("Зарядка: {}%", percent),
        AgentState::Emergency { reason } =>
            format!("АВАРІЯ: {}", reason),
    }
}
```

Кожна гілка match: шаблон `=>` вираз. Шаблон може деструктуризувати дані: `Airborne { altitude }` витягує значення поля `altitude` у змінну з тим самим ім'ям.

### Exhaustive matching — повнота

Якщо забути один варіант — помилка компіляції:

```rust
fn bad_describe(state: &AgentState) -> &str {
    match state {
        AgentState::Grounded => "На землі",
        AgentState::Airborne { .. } => "У повітрі",
        // Забули Moving, Charging, Emergency!
    }
}
```

```
error[E0004]: non-exhaustive patterns: `Moving { .. }`, `Charging { .. }`
              and `Emergency { .. }` not covered
```

Компілятор перелічує всі пропущені варіанти. Це гарантія: додавши новий варіант до enum (наприклад, `Patrolling`), ви отримаєте помилки у кожному `match`, що його не обробляє. Жоден варіант не "провалиться" необробленим.

### Wildcard `_` — решта варіантів

Якщо деякі варіанти обробляються однаково:

```rust
fn is_operational(state: &AgentState) -> bool {
    match state {
        AgentState::Emergency { .. } => false,
        _ => true,  // всі інші — operational
    }
}
```

`_` відповідає будь-якому варіанту, що не збігся з попередніми гілками. Використовуйте обережно: якщо додати новий варіант до enum, `_` "проковтне" його без попередження. Краще перелічувати варіанти явно, коли це важливо.

### match як вираз

`match` — це вираз, що повертає значення (як `if`):

```rust
let description = match state {
    AgentState::Grounded => "на землі",
    AgentState::Airborne { .. } => "у повітрі",
    _ => "інше",
};
println!("Стан: {}", description);
```

Всі гілки повинні повертати однаковий тип.

---

## 17.4. if let та while let

Якщо потрібно перевірити лише один варіант — `match` надто громіздкий. `if let` — спрощення:

```rust
let state = AgentState::Charging { percent: 75 };

// match — громіздко для одного варіанту
match state {
    AgentState::Charging { percent } => println!("Зарядка: {}%", percent),
    _ => {}  // нічого не робимо
}

// if let — компактніше
if let AgentState::Charging { percent } = state {
    println!("Зарядка: {}%", percent);
}
```

`if let` деструктуризує значення і виконує блок, лише якщо шаблон збігається. Інакше — пропускає. Можна додати `else`:

```rust
if let AgentState::Emergency { reason } = &state {
    println!("АВАРІЯ: {}", reason);
} else {
    println!("Стан нормальний");
}
```

`while let` — те саме для циклів. Корисно для обробки Option у циклі (секція 17.5).

---

## 17.5. Option: значення, що може бути відсутнім

### Проблема null

У багатьох мовах (Java, Python, C) "відсутність значення" позначається null/None/NULL. Це джерело помилок: ви отримуєте null там, де очікували дані, і програма аварійно завершується (NullPointerException, segfault).

Rust не має null. Замість цього — стандартний enum `Option<T>`:

```rust
enum Option<T> {
    Some(T),    // значення є
    None,       // значення відсутнє
}
```

`Option<T>` — це або `Some(значення)`, або `None`. Компілятор вимагає обробити обидва випадки:

```rust
fn find_agent(id: &str) -> Option<u8> {
    if id == "SCOUT-01" {
        Some(85)  // знайдено, батарея 85%
    } else {
        None      // не знайдено
    }
}

fn main() {
    let result = find_agent("SCOUT-01");
    
    match result {
        Some(battery) => println!("Знайдено! Батарея: {}%", battery),
        None => println!("Агент не знайдений"),
    }
    
    // if let — для швидкої перевірки
    if let Some(bat) = find_agent("TRANS-02") {
        println!("Батарея: {}%", bat);
    } else {
        println!("Не знайдений");
    }
}
```

Ви не можете "випадково" використати None як число — компілятор вимагає явно обробити обидва випадки. Це усуває цілий клас помилок, що у Java називаються NullPointerException, а у C — segfault через NULL pointer.

Щоб зрозуміти перевагу, порівняйте два підходи. У Python: `battery = find_agent("X")` може повернути None, і якщо ви забудете перевірити — `battery + 10` крашить програму в runtime. У Rust: `find_agent("X")` повертає `Option<u8>`. Ви не можете додати 10 до Option — потрібно спершу витягнути значення через match або if let. Компілятор не дозволить "забути" перевірку.

### Корисні методи Option

Option має багато корисних методів, що спрощують роботу без match:

```rust
fn main() {
    let some_val: Option<i32> = Some(42);
    let no_val: Option<i32> = None;
    
    // unwrap_or — значення за замовчуванням
    println!("{}", some_val.unwrap_or(0));  // 42
    println!("{}", no_val.unwrap_or(0));    // 0
    
    // is_some / is_none — перевірка
    println!("Є значення: {}", some_val.is_some());  // true
    println!("Порожній: {}", no_val.is_none());       // true
    
    // map — трансформація вмісту (якщо є)
    let doubled = some_val.map(|x| x * 2);  // Some(84)
    let nothing = no_val.map(|x| x * 2);    // None (map не робить нічого)
    println!("{:?}, {:?}", doubled, nothing);
    
    // unwrap — витягнути значення (паніка, якщо None!)
    // Використовуйте лише коли впевнені, що значення є
    println!("{}", some_val.unwrap());  // 42
    // no_val.unwrap();  // ПАНІКА: called unwrap() on None
}
```

`unwrap()` — небезпечний метод, що паніку при None. Використовуйте його лише у тестах або коли логічно гарантовано, що значення є. У production-коді — завжди `match`, `if let` або `unwrap_or`.

### Option у контексті БПЛА

Типове застосування: пошук агента у "групі":

```rust
fn find_nearest_agent(agents: &[(f64, f64)], target: (f64, f64)) -> Option<usize> {
    if agents.is_empty() {
        return None;  // немає агентів — немає результату
    }
    
    let mut best_index = 0;
    let mut best_dist = f64::MAX;
    
    for (i, &(ax, ay)) in agents.iter().enumerate() {
        let dist = ((ax - target.0).powi(2) + (ay - target.1).powi(2)).sqrt();
        if dist < best_dist {
            best_dist = dist;
            best_index = i;
        }
    }
    
    Some(best_index)
}

fn main() {
    let agents = vec![(10.0, 20.0), (30.0, 5.0), (15.0, 15.0)];
    
    match find_nearest_agent(&agents, (12.0, 18.0)) {
        Some(idx) => println!("Найближчий агент: #{}", idx),
        None => println!("Немає доступних агентів"),
    }
    
    let empty: Vec<(f64, f64)> = vec![];
    match find_nearest_agent(&empty, (0.0, 0.0)) {
        Some(idx) => println!("Знайдено: #{}", idx),
        None => println!("Немає агентів"),
    }
}
```

Вивід:

```
Найближчий агент: #2
Немає агентів
```

Без Option довелося б повертати якийсь "магічний" індекс (наприклад, -1 або usize::MAX), що вимагає окремої перевірки і легко забувається. З Option — компілятор нагадає.
```

---

## 17.6. Result: операція, що може завершитись помилкою

`Result<T, E>` — стандартний enum для операцій, що можуть не вдатись:

```rust
enum Result<T, E> {
    Ok(T),     // успіх, з даними типу T
    Err(E),    // помилка, з описом типу E
}
```

Приклад — парсинг числа з рядка:

```rust
fn parse_altitude(input: &str) -> Result<f64, String> {
    match input.trim().parse::<f64>() {
        Ok(value) => {
            if value >= 0.0 && value <= 10000.0 {
                Ok(value)
            } else {
                Err(format!("Висота {} поза межами 0–10000", value))
            }
        }
        Err(_) => Err(format!("'{}' — не число", input)),
    }
}

fn main() {
    match parse_altitude("350.5") {
        Ok(alt) => println!("Висота: {:.1} м", alt),
        Err(msg) => println!("Помилка: {}", msg),
    }
    
    match parse_altitude("abc") {
        Ok(alt) => println!("Висота: {:.1} м", alt),
        Err(msg) => println!("Помилка: {}", msg),
    }
}
```

Вивід:

```
Висота: 350.5 м
Помилка: 'abc' — не число
```

Result — це вступ до обробки помилок у Rust. Детально ми розглянемо її у Частині III (Розділи 24–26). Поки що — знайте, що Result існує і працює через той самий match.

---

## 17.7. Практика: машина станів БПЛА

Тепер об'єднаємо enum, struct та match у повноцінну машину станів агента:

```rust
enum AgentState {
    Grounded,
    Airborne { altitude: f64 },
    Charging { percent: u8 },
    Emergency { reason: String },
}

struct Agent {
    id: String,
    state: AgentState,
    battery: u8,
    pos_x: f64,
    pos_y: f64,
}

impl Agent {
    fn new(id: &str) -> Self {
        Self {
            id: String::from(id),
            state: AgentState::Grounded,
            battery: 100,
            pos_x: 0.0,
            pos_y: 0.0,
        }
    }
    
    fn status(&self) -> String {
        let state_desc = match &self.state {
            AgentState::Grounded => String::from("на землі"),
            AgentState::Airborne { altitude } =>
                format!("у повітрі ({:.0} м)", altitude),
            AgentState::Charging { percent } =>
                format!("зарядка {}%", percent),
            AgentState::Emergency { reason } =>
                format!("АВАРІЯ: {}", reason),
        };
        format!("{}: {} | батарея {}%", self.id, state_desc, self.battery)
    }
    
    fn takeoff(&mut self) {
        match self.state {
            AgentState::Grounded => {
                if self.battery >= 10 {
                    self.battery -= 10;
                    self.state = AgentState::Airborne { altitude: 50.0 };
                    println!("  {} — зліт!", self.id);
                } else {
                    println!("  {} — недостатньо батареї для зльоту", self.id);
                }
            }
            _ => println!("  {} — зліт можливий лише з землі", self.id),
        }
    }
    
    fn land(&mut self) {
        if let AgentState::Airborne { .. } = self.state {
            self.state = AgentState::Grounded;
            println!("  {} — посадка", self.id);
        } else {
            println!("  {} — посадка можлива лише з повітря", self.id);
        }
    }
    
    fn start_charging(&mut self) {
        match self.state {
            AgentState::Grounded => {
                self.state = AgentState::Charging { percent: self.battery };
                println!("  {} — початок зарядки", self.id);
            }
            _ => println!("  {} — зарядка можлива лише на землі", self.id),
        }
    }
    
    fn emergency(&mut self, reason: &str) {
        self.state = AgentState::Emergency {
            reason: String::from(reason),
        };
        println!("  {} — АВАРІЯ: {}", self.id, reason);
    }
}

fn main() {
    let mut scout = Agent::new("SCOUT-01");
    println!("{}", scout.status());
    
    scout.takeoff();
    println!("{}", scout.status());
    
    scout.land();
    scout.start_charging();
    println!("{}", scout.status());
    
    scout.emergency("втрата зв'язку");
    println!("{}", scout.status());
}
```

Вивід:

```
SCOUT-01: на землі | батарея 100%
  SCOUT-01 — зліт!
SCOUT-01: у повітрі (50 м) | батарея 90%
  SCOUT-01 — посадка
  SCOUT-01 — початок зарядки
SCOUT-01: зарядка 90% | батарея 90%
  SCOUT-01 — АВАРІЯ: втрата зв'язку
SCOUT-01: АВАРІЯ: втрата зв'язку | батарея 90%
```

Ключовий момент: `takeoff` дозволений лише з `Grounded`. `land` — лише з `Airborne`. `start_charging` — лише з `Grounded`. Будь-яка спроба порушити ці правила — повідомлення оператору, а не невалідний стан. Enum гарантує: агент завжди в одному з чотирьох визначених станів. Неможливо створити "п'ятий стан" або "між двома станами".

### Таблиця переходів

Машину станів можна описати таблицею допустимих переходів:

| Поточний стан | Дозволені переходи |
|---------------|-------------------|
| Grounded | Airborne (takeoff), Charging (start_charging), Emergency |
| Airborne | Grounded (land), Emergency |
| Charging | Grounded (коли зарядка завершена), Emergency |
| Emergency | Grounded (після усунення причини) |

Кожен перехід реалізовано як метод з match на поточному стані. Недозволені переходи (наприклад, з Charging в Airborne напряму) оброблюються гілкою `_` з повідомленням. Додавання нового стану (наприклад, `Patrolling`) вимагає:

Додати варіант до enum. Отримати помилки компіляції у всіх match, що його не обробляють. Додати обробку в кожен match. Реалізувати переходи. Це покроковий процес, керований компілятором — жодний match не буде "забутий".

### Порівняння: enum vs bool-прапорці vs рядки

| Підхід | Невалідний стан можливий? | Забути стан у match? | Type safety |
|--------|--------------------------|---------------------|-------------|
| Bool-прапорці | Так (airborne + charging) | Компілятор не бачить | Ні |
| Рядки ("grounded") | Так (помилка друку) | Компілятор не бачить | Ні |
| Enum | Ні (лише визначені варіанти) | Помилка компіляції | Так |

---

## Prompt Engineering: перевірка машини станів

### Промпт-шаблон

```
Ось мій enum AgentState та методи переходів:
[код]
Перевір:
1. Чи всі переходи між станами коректні?
2. Чи є невалідні переходи, які я пропустив?
3. Чи обробляю я кожен стан у кожному match?
```

### Приклад діалогу

```
Студент:
enum State { Grounded, Airborne, Charging }
fn takeoff(state: &mut State) {
    *state = State::Airborne;
}
Чи є проблеми?

AI: Так. takeoff не перевіряє поточний стан. Можна
викликати takeoff з Charging — а це недопустимо.
Виправлення:
  fn takeoff(state: &mut State) {
      match state {
          State::Grounded => *state = State::Airborne,
          _ => println!("Зліт можливий лише з землі"),
      }
  }
Також рекомендую зробити state приватним полем
структури Agent і давати доступ тільки через методи.
```

Оцінка: AI правильно знайшов проблему (відсутність перевірки поточного стану) та запропонував правильне рішення з match. Рекомендація зробити state приватним — це інкапсуляція, яку ми вивчимо у Розділі 18.

### Вправа з PE

Створіть свій enum з 4+ варіантами та реалізуйте переходи. Попросіть AI знайти пропущені або невалідні переходи. Запишіть у промпт-журнал.

---

## Лабораторна робота No17

### Мета

Створити машину станів агента через enum.

### Завдання

**Частина 1 (3 бали):** Створіть enum `Command` з варіантами: `Takeoff`, `Land`, `MoveTo { x: f64, y: f64 }`, `Scan { radius: f64 }`, `ReturnToBase`. Напишіть функцію `describe_command(cmd: &Command) -> String`.

**Частина 2 (4 бали):** Розширіть Agent: додайте стан `Patrolling { waypoint_index: usize }`. Реалізуйте метод `patrol(&mut self)`, що переводить агента з Grounded в Patrolling.

**Частина 3 (3 бали):** AI code review машини станів + промпт-журнал.

### Критерії

| Критерій | Бал |
|----------|-----|
| Command enum з describe | 30 |
| Patrolling стан з переходами | 40 |
| AI review + промпт-журнал | 30 |

---

## Troubleshooting

### `non-exhaustive patterns`

Найчастіша помилка з enum. Ви забули варіант у match:

```
error[E0004]: non-exhaustive patterns: `Emergency { .. }` not covered
```

Виправлення: додайте пропущений варіант або `_ =>` для решти.

### `cannot move out of borrowed content` при match на &enum

Якщо match на посиланні `&AgentState`, а варіант містить String:

```rust
fn get_reason(state: &AgentState) -> String {
    match state {
        AgentState::Emergency { reason } => reason.clone(),
        // reason тут — &String, бо state — посилання
        _ => String::from("немає"),
    }
}
```

При match на `&enum` дані всередині варіантів теж доступні через посилання. `reason` має тип `&String`, не `String`.

### Забули `&` перед self у match

```rust
fn status(&self) -> String {
    match self.state {  // може потребувати &self.state
        AgentState::Emergency { reason } => { ... }
        // reason: String — move! Але self позичено через &self
    }
}
```

Виправлення: `match &self.state` — тоді `reason` буде `&String`.

---

## Додатково

### Enum з impl

Enum може мати методи, як і struct:

```rust
impl AgentState {
    fn is_airborne(&self) -> bool {
        matches!(self, AgentState::Airborne { .. })
    }
    
    fn name(&self) -> &str {
        match self {
            AgentState::Grounded => "Grounded",
            AgentState::Airborne { .. } => "Airborne",
            AgentState::Charging { .. } => "Charging",
            AgentState::Emergency { .. } => "Emergency",
        }
    }
    
    fn can_accept_commands(&self) -> bool {
        !matches!(self, AgentState::Emergency { .. })
    }
}
```

`matches!` — макрос, що повертає bool: чи збігається значення з шаблоном. Зручна альтернатива match для простих перевірок.

### Pattern guards — додаткові умови в match

Гілка match може мати додаткову умову через `if`:

```rust
fn battery_action(state: &AgentState, battery: u8) -> &str {
    match state {
        AgentState::Airborne { altitude } if battery < 15 =>
            "Аварійна посадка — критичний заряд",
        AgentState::Airborne { altitude } if *altitude > 500.0 =>
            "Знизити висоту",
        AgentState::Airborne { .. } =>
            "Продовжувати політ",
        AgentState::Grounded if battery < 30 =>
            "Зарядити перед вильотом",
        _ => "Стан нормальний",
    }
}
```

Pattern guard `if battery < 15` додає умову до шаблону. Гілка спрацьовує, лише якщо І шаблон збігається, І умова виконана. Це дозволяє тонке розгалуження без вкладених if всередині гілки.

### Enum як тип помилки

```rust
enum MissionError {
    LowBattery(u8),
    OutOfBounds { x: f64, y: f64 },
    CommunicationLost,
    SensorFailure(String),
}

impl MissionError {
    fn severity(&self) -> &str {
        match self {
            MissionError::LowBattery(level) if *level < 5 => "CRITICAL",
            MissionError::LowBattery(_) => "WARNING",
            MissionError::CommunicationLost => "CRITICAL",
            MissionError::SensorFailure(_) => "WARNING",
            MissionError::OutOfBounds { .. } => "ERROR",
        }
    }
}
```

Кожен варіант помилки містить релевантні дані: LowBattery зберігає рівень заряду, OutOfBounds — координати, SensorFailure — опис проблеми. Метод `severity` класифікує помилки за критичністю, використовуючи pattern guards для різних рівнів батареї. Це основа системи обробки помилок, яку ви детально вивчите у Частині III.

### Вкладені enum

Enum може містити інші enum як дані варіантів:

```rust
enum Priority { Low, Medium, High, Critical }

enum Command {
    Move { x: f64, y: f64, priority: Priority },
    Scan { radius: f64, priority: Priority },
    ReturnToBase,
}
```

Кожна команда має свій пріоритет. Pattern matching може деструктуризувати обидва рівні:

```rust
fn is_urgent(cmd: &Command) -> bool {
    match cmd {
        Command::Move { priority: Priority::Critical, .. } => true,
        Command::Scan { priority: Priority::Critical, .. } => true,
        _ => false,
    }
}
```

---

## Контрольні запитання

### Рівень 1

1. Чим enum відрізняється від набору bool-прапорців?

Відповідь: enum гарантує, що значення — рівно один варіант. Bool-прапорці дозволяють невалідні комбінації.

2. Що таке exhaustive matching?

Відповідь: вимога match обробити кожен можливий варіант enum. Пропуск варіанту — помилка компіляції.

### Рівень 2

3. Чим `Option<T>` кращий за null?

Відповідь: Option — це тип. Компілятор вимагає обробити Some та None явно. Null — це значення, яке можна "забути" перевірити, що призводить до runtime-крашу.

4. Коли використовувати `if let` замість `match`?

Відповідь: коли потрібно перевірити лише один варіант, а решту — ігнорувати. Match — коли потрібно обробити всі або кілька варіантів.

### Рівень 3

5. Напишіть enum `TrafficLight` з варіантами Red, Yellow, Green та функцію `duration(light: &TrafficLight) -> u32` (Red=60, Yellow=5, Green=45).

### Рівень 4

6. Чому додавання нового варіанту до enum ламає всі match, що його не обробляють? Це перевага чи недолік?

Відповідь: перевага. Компілятор вказує кожне місце, де новий варіант не оброблений. Це запобігає ситуації, коли новий стан агента "провалюється" через необроблену гілку.

---

## Резюме

Enum — тип з фіксованим набором варіантів. Кожен варіант може містити різні дані (unit, tuple, struct variants). Це algebraic data type.

Match — exhaustive pattern matching. Компілятор вимагає обробити кожен варіант. Забути один — помилка компіляції. Match — вираз, що повертає значення.

if let — спрощення match для одного варіанту. while let — для циклів.

Option — замінює null. Some(T) або None. Компілятор вимагає обробити обидва.

Result — для операцій з помилками. Ok(T) або Err(E). Детально — у Частині III.

Enum ідеальний для машини станів: кожен стан — варіант, переходи — через match. Невалідні стани неможливі, пропущені переходи — помилка компіляції.

---

## Що далі

Агент тепер має структуру (Розділ 16) та машину станів (Розділ 17). Але весь код все ще в одному файлі — main.rs на сотні рядків. Додавання нових модулів (сенсори, навігація, комунікація) зробить файл некерованим.

У Розділі 18 (Модулі та видимість) ви розділите код на логічні одиниці: `mod agent`, `mod navigation`, `mod sensors`. Кожен модуль — окремий файл з чіткою відповідальністю. Ви дізнаєтесь про `pub`, `use`, та як контролювати, що видимо ззовні модуля, а що приховано.
