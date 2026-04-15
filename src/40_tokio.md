# Розділ 40. Tokio: async runtime

## Анотація

У Розділі 39 ви зрозуміли концепції async/await: Future як ледача обіцянка, state machine під капотом, tasks замість потоків. Але стандартна бібліотека Rust не включає runtime — механізм, що виконує futures. Це свідомий вибір: різні проєкти потребують різних runtime (мінімальний для embedded, повнофункціональний для серверів). Tokio — найпопулярніший async runtime для Rust, використовуваний у production системами Amazon, Discord, Cloudflare, Dropbox. Він надає: виконання async tasks на пулі потоків, таймери (sleep, interval, timeout), TCP/UDP мережу, файловий IO, та примітиви синхронізації. Цей розділ зосереджений на ядрі Tokio: запуск runtime, створення tasks, select для конкурентності, та таймери.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Налаштувати Tokio у проєкті та запустити async main через `#[tokio::main]`.
2. Створити async tasks через `tokio::spawn` та зібрати результати.
3. Використати `tokio::select!` для одночасного очікування кількох futures.
4. Використати `tokio::time::sleep`, `interval`, `timeout`.
5. Пояснити різницю між multi-thread та current-thread runtime.
6. Уникати блокуючих операцій в async-контексті.

---

## Ключові терміни

**Tokio** — async runtime для Rust. Надає планувальник tasks, IO-реактор, таймери та примітиви синхронізації.

**#[tokio::main]** — процедурний макрос, що перетворює `async fn main()` у звичайний main із запуском Tokio runtime.

**tokio::spawn** — створює нову async task на runtime. Аналог thread::spawn для async.

**select!** — макрос, що одночасно очікує кілька futures і виконує гілку першого, що завершився.

**JoinHandle** — "квиток" на результат spawned task. `.await` на JoinHandle чекає завершення task.

---

## Мотиваційний кейс

Координатор рою БПЛА повинен одночасно: слухати повідомлення від агентів (мережа), перевіряти таймаути heartbeat (таймер кожні 5с), обробляти команди оператора (stdin), та періодично зберігати стан (файл кожні 30с). З потоками — 4 потоки з синхронізацією. З Tokio — одна async fn з select!, де кожна гілка обробляє свою подію. Код компактніший, простіший для дебагу, ефективніший по пам'яті.

---

## 40.1. Налаштування Tokio

Cargo.toml:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

`features = ["full"]` вмикає всі можливості: runtime, мережу, файли, таймери, синхронізацію. Для мінімального проєкту можна вибрати окремі: `["rt", "macros", "time"]`.

Найпростіший async main:

```rust
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

`#[tokio::main]` — макрос, що обгортає ваш async main у код запуску runtime. Десахаризація:

```rust
// #[tokio::main] async fn main() { ... }
// перетворюється приблизно на:
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("Hello from Tokio!");
    });
}
```

Runtime::new() створює пул потоків (за замовчуванням — по кількості ядер). block_on запускає async-блок на цьому runtime і блокує main thread до завершення. Усередині async блоку доступні всі async-функції Tokio.

Є також `#[tokio::main(flavor = "current_thread")]` — однопотоковий runtime. Усі tasks на одному потоці (як JavaScript event loop). Простіший для дебагу, підходить для нескладних програм та тестів.

---

## 40.2. tokio::spawn: створення async tasks

`tokio::spawn` запускає async-блок як окрему task на runtime. Task виконується конкурентно з іншими tasks — runtime розподіляє час між ними:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let task1 = tokio::spawn(async {
        sleep(Duration::from_millis(200)).await;
        println!("Task 1 завершена");
        42
    });
    
    let task2 = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("Task 2 завершена");
        84
    });
    
    // Обидві tasks працюють конкурентно
    // Task 2 завершиться раніше (100мс < 200мс)
    
    let result1 = task1.await.unwrap(); // чекаємо task 1
    let result2 = task2.await.unwrap(); // task 2 вже завершена
    
    println!("Результати: {} + {} = {}", result1, result2, result1 + result2);
}
```

`tokio::spawn` повертає `JoinHandle<T>`, де T — тип повернення async-блоку. `.await` на JoinHandle блокує поточну task (не потік!) до завершення spawned task. unwrap — бо task могла панікувати.

Відмінності від thread::spawn. Семантично — аналогічно: spawn запускає, JoinHandle чекає, move для ownership. Але: thread::spawn створює OS-потік (мегабайти, мікросекунди). tokio::spawn створює async task (кілобайти, наносекунди). Тисячі tokio::spawn — нормально. Тисячі thread::spawn — проблема.

Вимоги до spawned async block: `Send + 'static`, як і thread::spawn. Send — бо task може мігрувати між потоками runtime. `'static` — бо task може жити довільно довго. Тому потрібен move або owned дані:

```rust
#[tokio::main]
async fn main() {
    let name = String::from("SCOUT-01");
    
    // move — як для thread::spawn
    let handle = tokio::spawn(async move {
        println!("Агент: {}", name);
    });
    
    handle.await.unwrap();
}
```

---

## 40.3. Конкурентність без spawn: join та try_join

Не завжди потрібен spawn. Для одночасного виконання кількох futures у тій самій task:

```rust
use tokio::time::{sleep, Duration};

