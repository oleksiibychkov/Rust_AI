# Розділ 24. Обробка помилок: Option та Result

## Анотація

До цього моменту наш агент працював у комфортному середовищі: дані завжди валідні, сенсори завжди відповідають, команди завжди коректні. Реальний БПЛА живе інакше. GPS-модуль може тимчасово втратити сигнал і не повернути координати. Команда оператора може містити некоректні параметри. Файл з маршрутом може бути відсутній або пошкоджений. У Розділах 10, 17, 21 ви вже зустрічали `Option` та `Result` — при парсингу вводу, при `pop()` з вектора, при `get()` з HashMap. Тепер час зрозуміти їх глибоко: як вони влаштовані, які комбінатори мають, і як будувати ланцюжки обробки помилок, що робить код одночасно безпечним та читабельним.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити, чому Rust використовує `Option` та `Result` замість null та виключень.
2. Обробляти `Option<T>` через `match`, `if let`, `unwrap_or`, `map`, `and_then`.
3. Обробляти `Result<T, E>` через `match`, `unwrap_or_else`, `map`, `map_err`, `and_then`.
4. Пояснити, чому `unwrap()` — антипатерн у продакшен-коді.
5. Перетворювати між Option та Result через `ok_or`, `ok`, `err`.
6. Будувати ланцюжки fallible-операцій через комбінатори.

---

## Ключові терміни

**Option\<T\>** — enum із двома варіантами: `Some(T)` (значення є) та `None` (значення відсутнє). Заміна null/nil з інших мов.

**Result\<T, E\>** — enum із двома варіантами: `Ok(T)` (успіх із результатом) та `Err(E)` (помилка з описом). Заміна виключень (exceptions) з інших мов.

**Combinator (комбінатор)** — метод Option або Result, що трансформує вміст без ручного match. Наприклад, `map`, `and_then`, `unwrap_or`.

**Unwrap** — метод, що витягує значення з Some/Ok або панікує при None/Err. Небезпечний у продакшен-коді.

---

## Мотиваційний кейс

У серпні 2004 року марсохід Spirit спробував відкрити файл, якого не існувало у файловій системі. Код не перевіряв результат операції відкриття — він передбачав, що файл завжди є. Результат: нескінченний цикл повторних спроб, переповнення пам'яті, перезавантаження системи. Інженери NASA витратили два тижні на діагностику та виправлення — дистанційно, з затримкою сигналу 20 хвилин в одну сторону.

Якби код перевіряв `Result` операції відкриття — марсохід виявив би помилку одразу, записав її в лог, і продовжив роботу з альтернативним планом. Дві тижні простою на Марсі — ціна одного пропущеного `match`.

---

## 24.1. Проблема: що повертати, коли відповіді немає

Розглянемо просту задачу: функція шукає елемент у слайсі.

```rust
fn find_agent(agents: &[&str], target: &str) -> &str {
    for &agent in agents {
        if agent == target {
            return agent;
        }
    }
    // Що повернути тут?
}
```

Якщо `target` не знайдено — що повертати? У різних мовах це вирішується по-різному, і кожне рішення має свої проблеми.

У C повертають спеціальне значення: -1 для індексу, NULL для вказівника. Проблема: програміст може забути перевірити. `char* name = find("X"); printf("%s", name);` — якщо find повернув NULL, це undefined behavior (краш, або гірше — випадкові дані).

У Java та Python кидають виключення (exception). Проблема: виключення невидимі у сигнатурі функції. Побачивши `String find(String target)`, ви не знаєте, що вона може кинути NoSuchElementException. Забули try/catch — програма падає у рантаймі.

У Go повертають пару (значення, помилка). Проблема: помилку можна проігнорувати `value, _ := find("X")` — нижнє підкреслення відкидає помилку, і код працює з нульовим значенням.

Rust обирає інший шлях: тип повернення явно каже "результату може не бути". `Option<T>` означає "або значення типу T, або нічого". `Result<T, E>` — "або значення T, або помилка E". Компілятор змушує обробити обидва варіанти: ви не можете випадково використати відсутнє значення чи проігнорувати помилку.

