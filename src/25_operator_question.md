# Розділ 25. Оператор ? та пропагація помилок

## Анотація

У Розділі 24 ви навчились обробляти Option та Result через match, if let та комбінатори. Це дає повний контроль, але для функцій із кількома послідовними fallible-операціями код стає багатослівним. Функція, що перевіряє батарею, зчитує GPS, обчислює маршрут та валідує його — це чотири match або чотири and_then. Кожен рівень вкладеності ускладнює читання. Оператор `?` вирішує цю проблему елегантно: він автоматично розгортає Ok/Some і пропагує Err/None нагору по стеку викликів. Один символ замість п'яти рядків match. Цей розділ пояснює, як `?` працює під капотом, які вимоги ставить до типів, і як trait `From` дозволяє автоматично конвертувати між різними типами помилок.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Застосувати оператор `?` до Result та Option у функціях з відповідним типом повернення.
2. Пояснити, у що компілятор "розгортає" (desugars) оператор `?`.
3. Пояснити роль trait `From<E>` в автоматичній конвертації помилок.
4. Написати `main() -> Result<(), Box<dyn Error>>` для використання `?` у main.
5. Побудувати ланцюжок fallible-операцій з `?` та `map_err`.
6. Обрати між `?`, `match` та комбінаторами для конкретної ситуації.

---

## Ключові терміни

**Оператор ? (question mark)** — синтаксичний оператор, що розгортає Ok(v)/Some(v) у v, або негайно повертає Err(e)/None з поточної функції.

**Error propagation (пропагація помилок)** — патерн, при якому функція не обробляє помилку сама, а передає її caller-у для прийняття рішення.

**From trait** — стандартний trait для конвертації між типами. Оператор `?` використовує `From` для автоматичного перетворення типу помилки.

**Box\<dyn Error\>** — trait object, що може містити будь-яку помилку. Використовується як "універсальний" тип помилки.

---

## Мотиваційний кейс

Автопілот БПЛА при підготовці до місії виконує ланцюжок перевірок: завантажити конфігурацію → перевірити батарею → ініціалізувати GPS → отримати координати бази → обчислити маршрут → валідувати маршрут. Шість кроків, кожен може провалитись з різною помилкою. Без оператора `?` це або шість вкладених match (пірамідна структура, що зсувається вправо з кожним рівнем), або шість and_then з великими замиканнями. З `?` — шість рядків, що читаються як звичайна послідовність команд, з автоматичною зупинкою при першій помилці.

---

## 25.1. Проблема: каскад match

Повернемося до функції start_mission з Розділу 24 та розглянемо, як би вона виглядала без оператора `?`, з повним розгортанням через match.

Кожна з допоміжних функцій повертає Result:

```rust
fn check_battery(level: u8) -> Result<u8, String> {
    if level < 20 {
        Err(format!("Батарея критична: {}%", level))
    } else {
        Ok(level)
    }
}

fn get_gps_position() -> Result<(f64, f64), String> {
    Ok((47.12, 35.45)) // симуляція
}

fn calculate_route(from: (f64, f64), to: (f64, f64)) -> Result<Vec<(f64, f64)>, String> {
    let dist = ((to.0 - from.0).powi(2) + (to.1 - from.1).powi(2)).sqrt();
    if dist > 100.0 {
        Err(format!("Ціль занадто далеко: {:.1}", dist))
    } else {
        Ok(vec![from, to])
    }
}
```

Без `?` — версія з вкладеними match:

```rust
fn start_mission_verbose(battery: u8, target: (f64, f64)) -> Result<Vec<(f64, f64)>, String> {
    match check_battery(battery) {
        Ok(_bat) => {
            match get_gps_position() {
                Ok(pos) => {
                    match calculate_route(pos, target) {
                        Ok(route) => Ok(route),
                        Err(e) => Err(e),
                    }
                }
                Err(e) => Err(e),
            }
        }
        Err(e) => Err(e),
    }
}
```

