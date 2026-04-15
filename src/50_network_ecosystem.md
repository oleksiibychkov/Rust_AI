# Розділ 50. Мережева комунікація та екосистема

## Анотація

До цього моменту всі агенти рою працювали в одному процесі — комунікація через in-process channels. Реальний рій — це окремі пристрої (або процеси), з'єднані мережею. Агент на дроні #1 надсилає дані координатору на базовій станції через Wi-Fi або радіоканал. Координатор відповідає командами. Для цього потрібна мережева комунікація: TCP/UDP з'єднання, серіалізація повідомлень для передачі по мережі (serde → bytes → мережа → bytes → serde), та протокол (формат повідомлень, порядок обміну, обробка помилок).

Цей розділ — останній технічний розділ перед фінальним проєктом. Він показує, як перейти від in-process рою до розподіленого, та дає огляд екосистеми Rust для подальшого розвитку.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Створити TCP-сервер та клієнт через tokio::net.
2. Серіалізувати та десеріалізувати повідомлення для мережевої передачі.
3. Реалізувати базовий протокол: length-prefixed messages.
4. Побудувати розподілений рій: агенти та координатор у різних процесах.
5. Орієнтуватись в екосистемі Rust: ключові крейти для різних доменів.

---

## Ключові терміни

**TCP (Transmission Control Protocol)** — протокол надійної передачі даних. Гарантує доставку та порядок. Основа для більшості мережевих протоколів.

**tokio::net** — async мережевий модуль Tokio: TcpListener (сервер), TcpStream (з'єднання).

**Length-prefixed protocol** — формат повідомлень, де перші N байтів — довжина, потім — тіло повідомлення. Дозволяє розділяти потік байтів на окремі повідомлення.

**Framing** — процес розділення безперервного TCP-потоку на окремі повідомлення (фрейми).

---

## Мотиваційний кейс

Координатор рою працює на сервері. 10 агентів — на 10 БПЛА, кожен зі своїм комп'ютером. Комунікація — через Wi-Fi. Кожен агент підключається до координатора по TCP, надсилає звіти (серіалізовані AgentReport), отримує команди (серіалізовані Command). Ті самі типи повідомлень, що й у Розділі 45 — але замість mpsc channel — TCP-з'єднання. Serde серіалізує у JSON (або bincode), tokio::net передає по мережі.

---

## 50.1. TCP з Tokio: сервер та клієнт

TCP-сервер слухає підключення, для кожного — async task:

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Сервер слухає на :8080");
    
    loop {
        let (mut socket, addr) = listener.accept().await?;
        println!("Нове з'єднання: {}", addr);
        
        tokio::spawn(async move {
            let mut buf = vec![0u8; 1024];
            loop {
                let n = match socket.read(&mut buf).await {
                    Ok(0) => { println!("{} від'єднався", addr); return; }
                    Ok(n) => n,
                    Err(e) => { println!("Помилка: {}", e); return; }
                };
                
                let msg = String::from_utf8_lossy(&buf[..n]);
                println!("[{}] {}", addr, msg);
                
                socket.write_all(b"OK\n").await.unwrap();
            }
        });
    }
}
```

TCP-клієнт:

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncWriteExt, AsyncReadExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
    println!("Підключено до координатора");
    
    stream.write_all(b"SCOUT-01: position (10, 20)").await?;
    
    let mut buf = vec![0u8; 1024];
    let n = stream.read(&mut buf).await?;
    println!("Відповідь: {}", String::from_utf8_lossy(&buf[..n]));
    
    Ok(())
}
```

TcpListener::bind — створює серверний сокет. accept() — чекає нове з'єднання (async). Для кожного — tokio::spawn з async task. TcpStream — з'єднання між клієнтом та сервером. read/write — async IO.

Проблема цього простого підходу: TCP — потік байтів, не потік повідомлень. Якщо клієнт надіслав два повідомлення швидко — вони можуть прийти в одному read як один блок. Або одне довге повідомлення може бути розбите на два read. Потрібен framing.

---

## 50.2. Framing: length-prefixed messages

Стандартний підхід: перед кожним повідомленням — 4 байти з його довжиною (u32 big-endian):

```
[4 bytes: length][N bytes: message body]
[4 bytes: length][N bytes: message body]
...
```

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