---

## 24.2. Option\<T\>: значення, якого може не бути

Option — це enum з двома варіантами:

```rust
enum Option<T> {
    Some(T),  // значення є
    None,     // значення відсутнє
}
```

Це не спеціальний тип компілятора — це звичайний enum зі стандартної бібліотеки. `Some` та `None` імпортовані у prelude, тому їх можна використовувати без `Option::`.

Повернемось до задачі пошуку:

```rust
fn find_agent<'a>(agents: &[&'a str], target: &str) -> Option<&'a str> {
    for &agent in agents {
        if agent == target {
            return Some(agent);
        }
    }
    None
}

fn main() {
    let team = vec!["SCOUT-01", "RECON-02", "SCOUT-03"];
    
    match find_agent(&team, "RECON-02") {
        Some(agent) => println!("Знайдено: {}", agent),
        None => println!("Агент не в команді"),
    }
    
    match find_agent(&team, "CARGO-05") {
        Some(agent) => println!("Знайдено: {}", agent),
        None => println!("Агент не в команді"),
    }
}
```

Тип `Option<&str>` у сигнатурі каже: "ця функція може повернути рядок, а може не повернути". Caller змушений обробити обидва випадки — `match` не скомпілюється без гілки `None`.

Де ви вже зустрічали Option: `Vec::pop()` повертає `Option<T>`, бо вектор може бути порожній. `HashMap::get()` повертає `Option<&V>`, бо ключ може бути відсутній. `&[T]::first()` та `last()` повертають `Option<&T>`. `str::parse::<i32>()` повертає `Result`, але `.ok()` перетворює його на `Option`. Iterator::find() та position() повертають Option.

### Обробка Option: від match до комбінаторів

`match` — найповніший спосіб, але для простих випадків надмірний. Rust пропонує кілька альтернатив.

`if let` — коли цікавить лише Some, а None можна проігнорувати або обробити просто:

```rust
fn main() {
    let readings = vec![10.5, 20.3, 15.8];
    
    if let Some(&max) = readings.iter().max_by(|a, b| a.partial_cmp(b).unwrap()) {
        println!("Максимум: {:.1}", max);
    }
    // Якщо вектор порожній — нічого не виведеться
}
```

`unwrap()` — витягує значення з Some або панікує при None. Корисний у тестах та прототипах, небезпечний у продакшені:

```rust
fn main() {
    let data = vec![1, 2, 3];
    let first = data.first().unwrap(); // Ok: вектор не порожній
    println!("{}", first);
    
    let empty: Vec<i32> = vec![];
    // let boom = empty.first().unwrap(); // ПАНІКА: called unwrap() on None
}
```

`expect("повідомлення")` — як unwrap, але з кастомним повідомленням паніки. Трохи краще для дебагу:

```rust
let config = load_config().expect("Не вдалось завантажити конфігурацію");
```

`unwrap_or(default)` — повертає значення з Some або значення за замовчуванням:

```rust
fn main() {
    let altitude: Option<f64> = None; // GPS недоступний
    
    let safe_alt = altitude.unwrap_or(0.0); // 0.0 як fallback
    println!("Висота: {:.0}", safe_alt);
}
```

`unwrap_or_else(|| ...)` — як unwrap_or, але значення за замовчуванням обчислюється ледачо (лише коли потрібно):

```rust
fn main() {
    let cached_route: Option<Vec<(f64, f64)>> = None;
    
    let route = cached_route.unwrap_or_else(|| {
        println!("Кеш порожній, обчислюю маршрут...");
        vec![(0.0, 0.0), (10.0, 5.0)] // дорога операція
    });
    println!("Маршрут: {} точок", route.len());
}
```

---

## 24.3. Комбінатори Option: трансформація без match

Комбінатори — це методи, що трансформують вміст Option без ручного match. Вони аналогічні адаптерам ітераторів: кожен робить одну операцію, і їх можна з'єднувати в ланцюжки.

### map: трансформувати значення всередині Some

