# Розділ 38. Channels: обмін повідомленнями

## Анотація

У Розділі 37 потоки ділили спільний стан через Arc\<Mutex\<T\>\>. Це працює, але має проблеми: contention (потоки чекають один одного на lock), ризик deadlocks (при кількох мьютексах), складність дебагу (хто коли тримав lock?). Існує альтернативний підхід до комунікації між потоками: замість спільного стану — обмін повідомленнями. Потік A не записує дані у спільну HashMap. Він відправляє повідомлення "я знайшов об'єкт на координатах (X, Y)" через канал. Потік B отримує це повідомлення та оновлює свою локальну карту. Жодного спільного стану, жодного Mutex, жодного ризику deadlock.

Цей підхід відомий як CSP (Communicating Sequential Processes) і втілений у слогані з мови Go: "Don't communicate by sharing memory; share memory by communicating." Rust підтримує обидва підходи (Mutex та channels), і вибір залежить від архітектури. Цей розділ пояснює channels у стандартній бібліотеці (`mpsc`), їхню семантику ownership, та як побудувати систему комунікації рою БПЛА.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити філософію "share memory by communicating" та її переваги.
2. Створити канал через `mpsc::channel()` і використати Sender/Receiver.
3. Клонувати Sender для multiple producers.
4. Використати `sync_channel` для обмеження розміру буфера.
5. Обрати між channels та shared state (Mutex) для конкретної задачі.
6. Побудувати систему повідомлень для рою БПЛА.

---

## Ключові терміни

**Channel (канал)** — механізм передачі повідомлень між потоками. Має два кінці: Sender (відправник) та Receiver (отримувач).

**mpsc** — Multiple Producer, Single Consumer — тип каналу, де кілька потоків можуть відправляти, але лише один отримує.

**Sender\<T\>** — відправний кінець каналу. Можна клонувати для кількох producers. Send передає ownership повідомлення.

**Receiver\<T\>** — приймальний кінець каналу. Не можна клонувати (single consumer). recv() блокує до отримання повідомлення.

**Bounded channel (обмежений канал)** — канал із фіксованим розміром буфера. send() блокує, коли буфер повний (backpressure).

**Unbounded channel (необмежений канал)** — канал без обмеження буфера. send() ніколи не блокує, але може споживати необмежену пам'ять.

---

## Мотиваційний кейс

Координатор рою БПЛА отримує звіти від десятків агентів одночасно. З Arc\<Mutex\<Vec\<Report\>\>\> кожен агент блокує мьютекс для push — при 20 агентах це 20 потоків, що конкурують за один lock. Координатор теж конкурує за lock, щоб прочитати звіти. Contention зростає, продуктивність падає.

З channels: кожен агент відправляє Report через свій Sender. Координатор приймає через один Receiver. Жодного lock — кожен send та recv працює незалежно. Координатор обробляє повідомлення по черзі міру надходження, як поштову скриньку. Навантаження агентів не впливає на координатора і навпаки.

---

## 38.1. Два підходи до міжпотокової комунікації

Є два фундаментально різних підходи до комунікації між потоками. Розуміння їхніх trade-offs — ключ до правильного вибору.

Shared state (спільний стан) — Arc\<Mutex\<T\>\>. Потоки ділять одну структуру даних і синхронізують доступ через locks. Плюси: прямий доступ до актуальних даних, ефективний для read-heavy сценаріїв (RwLock). Мінуси: contention при частих записах, ризик deadlocks, складність дебагу ("хто тримає lock?").

Message passing (обмін повідомленнями) — channels. Потоки не ділять дані — вони надсилають копії або переміщують ownership через канал. Плюси: немає locks → немає contention та deadlocks, кожен потік працює з локальними даними, простіший для розуміння (потік даних лінійний). Мінуси: overhead копіювання даних (або втрата ownership), складніше отримати "актуальний стан" (дані у каналі — це історія, не поточний стан).