async fn fetch_position() -> (f64, f64) {
    sleep(Duration::from_millis(100)).await; // імітація GPS
    (47.5, 35.2)
}

async fn fetch_battery() -> u8 {
    sleep(Duration::from_millis(80)).await; // імітація сенсора
    85
}

async fn fetch_altitude() -> f64 {
    sleep(Duration::from_millis(120)).await; // імітація барометра
    150.0
}

#[tokio::main]
async fn main() {
    // ПОСЛІДОВНО: 100 + 80 + 120 = 300мс
    let pos = fetch_position().await;
    let bat = fetch_battery().await;
    let alt = fetch_altitude().await;
    println!("Послідовно: {:?}, {}, {:.0}", pos, bat, alt);
    
    // КОНКУРЕНТНО: max(100, 80, 120) = 120мс
    let (pos, bat, alt) = tokio::join!(
        fetch_position(),
        fetch_battery(),
        fetch_altitude()
    );
    println!("Конкурентно: {:?}, {}, {:.0}", pos, bat, alt);
}
```

`tokio::join!` виконує всі futures конкурентно всередині однієї task. Завершується, коли ВСІ futures завершились. Загальний час — максимум із часів окремих futures, а не сума.

Різниця між join! та spawn: join! виконує futures у поточній task — вони ділять ownership контекст (не потрібен move, можна використовувати &). spawn створює нову task — потрібен move, Send + 'static. join! — для "зробити кілька речей одночасно у цій функції". spawn — для "запустити незалежну задачу, що може жити довше за мене".

`tokio::try_join!` — як join!, але для futures, що повертають Result. Якщо будь-який повертає Err — try_join! одразу повертає цей Err, скасовуючи інші:

```rust
async fn risky_operation() -> Result<i32, String> {
    Err(String::from("збій"))
}

#[tokio::main]
async fn main() {
    let result = tokio::try_join!(
        async { Ok::<_, String>(42) },
        risky_operation(),
    );
    
    match result {
        Ok((a, b)) => println!("Обидва: {}, {}", a, b),
        Err(e) => println!("Помилка: {}", e), // "збій"
    }
}
```

---

## 40.4. tokio::select!: перший із кількох

`select!` — одночасно очікує кілька futures і виконує гілку першого, що завершився. Решта — скасовуються (dropped):

```rust
use tokio::time::{sleep, Duration};
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(32);
    
    // Агент надсилає повідомлення через 200мс
    tokio::spawn(async move {
        sleep(Duration::from_millis(200)).await;
        tx.send(String::from("Об'єкт виявлено!")).await.unwrap();
    });
    
    // Координатор чекає повідомлення АБО таймаут
    tokio::select! {
        msg = rx.recv() => {
            match msg {
                Some(text) => println!("Повідомлення: {}", text),
                None => println!("Канал закрито"),
            }
        }
        _ = sleep(Duration::from_millis(500)) => {
            println!("Таймаут: агент не відповів за 500мс");
        }
    }
}
```

select! запускає обидва futures одночасно: очікування повідомлення та таймаут. Який завершиться першим — та гілка виконається. Інша — скасовується (future dropped, ресурси звільнені).

Це ключовий патерн для координатора рою: "чекай повідомлення від агента, але не довше 5 секунд". Або: "чекай команду від оператора АБО тривогу від агента АБО таймер heartbeat — що б не прийшло першим".

select! з циклом — основа event loop координатора:

```rust
use tokio::time::{sleep, interval, Duration};
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(100);
    
    // Симуляція агентів
    for i in 0..3 {
        let tx = tx.clone();
        tokio::spawn(async move {
            for step in 1..=3 {
                sleep(Duration::from_millis(100 + i * 50)).await;
                tx.send(format!("Agent-{}: крок {}", i, step)).await.unwrap();
            }
        });
    }
    drop(tx); // закриваємо оригінальний sender
    
    let mut heartbeat = interval(Duration::from_millis(250));
    let mut message_count = 0;
    
    loop {
        tokio::select! {
            msg = rx.recv() => {
                match msg {
                    Some(text) => {
                        message_count += 1;
                        println!("[MSG #{}] {}", message_count, text);
                    }
                    None => {
                        println!("Всі канали закриті. Завершення.");
                        break;
                    }
                }
            }
            _ = heartbeat.tick() => {
                println!("[HEARTBEAT] Отримано {} повідомлень", message_count);
            }
        }
    }
}
```

Цикл `loop { select! { ... } }` — серце координатора. На кожній ітерації: якщо є повідомлення — обробити. Якщо спрацював heartbeat таймер — вивести статистику. Якщо канал закрито (всі агенти завершили) — вийти з циклу.

---

## 40.5. tokio::time: таймери та таймаути

Tokio надає три часових примітиви, кожен для своєї задачі:

`sleep(duration)` — призупинити task на вказаний час. Неблокуючий — потік runtime обслуговує інші tasks:

```rust
use tokio::time::{sleep, Duration};