`map(f)` застосовує функцію до значення всередині Some. Якщо Option — None, map повертає None без виклику функції:

```rust
fn main() {
    let altitude: Option<f64> = Some(150.0);
    
    // Перетворити метри у фути
    let in_feet = altitude.map(|m| m * 3.28084);
    println!("{:?}", in_feet); // Some(492.126)
    
    let no_data: Option<f64> = None;
    let in_feet = no_data.map(|m| m * 3.28084);
    println!("{:?}", in_feet); // None — функція не викликалась
}
```

map зберігає "обгортку": `Some(x)` → `Some(f(x))`, `None` → `None`. Це як map для ітераторів, але для одного можливого значення замість послідовності.

### and_then: ланцюжок операцій, кожна з яких може провалитись

`and_then(f)` — як map, але замикання повертає Option. Це для ситуацій, де трансформація теж може "не вдатись":

```rust
fn parse_altitude(s: &str) -> Option<f64> {
    s.parse::<f64>().ok() // Result → Option
}

fn validate_altitude(alt: f64) -> Option<f64> {
    if alt >= 0.0 && alt <= 15000.0 {
        Some(alt)
    } else {
        None // невалідна висота
    }
}

fn main() {
    let input = "250.5";
    
    let result = parse_altitude(input)
        .and_then(validate_altitude);
    
    println!("{:?}", result); // Some(250.5)
    
    let bad_input = "not_a_number";
    let result = parse_altitude(bad_input)
        .and_then(validate_altitude);
    
    println!("{:?}", result); // None (парсинг провалився)
    
    let extreme = "99999.0";
    let result = parse_altitude(extreme)
        .and_then(validate_altitude);
    
    println!("{:?}", result); // None (валідація провалилась)
}
```

Різниця між map та and_then: map загортає результат у Some автоматично (`Some(x)` → `Some(f(x))`). and_then НЕ загортає — замикання саме повертає Option. Якщо використати map замість and_then із функцією, що повертає Option, отримаєте `Option<Option<T>>` — вкладений Option, що зазвичай не те, що потрібно.

### or та or_else: альтернативні значення

`or(other)` — якщо self є Some, повертає self. Якщо None — повертає other:

```rust
fn main() {
    let primary_gps: Option<(f64, f64)> = None;
    let backup_gps: Option<(f64, f64)> = Some((47.12, 35.45));
    
    let position = primary_gps.or(backup_gps);
    println!("Позиція: {:?}", position); // Some((47.12, 35.45))
}
```

Це патерн "спробувати основне джерело, якщо не працює — запасне".

### Ланцюжок комбінаторів

Комбінатори справді розкривають свою силу у ланцюжках:

```rust
fn main() {
    let raw_input: Option<&str> = Some("  42.5  ");
    
    let result: Option<f64> = raw_input
        .map(|s| s.trim())                    // "  42.5  " → "42.5"
        .and_then(|s| s.parse::<f64>().ok())  // "42.5" → Some(42.5)
        .map(|x| x * 1000.0)                 // 42.5 → 42500.0
        .filter(|&x| x > 0.0);               // залишити лише додатні
    
    println!("{:?}", result); // Some(42500.0)
}
```

Кожна ланка — одна операція. Якщо будь-яка ланка "провалюється" (повертає None) — весь ланцюжок коротко замикається і повертає None. Не потрібні вкладені if або match.

---

## 24.4. Result\<T, E\>: успіх або помилка з деталями

Option каже "є чи немає". Але іноді потрібно знати чому немає. GPS не відповідає — чому? Таймаут? Апаратний збій? Немає супутників? Для таких випадків є Result:

```rust
enum Result<T, E> {
    Ok(T),   // успіх із результатом типу T
    Err(E),  // помилка з описом типу E
}
```