Коли що обирати? Shared state — коли кілька потоків потребують читання актуального стану (карта світу: "що ми знаємо зараз?"). Message passing — коли потоки виконують роботу та повідомляють про результати (producer-consumer: агенти → координатор). На практиці часто комбінують обидва: channels для передачі повідомлень, shared state для спільного кешу.

Rust підтримує обидва підходи однаково добре. Ownership system гарантує безпеку в обох випадках: Mutex забезпечує ексклюзивний доступ до shared state, а channels переміщують ownership повідомлення від sender до receiver — жоден потік не може звернутись до повідомлення після відправки.

---

## 38.2. mpsc::channel: базовий канал

`std::sync::mpsc::channel()` створює пару (Sender, Receiver) для передачі значень типу T:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // Створюємо канал для String
    let (sender, receiver) = mpsc::channel();
    
    // Потік-відправник
    thread::spawn(move || {
        let message = String::from("Привіт з іншого потоку!");
        sender.send(message).unwrap();
        // message переміщено у канал — тут більше не доступне
        // println!("{}", message); // ПОМИЛКА: value used after move
        println!("Відправлено");
    });
    
    // Main — отримувач
    let received = receiver.recv().unwrap();
    println!("Отримано: {}", received);
}
```

Розберемо механізм покроково.

`mpsc::channel()` повертає кортеж (Sender\<T\>, Receiver\<T\>). Тип T виводиться з першого send. Канал — необмежений (unbounded): send ніколи не блокує, повідомлення накопичуються у внутрішньому буфері.

`sender.send(message)` переміщує (move) ownership повідомлення у канал. Після send відправник більше не має доступу до message — це та сама ownership-семантика, що й передача у функцію. send повертає `Result<(), SendError<T>>`: Ok якщо receiver ще існує, Err якщо receiver знищений (канал "закритий").

`receiver.recv()` блокує поточний потік до отримання повідомлення. Повертає `Result<T, RecvError>`: Ok з повідомленням, або Err якщо всі sender-и знищені (канал "закритий" — більше повідомлень не буде). Ownership повідомлення переходить від каналу до receiver.

Ownership flow: відправник створює → send (move у канал) → канал зберігає → recv (move з каналу) → отримувач володіє. У кожен момент лише один "місце" володіє повідомленням. Data race неможливий — не через Mutex, а через ownership transfer.

---

## 38.3. Кілька повідомлень та ітерація

Sender можна використовувати багаторазово. Receiver можна ітерувати як ітератор — він видає повідомлення, поки канал не закриється:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    thread::spawn(move || {
        let messages = vec![
            "Зліт завершено",
            "Висота 100м",
            "Скан зони A",
            "Об'єкт виявлено",
            "Повернення на базу",
        ];
        
        for msg in messages {
            tx.send(String::from(msg)).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
        println!("Потік: всі повідомлення відправлені");
        // tx знищується тут → канал закривається
    });
    
    // Ітерація по каналу — блокує до наступного повідомлення
    // Завершується коли канал закритий (всі sender-и знищені)
    for message in rx {
        println!("Отримано: {}", message);
    }
    
    println!("Канал закритий, всі повідомлення отримані");
}
```

`for message in rx` — Receiver реалізує Iterator. Кожна ітерація блокується до наступного повідомлення. Коли tx знищується (потік завершився) — канал закривається, ітератор повертає None, цикл завершується. Це природний патерн "producer produces until done, consumer consumes until channel closed".