async fn delayed_action() {
    println!("Початок");
    sleep(Duration::from_secs(2)).await; // task спить, потік працює
    println!("Через 2 секунди");
}
```

`interval(period)` — створює потік "тіків" із заданим періодом. Ідеальний для періодичних задач (heartbeat, збереження стану, перевірка сенсорів):

```rust
use tokio::time::{interval, Duration};

async fn periodic_check() {
    let mut timer = interval(Duration::from_secs(5));
    loop {
        timer.tick().await; // чекає наступного тіку
        println!("Перевірка сенсорів...");
    }
}
```

Перший `tick()` спрацьовує одразу (без затримки). Наступні — через period. Якщо обробка тіку затягується довше за period — наступний тік спрацьовує одразу (щоб "наздогнати"). Це можна налаштувати через `interval.set_missed_tick_behavior(...)`.

`timeout(duration, future)` — обгортає future із максимальним часом очікування:

```rust
use tokio::time::{timeout, Duration};

async fn fetch_data() -> String {
    tokio::time::sleep(Duration::from_secs(10)).await; // повільна операція
    String::from("дані")
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_secs(3), fetch_data()).await {
        Ok(data) => println!("Отримано: {}", data),
        Err(_) => println!("Таймаут! Операція не завершилась за 3с"),
    }
}
```

timeout повертає `Result<T, Elapsed>`: Ok з результатом future, або Err якщо час вичерпано. При таймауті inner future скасовується (dropped). Це критичний патерн для мережевих операцій: "спробувати отримати дані, але не чекати безкінечно".

---

## 40.6. Блокуючі операції в async: spawn_blocking

Деякі операції блокують потік і не мають async-версій: CPU-інтенсивні обчислення, деякі бібліотеки з sync API, файловий IO через std. Виконання їх напряму в async-контексті блокує потік runtime — інші tasks на цьому потоці зупиняються.

Рішення — `tokio::task::spawn_blocking`: виконує замикання на окремому потоці, призначеному для блокуючих операцій:

```rust
#[tokio::main]
async fn main() {
    // CPU-інтенсивне обчислення в окремому потоці
    let result = tokio::task::spawn_blocking(|| {
        // Це виконується НЕ на async-потоці
        let sum: u64 = (1..=10_000_000).sum();
        sum
    }).await.unwrap();
    
    println!("Сума: {}", result);
    
    // Async tasks продовжують працювати, поки spawn_blocking обчислює
}
```

spawn_blocking повертає JoinHandle, що можна .await в async-контексті. Обчислення відбувається в окремому thread pool, не блокуючи async runtime.

Правило: будь-яка операція, що займає більше кількох мілісекунд без .await — потенційно блокуюча. Обчислення, sync IO, виклики C-бібліотек — виносьте у spawn_blocking. Async IO (tokio::fs, tokio::net) — використовуйте напряму з .await.

---

## 40.7. Практика: async координатор БПЛА

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, interval, timeout, Duration};
use std::collections::HashMap;

#[derive(Debug)]
enum AgentReport {
    Position { id: String, x: f64, y: f64 },
    Detection { id: String, object: String, x: f64, y: f64 },
    Complete { id: String },
}

async fn simulate_agent(id: String, tx: mpsc::Sender<AgentReport>) {
    for step in 0..5 {
        sleep(Duration::from_millis(80)).await;
        
        let x = step as f64 * 10.0;
        let y = step as f64 * 5.0;
        tx.send(AgentReport::Position { id: id.clone(), x, y }).await.unwrap();
        
        if step == 2 {
            tx.send(AgentReport::Detection {
                id: id.clone(),
                object: format!("OBJ-{}", id),
                x, y,
            }).await.unwrap();
        }
    }
    tx.send(AgentReport::Complete { id }).await.unwrap();
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<AgentReport>(256);
    
    let agents = vec!["SCOUT-01", "SCOUT-02", "RECON-01"];
    
    for agent_id in &agents {
        let tx = tx.clone();
        let id = agent_id.to_string();
        tokio::spawn(async move {
            simulate_agent(id, tx).await;
        });
    }
    drop(tx);
    
    let mut positions: HashMap<String, (f64, f64)> = HashMap::new();
    let mut detections = 0_u32;
    let mut completed = 0_u32;
    let mut heartbeat = interval(Duration::from_millis(200));
    
    loop {
        tokio::select! {
            report = rx.recv() => {
                match report {
                    Some(AgentReport::Position { id, x, y }) => {
                        positions.insert(id, (x, y));
                    }
                    Some(AgentReport::Detection { id, object, x, y }) => {
                        detections += 1;
                        println!("[DETECT] {} знайшов {} на ({:.0}, {:.0})", id, object, x, y);
                    }
                    Some(AgentReport::Complete { id }) => {
                        completed += 1;
                        println!("[DONE] {} завершив", id);
                        if completed == agents.len() as u32 {
                            println!("Всі агенти завершили!");
                            break;
                        }
                    }
                    None => break,
                }
            }
            _ = heartbeat.tick() => {
                println!("[HB] Агентів: {}, виявлень: {}, завершено: {}",
                         positions.len(), detections, completed);
            }
        }
    }
    
    println!("\n=== Фінальний звіт ===");
    println!("Виявлень: {}", detections);
    for (id, (x, y)) in &positions {
        println!("  {} → ({:.0}, {:.0})", id, x, y);
    }
}
```