Result — теж звичайний enum. `Ok` та `Err` у prelude.

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Ділення на нуль"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10.0, 3.0) {
        Ok(result) => println!("10 / 3 = {:.2}", result),
        Err(msg) => println!("Помилка: {}", msg),
    }
    
    match divide(10.0, 0.0) {
        Ok(result) => println!("10 / 0 = {:.2}", result),
        Err(msg) => println!("Помилка: {}", msg),
    }
}
```

Сигнатура `Result<f64, String>` каже: "ця функція або поверне f64, або поверне String з описом помилки". Caller бачить це у типі і змушений обробити обидва варіанти.

### unwrap та expect для Result

Працюють аналогічно до Option:

```rust
fn main() {
    let ok_val: Result<i32, String> = Ok(42);
    println!("{}", ok_val.unwrap()); // 42
    
    let err_val: Result<i32, String> = Err(String::from("збій"));
    // println!("{}", err_val.unwrap()); // ПАНІКА: called unwrap() on Err("збій")
}
```

### Чому unwrap — антипатерн у продакшені

`unwrap()` перетворює recoverable error (помилку, яку можна обробити) на unrecoverable panic (аварійне завершення). Для БПЛА це різниця між "GPS тимчасово недоступний, використовую інерціальну навігацію" та "програма впала, дрон без керування".

Коли unwrap прийнятний: у тестах (`assert!` панікує теж — це нормально для тестів), у прототипах ("пізніше замінити на нормальну обробку"), коли ви логічно впевнені, що помилка неможлива (і залишаєте коментар чому). В усіх інших випадках — обробляйте через match, if let, або комбінатори.

---

## 24.5. Комбінатори Result

Result має набір комбінаторів, аналогічний Option.

### map та map_err

`map(f)` трансформує Ok-значення, не чіпаючи Err. `map_err(f)` — навпаки, трансформує Err, не чіпаючи Ok:

```rust
fn main() {
    let result: Result<f64, String> = Ok(150.0);
    
    let in_km = result.map(|m| m / 1000.0);
    println!("{:?}", in_km); // Ok(0.15)
    
    let err: Result<f64, String> = Err(String::from("timeout"));
    let with_context = err.map_err(|e| format!("GPS помилка: {}", e));
    println!("{:?}", with_context); // Err("GPS помилка: timeout")
}
```

### and_then: ланцюжок fallible операцій

```rust
fn parse_coordinate(s: &str) -> Result<f64, String> {
    s.parse::<f64>().map_err(|e| format!("Невалідне число: {}", e))
}

fn validate_latitude(lat: f64) -> Result<f64, String> {
    if lat >= -90.0 && lat <= 90.0 {
        Ok(lat)
    } else {
        Err(format!("Широта {} поза межами [-90, 90]", lat))
    }
}

fn main() {
    let input = "47.5";
    
    let result = parse_coordinate(input)
        .and_then(validate_latitude);
    
    println!("{:?}", result); // Ok(47.5)
    
    let bad = "abc";
    let result = parse_coordinate(bad)
        .and_then(validate_latitude);
    
    println!("{:?}", result); // Err("Невалідне число: invalid float literal")
    
    let extreme = "100.0";
    let result = parse_coordinate(extreme)
        .and_then(validate_latitude);
    
    println!("{:?}", result); // Err("Широта 100 поза межами [-90, 90]")
}
```

Ланцюжок `parse → validate`: якщо parse провалюється — validate навіть не викликається. Як тільки будь-яка ланка повертає Err — весь ланцюжок зупиняється і повертає цей Err.

### unwrap_or та unwrap_or_else

```rust
fn main() {
    let altitude: Result<f64, String> = Err(String::from("GPS offline"));
    
    // Фіксоване значення за замовчуванням
    let safe = altitude.unwrap_or(0.0);
    println!("Висота: {:.0}", safe); // 0.0
    
    // Обчислене значення за замовчуванням
    let smart: Result<f64, String> = Err(String::from("timeout"));
    let alt = smart.unwrap_or_else(|err| {
        println!("Помилка: {}. Використовую останнє відоме значення.", err);
        last_known_altitude()
    });
    println!("Висота: {:.0}", alt);
}