Крім recv (блокуючий), є `try_recv` (неблокуючий):

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    tx.send(42).unwrap();
    
    // try_recv: повертає одразу, не блокує
    match rx.try_recv() {
        Ok(msg) => println!("Є повідомлення: {}", msg),
        Err(mpsc::TryRecvError::Empty) => println!("Канал порожній"),
        Err(mpsc::TryRecvError::Disconnected) => println!("Канал закритий"),
    }
}
```

try_recv корисний для неблокуючих циклів: потік робить свою роботу, періодично перевіряє "чи є нові повідомлення?", і обробляє їх якщо є.

Є також `recv_timeout(Duration)` — блокує до повідомлення або таймауту. Корисно для heartbeat-моніторингу: "якщо агент не відповів за 5 секунд — позначити як втрачений".

---

## 38.4. Multiple Producers: клонування Sender

mpsc означає Multiple Producer, Single Consumer. Кілька потоків можуть відправляти в один канал — через clone Sender:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    // 4 потоки-відправники
    for i in 0..4 {
        let tx_clone = tx.clone(); // кожен потік отримує свій Sender
        thread::spawn(move || {
            for step in 1..=3 {
                let msg = format!("Agent-{}: крок {}", i, step);
                tx_clone.send(msg).unwrap();
                thread::sleep(Duration::from_millis(50 + i * 20));
            }
            // tx_clone знищується при завершенні потоку
        });
    }
    
    // Важливо: знищити оригінальний tx!
    drop(tx); // інакше канал ніколи не закриється
    
    // Отримувач збирає всі повідомлення
    let mut count = 0;
    for message in rx {
        println!("[{}] {}", count, message);
        count += 1;
    }
    
    println!("Всього отримано: {} повідомлень", count); // 12
}
```

Кожен `tx.clone()` створює новий Sender, що вказує на той самий канал. Канал закривається, коли ВСІ Sender-и знищені. Тому `drop(tx)` — критичний: оригінальний tx залишається у main, і без явного drop канал ніколи не закриється, `for message in rx` буде чекати вічно.

Чому Receiver не можна клонувати? mpsc — single consumer. Одне повідомлення отримує рівно один потік. Якби два Receiver-и існували — хто отримає повідомлення? Для multiple consumers потрібні інші бібліотеки (crossbeam-channel з mpmc — multiple producer, multiple consumer).

---

## 38.5. sync_channel: обмежений буфер та backpressure

`mpsc::channel()` — необмежений: send ніколи не блокує, повідомлення накопичуються у буфері. Якщо producer швидший за consumer — буфер зростає безмежно, з'їдаючи пам'ять. Для реального БПЛА це небезпечно: сенсор генерує 1000 показань/с, обробник встигає 100/с — за хвилину 54000 показань у буфері.

