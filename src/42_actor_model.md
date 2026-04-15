# Розділ 42. Actor Model: концепція та реалізація

## Анотація

У попередніх розділах ви використовували різні примітиви конкурентності: spawn, channels, Mutex, Semaphore. Кожен вирішує конкретну задачу, але складна система з десятками компонентів потребує архітектурного патерну — способу організувати взаємодію компонентів. Actor Model — один із найуспішніших таких патернів. Кожен компонент — актор: ізольована async task із власним станом та mailbox (mpsc channel). Актори не ділять стан — вони комунікують лише через повідомлення. Це усуває data races архітектурно (немає спільної пам'яті) та робить систему модульною: додати нового актора — створити task з channel, не змінюючи існуючих.

Erlang побудував на Actor Model телекомунікаційні системи з uptimе 99.9999999% (дев'ять дев'яток). Akka (Scala/Java) використовується у LinkedIn, PayPal, Walmart. Rust не має Actor Model у стандартній бібліотеці, але Tokio дає всі будівельні блоки: spawned task = актор, mpsc channel = mailbox, Sender = handle. Цей розділ показує, як побудувати акторну систему вручну — без зовнішніх фреймворків, лише з Tokio.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити Actor Model: актор, mailbox, handle, повідомлення.
2. Реалізувати актора як async task з mpsc receiver loop.
3. Реалізувати handle як struct із Sender для відправки повідомлень актору.
4. Використати oneshot для request-response між акторами.
5. Реалізувати graceful shutdown для акторної системи.
6. Побудувати рій БПЛА як систему акторів.

---

## Ключові терміни

**Actor (актор)** — ізольована одиниця обчислення зі своїм станом. Приймає повідомлення, обробляє їх по одному, може відправляти повідомлення іншим акторам.

**Mailbox (поштова скринька)** — черга вхідних повідомлень актора. У Rust — mpsc::Receiver.

**Handle (дескриптор)** — інтерфейс для відправки повідомлень актору. У Rust — struct із mpsc::Sender. Можна клонувати для кількох відправників.

**Message loop (цикл обробки)** — нескінченний цикл, де актор отримує повідомлення з mailbox та обробляє кожне.

**Graceful shutdown** — контрольоване завершення: актор завершує поточну роботу, зберігає стан, закриває ресурси.

---

## Мотиваційний кейс

Без архітектурного патерну код координатора рою перетворюється на "великий switch": одна async fn з select!, де 15 гілок обробляють різні типи повідомлень, зі спільними мутабельними змінними. Додавання нового типу агента вимагає зміни цієї монолітної функції. Тестування — нічний кошмар: потрібно підняти всю систему для перевірки одного компонента.

Actor Model розбиває монолійт на ізольовані одиниці. Координатор — актор зі своїм mailbox та станом. Кожен агент — актор. Логер — актор. Кожен тестується окремо: створити актора, відправити повідомлення через handle, перевірити відповідь. Додати нового актора — створити task і channel, не чіпаючи існуючих.

---

## 42.1. Actor Model: три правила

Actor Model визначає три простих правила. Актор може:

1. Обробити повідомлення з mailbox.
2. Створити нових акторів.
3. Відправити повідомлення іншим акторам.

І нічого більше. Актор не може напряму читати стан іншого актора. Не може модифікувати чужий стан. Не може ділити мутабельну пам'ять. Єдиний спосіб взаємодії — повідомлення.

Наслідки цих правил. Ізоляція стану: стан актора — приватний, доступний лише йому. Жоден інший актор не може його зіпсувати. Data race неможливий — немає спільних даних. Послідовна обробка: актор обробляє повідомлення по одному. Жодних конкурентних доступів до внутрішнього стану — не потрібен Mutex. Location transparency: актор не знає, де фізично знаходиться інший актор — у тому ж процесі, на іншій машині, у хмарі. Спілкування через повідомлення працює однаково.

Для БПЛА-рою це природна модель. Кожен дрон — автономний актор зі своїм станом (позиція, батарея, завдання). Координатор — актор, що приймає звіти та надсилає команди. Логер — актор, що приймає повідомлення та записує у файл. Кожен працює незалежно, комунікує через повідомлення.

---

## 42.2. Анатомія актора у Rust

Актор складається з трьох частин:

Стан — структура з полями, що актор зберігає між повідомленнями. Приватний, не ділиться.

Mailbox — `mpsc::Receiver<Message>`, звідки актор отримує повідомлення.

Handle — структура з `mpsc::Sender<Message>`, через яку зовнішній код відправляє повідомлення.

```rust
use tokio::sync::{mpsc, oneshot};

// --- Повідомлення ---
enum CoordinatorMessage {
    AgentReport { agent_id: String, objects_found: u32 },
    GetStatus { reply: oneshot::Sender<CoordinatorStatus> },
    Shutdown,
}

#[derive(Debug, Clone)]
struct CoordinatorStatus {
    total_objects: u32,
    active_agents: usize,
}

// --- Актор ---
struct CoordinatorActor {
    receiver: mpsc::Receiver<CoordinatorMessage>,
    total_objects: u32,
    active_agents: Vec<String>,
}

impl CoordinatorActor {
    fn new(receiver: mpsc::Receiver<CoordinatorMessage>) -> Self {
        CoordinatorActor {
            receiver,
            total_objects: 0,
            active_agents: Vec::new(),
        }
    }
    
    async fn run(mut self) {
        println!("[Coordinator] Запущено");
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                CoordinatorMessage::AgentReport { agent_id, objects_found } => {
                    self.total_objects += objects_found;
                    if !self.active_agents.contains(&agent_id) {
                        self.active_agents.push(agent_id.clone());
                    }
                    println!("[Coordinator] Звіт від {}: +{} об'єктів (всього: {})",
                             agent_id, objects_found, self.total_objects);
                }
                CoordinatorMessage::GetStatus { reply } => {
                    let status = CoordinatorStatus {
                        total_objects: self.total_objects,
                        active_agents: self.active_agents.len(),
                    };
                    let _ = reply.send(status);
                }
                CoordinatorMessage::Shutdown => {
                    println!("[Coordinator] Завершення. Об'єктів: {}", self.total_objects);
                    break;
                }
            }
        }
    }
}

// --- Handle ---
#[derive(Clone)]
struct CoordinatorHandle {
    sender: mpsc::Sender<CoordinatorMessage>,
}

impl CoordinatorHandle {
    fn new(buffer: usize) -> (Self, CoordinatorActor) {
        let (sender, receiver) = mpsc::channel(buffer);
        let handle = CoordinatorHandle { sender };
        let actor = CoordinatorActor::new(receiver);
        (handle, actor)
    }
    
    async fn report(&self, agent_id: &str, objects: u32) {
        let _ = self.sender.send(CoordinatorMessage::AgentReport {
            agent_id: agent_id.to_string(),
            objects_found: objects,
        }).await;
    }
    
    async fn get_status(&self) -> CoordinatorStatus {
        let (tx, rx) = oneshot::channel();
        let _ = self.sender.send(CoordinatorMessage::GetStatus { reply: tx }).await;
        rx.await.unwrap()
    }
    
    async fn shutdown(&self) {
        let _ = self.sender.send(CoordinatorMessage::Shutdown).await;
    }
}
```

Розберемо патерн. `CoordinatorActor` містить стан (total_objects, active_agents) та receiver (mailbox). Метод `run` — нескінченний цикл, що читає повідомлення з mailbox та обробляє кожне через match. Стан модифікується лише всередині run — жоден зовнішній код не має доступу.

`CoordinatorHandle` містить sender. Методи handle — зручний API для відправки повідомлень: `report()` обгортає дані у CoordinatorMessage::AgentReport, `get_status()` використовує oneshot для request-response, `shutdown()` надсилає сигнал завершення. Handle можна clone — кілька компонентів можуть відправляти повідомлення одному актору.

`new()` створює і handle, і actor. Caller запускає actor через tokio::spawn і використовує handle:

```rust
#[tokio::main]
async fn main() {
    let (handle, actor) = CoordinatorHandle::new(32);
    tokio::spawn(actor.run()); // запускаємо актора
    
    // Відправляємо повідомлення через handle
    handle.report("SCOUT-01", 3).await;
    handle.report("RECON-02", 5).await;
    
    let status = handle.get_status().await;
    println!("Статус: {:?}", status);
    
    handle.shutdown().await;
    tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
}
```

Зовнішній код бачить лише handle. Не знає про внутрішній стан актора, не має доступу до receiver, не може порушити інваріанти. Це інкапсуляція на рівні архітектури.

---

## 42.3. Агент як актор

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

enum AgentMessage {
    ExecuteStep,
    ChangeTarget(f64, f64),
    Shutdown,
}

struct AgentActor {
    id: String,
    receiver: mpsc::Receiver<AgentMessage>,
    coordinator: CoordinatorHandle,
    position: (f64, f64),
    target: (f64, f64),
    battery: u8,
}

impl AgentActor {
    async fn run(mut self) {
        println!("[{}] Запущено", self.id);
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                AgentMessage::ExecuteStep => {
                    if self.battery < 10 { continue; }
                    
                    // Рухаємось до цілі
                    let dx = (self.target.0 - self.position.0).signum();
                    let dy = (self.target.1 - self.position.1).signum();
                    self.position.0 += dx;
                    self.position.1 += dy;
                    self.battery = self.battery.saturating_sub(2);
                    
                    // Симуляція виявлення
                    let found = if self.battery % 7 == 0 { 1 } else { 0 };
                    if found > 0 {
                        self.coordinator.report(&self.id, found).await;
                    }
                }
                AgentMessage::ChangeTarget(x, y) => {
                    println!("[{}] Нова ціль: ({:.0}, {:.0})", self.id, x, y);
                    self.target = (x, y);
                }
                AgentMessage::Shutdown => {
                    println!("[{}] Завершення (bat={}%)", self.id, self.battery);
                    break;
                }
            }
        }
    }
}