fn last_known_altitude() -> f64 { 120.0 }
```

---

## 24.6. Конвертація між Option та Result

Option та Result часто потрібно перетворювати одне в одне.

`Option → Result` через `ok_or` та `ok_or_else`:

```rust
fn main() {
    let maybe_value: Option<f64> = Some(42.0);
    let result: Result<f64, String> = maybe_value.ok_or(String::from("відсутнє значення"));
    println!("{:?}", result); // Ok(42.0)
    
    let nothing: Option<f64> = None;
    let result: Result<f64, String> = nothing.ok_or(String::from("відсутнє значення"));
    println!("{:?}", result); // Err("відсутнє значення")
}
```

`Result → Option` через `ok()` та `err()`:

```rust
fn main() {
    let result: Result<i32, String> = Ok(42);
    let option: Option<i32> = result.ok(); // Some(42), помилка відкидається
    
    let error: Result<i32, String> = Err(String::from("збій"));
    let option: Option<i32> = error.ok(); // None
}
```

`ok()` корисний коли вам не важлива причина помилки — лише факт, що значення є чи немає. Типовий приклад: `"42".parse::<i32>().ok()` — перетворити Result парсингу на Option.

## 24.7. Практика: безпечний агент

Перепишемо кілька операцій агента з використанням Option та Result замість unwrap і паніки.

Перша задача — безпечне зчитування сенсора. Сенсор може бути offline або повернути невалідне значення:

```rust
fn read_sensor(sensor_id: u8) -> Result<f64, String> {
    // Симуляція: сенсор 3 — offline, сенсор 5 — невалідне значення
    match sensor_id {
        3 => Err(String::from("Сенсор offline")),
        5 => Err(String::from("Невалідне показання: NaN")),
        id => Ok(id as f64 * 10.0 + 5.0), // нормальне показання
    }
}

fn main() {
    let sensors = [1, 2, 3, 4, 5, 6];
    
    println!("Зчитування сенсорів:");
    for &id in &sensors {
        match read_sensor(id) {
            Ok(value) => println!("  Сенсор {}: {:.1}", id, value),
            Err(msg) => println!("  Сенсор {}: ПОМИЛКА — {}", id, msg),
        }
    }
    
    // Зібрати лише успішні показання
    let valid_readings: Vec<f64> = sensors.iter()
        .filter_map(|&id| read_sensor(id).ok())
        .collect();
    
    println!("Валідних показань: {}", valid_readings.len());
    
    if !valid_readings.is_empty() {
        let avg: f64 = valid_readings.iter().sum::<f64>() / valid_readings.len() as f64;
        println!("Середнє: {:.1}", avg);
    }
}
```

Зверніть увагу на `filter_map(|&id| read_sensor(id).ok())`: для кожного сенсора викликаємо read_sensor, перетворюємо Result на Option через `.ok()`, і filter_map пропускає лише Some. Один рядок замість циклу з match та тимчасовим вектором.

Друга задача — валідація команди оператора. Команда містить рядок, який потрібно розпарсити та перевірити:

```rust
fn parse_command(input: &str) -> Result<(&str, f64), String> {
    let parts: Vec<&str> = input.trim().split_whitespace().collect();
    
    if parts.len() != 2 {
        return Err(format!("Очікувалось 2 частини, отримано {}", parts.len()));
    }
    
    let command = parts[0];
    let value = parts[1].parse::<f64>()
        .map_err(|_| format!("Невалідне число: '{}'", parts[1]))?;
    
    // Валідація
    match command {
        "висота" if value >= 0.0 && value <= 500.0 => Ok((command, value)),
        "висота" => Err(format!("Висота {} поза межами [0, 500]", value)),
        "швидкість" if value >= 0.0 && value <= 100.0 => Ok((command, value)),
        "швидкість" => Err(format!("Швидкість {} поза межами [0, 100]", value)),
        other => Err(format!("Невідома команда: '{}'", other)),
    }
}