Архітектура: 3 async tasks-агенти відправляють AgentReport через mpsc channel. Main task — координатор із select! loop: обробляє звіти та виводить heartbeat. Все асинхронне: sleep замість thread::sleep, tokio::sync::mpsc замість std::sync::mpsc.

---

## 40.8. Prompt Engineering: scaffold async системи

### Промпт-шаблон

```
Я будую async систему з Tokio:
- N агентів (async tasks), кожен читає сенсори та надсилає звіти
- Координатор (async task) приймає звіти та надсилає команди
- Періодичне збереження стану кожні 30с
- Таймаут: якщо агент не відповів 10с — позначити як lost

Згенеруй каркас: типи повідомлень (enum), структуру 
async fn для кожного компонента, select! loop координатора.
Я заповню бізнес-логіку.
```

---

## Лабораторна робота No40

### Мета

Побудувати async координатор рою на Tokio.

### Завдання базового рівня

1. 4+ async tasks-агенти через tokio::spawn.
2. mpsc channel для звітів (enum з 3+ варіантами).
3. select! loop у координаторі: повідомлення + heartbeat interval.
4. timeout для операцій: якщо агент не відповів за N секунд — логувати.

### Варіанти

**A.** Двонаправлена комунікація: координатор → агенти через окремі mpsc channels. Координатор надсилає "зміна цілі" через 2с.

**B.** Graceful shutdown: координатор приймає Ctrl+C (tokio::signal) через select! та завершує всі tasks.

**C.** Конкурентні запити: координатор одночасно запитує стан у 5 агентів через join! з timeout на кожен.

### Критерії

| Критерій | Бал |
|----------|-----|
| tokio::spawn для tasks | 15 |
| mpsc channel з enum повідомлень | 20 |
| select! loop із 2+ гілками | 25 |
| timeout або interval | 15 |
| Тести (tokio::test) | 15 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `there is no reactor running`

Виклик async-функції Tokio поза runtime:

```rust
fn main() {
    tokio::time::sleep(Duration::from_secs(1)); // ПОМИЛКА
}
```

Виправлення: `#[tokio::main] async fn main()` або `Runtime::new().unwrap().block_on(...)`.

### Помилка 2: task ніколи не завершується у select!

select! скасовує гілки, що не "перемогли". Якщо future у гілці — нескінченний (наприклад, recv без close) — це нормально: select! обере іншу гілку. Але якщо ВСІ гілки нескінченні та жодна не готова — select! блокується.

Виправлення: завжди мати "вихідну" гілку (таймаут або сигнал shutdown).

### Помилка 3: `Send` bound не задоволено для spawned task

```rust
let data = Rc::new(42);
tokio::spawn(async move { println!("{}", data); }); // ПОМИЛКА: Rc не Send
```