`sync_channel(bound)` створює канал з обмеженим буфером:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // Буфер на 3 повідомлення
    let (tx, rx) = mpsc::sync_channel(3);
    
    thread::spawn(move || {
        for i in 1..=10 {
            println!("Відправляю {}...", i);
            tx.send(i).unwrap(); // БЛОКУЄ, якщо буфер повний
            println!("Відправлено {}", i);
        }
    });
    
    // Повільний споживач
    for msg in rx {
        thread::sleep(Duration::from_millis(200));
        println!("  Отримано: {}", msg);
    }
}
```

Коли буфер повний (3 повідомлення), send блокується до тих пір, поки receiver не забере одне повідомлення. Це backpressure — механізм, що не дозволяє producer-у "загнати" consumer-а. Producer сповільнюється до швидкості consumer-а. Це запобігає необмеженому зростанню пам'яті.

`sync_channel(0)` — rendezvous channel: send блокує до тих пір, поки receiver не викличе recv. Кожне повідомлення передається "з рук у руки" — синхронно. Максимальне backpressure, мінімальний буфер.

Вибір: `channel()` (unbounded) — коли producer рідко швидший за consumer, або коли пам'ять не обмежена. `sync_channel(N)` — коли потрібно контролювати пам'ять та забезпечити backpressure. N — компроміс: більший буфер = менше блокувань producer-а, але більше пам'яті.

---

## 38.6. Типізовані повідомлення: enum як протокол

Канал передає один тип T. Для різних видів повідомлень — використовуйте enum:

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
enum AgentMessage {
    StatusReport {
        agent_id: String,
        battery: u8,
        position: (f64, f64),
    },
    ObjectDetected {
        agent_id: String,
        object_id: String,
        position: (f64, f64),
        kind: String,
    },
    MissionComplete {
        agent_id: String,
        steps: u32,
    },
    Emergency {
        agent_id: String,
        reason: String,
    },
}

fn simulate_agent(id: &str, tx: mpsc::Sender<AgentMessage>) {
    let id = String::from(id);
    
    tx.send(AgentMessage::StatusReport {
        agent_id: id.clone(),
        battery: 95,
        position: (0.0, 0.0),
    }).unwrap();
    
    thread::sleep(Duration::from_millis(100));
    
    tx.send(AgentMessage::ObjectDetected {
        agent_id: id.clone(),
        object_id: String::from("OBJ-42"),
        position: (15.0, 25.0),
        kind: String::from("будівля"),
    }).unwrap();
    
    thread::sleep(Duration::from_millis(100));
    
    tx.send(AgentMessage::MissionComplete {
        agent_id: id,
        steps: 10,
    }).unwrap();
}

fn main() {
    let (tx, rx) = mpsc::channel();
    
    let agents = vec!["SCOUT-01", "SCOUT-02", "RECON-01"];
    
    for agent_id in &agents {
        let tx_clone = tx.clone();
        let id = agent_id.to_string();
        thread::spawn(move || simulate_agent(&id, tx_clone));
    }
    drop(tx);
    
    // Координатор обробляє повідомлення
    let mut total_objects = 0;
    let mut completed = 0;
    
    for msg in rx {
        match msg {
            AgentMessage::StatusReport { agent_id, battery, position } => {
                println!("[STATUS] {} — bat={}%, pos={:?}", agent_id, battery, position);
            }
            AgentMessage::ObjectDetected { agent_id, object_id, position, kind } => {
                println!("[DETECT] {} знайшов {} ({}) на {:?}", 
                         agent_id, object_id, kind, position);
                total_objects += 1;
            }
            AgentMessage::MissionComplete { agent_id, steps } => {
                println!("[DONE] {} завершив за {} кроків", agent_id, steps);
                completed += 1;
            }
            AgentMessage::Emergency { agent_id, reason } => {
                println!("[ALERT] {} АВАРІЯ: {}", agent_id, reason);
            }
        }
    }
    
    println!("\n=== Підсумки ===");
    println!("Об'єктів виявлено: {}", total_objects);
    println!("Агентів завершили: {}/{}", completed, agents.len());
}
```

Enum AgentMessage — це протокол комунікації рою. Кожен варіант — тип повідомлення з відповідними даними. match на receiver-і — обробка кожного типу з конкретною логікою. Exhaustive match гарантує: додавання нового типу повідомлення (наприклад, LowBattery) одразу спричинить помилку компіляції у кожному match — жодне повідомлення не буде проігноровано.

Зверніть увагу: AgentMessage derive Serialize/Deserialize (Розділ 35). Це означає, що ті самі повідомлення можна серіалізувати у JSON для передачі по мережі — локальний канал та мережевий протокол використовують однакову структуру даних.

---

## 38.7. Prompt Engineering: проєктування системи повідомлень

### Промпт-шаблон

```
Я проєктую систему комунікації для рою БПЛА.
Учасники: координатор, N агентів-розвідників, M агентів-транспортників.
Потоки повідомлень:
- Агенти → координатор: звіти, виявлення, аварії
- Координатор → агенти: команди, оновлення місії

Спроєктуй:
1. Enum для кожного потоку повідомлень
2. Архітектуру каналів (скільки каналів, хто sender, хто receiver)
3. Чи потрібен sync_channel і з яким буфером
```

### Вправа з PE

Попросіть AI порівняти Arc<Mutex<Vec<Report>>> та mpsc::channel для збору звітів від 10 агентів. Чи правильно AI оцінив trade-offs (contention vs ownership transfer)? Чи згадав про drop(tx)?

---

## Лабораторна робота No38

### Мета

Побудувати систему комунікації рою через channels.

### Завдання базового рівня

1. Enum AgentMessage з 4+ варіантами (StatusReport, ObjectDetected, MissionComplete, Emergency).
2. 4+ потоки-агенти, кожен відправляє кілька повідомлень через Sender.
3. Main (координатор) отримує через Receiver та обробляє match.
4. Статистика: кількість повідомлень за типом, кількість завершених агентів.