fn main() {
    let inputs = vec!["висота 150.0", "швидкість abc", "висота 999", "курс 90"];
    
    for input in &inputs {
        match parse_command(input) {
            Ok((cmd, val)) => println!("Прийнято: {} = {:.0}", cmd, val),
            Err(msg) => println!("Відхилено '{}': {}", input, msg),
        }
    }
}
```

Вивід:

```
Прийнято: висота = 150
Відхилено 'швидкість abc': Невалідне число: 'abc'
Відхилено 'висота 999': Висота 999 поза межами [0, 500]
Відхилено 'курс 90': Невідома команда: 'курс'
```

Тут з'явився оператор `?` (рядок з `.map_err(...)?`). Він коротко означає: "якщо Result — Err, повернути цей Err з функції одразу". Це тема наступного Розділу 25, але базове використання вже видно: замість match на кожному кроці — `?` для швидкого пропагування помилки.

Третя задача — ланцюжок операцій місії. Кожен крок може провалитись, і помилка на будь-якому кроці зупиняє всю місію:

```rust
fn check_battery(level: u8) -> Result<u8, String> {
    if level < 20 {
        Err(format!("Батарея критична: {}%", level))
    } else {
        Ok(level)
    }
}

fn check_gps() -> Result<(f64, f64), String> {
    // Симуляція: GPS доступний
    Ok((47.12, 35.45))
}

fn calculate_route(from: (f64, f64), to: (f64, f64)) -> Result<Vec<(f64, f64)>, String> {
    let dist = ((to.0 - from.0).powi(2) + (to.1 - from.1).powi(2)).sqrt();
    if dist > 100.0 {
        Err(format!("Ціль занадто далеко: {:.1} км", dist))
    } else {
        Ok(vec![from, to])
    }
}

fn start_mission(battery: u8, target: (f64, f64)) -> Result<Vec<(f64, f64)>, String> {
    let _bat = check_battery(battery)?;
    let pos = check_gps()?;
    let route = calculate_route(pos, target)?;
    Ok(route)
}