Три функції — три рівні вкладеності. Кожна гілка Err робить однакову річ: просто пробрасити помилку нагору. Це шаблонний код, що нічого не додає до розуміння логіки. Якщо операцій шість — пірамідна структура стає нечитабельною.

Версія з and_then — площинніша, але все одно громіздка:

```rust
fn start_mission_andthen(battery: u8, target: (f64, f64)) -> Result<Vec<(f64, f64)>, String> {
    check_battery(battery)
        .and_then(|_| get_gps_position())
        .and_then(|pos| calculate_route(pos, target))
}
```

Це краще, але замикання |_| та |pos| додають синтаксичного шуму. Для простих ланцюжків and_then прийнятний. Для складніших, де проміжні результати використовуються у кількох наступних кроках — and_then стає заплутаним: кожне замикання "бачить" лише результат попереднього кроку, а не всіх попередніх.

Тепер — версія з `?`:

```rust
fn start_mission(battery: u8, target: (f64, f64)) -> Result<Vec<(f64, f64)>, String> {
    let _bat = check_battery(battery)?;
    let pos = get_gps_position()?;
    let route = calculate_route(pos, target)?;
    Ok(route)
}
```

Чотири рядки замість дванадцяти. Жодного вкладеного match. Логіка читається зверху вниз: перевірити батарею, отримати позицію, обчислити маршрут, повернути маршрут. Знак `?` після кожного виклику каже: "якщо Ok — продовжити з розгорнутим значенням; якщо Err — повернути цей Err з функції одразу".

Це і є пропагація помилок (error propagation): функція не обробляє помилку сама, а передає її вище по ланцюжку викликів. Рішення про обробку приймає той, хто знає контекст: main може вивести повідомлення, тест може паніканути, веб-сервер може повернути HTTP 500. Функція start_mission лише каже: "якщо щось пішло не так — ось що саме".

---

## 25.2. Як ? працює: десахаризація

Оператор `?` — це синтаксичний цукор (syntactic sugar). Компілятор перетворює його на еквівалентний match. Розуміння цієї трансформації — ключ до розуміння обмежень та можливостей оператора.

Для Result десахаризація виглядає так:

```rust
// Ви пишете:
let value = some_function()?;

// Компілятор перетворює на:
let value = match some_function() {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),
};
```

Ця трансформація містить три ключових механізми, і кожен заслуговує окремого пояснення.

Перший механізм — розгортання (unwrapping). Якщо результат — `Ok(v)`, оператор "розгортає" його і повертає `v`. Виконання продовжується на наступному рядку як ні в чому не бувало. Змінна `value` отримує тип `T` (не `Result<T, E>`). Це те, що робить `?` таким зручним: після `let pos = get_gps()?;` змінна `pos` має тип `(f64, f64)`, а не `Result<(f64, f64), Error>`. Ви працюєте зі "щасливим шляхом" (happy path) без вкладеності.

Другий механізм — ранній вихід (early return). Якщо результат — `Err(e)`, оператор виконує `return Err(...)` — негайне повернення з поточної функції. Жоден наступний рядок не виконується. Це ранній вихід (early return), аналогічний guard clauses з Розділу 11. Різниця в тому, що guard clause ви пишете явно (`if condition { return Err(...); }`), а `?` робить це автоматично.

Третій механізм — конвертація через From. Перед поверненням помилки викликається `From::from(e)`. Це конвертація типу помилки. Чому це потрібно? Уявіть функцію, що повертає `Result<T, MyError>`, і всередині викликає `std::fs::read_to_string(path)?`, яка повертає `Result<String, io::Error>`. Типи помилок різні: `io::Error` та `MyError`. Без конвертації `?` не скомпілюється — не можна повернути `io::Error` звідки очікують `MyError`. `From::from` автоматично конвертує `io::Error` у `MyError`, якщо існує реалізація `impl From<io::Error> for MyError`. Саме це дозволяє використовувати `?` з функціями, що повертають різні типи помилок — компілятор "склеює" їх через From.

Для Option десахаризація простіша — немає конвертації:

```rust
// Ви пишете:
let value = some_option()?;

// Компілятор перетворює на:
let value = match some_option() {
    Some(v) => v,
    None => return None,
};
```

None просто пропагується нагору. Немає From, бо None не містить інформації для конвертації.

Цей механізм пояснює, чому `?` не працює у функціях, що повертають `()` або інші типи: `return Err(...)` чи `return None` мають бути сумісними з типом повернення функції. Якщо функція повертає `i32`, `return Err(...)` не скомпілюється — тип i32 не є ні Result, ні Option.

---

## 25.3. Вимоги до типів: коли ? можна використовувати

Оператор `?` можна використовувати у функції лише якщо її тип повернення сумісний з пропагацією. Конкретно:

Для `Result`: функція повинна повертати `Result<SomeType, SomeErrorType>`. Тип помилки `SomeErrorType` повинен бути сумісним із типом помилки виразу, до якого застосовується `?` (або через точний збіг, або через `From`).

Для `Option`: функція повинна повертати `Option<SomeType>`.

Не можна змішувати: `?` на Result у функції, що повертає Option (і навпаки). Для конвертації використовуйте `.ok()` (Result → Option) або `.ok_or("...")` (Option → Result) перед `?`.

```rust
// Правильно: ? на Result у функції, що повертає Result
fn read_config() -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string("config.txt")?;
    Ok(content)
}

// Правильно: ? на Option у функції, що повертає Option
fn first_positive(data: &[i32]) -> Option<i32> {
    let &first = data.first()?; // first() повертає Option<&i32>
    if first > 0 { Some(first) } else { None }
}

// ПОМИЛКА: ? на Result у функції, що повертає Option
// fn mixed(s: &str) -> Option<i32> {
//     let n = s.parse::<i32>()?; // parse повертає Result, а функція — Option
//     Some(n)
// }

// Виправлення: конвертація Result → Option через .ok()
fn mixed_fixed(s: &str) -> Option<i32> {
    let n = s.parse::<i32>().ok()?; // .ok() перетворює Result на Option
    Some(n)
}
```

---

## 25.4. From trait: автоматична конвертація помилок

У реальних програмах різні функції повертають різні типи помилок: `std::io::Error` для файлових операцій, `std::num::ParseIntError` для парсингу чисел, `serde_json::Error` для JSON. Якщо ваша функція використовує `?` для всіх трьох — типи помилок повинні бути сумісні.

Trait `From` — стандартний механізм конвертації між типами в Rust. Він оголошений так:

```rust
trait From<T> {
    fn from(value: T) -> Self;
}
```

Коли `?` зустрічає `Err(e)`, він фактично виконує `return Err(TargetError::from(e))`, де `TargetError` — тип помилки функції. Якщо `TargetError` реалізує `From<SourceError>`, конвертація відбувається автоматично. Якщо ні — помилка компіляції.

Розглянемо конкретний приклад. Припустимо, у нас є функція, що читає файл та парсить число з нього:

```rust
use std::io;
use std::num::ParseIntError;

// Ця функція НЕ скомпілюється:
// fn read_number(path: &str) -> Result<i32, io::Error> {
//     let text = std::fs::read_to_string(path)?;  // Ok: io::Error → io::Error
//     let num: i32 = text.trim().parse()?;          // ПОМИЛКА: ParseIntError → io::Error ???
//     Ok(num)
// }
```

Другий `?` не спрацює: `parse()` повертає `Result<i32, ParseIntError>`, а функція — `Result<i32, io::Error>`. Немає реалізації `From<ParseIntError> for io::Error>`, тому компілятор не знає, як конвертувати.

Є кілька рішень. Перше — `Box<dyn std::error::Error>`, "контейнер" для будь-якої помилки. Будь-який тип, що реалізує trait `Error`, автоматично конвертується у `Box<dyn Error>` через бланкетну реалізацію From:

```rust
use std::error::Error;
use std::fs;

fn load_mission(path: &str) -> Result<Vec<(f64, f64)>, Box<dyn Error>> {
    let content = fs::read_to_string(path)?;          // io::Error → Box<dyn Error>
    let count: usize = content.lines().count();
    
    if count == 0 {
        return Err("Порожній файл маршруту".into());  // &str → Box<dyn Error>
    }
    
    let mut waypoints = Vec::new();
    for line in content.lines() {
        let parts: Vec<&str> = line.split(',').collect();
        if parts.len() != 2 {
            return Err(format!("Невалідний рядок: {}", line).into());
        }
        let x: f64 = parts[0].trim().parse()?;        // ParseFloatError → Box<dyn Error>
        let y: f64 = parts[1].trim().parse()?;        // ParseFloatError → Box<dyn Error>
        waypoints.push((x, y));
    }
    
    Ok(waypoints)
}
```

У цій функції три різних типи помилок: `io::Error` (файл не знайдено), `String`/`&str` (невалідний формат), `ParseFloatError` (невалідне число). Усі конвертуються у `Box<dyn Error>` автоматично через `?` та `.into()`. Функція не знає деталей кожної помилки — вона просто пропагує їх нагору.

Друге рішення — `map_err` для ручної конвертації кожної помилки у спільний тип (наприклад, String):

```rust
fn read_number(path: &str) -> Result<i32, String> {
    let text = std::fs::read_to_string(path)
        .map_err(|e| format!("Помилка читання '{}': {}", path, e))?;
    let num: i32 = text.trim().parse()
        .map_err(|e| format!("Невалідне число у '{}': {}", path, e))?;
    Ok(num)
}
```

Цей підхід найбільш явний: кожна помилка отримує контекст, і caller бачить зрозуміле повідомлення.

Третє рішення — власний enum помилок з реалізацією From для кожного вхідного типу. Це тема Розділу 26, і саме цей підхід використовується у продакшен-бібліотеках.

`Box<dyn Error>` — зручний для прототипування та невеликих програм. Недолік: caller не може перевірити конкретний тип помилки через match (лише через downcast, який незручний і може провалитись). У Розділі 26 ви навчитесь створювати власні enum помилок, що дають і зручність `?`, і можливість match на конкретні варіанти.

---

## 25.5. Використання ? у main

За замовчуванням `main()` повертає `()`, і `?` у ній не працює. Але Rust дозволяє main повертати Result:

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let config = std::fs::read_to_string("config.txt")?;
    println!("Конфігурація: {} байтів", config.len());
    
    let port: u16 = config.trim().parse()?;
    println!("Порт: {}", port);
    
    Ok(()) // успішне завершення
}
```

Якщо будь-яка операція з `?` повертає Err — main завершиться з повідомленням про помилку (без паніки, але з ненульовим кодом виходу). Це зручно для утиліт командного рядка та скриптів.

`Box<dyn Error>` у main — стандартний підхід: приймає будь-яку помилку, виводить її через Display при завершенні. Для продакшен-коду краще використовувати конкретний тип помилки (Розділ 26).

---

## 25.6. map_err: додавання контексту до помилки

Іноді `?` пропагує помилку, яка занадто низькорівнева для caller-а. "ParseFloatError" не каже, яке саме поле не вдалось парсити. `map_err` додає контекст перед пропагацією:

```rust
fn parse_waypoint(line: &str) -> Result<(f64, f64), String> {
    let parts: Vec<&str> = line.split(',').collect();
    if parts.len() != 2 {
        return Err(format!("Очікувалось 'x,y', отримано: '{}'", line));
    }
    
    let x: f64 = parts[0].trim().parse()
        .map_err(|e| format!("Невалідна координата X '{}': {}", parts[0].trim(), e))?;
    let y: f64 = parts[1].trim().parse()
        .map_err(|e| format!("Невалідна координата Y '{}': {}", parts[1].trim(), e))?;
    
    Ok((x, y))
}