async fn send_message(stream: &mut TcpStream, data: &[u8]) -> tokio::io::Result<()> {
    let len = (data.len() as u32).to_be_bytes();
    stream.write_all(&len).await?;     // 4 байти довжини
    stream.write_all(data).await?;     // тіло
    Ok(())
}

async fn recv_message(stream: &mut TcpStream) -> tokio::io::Result<Vec<u8>> {
    let mut len_buf = [0u8; 4];
    stream.read_exact(&mut len_buf).await?;   // читаємо рівно 4 байти
    let len = u32::from_be_bytes(len_buf) as usize;
    
    let mut body = vec![0u8; len];
    stream.read_exact(&mut body).await?;      // читаємо рівно len байтів
    Ok(body)
}
```

read_exact — читає рівно N байтів, чекає доки всі не прийдуть. Це гарантує: кожен recv_message повертає рівно одне повне повідомлення, незалежно від того, як TCP розбив байти.

---

## 50.3. Серіалізація повідомлень для мережі

Комбінуємо framing з serde для передачі Rust-структур:

```rust
use serde::{Serialize, Deserialize};
use tokio::net::TcpStream;

#[derive(Debug, Serialize, Deserialize)]
enum AgentMessage {
    Report { agent_id: String, x: f64, y: f64, battery: u8 },
    Detection { agent_id: String, object_id: String },
    Complete { agent_id: String },
}

async fn send_typed<T: Serialize>(stream: &mut TcpStream, msg: &T) -> Result<(), Box<dyn std::error::Error>> {
    let json = serde_json::to_vec(msg)?;
    send_message(stream, &json).await?;
    Ok(())
}