fn main() {
    // Успішна місія
    match start_mission(85, (47.15, 35.50)) {
        Ok(route) => println!("Місія розпочата, {} точок маршруту", route.len()),
        Err(msg) => println!("Місія відхилена: {}", msg),
    }
    
    // Низька батарея
    match start_mission(10, (47.15, 35.50)) {
        Ok(route) => println!("Місія розпочата, {} точок маршруту", route.len()),
        Err(msg) => println!("Місія відхилена: {}", msg),
    }
}
```

Функція `start_mission` — ланцюжок перевірок через `?`. Якщо батарея низька — одразу Err, GPS не перевіряється. Якщо GPS fail — маршрут не обчислюється. Кожен `?` — потенційний ранній вихід. Це набагато чистіше, ніж вкладені match.

---

## 24.8. Prompt Engineering: заміна unwrap на безпечну обробку

### Промпт-шаблон: рефакторинг unwrap

```
Ось мій код з unwrap():
[вставити код]
Заміни кожен unwrap() на безпечну обробку.
Для кожної заміни поясни: який варіант обрав 
(unwrap_or, match, if let, ?) і чому саме він підходить тут.
```

### Промпт-шаблон: вибір між Option та Result

```
Я пишу функцію [опис].
Вона може провалитись через [причини].
Що краще для типу повернення: Option чи Result? Чому?
```

Правило: якщо caller потребує лише факт "є чи немає" — Option. Якщо потрібна причина відсутності (для логування, відображення, або різної обробки різних помилок) — Result.

### Вправа з PE

Візьміть код із Розділу 20 (Практикум v1.0) і знайдіть усі місця з `unwrap()`, `expect()`, або потенційною панікою (індексація вектора без перевірки). Попросіть AI замінити їх на безпечну обробку. Оцініть кожну пропозицію: чи доречний обраний метод? Чи не втрачається інформація про помилку?

---

## Лабораторна робота No24

### Мета

Зробити агента стійким до помилок через Option та Result.

### Завдання базового рівня

1. Замінити всі `unwrap()` в коді агента з Розділу 20–23 на безпечну обробку.
2. Функція `read_waypoints(input: &str) -> Result<Vec<(f64, f64)>, String>` — парсить рядок "x1,y1;x2,y2;..." у вектор координат. Кожна точка може мати помилку парсингу.
3. Функція `find_nearest(objects: &[(String, f64, f64)], pos: (f64, f64)) -> Option<&str>` — найближчий об'єкт, або None якщо список порожній.

### Варіанти

**A.** Повна валідація: створити набір функцій валідації для кожного поля агента (батарея: 0-100, висота: 0-15000, швидкість: >= 0, координати: валідний діапазон). Кожна повертає Result. Об'єднати у `validate_agent_state`.

**B.** Graceful degradation: агент має основний та запасний сенсор для кожного виміру. Якщо основний — Err, використати запасний. Якщо обидва — Err, використати останнє відоме значення. Продемонструвати через ланцюжок `.or_else(...)`.

**C.** Парсер конфігурації: прочитати конфігурацію агента з текстового формату "ключ=значення" (по рядку). Обробити: відсутні обов'язкові ключі, невалідні значення, дублікати.

### Критерії

| Критерій | Бал |
|----------|-----|
| Жоден unwrap у продакшен-коді | 20 |
| Правильний вибір Option vs Result | 20 |
| Використання комбінаторів де доречно | 20 |
| Інформативні повідомлення у Err | 15 |
| Тести (включаючи помилкові сценарії) | 15 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `cannot use ? in function that returns ()`

Оператор `?` у функції, що не повертає Result:

```rust
fn main() {
    let x: i32 = "42".parse()?; // ПОМИЛКА: main повертає ()
}
```

Виправлення: або обробити через match/unwrap_or, або змінити сигнатуру main: `fn main() -> Result<(), Box<dyn std::error::Error>>`. Оператор `?` детально розглянемо у Розділі 25.

### Помилка 2: `type mismatch: expected String, found &str` у Err

Err очікує owned String, а ви передаєте літерал:

```rust
fn check() -> Result<(), String> {
    Err("помилка") // ПОМИЛКА: &str, не String
}
```

Виправлення: `Err(String::from("помилка"))` або `Err("помилка".to_string())`.

### Помилка 3: `Option<Option<T>>` — подвійна обгортка

Використали map замість and_then з функцією, що повертає Option:

```rust
let result = Some("42").map(|s| s.parse::<i32>().ok());
// Тип: Option<Option<i32>> — не те, що потрібно
```

Виправлення: `Some("42").and_then(|s| s.parse::<i32>().ok())` — тип `Option<i32>`.

### Помилка 4: забутий `?` або `match` — помилка ігнорується

```rust
fn main() {
    let mut data = String::new();
    std::io::stdin().read_line(&mut data); // WARNING: unused Result
}
```

Компілятор попередить: "unused Result that must be used". Це не помилка компіляції, але серйозне попередження — ви ігноруєте можливу помилку вводу/виводу.

Виправлення: `.expect("Помилка читання")` або match/if let.

---

## Додатково

### Option та Result як монади

Якщо ви знайомі з функціональним програмуванням — Option та Result у Rust є монадами. `map` — це functor, `and_then` — це monadic bind. Ланцюжок `and_then` — це монадична композиція, де кожен крок може "провалитись" і коротко замкнути весь ланцюжок.

Rust не використовує цю термінологію у документації — замість "monad" каже "combinator", замість "bind" — "and_then". Але якщо ви вивчали Haskell або Scala — концепції ідентичні. Maybe = Option, Either = Result.

### Порівняння підходів до помилок

| Мова | Механізм | Видимий у типі? | Можна ігнорувати? | Runtime overhead |
|------|----------|-----------------|-------------------|-----------------|
| C | return code (-1, NULL) | Частково | Так | Ні |
| Java | checked exceptions | Так | Ні (для checked) | Так (stack unwinding) |
| Python | exceptions | Ні | Так | Так |
| Go | (value, error) tuple | Ні (конвенція) | Так (`_, err`) | Ні |
| Rust | Option/Result | Так (у типі) | Ні (warning) | Ні |

Rust — єдина з цих мов, де тип повернення повністю описує можливість помилки, ігнорування генерує попередження компілятора, і обробка не має runtime overhead (ніякого stack unwinding).

---

## Контрольні запитання

### Рівень 1

1. Чим Option відрізняється від Result?

Відповідь: Option має два варіанти: Some(T) та None. Result — Ok(T) та Err(E). Option каже "є чи немає", Result — "успіх чи помилка з описом".

2. Що зробить `unwrap()` при None/Err?

Відповідь: паніка — аварійне завершення програми з повідомленням.

### Рівень 2

3. Чим `map` відрізняється від `and_then` для Option?

Відповідь: `map(f)` застосовує f до значення всередині Some і автоматично загортає результат у Some: `Some(x) → Some(f(x))`. `and_then(f)` передає значення у f, яка сама повертає Option: `Some(x) → f(x)`. and_then уникає подвійної обгортки Option<Option<T>>.

4. Коли використовувати `unwrap_or` vs `unwrap_or_else`?

Відповідь: `unwrap_or(default)` — коли значення за замовчуванням дешеве (літерал, просте значення). `unwrap_or_else(|| ...)` — коли обчислення дорогого значення за замовчуванням потрібно відкласти до моменту, коли воно дійсно знадобиться (ледача ініціалізація).

### Рівень 3

5. Перепишіть через комбінатори:

```rust
fn process(input: Option<&str>) -> Option<i32> {
    match input {
        Some(s) => {
            let trimmed = s.trim();
            match trimmed.parse::<i32>() {
                Ok(n) if n > 0 => Some(n),
                _ => None,
            }
        }
        None => None,
    }
}
```

Відповідь:

```rust
fn process(input: Option<&str>) -> Option<i32> {
    input
        .map(|s| s.trim())
        .and_then(|s| s.parse::<i32>().ok())
        .filter(|&n| n > 0)
}
```

### Рівень 4

6. Чому Rust не має null і не використовує exceptions? Які переваги та недоліки?

Відповідь: null (Tony Hoare назвав його "billion-dollar mistake") дозволяє будь-якому значенню бути "нічим" без видимості у типі, що призводить до NullPointerException у рантаймі. Option робить відсутність видимою у типі і перевіряється компілятором. Exceptions невидимі у сигнатурі (крім Java checked exceptions), мають runtime overhead (stack unwinding), і можуть бути проігноровані. Result видимий у типі, не має runtime overhead, і компілятор попереджає про ігнорування. Недолік: більш verbose код (match/if let замість try/catch), крива навчання для початківців.

---

## Резюме

`Option<T>` формалізує "значення може бути відсутнім": `Some(T)` або `None`. Використовується коли відсутність — нормальна ситуація (пошук, опціональні параметри, перший/останній елемент).

`Result<T, E>` формалізує "операція може провалитись": `Ok(T)` або `Err(E)`. Використовується коли потрібна причина помилки (IO, парсинг, валідація).

Комбінатори трансформують вміст без ручного match: `map` (перетворити значення), `and_then` (ланцюжок fallible операцій), `unwrap_or`/`unwrap_or_else` (значення за замовчуванням), `or`/`or_else` (альтернативне джерело), `ok_or` (Option → Result), `ok` (Result → Option).

`unwrap()` та `expect()` — антипатерн у продакшен-коді. Вони перетворюють recoverable error на unrecoverable panic. Використовуйте лише у тестах або де ви логічно впевнені у результаті.

---

## Що далі

Комбінатори та match дають повний контроль над помилками, але для функцій із кількома fallible операціями код стає багатослівним. Функція, що відкриває файл, читає його, парсить JSON та валідує дані — це чотири вкладених match або чотири and_then. Оператор `?` (Розділ 25) скорочує це до чотирьох рядків: кожна операція з `?` на кінці автоматично пропагує помилку нагору. А у Розділі 26 ви навчитесь створювати власні типи помилок — замість `String` у кожному Err — ієрархія `DroneError`, `NavigationError`, `SensorError` із автоматичними конвертаціями.