fn main() {
    match parse_waypoint("47.5, abc") {
        Ok(wp) => println!("Waypoint: {:?}", wp),
        Err(msg) => println!("Помилка: {}", msg),
    }
    // Помилка: Невалідна координата Y 'abc': invalid float literal
}
```

Без `map_err` помилка була б "invalid float literal" — без вказівки, що саме пішло не так і в якому контексті. З `map_err` — "Невалідна координата Y 'abc': invalid float literal" — значно інформативніше для дебагу.

Патерн `operation().map_err(|e| format!("контекст: {}", e))?` — один із найуживаніших у Rust. Він додає прошарок контексту до кожної помилки, не втрачаючи оригінального повідомлення.

---

## 25.7. Коли ? не підходить: стратегія обробки помилок

Оператор `?` реалізує одну конкретну стратегію: пропагація — "я не знаю, що робити з цією помилкою, передаю вирішувати caller-у". Це правильна стратегія для більшості випадків у бібліотечному коді та middleware. Але існують ситуації, де потрібні інші стратегії.

Стратегія "відновлення" (recovery) — при помилці спробувати альтернативу та продовжити. Типовий приклад: основний та запасний сенсор:

```rust
fn read_with_fallback(primary: &str, backup: &str) -> String {
    match std::fs::read_to_string(primary) {
        Ok(content) => content,
        Err(e) => {
            println!("Основний файл недоступний: {}. Використовую запасний.", e);
            std::fs::read_to_string(backup)
                .unwrap_or_else(|e2| {
                    println!("Запасний теж недоступний: {}. Використовую замовчування.", e2);
                    String::from("default_config")
                })
        }
    }
}
```

Тут `?` не підходить: функція повертає `String`, а не `Result`. І логіка вимагає спробувати альтернативу, а не здаватись при першій помилці.

Стратегія "збору всіх помилок" (collect all errors) — не зупинятись на першій помилці, а зібрати все:

```rust
fn validate_all(data: &[(String, f64)]) -> Vec<String> {
    let mut errors = Vec::new();
    for (name, value) in data {
        if name.is_empty() {
            errors.push(String::from("Порожнє ім'я"));
        }
        if *value < 0.0 {
            errors.push(format!("Від'ємне значення: {}", value));
        }
    }
    errors
}
```

`?` зупинився б на першій помилці. А тут потрібно показати всі помилки одразу, щоб користувач виправив усе за один раз.

Стратегія "логування та продовження" (log and continue) — записати помилку в лог, але не зупиняти обробку:

```rust
fn process_readings(readings: &[&str]) -> Vec<f64> {
    let mut results = Vec::new();
    for (i, &reading) in readings.iter().enumerate() {
        match reading.parse::<f64>() {
            Ok(value) => results.push(value),
            Err(e) => {
                println!("Показання #{} невалідне ('{}'): {}. Пропущено.", i, reading, e);
            }
        }
    }
    results
}
```

`?` зупинив би обробку при першому невалідному показанні. А нам потрібно обробити всі, пропустивши погані.

Вибір стратегії залежить від контексту. Для БПЛА загальне правило: критичні помилки (немає батареї, апаратний збій) — пропагувати через `?` до верхнього рівня, який приймає рішення про аварійну посадку. Некритичні (один невалідний сенсор з десяти) — записати в лог, використати fallback, продовжити місію. Оператор `?` — інструмент для першої категорії. match/unwrap_or/or_else — для другої.

---

## 25.8. Ланцюжки fallible операцій у БПЛА-агенті

Тепер побудуємо реалістичний pipeline підготовки місії, де кожен крок може провалитись:

```rust
use std::error::Error;

struct MissionConfig {
    target: (f64, f64),
    min_battery: u8,
    max_distance: f64,
}

struct AgentState {
    battery: u8,
    position: (f64, f64),
    altitude: f64,
}