### Варіанти

**A.** Двонаправлена комунікація: координатор → агенти через окремий канал. Кожен агент має свій Receiver для команд. Координатор надсилає "зміна цілі" — агент змінює поведінку.

**B.** sync_channel з backpressure: агент генерує показання сенсорів з високою частотою (1000/с). sync_channel(100) обмежує буфер. Виміряти, чи сповільнюється producer.

**C.** Pipeline: три стадії обробки — producer → transformer → consumer. Два канали: producer відправляє сирі дані → transformer фільтрує та перетворює → consumer зберігає. Кожна стадія — окремий потік.

### Критерії

| Критерій | Бал |
|----------|-----|
| Enum повідомлень з 4+ варіантами | 15 |
| Multiple producers (clone Sender) | 20 |
| Коректний drop(tx) для закриття каналу | 10 |
| match для кожного варіанту повідомлення | 20 |
| Агрегація результатів після закриття каналу | 15 |
| Тести | 10 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `for message in rx` зависає назавжди

Канал не закривається, бо існує Sender, що не використовується:

```rust
let (tx, rx) = mpsc::channel();
let tx_clone = tx.clone();
thread::spawn(move || { tx_clone.send(42).unwrap(); });
// tx все ще існує → канал відкритий → rx.recv() чекає вічно
for msg in rx { /* зависне */ }
```

Виправлення: `drop(tx)` перед ітерацією — знищити оригінальний Sender.

### Помилка 2: `sending on a closed channel`

Receiver знищений, а Sender намагається відправити:

```rust
let (tx, rx) = mpsc::channel::<i32>();
drop(rx); // закриваємо receiver
tx.send(42); // Err(SendError(42))
```

Виправлення: перевіряти результат send: `if tx.send(msg).is_err() { break; }` — якщо receiver закритий, припинити відправку.

### Помилка 3: повідомлення приходять у непередбачуваному порядку

Кілька producers — порядок повідомлень від різних потоків не визначений. Повідомлення від одного потоку зберігають порядок між собою, але між потоками — ні.

Це не помилка — це фундаментальна властивість паралелізму. Якщо потрібен порядок — додайте timestamp до повідомлення та сортуйте на receiver.

### Помилка 4: `Receiver<T> cannot be shared between threads`

Спроба використати один Receiver у кількох потоках:

```rust
let (tx, rx) = mpsc::channel();
let rx2 = rx; // move, не clone — Receiver не Clone
```

mpsc — single consumer. Для multiple consumers використовуйте crossbeam-channel або створіть dispatcher: один потік отримує з rx і розподіляє по окремих каналах для кожного consumer.

---

## Додатково

### crossbeam-channel: розширені можливості

Стандартний mpsc має обмеження: single consumer, немає select (отримати з першого з кількох каналів). Крейт `crossbeam-channel` надає:

mpmc (multiple producer, multiple consumer) — кілька потоків можуть отримувати з одного каналу. Корисно для thread pool: кілька worker-ів беруть завдання з однієї черги.

select! — отримати повідомлення з першого каналу, що має дані. Корисно для координатора, що слухає кілька джерел: канал звітів, канал команд, канал heartbeat.

never та after — спеціальні "канали" для таймаутів та логіки "ніколи не отримає".

Для навчальних проєктів стандартний mpsc достатній. crossbeam-channel — для production-систем.

### Channels та ownership: чому це безпечно

Channels у Rust безпечні не через runtime перевірки (як Mutex), а через ownership. send(message) переміщує ownership повідомлення з відправника у канал. recv() переміщує ownership з каналу в отримувач. У кожен момент рівно одне "місце" володіє повідомленням. Data race неможливий без shared mutable state. Внутрішньо канал використовує lock-free алгоритми для буфера — тому він ефективніший за naive Mutex<VecDeque>.

### Порівняння channels та shared state