async fn recv_typed<T: for<'de> Deserialize<'de>>(stream: &mut TcpStream) -> Result<T, Box<dyn std::error::Error>> {
    let data = recv_message(stream).await?;
    let msg: T = serde_json::from_slice(&data)?;
    Ok(msg)
}
```

`send_typed` серіалізує Rust-структуру → JSON bytes → length-prefix → TCP. `recv_typed` — зворотно: TCP → length-prefix → JSON bytes → Rust-структура. Типобезпечна мережева комунікація: відправляєте AgentMessage — отримуєте AgentMessage. Якщо формат не збігається — десеріалізація поверне Err замість пошкоджених даних.

Для production замість JSON (текстовий, повільний) краще bincode (бінарний, швидкий):

```rust
let bytes = bincode::serialize(msg)?;   // замість serde_json::to_vec
let msg: T = bincode::deserialize(&data)?; // замість serde_json::from_slice
```

---

## 50.4. Розподілений рій: координатор та агенти

Координатор (сервер):

```rust
use tokio::net::TcpListener;
use tokio::sync::mpsc;
use tracing::{info, warn};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt::init();
    
    let listener = TcpListener::bind("0.0.0.0:9000").await?;
    info!("Координатор на :9000");
    
    let (tx, mut rx) = mpsc::channel::<AgentMessage>(256);
    
    // Task для обробки повідомлень
    tokio::spawn(async move {
        let mut total_objects = 0;
        while let Some(msg) = rx.recv().await {
            match msg {
                AgentMessage::Detection { agent_id, object_id } => {
                    total_objects += 1;
                    info!(agent = %agent_id, object = %object_id, total = total_objects, "Виявлення");
                }
                AgentMessage::Complete { agent_id } => {
                    info!(agent = %agent_id, "Агент завершив");
                }
                _ => {}
            }
        }
    });
    
    // Приймаємо з'єднання
    loop {
        let (mut socket, addr) = listener.accept().await?;
        let tx = tx.clone();
        info!(%addr, "Агент підключився");
        
        tokio::spawn(async move {
            loop {
                match recv_typed::<AgentMessage>(&mut socket).await {
                    Ok(msg) => { let _ = tx.send(msg).await; }
                    Err(_) => { info!(%addr, "Від'єднано"); break; }
                }
            }
        });
    }
}
```

Агент (клієнт):

```rust
use tokio::net::TcpStream;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent_id = std::env::args().nth(1).unwrap_or(String::from("AGENT-0"));
    
    let mut stream = TcpStream::connect("127.0.0.1:9000").await?;
    println!("[{}] Підключено до координатора", agent_id);
    
    for step in 0..5 {
        sleep(Duration::from_millis(200)).await;
        
        let msg = AgentMessage::Report {
            agent_id: agent_id.clone(),
            x: step as f64 * 10.0,
            y: step as f64 * 5.0,
            battery: 100 - step * 15,
        };
        send_typed(&mut stream, &msg).await?;
        
        if step == 3 {
            let det = AgentMessage::Detection {
                agent_id: agent_id.clone(),
                object_id: format!("OBJ-{}", agent_id),
            };
            send_typed(&mut stream, &det).await?;
        }
    }
    
    send_typed(&mut stream, &AgentMessage::Complete { agent_id }).await?;
    Ok(())
}
```

Координатор та агент — окремі бінарники (або процеси). Комунікація через TCP із серіалізацією serde. Ті самі enum AgentMessage, що й в in-process версії. Перехід від channels до мережі — зміна транспорту, не логіки.

---

## 50.5. Екосистема Rust: крейти для подальшого розвитку

Rust має багату екосистему крейтів. Огляд ключових напрямків для БПЛА та мультиагентних систем:

Мережа та веб: `tokio` (async runtime), `hyper` (HTTP), `reqwest` (HTTP-клієнт), `tonic` (gRPC), `warp`/`axum` (веб-фреймворки). Для координатора рою з HTTP API — axum + serde.

Серіалізація: `serde` (фреймворк), `serde_json`, `toml`, `bincode` (бінарний), `protobuf`/`prost` (Protocol Buffers), `flatbuffers` (zero-copy серіалізація).

Конкурентність: `tokio` (async), `rayon` (data parallelism), `crossbeam` (lock-free структури, scoped threads), `dashmap` (concurrent HashMap).

Embedded та робототехніка: `embedded-hal` (Hardware Abstraction Layer), `rppal` (Raspberry Pi GPIO), `serialport` (UART/serial), `ros2-client` (ROS2 інтеграція).

Математика та ML: `nalgebra` (лінійна алгебра), `ndarray` (N-dimensional масиви), `burn` (ML фреймворк на Rust), `candle` (ML від Hugging Face).

CLI та інструменти: `clap` (парсинг аргументів), `dialoguer` (інтерактивний CLI), `indicatif` (прогрес-бари).

Тестування: `criterion` (бенчмарки), `proptest` (property-based testing), `mockall` (mock-об'єкти), `wiremock` (HTTP mock).

Якість коду: `clippy` (лінтер), `rustfmt` (форматування), `cargo-audit` (вразливості у залежностях), `cargo-deny` (policy enforcement).

---

## 50.6. Ресурси для подальшого навчання

Книги: "The Rust Programming Language" (офіційна, безкоштовна), "Rust in Action" (Tim McNamara), "Programming Rust" (O'Reilly), "Rust for Rustaceans" (Jon Gjengset, просунутий рівень).

Онлайн: docs.rs (документація крейтів), lib.rs (пошук крейтів), This Week in Rust (щотижневий дайджест), Rust Blog (офіційний блог), Rust Playground (play.rust-lang.org).

Відео: Jon Gjengset YouTube (глибокі технічні стріми), Rust Conf записи, Let's Get Rusty (навчальний канал).

Спільноти: Rust Users Forum, Rust Discord, Reddit r/rust, Telegram Rust UA.

Практика: Exercism (Rust track), Advent of Code, rustlings (інтерактивні вправи).

---

## 50.7. Prompt Engineering: AI-інструменти для Rust

Claude Code — CLI-інструмент для роботи з кодовою базою через AI. Розуміє структуру проєкту, може редагувати файли, запускати тести.

GitHub Copilot — автодоповнення коду в IDE. Ефективний для boilerplate, менш — для складної логіки.

Cursor — IDE з вбудованим AI. Розуміє контекст проєкту, може рефакторити та пояснювати код.

AI у CI/CD: автоматичний code review через AI (PR review bots), генерація changelog, аналіз покриття тестами.

Правило: AI-інструменти найефективніші у комбінації. Copilot для автодоповнення під час написання. Claude для архітектурних рішень та дебагу. CI/CD AI для review. Жоден не замінює розуміння коду програмістом.

---

## Лабораторна робота No50

### Мета

Побудувати розподілений рій з мережевою комунікацією.

### Завдання базового рівня

1. TCP-сервер (координатор): приймає з'єднання, десеріалізує AgentMessage.
2. TCP-клієнт (агент): підключається, серіалізує та надсилає звіти.
3. Length-prefixed framing для розділення повідомлень.
4. Запуск: 1 координатор + 3 агенти (окремі процеси або термінали).

### Варіанти

**A.** Двонаправлена комунікація: координатор надсилає Command агентам через те саме TCP-з'єднання.

**B.** UDP для telemetry: агенти надсилають часті показання через UDP (без гарантії доставки, але швидше), команди — через TCP.

**C.** TLS: додати шифрування через tokio-rustls для захисту комунікації.

### Критерії

| Критерій | Бал |
|----------|-----|
| TCP сервер + клієнт на Tokio | 25 |
| Серіалізація serde по мережі | 20 |
| Length-prefixed framing | 20 |
| Кілька агентів одночасно | 15 |
| Обробка disconnect | 10 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `Connection refused`

Сервер не запущений або слухає інший порт/адресу. Перевірте: сервер запущений, порт збігається, "0.0.0.0" (всі інтерфейси) а не "127.0.0.1" (лише localhost).

### Помилка 2: повідомлення "склеюються" або "розрізаються"

TCP — потік байтів, не повідомлень. Два send поспіль можуть прийти як один read.

Виправлення: length-prefixed framing з read_exact (секція 50.2).

### Помилка 3: десеріалізація провалюється

JSON обрізаний (прочитали не все повідомлення) або зіпсований.

Виправлення: framing гарантує повноту повідомлення. Якщо проблема залишається — перевірте encoding (UTF-8) та версію serde-структур на сервері та клієнті.

---

## Контрольні запитання

### Рівень 1

1. Чому TCP-потік потрібно "фреймувати"?

Відповідь: TCP передає потік байтів, не повідомлення. Два send можуть прийти як один read. Length-prefixed framing додає 4-байтну довжину перед кожним повідомленням — receiver знає, скільки байтів читати для одного повідомлення.

2. Як Rust-структура передається по мережі?

Відповідь: серіалізація (serde → JSON/bincode bytes) → framing (length prefix) → TCP send. На іншому боці: TCP recv → deframing → десеріалізація (bytes → serde → Rust struct).

### Рівень 2

3. Чому bincode кращий за JSON для мережевої комунікації рою?

Відповідь: bincode — бінарний, компактніший (менше байтів), швидший (немає парсингу тексту). JSON — текстовий, читабельний (зручно для дебагу), але більший та повільніший. Для production рою з тисячами повідомлень/с — bincode. Для дебагу та розробки — JSON.

### Рівень 4

4. Як масштабувати координатор для 1000 агентів?

Відповідь: один tokio::spawn per connection — 1000 tasks, прийнятно для Tokio. Bottleneck — обробка повідомлень: mpsc канал із async обробником. Якщо один обробник не встигає — шардування по зонах (кілька обробників). Load balancing — кілька координаторів за балансувальником. Для 10000+ — розглянути UDP для telemetry та message broker (NATS, RabbitMQ) замість прямих TCP.

---

## Резюме

tokio::net надає async TCP (TcpListener, TcpStream). Length-prefixed framing розділяє потік на повідомлення. serde серіалізує Rust-структури для передачі (JSON для дебагу, bincode для production).

Розподілений рій: ті самі типи повідомлень, що й in-process, але транспорт — TCP замість channels. Зміна транспорту — не зміна логіки.

Екосистема Rust: сотні крейтів для мережі, серіалізації, конкурентності, embedded, ML, CLI, тестування. crates.io та lib.rs — пошук. docs.rs — документація.

---

## Що далі

Частина VI завершена. Ви маєте повний інструментарій: від базового синтаксису до макросів, від ownership до async, від одного агента до розподіленого рою з мережевою комунікацією.

Частина VII — фінальний проєкт. Розділ 51 формулює вимоги до автономного рою БПЛА: актори, канали, серіалізація, тести, tracing, документація. Розділ 52 — рекомендації до захисту, критерії оцінювання, та завершальні слова курсу. Це ваша можливість інтегрувати все, що вивчили, у production-ready систему.