fn load_config(text: &str) -> Result<MissionConfig, Box<dyn Error>> {
    // Спрощений парсинг: "target_x,target_y;min_battery;max_distance"
    let parts: Vec<&str> = text.split(';').collect();
    if parts.len() != 3 {
        return Err("Формат: 'x,y;battery;distance'".into());
    }
    
    let coords: Vec<&str> = parts[0].split(',').collect();
    if coords.len() != 2 {
        return Err("Координати: 'x,y'".into());
    }
    
    Ok(MissionConfig {
        target: (
            coords[0].trim().parse().map_err(|e| format!("target_x: {}", e))?,
            coords[1].trim().parse().map_err(|e| format!("target_y: {}", e))?,
        ),
        min_battery: parts[1].trim().parse().map_err(|e| format!("battery: {}", e))?,
        max_distance: parts[2].trim().parse().map_err(|e| format!("distance: {}", e))?,
    })
}

fn validate_state(state: &AgentState, config: &MissionConfig) -> Result<(), Box<dyn Error>> {
    if state.battery < config.min_battery {
        return Err(format!(
            "Батарея {}% нижче мінімуму {}%", state.battery, config.min_battery
        ).into());
    }
    
    let dist = ((config.target.0 - state.position.0).powi(2) 
              + (config.target.1 - state.position.1).powi(2)).sqrt();
    if dist > config.max_distance {
        return Err(format!(
            "Ціль на відстані {:.1}, максимум {:.1}", dist, config.max_distance
        ).into());
    }
    
    if state.altitude < 0.0 {
        return Err("Некоректна висота (від'ємна)".into());
    }
    
    Ok(())
}

fn prepare_mission(config_text: &str, state: &AgentState) -> Result<Vec<(f64, f64)>, Box<dyn Error>> {
    let config = load_config(config_text)?;
    validate_state(state, &config)?;
    
    // Простий маршрут: пряма лінія
    let route = vec![state.position, config.target];
    
    println!("Місія підготовлена: {} точок маршруту", route.len());
    Ok(route)
}