Виправлення: Arc замість Rc. Або `tokio::task::spawn_local` (single-thread runtime, не потребує Send).

### Помилка 4: `std::sync::Mutex` блокує async runtime

```rust
let mutex = std::sync::Mutex::new(42);
tokio::spawn(async move {
    let guard = mutex.lock().unwrap(); // БЛОКУЄ потік runtime!
});
```

Виправлення: `tokio::sync::Mutex` — async-версія, що не блокує потік.

---

## Додатково

### Multi-thread vs current-thread runtime

```rust
// Multi-thread (за замовчуванням): пул потоків = кількість ядер
#[tokio::main]
async fn main() { }

// Current-thread: один потік, всі tasks на ньому
#[tokio::main(flavor = "current_thread")]
async fn main() { }
```

Multi-thread — для production: tasks розподіляються по ядрах. Current-thread — для простих програм та тестів: простіший для дебагу, не потребує Send для spawned tasks.

### tokio::sync::Mutex vs std::sync::Mutex

std::sync::Mutex блокує OS-потік при lock() — інші tasks на цьому потоці не виконуються. tokio::sync::Mutex — async: lock().await призупиняє task, потік бере іншу task. Правило: якщо lock тримається коротко (наносекунди) — std::sync::Mutex прийнятний і навіть швидший. Якщо lock може триматись довго (через .await всередині critical section) — тільки tokio::sync::Mutex.

---

## Контрольні запитання

### Рівень 1

1. Що робить `#[tokio::main]`?

Відповідь: створює Tokio runtime та запускає async fn main() на ньому. Десахаризується у Runtime::new() + block_on().

2. Чим tokio::spawn відрізняється від thread::spawn?

Відповідь: tokio::spawn створює async task (кілобайти, наносекунди переключення). thread::spawn — OS-потік (мегабайти, мікросекунди). Обидва потребують move, Send, 'static.

### Рівень 2

3. Коли join!, а коли spawn?

Відповідь: join! — конкурентне виконання у поточній task (не потрібен Send, можна &). spawn — нова незалежна task (потрібен Send + 'static, може жити довше за caller-а). join! — "зроби це разом зі мною". spawn — "зроби це окремо від мене".

4. Чому блокуючий код у async — проблема?

Відповідь: async runtime використовує кілька потоків для тисяч tasks. Блокуючий код (std::thread::sleep, CPU-обчислення, sync IO) зупиняє потік — всі tasks на ньому заблоковані. spawn_blocking виносить блокуючий код на окремий потік.

### Рівень 3

5. Реалізуйте timeout для отримання з каналу:

Відповідь:

```rust
match timeout(Duration::from_secs(5), rx.recv()).await {
    Ok(Some(msg)) => println!("Отримано: {:?}", msg),
    Ok(None) => println!("Канал закрито"),
    Err(_) => println!("Таймаут: 5 секунд без відповіді"),
}
```

### Рівень 4

6. Порівняйте архітектуру "thread per agent" vs "async task per agent" для 1000 агентів.

Відповідь: 1000 потоків: ~4 ГБ стеків, тисячі context switches ОС, серйозне навантаження на планувальник. 1000 async tasks: ~1 МБ, переключення у userspace наносекундами, 4-8 потоків runtime обслуговують усіх. Для IO-bound (мережева комунікація агентів) async у 1000× ефективніший по пам'яті та значно кращий по латентності.

---

## Резюме

Tokio — async runtime для Rust. `#[tokio::main]` запускає runtime. `tokio::spawn` створює async task. Обидва потребують Send + 'static (як thread::spawn).

`join!` — конкурентне виконання кількох futures у одній task (не потребує Send). `select!` — виконати гілку першого futures, що завершився; решта скасовуються. select! у циклі — основа event loop координатора.

Таймери: `sleep` (призупинити), `interval` (періодичні тіки), `timeout` (максимальний час). Всі неблокуючі — task спить, потік працює.

spawn_blocking — для блокуючих операцій (CPU, sync IO). Виконується на окремому потоці, не блокує async runtime.

std::sync::Mutex vs tokio::sync::Mutex: перший блокує потік, другий — task. Використовуйте async Mutex якщо lock тримається через .await.

---

## Що далі

tokio::spawn та tokio::sync::mpsc дали інструменти для async комунікації. Розділ 41 розширить це: async channels (broadcast для "один до багатьох", watch для "останнє значення", oneshot для "одна відповідь"), Streams (async аналог ітераторів), та tokio::sync примітиви (Semaphore для обмеження конкурентності). Агент отримає потокову обробку даних сенсорів через Stream та координацію рою через broadcast.