| Критерій | Channels | Arc\<Mutex\<T\>\> |
|----------|----------|-------------------|
| Модель | Ownership transfer | Shared access |
| Lock contention | Немає | Є при частих lock |
| Deadlock ризик | Немає (один ресурс) | Є (при кількох Mutex) |
| Актуальний стан | Ні (повідомлення = історія) | Так (read = поточний стан) |
| Складність дебагу | Простіший (потік даних лінійний) | Складніший (хто тримає lock?) |
| Overhead | Allocation на повідомлення | Lock/unlock на доступ |
| Типовий use case | Producer-consumer, events | Shared cache, shared map |

---

## Контрольні запитання

### Рівень 1

1. Що означає mpsc?

Відповідь: Multiple Producer, Single Consumer — кілька потоків відправляють (clone Sender), один отримує (Receiver не клонується).

2. Що повертає `recv()` коли канал закритий?

Відповідь: `Err(RecvError)` — всі Sender-и знищені, більше повідомлень не буде.

### Рівень 2

3. Чому потрібен `drop(tx)` перед `for msg in rx`?

Відповідь: for msg in rx завершиться лише коли канал закритий (всі Sender-и знищені). Якщо оригінальний tx не знищений — він тримає канал відкритим, навіть якщо всі клони вже знищені. drop(tx) знищує оригінальний Sender, і коли останній clone у потоці теж знищиться — канал закриється.

4. Чим `sync_channel(3)` відрізняється від `channel()`?

Відповідь: `channel()` — необмежений буфер, send ніколи не блокує (пам'ять може зростати необмежено). `sync_channel(3)` — буфер на 3 повідомлення, send блокує коли буфер повний (backpressure). sync_channel контролює пам'ять та сповільнює producer до швидкості consumer.

### Рівень 3

5. Спроєктуйте enum для протоколу "координатор → агент" з 4 варіантами.

Відповідь:

```rust
enum CoordinatorCommand {
    MoveTo { x: f64, y: f64 },
    ChangeAltitude { meters: f64 },
    UpdateMission { new_target: (f64, f64), priority: u8 },
    ReturnToBase,
}
```

### Рівень 4

6. Коли обирати channels, а коли Arc\<Mutex\<T\>\>?

Відповідь: channels — коли дані "потікають" від producer до consumer (звіти, події, команди). Один потік створює, інший обробляє. Немає потреби у "поточному стані". Arc\<Mutex\<T\>\> — коли кілька потоків потребують доступу до актуального стану (карта світу, конфігурація). Потоки читають/пишуть shared structure, а не обмінюються повідомленнями. Часто комбінуються: channels для events, shared state для cache.

---

## Резюме

Channels — альтернатива shared state для міжпотокової комунікації. Замість спільного стану з Mutex — передача ownership повідомлень. Жодних locks, жодного contention, жодних deadlocks.

`mpsc::channel()` — необмежений канал. Sender::send() переміщує ownership. Receiver::recv() блокує до повідомлення. for msg in rx — ітерація до закриття каналу.

Multiple producers — clone Sender. Single consumer — Receiver не клонується. Не забути drop(tx) для закриття каналу.

`sync_channel(N)` — обмежений буфер з backpressure. send блокує при повному буфері. Контролює пам'ять.

Enum як протокол: кожен варіант — тип повідомлення. Exhaustive match на receiver гарантує обробку всіх типів.

Вибір: channels для producer-consumer та events. Arc\<Mutex\<T\>\> для shared state та cache. Часто комбінують обидва.

---

## Що далі

Частина IV завершена. Ваш агент еволюціонував від однопотокового об'єкта до елемента паралельного рою: closures як "мозок" потоку, Box для heap-розміщення, Rc/Arc для спільного володіння, Mutex/RwLock для синхронізації, channels для комунікації. Фундамент для паралельності побудований.

Частина V ("Асинхронність та масштабування") піднімає рівень: замість потоків — async/await та Tokio runtime, де тисячі задач обслуговуються кількома потоками. Агент стане асинхронним — виконуватиме IO (мережа, файли, сенсори) без блокування потоку. Рій масштабуватиметься до сотень агентів на одному сервері.