#[derive(Clone)]
struct AgentHandle {
    sender: mpsc::Sender<AgentMessage>,
}

impl AgentHandle {
    fn new(id: &str, coordinator: CoordinatorHandle) -> (Self, AgentActor) {
        let (sender, receiver) = mpsc::channel(32);
        let actor = AgentActor {
            id: id.to_string(),
            receiver,
            coordinator,
            position: (0.0, 0.0),
            target: (50.0, 50.0),
            battery: 100,
        };
        (AgentHandle { sender }, actor)
    }
    
    async fn step(&self) {
        let _ = self.sender.send(AgentMessage::ExecuteStep).await;
    }
    
    async fn change_target(&self, x: f64, y: f64) {
        let _ = self.sender.send(AgentMessage::ChangeTarget(x, y)).await;
    }
    
    async fn shutdown(&self) {
        let _ = self.sender.send(AgentMessage::Shutdown).await;
    }
}
```

Агент-актор має handle на координатора — відправляє звіти через нього. Координатор не знає про внутрішній стан агента. Агент не знає про стан координатора. Комунікація — лише через повідомлення.

---

## 42.4. Збираємо рій: повна акторна система

```rust
#[tokio::main]
async fn main() {
    // Створюємо координатора
    let (coord_handle, coord_actor) = CoordinatorHandle::new(256);
    tokio::spawn(coord_actor.run());
    
    // Створюємо агентів
    let mut agent_handles = Vec::new();
    for i in 0..4 {
        let id = format!("AGENT-{}", i);
        let (handle, actor) = AgentHandle::new(&id, coord_handle.clone());
        tokio::spawn(actor.run());
        agent_handles.push(handle);
    }
    
    // Симуляція місії: 10 кроків
    for step in 0..10 {
        for handle in &agent_handles {
            handle.step().await;
        }
        sleep(Duration::from_millis(50)).await;
        
        if step == 5 {
            // Зміна цілі для всіх агентів
            for handle in &agent_handles {
                handle.change_target(80.0, 20.0).await;
            }
        }
    }
    
    // Запитуємо статус
    let status = coord_handle.get_status().await;
    println!("\nСтатус: {} об'єктів від {} агентів", 
             status.total_objects, status.active_agents);
    
    // Graceful shutdown
    for handle in &agent_handles {
        handle.shutdown().await;
    }
    coord_handle.shutdown().await;
    sleep(Duration::from_millis(100)).await;
}
```

Чотири актори-агенти + один координатор. Кожен — окрема task. Комунікація лише через handles. Main — "режисер", що керує місією через handles. Graceful shutdown — Shutdown повідомлення кожному актору.

---

## 42.5. Graceful shutdown: контрольоване завершення

У реальних системах важливо завершувати акторів коректно: зберегти стан, закрити з'єднання, повідомити залежних. Патерни:

Shutdown повідомлення (як вище) — явний варіант enum. Актор завершує цикл через break.

Drop handle — коли всі handles знищені, sender закривається, receiver.recv() повертає None, цикл while let Some завершується природно. Це "автоматичний" shutdown.

Cancellation token — `tokio_util::sync::CancellationToken` для координованого shutdown кількох акторів одночасно:

```rust
use tokio_util::sync::CancellationToken;