fn main() {
    let state = AgentState {
        battery: 85,
        position: (0.0, 0.0),
        altitude: 50.0,
    };
    
    // Успішна місія
    match prepare_mission("10.0,20.0;50;100.0", &state) {
        Ok(route) => println!("Маршрут: {:?}", route),
        Err(e) => println!("Помилка підготовки: {}", e),
    }
    
    // Помилка: низька батарея
    let weak = AgentState { battery: 30, position: (0.0, 0.0), altitude: 50.0 };
    match prepare_mission("10.0,20.0;50;100.0", &weak) {
        Ok(_) => println!("Маршрут готовий"),
        Err(e) => println!("Помилка: {}", e),
    }
    
    // Помилка: невалідна конфігурація
    match prepare_mission("not,a;valid;config", &state) {
        Ok(_) => println!("Маршрут готовий"),
        Err(e) => println!("Помилка: {}", e),
    }
}
```

Функція `prepare_mission` — дві операції з `?`. Якщо конфігурація невалідна — валідація не виконується. Якщо стан не проходить валідацію — маршрут не обчислюється. Потік помилок прозорий: caller бачить конкретну причину відмови.

---

## 25.9. Prompt Engineering: рефакторинг match → ?

### Промпт-шаблон

```
Перепиши цю функцію, замінивши вкладені match на оператор ?:
[вставити код]
Для кожної заміни поясни: 
чи потрібен map_err для додавання контексту?
Чи не втрачається інформація про помилку?
```

### Вправа з PE

Візьміть функцію parse_command з Розділу 24 та попросіть AI переписати її з максимальним використанням `?`. Порівняйте: чи стала функція коротшою? Чи залишились інформативні повідомлення про помилки? Чи не додав AI зайвих unwrap?

---

## Лабораторна робота No25

### Мета

Побудувати pipeline підготовки та виконання місії з повною пропагацією помилок.

### Завдання базового рівня

Реалізуйте `fn execute_mission(config: &str) -> Result<MissionReport, Box<dyn Error>>` з кроками:
1. Парсинг конфігурації (формат на ваш вибір).
2. Валідація параметрів.
3. Завантаження списку waypoints (з рядка, імітуючи файл).
4. Перевірка досяжності кожного waypoint.
5. Повернення MissionReport з підсумком.

Кожен крок — окрема функція, що повертає Result. Головна функція використовує лише `?`.

### Варіанти

**A.** Парсер телеметрії: прочитати рядки "timestamp,sensor,value", обробити помилки парсингу кожного поля через map_err з контекстом (номер рядка, назва поля).

**B.** Валідатор плану місії: перевірити 5+ умов (батарея, відстань, погода, дозвіл, наявність waypoints). Кожна перевірка — окрема функція з Result. Зібрати ВСІ помилки (не зупинятись на першій) через цикл з match.

**C.** Graceful pipeline: функція, де деякі кроки критичні (використовують `?`), а деякі — опціональні (обробляються через match з fallback). Наприклад: конфігурація обов'язкова (?), завантаження кешу опціональне (unwrap_or_default).

### Критерії

| Критерій | Бал |
|----------|-----|
| Жоден unwrap у продакшен-коді | 15 |
| Кожен крок — окрема функція з Result | 20 |
| Використання ? для пропагації | 20 |
| map_err з інформативним контекстом | 15 |
| Тести (і успішних, і помилкових сценаріїв) | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `the ? operator can only be used in a function that returns Result or Option`

```rust
fn main() {
    let x: i32 = "42".parse()?; // ПОМИЛКА
}
```

Виправлення: змінити сигнатуру на `fn main() -> Result<(), Box<dyn std::error::Error>>` і додати `Ok(())` в кінці.

### Помилка 2: `the trait From<TypeA> is not implemented for TypeB`

Оператор `?` не може автоматично конвертувати тип помилки:

```rust
fn f() -> Result<(), String> {
    let _file = std::fs::read_to_string("x")?; // io::Error → String: немає From
}
```

Виправлення: додати `map_err` для ручної конвертації: `.map_err(|e| e.to_string())?`, або змінити тип повернення на `Box<dyn Error>`.

### Помилка 3: `cannot use ? for Option in function returning Result`

```rust
fn f(data: &[i32]) -> Result<i32, String> {
    let first = data.first()?; // first() → Option, функція → Result
}
```

Виправлення: конвертувати Option → Result: `data.first().ok_or("Порожній масив")?`.

### Помилка 4: `? на кожному рядку` — втрата контексту

Не помилка компіляції, але проблема: кожен `?` пропагує "сиру" помилку без контексту.

```rust
fn load(path: &str) -> Result<Config, Box<dyn Error>> {
    let text = std::fs::read_to_string(path)?; // "No such file" — який файл?
    let port: u16 = text.trim().parse()?;       // "invalid digit" — яке поле?
    Ok(Config { port })
}
```

Виправлення: додати map_err перед кожним `?`:

```rust
let text = std::fs::read_to_string(path)
    .map_err(|e| format!("Не вдалось прочитати '{}': {}", path, e))?;
```

---

## Додатково

### try блоки (nightly)

В нічних збірках Rust доступний `try {}` — блок, у якому можна використовувати `?`, не вимагаючи щоб вся функція повертала Result:

```rust
// Лише в nightly Rust!
#![feature(try_blocks)]