async fn actor_with_token(mut rx: mpsc::Receiver<String>, token: CancellationToken) {
    loop {
        tokio::select! {
            msg = rx.recv() => {
                match msg {
                    Some(m) => println!("Отримано: {}", m),
                    None => break,
                }
            }
            _ = token.cancelled() => {
                println!("Отримано сигнал shutdown");
                break;
            }
        }
    }
    println!("Актор завершив gracefully");
}
```

CancellationToken — один об'єкт, що сигналізує shutdown всім акторам одночасно. `token.cancel()` — сигнал. `token.cancelled().await` — очікування сигналу в select!.

---

## 42.6. Prompt Engineering: проєктування акторної системи

### Промпт-шаблон

```
Проєктую систему для рою БПЛА з такими компонентами:
- Координатор: приймає звіти, надсилає команди, веде статистику
- N агентів: сканують, рухаються, звітують
- Логер: записує всі події
- Моніторинг: перевіряє heartbeat кожного агента

Для кожного компонента спроєктуй:
1. enum Message (вхідні повідомлення)
2. struct ActorState (внутрішній стан)
3. struct Handle (методи для відправки)
4. Зв'язки між акторами (хто кому відправляє)
```

---

## Лабораторна робота No42

### Мета

Побудувати рій БПЛА як систему акторів.

### Завдання базового рівня

1. CoordinatorActor з handle, enum з 3+ варіантами, request-response через oneshot.
2. AgentActor з handle, зв'язок із координатором.
3. 4+ агенти + координатор, 10+ кроків симуляції.
4. Graceful shutdown через Shutdown повідомлення.

### Варіанти

**A.** LoggerActor: приймає LogMessage від координатора та агентів, записує у Vec (або файл). Handle з методами log_info, log_warning, log_error.

**B.** MonitorActor: перевіряє heartbeat агентів. Якщо агент не відповів за 5с — логує warning. Використовує interval + timeout.

**C.** Dynamic actors: координатор може створювати нових агентів під час місії (spawn actor + зберегти handle) та завершувати існуючих (shutdown).

### Критерії

| Критерій | Бал |
|----------|-----|
| Actor + Handle патерн для 2+ акторів | 25 |
| enum Message з 3+ варіантами | 15 |
| Request-response через oneshot | 15 |
| Graceful shutdown | 15 |
| Тести (відправити повідомлення → перевірити стан) | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: Actor task завершується одразу

Всі sender-и знищені до першого повідомлення → receiver.recv() повертає None → цикл завершується.

Виправлення: переконайтесь, що хоча б один handle живе поки актор потрібен. Зберігайте handles у Vec або структурі.

### Помилка 2: oneshot response ніколи не приходить

Актор не обробив повідомлення (помилка в match, паніка, або receiver перевантажений).

Виправлення: timeout на oneshot::Receiver: `timeout(Duration::from_secs(5), rx).await`.

### Помилка 3: handle.method().await зависає

mpsc канал повний (bounded channel) і актор не читає.

Виправлення: збільшити буфер каналу або перевірити, що актор працює (не завис у довгій операції).

---

## Додатково

### Actor Model vs Object-Oriented

В ООП об'єкт має методи, що викликаються синхронно та повертають результат одразу. Caller блокується до повернення. У Actor Model "виклик методу" — відправка повідомлення у mailbox. Caller не блокується (send повертає одразу). Результат приходить пізніше (через oneshot або callback).

Це фундаментальна різниця: ООП — синхронний call/return. Actors — асинхронний send/receive. Actors природно підходять для конкурентних систем, де компоненти працюють паралельно. ООП потребує додаткових механізмів (locks, monitors) для безпечної конкурентності.

### Коли Actor Model, а коли просто channels

Не кожна задача потребує повноцінних акторів. Проста система з 2–3 tasks і 1 каналом — не потребує abstractions handle/actor. Overhead не виправданий.

Actor Model окупається при: 5+ компонентах, складних взаємодіях (request-response, broadcast + collect), потребі у тестуванні компонентів окремо, потребі у graceful shutdown. Для простих producer-consumer — достатньо mpsc.

---

## Контрольні запитання

### Рівень 1

1. Що таке актор?

Відповідь: ізольована одиниця обчислення зі своїм станом та mailbox. Приймає повідомлення, обробляє по одному, може відправляти повідомлення іншим акторам. Не ділить стан.

2. Навіщо handle, якщо можна використати Sender напряму?

Відповідь: handle надає типізований API: `handle.report("id", 5)` замість `sender.send(Message::Report { ... })`. Ховає деталі enum від caller-а. Може містити додаткову логіку (валідація, retry, timeout).

### Рівень 2

3. Чому актор обробляє повідомлення по одному, а не паралельно?

Відповідь: послідовна обробка усуває потребу у Mutex для внутрішнього стану. Актор — єдиний, хто змінює свій стан. Жодних data races, жодних lock-ів. Паралельна обробка повідомлень потребувала б синхронізації — це ускладнило б модель без суттєвих переваг (більшість акторів не bottleneck).

4. Як реалізувати request-response в Actor Model?

Відповідь: caller створює oneshot::channel(), надсилає Sender у повідомленні актору, чекає Receiver. Актор отримує повідомлення, обчислює відповідь, відправляє через oneshot Sender. Кожен запит — окремий oneshot.

### Рівень 3

5. Спроєктуйте enum Message для актора-логера.

Відповідь:

```rust
enum LogMessage {
    Info { source: String, text: String },
    Warning { source: String, text: String },
    Error { source: String, text: String },
    Flush,
    GetStats { reply: oneshot::Sender<LogStats> },
    Shutdown,
}
```

### Рівень 4

6. Порівняйте Actor Model із Arc\<Mutex\<SharedState\>\> для координатора рою.

Відповідь: Arc<Mutex> — спільний стан, кожен потік/task блокує mutex для доступу. Contention при багатьох writers. Стан "розмазаний" по всій програмі — хто завгодно може заблокувати та модифікувати. Actor — стан ізольований в одній task, доступ лише через повідомлення. Жодного contention — повідомлення обробляються послідовно. Стан інкапсульований — лише актор знає про нього. Actor краще для складних систем із багатьма компонентами. Mutex — для простого shared counter чи cache.

---

## Резюме

Actor Model: кожен компонент — ізольований актор зі станом та mailbox. Комунікація лише через повідомлення. Жодного спільного стану → жодних data races → жодних locks.

Реалізація у Rust: actor = struct + async fn run() з mpsc::Receiver loop. Handle = struct із mpsc::Sender + типізовані методи. new() створює handle + actor. tokio::spawn(actor.run()) запускає.

Request-response: oneshot::channel() у повідомленні. Graceful shutdown: Shutdown варіант enum або CancellationToken.

Actor окупається при 5+ компонентах, складних взаємодіях, потребі у тестуванні та graceful shutdown.

---

## Що далі

Акторна система дає архітектуру для IO-bound конкурентності. Але деякі задачі — CPU-bound: обчислення маршрутів для 100 агентів, обробка зображень з камер, аналіз великих наборів даних. Async/Tokio для цього не підходить — тривале обчислення блокує runtime. Розділ 43 представляє Rayon — бібліотеку паралелізму даних: `par_iter()` розподіляє обчислення по ядрах процесора з мінімальним зусиллям. Одна зміна `.iter()` → `.par_iter()` — і код працює паралельно.