fn main() {
    let result: Result<i32, String> = try {
        let x: i32 = "42".parse().map_err(|e: std::num::ParseIntError| e.to_string())?;
        let y: i32 = "17".parse().map_err(|e: std::num::ParseIntError| e.to_string())?;
        x + y
    };
    println!("{:?}", result); // Ok(59)
}
```

Це стабілізується у майбутніх версіях Rust і дозволить використовувати `?` у довільних блоках коду, а не лише у функціях з відповідним типом повернення.

### ? та performance

Оператор `?` не має runtime overhead порівняно з ручним match. Обидва компілюються в ідентичний машинний код — перевірка дискримінанту enum та умовний стрибок. Немає stack unwinding (як у C++ exceptions), немає виділення пам'яті. Пропагація помилки — це просто повернення значення з функції.

---

## Контрольні запитання

### Рівень 1

1. У що компілятор перетворює `let x = f()?;`?

Відповідь: `let x = match f() { Ok(v) => v, Err(e) => return Err(From::from(e)) };`

2. Чому `?` не працює у `fn main()` без зміни сигнатури?

Відповідь: за замовчуванням main повертає `()`. `?` потребує `Result` або `Option` як тип повернення, бо виконує `return Err(...)` або `return None`.

### Рівень 2

3. Яку роль відіграє `From::from(e)` у десахаризації `?`?

Відповідь: конвертує тип помилки виразу у тип помилки функції. Наприклад, якщо вираз повертає `io::Error`, а функція — `Result<T, MyError>`, то `From::from` перетворить io::Error на MyError (якщо реалізовано `From<io::Error> for MyError`).

4. Навіщо map_err, якщо ? і так пропагує помилку?

Відповідь: `?` пропагує помилку as-is (або з From-конвертацією). map_err додає контекст: "Помилка парсингу координати X: invalid float" замість просто "invalid float". Без контексту caller не знає, що саме пішло не так.

### Рівень 3

5. Перепишіть через `?`:

```rust
fn process(s: &str) -> Result<f64, String> {
    match s.trim().parse::<f64>() {
        Ok(x) => {
            if x >= 0.0 {
                Ok(x.sqrt())
            } else {
                Err(String::from("Від'ємне число"))
            }
        }
        Err(e) => Err(format!("Парсинг: {}", e)),
    }
}
```

Відповідь:

```rust
fn process(s: &str) -> Result<f64, String> {
    let x: f64 = s.trim().parse()
        .map_err(|e| format!("Парсинг: {}", e))?;
    if x < 0.0 {
        return Err(String::from("Від'ємне число"));
    }
    Ok(x.sqrt())
}
```

### Рівень 4

6. Чому `Box<dyn Error>` зручний для прототипування, але поганий для бібліотек?

Відповідь: `Box<dyn Error>` приймає будь-яку помилку, що зручно — не потрібно створювати власні типи. Але caller не може зробити match на конкретний тип помилки (лише downcast, який незручний і може провалитись). Бібліотека повинна надати caller-у можливість по-різному обробляти різні помилки: "файл не знайдено" vs "формат невалідний" vs "мережа недоступна". Для цього потрібен конкретний enum помилок (Розділ 26).

---

## Резюме

Оператор `?` автоматично розгортає Ok/Some і пропагує Err/None з поточної функції. Це синтаксичний цукор для match з early return. При пропагації Result викликається `From::from(e)` для конвертації типу помилки.

Вимоги: функція повинна повертати Result або Option. Не можна змішувати ? для Result та Option в одній функції без конвертації (.ok(), .ok_or()).

`main() -> Result<(), Box<dyn Error>>` дозволяє використовувати `?` у main. `Box<dyn Error>` приймає будь-яку помилку — зручно для прототипування.

`map_err` додає контекст до помилки перед `?`: замість "invalid float" — "Помилка парсингу координати Y 'abc': invalid float". Використовуйте map_err для кожної операції, де "сира" помилка недостатньо інформативна.

`?` — для пропагації (передати помилку caller-у). match/unwrap_or/or_else — для обробки на місці (спробувати альтернативу, записати в лог, використати значення за замовчуванням).

---

## Що далі

`Box<dyn Error>` та `String` у ролі типу помилки — це прийнятно для навчальних прикладів. Але для реального проєкту потрібна структура: різні помилки вимагають різної обробки. "GPS offline" — спробувати через 5 секунд. "Батарея критична" — негайне повернення на базу. "Невалідна конфігурація" — зупинити запуск.

У Розділі 26 ви навчитесь створювати власні enum помилок: `DroneError::GpsOffline`, `DroneError::LowBattery(u8)`, `DroneError::InvalidConfig(String)`. Крейт `thiserror` автоматизує реалізацію Display та From. Caller зможе зробити match на конкретний варіант і обробити кожну помилку по-різному. А `anyhow` дасть зручну альтернативу для програм (не бібліотек), де конкретний тип помилки менш важливий.
