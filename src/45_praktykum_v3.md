# Розділ 45. Практикум: Асинхронний рій v3.0

## Анотація

Це фінальний розділ Частини V та кульмінація підручника у частині практичного програмування. У Розділах 39–44 ви вивчили: async/await та futures (39), Tokio runtime (40), async channels та streams (41), Actor Model (42), Rayon для CPU-bound (43), tracing для логування (44). Кожна тема — окремий інструмент. Тепер час об'єднати їх у повноцінну систему: асинхронний рій з 10+ агентів-акторів, координатором, логером, паралельними обчисленнями та структурованим логуванням.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Спроєктувати архітектуру асинхронної мультиагентної системи.
2. Реалізувати акторів (координатор, агенти, логер) із відповідними каналами.
3. Інтегрувати Tokio (IO), Rayon (CPU), tracing (логування) в одному проєкті.
4. Провести AI-assisted проєктування та code review.

---

## 45.1. Що змінюється: v2.0 → v3.0

| Аспект | v2.0 (Розділ 30) | v3.0 (цей розділ) |
|--------|------------------|-------------------|
| Виконання | Однопотоковий | Async (Tokio) + CPU (Rayon) |
| Архітектура | Struct + methods | Actor Model |
| Комунікація | Прямі виклики | Channels (mpsc, broadcast, watch) |
| Агенти | Послідовно | Конкурентно (async tasks) |
| Обчислення | iter() | par_iter() для CPU-bound |
| Логування | println | tracing зі spans |
| Shutdown | drop | Graceful (CancellationToken) |
| Масштаб | 1–4 агенти | 10+ агентів |

---

## 45.2. Архітектура v3.0

```
┌─────────────────────────────────────────────┐
│                   main()                     │
│  - Ініціалізація tracing                     │
│  - Створення акторів                         │
│  - Запуск місії                              │
│  - Graceful shutdown                         │
└────┬──────────┬──────────┬──────────────────┘
     │          │          │
     ▼          ▼          ▼
┌─────────┐ ┌──────┐ ┌──────────────────┐
│Coordinator│ │Logger│ │  Agent-0..N      │
│  Actor   │ │Actor │ │  Actors          │
├─────────┤ ├──────┤ ├──────────────────┤
│ mpsc ◄──┼─┤ mpsc │ │ broadcast ◄──────┤ commands
│ (reports)│ │(logs)│ │ mpsc ──────────► │ reports
│ watch ──►│ │      │ │ watch ◄────────  │ config
│ (config) │ └──────┘ └──────────────────┘
│ broadcast│
│ (cmds)──►│
└─────────┘
```

Чотири типи каналів:
- mpsc: агенти → координатор (звіти), всі → логер (логи)
- broadcast: координатор → агенти (команди)
- watch: координатор → агенти (поточна конфігурація місії)
- oneshot: координатор ← агент (request-response для статусу)

---

## 45.3. Типи повідомлень

```rust
use serde::{Serialize, Deserialize};
use tokio::sync::oneshot;

// Звіти від агентів до координатора
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AgentReport {
    Position { agent_id: String, x: f64, y: f64 },
    Detection { agent_id: String, object_id: String, x: f64, y: f64 },
    StatusUpdate { agent_id: String, battery: u8, state: String },
    MissionComplete { agent_id: String, steps: u32, objects_found: u32 },
}

// Команди від координатора до агентів
#[derive(Debug, Clone)]
pub enum Command {
    ChangeTarget { x: f64, y: f64 },
    ChangeAltitude { meters: f64 },
    ReturnToBase,
    Shutdown,
}

// Конфігурація місії (через watch)
#[derive(Debug, Clone)]
pub struct MissionConfig {
    pub target_zone: (f64, f64, f64, f64), // min_x, min_y, max_x, max_y
    pub max_agents: usize,
    pub scan_radius: f64,
}

// Повідомлення координатору (включаючи request-response)
pub enum CoordinatorMessage {
    Report(AgentReport),
    GetStatus { reply: oneshot::Sender<MissionStatus> },
}

#[derive(Debug, Clone)]
pub struct MissionStatus {
    pub active_agents: usize,
    pub total_objects: u32,
    pub completed: u32,
}
```

AgentReport derive Serialize/Deserialize (Розділ 35) — ті самі повідомлення можна зберегти у файл або передати по мережі.

---

## 45.4. Координатор-актор

```rust
use tokio::sync::{mpsc, broadcast, watch, oneshot};
use tracing::{info, warn, instrument, span, Level};
use std::collections::HashMap;

pub struct CoordinatorActor {
    reports_rx: mpsc::Receiver<CoordinatorMessage>,
    commands_tx: broadcast::Sender<Command>,
    config_tx: watch::Sender<MissionConfig>,
    // Стан
    agent_positions: HashMap<String, (f64, f64)>,
    total_objects: u32,
    completed_agents: u32,
    expected_agents: u32,
}

impl CoordinatorActor {
    #[instrument(name = "coordinator", skip(self))]
    pub async fn run(mut self) {
        info!("Координатор запущено");
        
        while let Some(msg) = self.reports_rx.recv().await {
            match msg {
                CoordinatorMessage::Report(report) => {
                    self.handle_report(report);
                }
                CoordinatorMessage::GetStatus { reply } => {
                    let status = MissionStatus {
                        active_agents: self.agent_positions.len(),
                        total_objects: self.total_objects,
                        completed: self.completed_agents,
                    };
                    let _ = reply.send(status);
                }
            }
            
            // Перевірка завершення
            if self.completed_agents >= self.expected_agents {
                info!(total_objects = self.total_objects, "Всі агенти завершили");
                let _ = self.commands_tx.send(Command::Shutdown);
                break;
            }
        }
        
        info!("Координатор завершив");
    }
    
    fn handle_report(&mut self, report: AgentReport) {
        match report {
            AgentReport::Position { agent_id, x, y } => {
                self.agent_positions.insert(agent_id, (x, y));
            }
            AgentReport::Detection { agent_id, object_id, x, y } => {
                self.total_objects += 1;
                info!(agent = %agent_id, object = %object_id, x, y, "Об'єкт виявлено");
            }
            AgentReport::StatusUpdate { agent_id, battery, state } => {
                if battery < 20 {
                    warn!(agent = %agent_id, battery, "Низька батарея");
                }
            }
            AgentReport::MissionComplete { agent_id, steps, objects_found } => {
                self.completed_agents += 1;
                info!(
                    agent = %agent_id, steps, objects_found,
                    completed = self.completed_agents,
                    total = self.expected_agents,
                    "Агент завершив місію"
                );
            }
        }
    }
}

// Handle
#[derive(Clone)]
pub struct CoordinatorHandle {
    reports_tx: mpsc::Sender<CoordinatorMessage>,
    pub commands_rx_factory: broadcast::Sender<Command>,
    pub config_rx: watch::Receiver<MissionConfig>,
}

impl CoordinatorHandle {
    pub async fn report(&self, report: AgentReport) {
        let _ = self.reports_tx.send(CoordinatorMessage::Report(report)).await;
    }
    
    pub async fn get_status(&self) -> Option<MissionStatus> {
        let (tx, rx) = oneshot::channel();
        self.reports_tx.send(CoordinatorMessage::GetStatus { reply: tx }).await.ok()?;
        rx.await.ok()
    }
}
```

---

## 45.5. Агент-актор

```rust
use tokio::sync::{mpsc, broadcast, watch};
use tokio::time::{sleep, interval, Duration};
use tracing::{info, warn, debug, instrument};

pub struct AgentActor {
    id: String,
    coordinator: mpsc::Sender<CoordinatorMessage>,
    commands: broadcast::Receiver<Command>,
    config: watch::Receiver<MissionConfig>,
    // Стан
    position: (f64, f64),
    battery: u8,
    target: (f64, f64),
}

impl AgentActor {
    pub fn new(
        id: &str,
        coordinator: mpsc::Sender<CoordinatorMessage>,
        commands: broadcast::Receiver<Command>,
        config: watch::Receiver<MissionConfig>,
    ) -> Self {
        AgentActor {
            id: id.to_string(),
            coordinator,
            commands,
            config,
            position: (0.0, 0.0),
            battery: 100,
            target: (50.0, 50.0),
        }
    }
    
    #[instrument(name = "agent", fields(id = %self.id), skip(self))]
    pub async fn run(mut self) {
        info!("Запущено");
        
        let mut step_timer = interval(Duration::from_millis(100));
        let mut steps = 0_u32;
        let mut objects_found = 0_u32;
        
        loop {
            tokio::select! {
                _ = step_timer.tick() => {
                    if self.battery < 5 {
                        warn!(battery = self.battery, "Батарея критична, завершую");
                        break;
                    }
                    
                    self.execute_step();
                    steps += 1;
                    
                    // Звіт позиції
                    let _ = self.coordinator.send(CoordinatorMessage::Report(
                        AgentReport::Position {
                            agent_id: self.id.clone(),
                            x: self.position.0,
                            y: self.position.1,
                        }
                    )).await;
                    
                    // Симуляція виявлення
                    if steps % 5 == 0 {
                        objects_found += 1;
                        let _ = self.coordinator.send(CoordinatorMessage::Report(
                            AgentReport::Detection {
                                agent_id: self.id.clone(),
                                object_id: format!("OBJ-{}-{}", self.id, objects_found),
                                x: self.position.0,
                                y: self.position.1,
                            }
                        )).await;
                    }
                    
                    if steps >= 20 { break; }
                }
                cmd = self.commands.recv() => {
                    match cmd {
                        Ok(Command::ChangeTarget { x, y }) => {
                            info!(x, y, "Нова ціль");
                            self.target = (x, y);
                        }
                        Ok(Command::Shutdown) | Err(_) => {
                            info!("Отримано shutdown");
                            break;
                        }
                        Ok(Command::ReturnToBase) => {
                            self.target = (0.0, 0.0);
                        }
                        _ => {}
                    }
                }
            }
        }
        
        // Фінальний звіт
        let _ = self.coordinator.send(CoordinatorMessage::Report(
            AgentReport::MissionComplete {
                agent_id: self.id,
                steps,
                objects_found,
            }
        )).await;
    }
    
    fn execute_step(&mut self) {
        let dx = (self.target.0 - self.position.0).signum() * 2.0;
        let dy = (self.target.1 - self.position.1).signum() * 2.0;
        self.position.0 += dx;
        self.position.1 += dy;
        self.battery = self.battery.saturating_sub(3);
    }
}
```

select! loop — серце агента: на кожній ітерації або виконує крок (таймер), або обробляє команду (broadcast). Shutdown через Command::Shutdown або закриття каналу (Err).

---

## 45.6. main: збирання системи

```rust
use tokio::sync::{mpsc, broadcast, watch};
use tokio::time::{sleep, Duration};
use tracing::{info, instrument};
use tracing_subscriber::EnvFilter;

#[tokio::main]
async fn main() {
    // Ініціалізація tracing
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .init();
    
    info!("=== Рій v3.0 ===");
    
    let num_agents = 10;
    
    // Канали
    let (reports_tx, reports_rx) = mpsc::channel::<CoordinatorMessage>(256);
    let (commands_tx, _) = broadcast::channel::<Command>(64);
    let (config_tx, config_rx) = watch::channel(MissionConfig {
        target_zone: (0.0, 0.0, 100.0, 100.0),
        max_agents: num_agents,
        scan_radius: 10.0,
    });
    
    // Координатор
    let coordinator = CoordinatorActor {
        reports_rx,
        commands_tx: commands_tx.clone(),
        config_tx,
        agent_positions: std::collections::HashMap::new(),
        total_objects: 0,
        completed_agents: 0,
        expected_agents: num_agents as u32,
    };
    tokio::spawn(coordinator.run());
    
    // Агенти
    for i in 0..num_agents {
        let agent = AgentActor::new(
            &format!("AGENT-{:02}", i),
            reports_tx.clone(),
            commands_tx.subscribe(),
            config_rx.clone(),
        );
        tokio::spawn(agent.run());
    }
    drop(reports_tx); // координатор закриється коли всі агенти завершать
    
    // Зміна цілі через 1 секунду
    let cmd_tx = commands_tx.clone();
    tokio::spawn(async move {
        sleep(Duration::from_secs(1)).await;
        info!("Зміна цілі для всіх агентів");
        let _ = cmd_tx.send(Command::ChangeTarget { x: 80.0, y: 20.0 });
    });
    
    // Чекаємо завершення (координатор завершиться коли всі агенти звітують)
    sleep(Duration::from_secs(5)).await;
    info!("=== Місія завершена ===");
}
```

10 агентів-акторів + координатор-актор + broadcast для команд + mpsc для звітів + watch для конфігурації + tracing для логування. Все async, все конкурентно.

---

## 45.7. Де що використано: мапа концепцій

| Концепція | Де у v3.0 | Розділ |
|-----------|-----------|--------|
| async/await | Кожна fn актора | 39 |
| Tokio runtime | #[tokio::main], tokio::spawn | 40 |
| tokio::select! | Agent message loop | 40 |
| mpsc channel | Звіти агентів → координатор | 41 |
| broadcast | Команди координатор → агенти | 41 |
| watch | Конфігурація місії | 41 |
| oneshot | GetStatus request-response | 41 |
| Actor Model | Coordinator, Agent як актори | 42 |
| Handle pattern | CoordinatorHandle | 42 |
| Graceful shutdown | Command::Shutdown через broadcast | 42 |
| Serde | AgentReport Serialize/Deserialize | 35 |
| tracing | #[instrument], info!, warn!, spans | 44 |
| interval | Крок агента кожні 100мс | 40 |

---

## 45.8. Prompt Engineering: AI-assisted архітектура

PE-фокус цього практикуму — повний цикл AI-assisted розробки системи.

### Крок 1: проєктування архітектури

```
Проєктую async рій БПЛА на Tokio:
- 10 агентів-акторів (сканують, рухаються, звітують)
- Координатор-актор (збирає звіти, надсилає команди)
- Логер-актор (записує всі події)

Спроєктуй:
1. Enum для кожного типу повідомлень
2. Канали між акторами (mpsc/broadcast/watch)
3. Структуру кожного актора (стан, mailbox)
```

### Крок 2: генерація каркасу

```
На основі архітектури вище, згенеруй каркас коду:
- struct та impl для кожного актора
- Handle для кожного
- async fn run() з select! loop
- main() з ініціалізацією
Без бізнес-логіки — лише структура.
```

### Крок 3: review

```
Ось мій повний код v3.0:
[вставити код]
Review:
1. Чи правильна архітектура каналів?
2. Чи є потенційні deadlocks?
3. Чи коректний graceful shutdown?
4. Чи правильні рівні tracing?
5. Які тести відсутні?
```

### Промпт-журнал

Мінімум 4 записи: архітектура, каркас, бізнес-логіка, review. Для кожного: промпт, ключові моменти відповіді AI, що прийнято / відхилено / модифіковано, обґрунтування.

---

## Лабораторна робота No45

### Завдання

Реалізувати асинхронний рій v3.0.

### Обов'язкові компоненти

1. CoordinatorActor з handle, mpsc для звітів, broadcast для команд.
2. Мінімум 5 AgentActor із select! loop (таймер + команди).
3. watch для конфігурації місії.
4. tracing з #[instrument] для кожного актора, EnvFilter.
5. Graceful shutdown (Shutdown команда через broadcast).
6. Мінімум 8 тестів.
7. Промпт-журнал з 4+ записами.

### Варіанти розширення

**A.** LoggerActor: окремий актор, що отримує LogEntry через mpsc від усіх інших та записує у Vec (або файл через tokio::fs).

**B.** MonitorActor: перевіряє heartbeat кожного агента через interval + HashMap<agent_id, Instant>. Якщо агент не відповів 2с — warn.

**C.** Rayon інтеграція: координатор через spawn_blocking + par_iter обчислює оптимальний розподіл агентів по зонах.

**D.** Serde checkpoint: координатор кожні 5с серіалізує MissionStatus у JSON-файл через tokio::fs.

### Критерії

| Критерій | Бал |
|----------|-----|
| Actor + Handle для 2+ акторів | 15 |
| 3+ типи каналів (mpsc, broadcast, watch) | 15 |
| select! loop в агенті | 10 |
| tracing (#[instrument], рівні, EnvFilter) | 10 |
| Graceful shutdown | 10 |
| 5+ async tasks працюють конкурентно | 10 |
| Тести (8+) | 10 |
| Промпт-журнал (4+ записи) | 10 |
| Читабельність та модульність | 10 |

---

## Контрольні запитання

### Рівень 2

1. Порівняйте v1.0 (Розділ 20), v2.0 (Розділ 30) та v3.0. Що покращилось на кожному етапі?

Відповідь: v1.0 → v2.0: фіксовані дані → динамічні колекції, паніка → Result, конкретний struct → trait Agent. v2.0 → v3.0: однопотоковий → async, прямі виклики → actors/channels, println → tracing, послідовний → конкурентний рій. Кожна версія вирішує обмеження попередньої.

### Рівень 4

2. Яке архітектурне рішення v3.0 ви б змінили для рою з 10000 агентів?

Відповідь: mpsc канал із буфером 256 — може переповнитись при 10000 агентах. Рішення: збільшити буфер або backpressure. Один координатор — bottleneck. Рішення: ієрархія координаторів (squad leader → platoon leader → mission coordinator). HashMap agent_positions — contention при частому оновленні. Рішення: шардування по зонах. broadcast для команд — 10000 підписників. Рішення: multicast по групах.

---

## Резюме

v3.0 інтегрує всі концепції Частини V: Tokio runtime для async виконання (39–40), channels (mpsc, broadcast, watch, oneshot) для комунікації (41), Actor Model для архітектури (42), tracing для логування (44).

Архітектура: кожен компонент — актор із handle. Комунікація лише через повідомлення. Жодного спільного мутабельного стану. Graceful shutdown через broadcast.

AI-assisted workflow: проєктування → каркас → бізнес-логіка → review → тести. Промпт-журнал фіксує рішення.

---

## Що далі

Частина V завершена. Ваш рій — повноцінна асинхронна мультиагентна система: актори, канали, паралельні обчислення, структуроване логування. Це вже не навчальний проєкт — це архітектура, близька до production.

Частина VI ("Архітектура, макроси та екосистема") переходить від коду до системного мислення: макроси для скорочення boilerplate (Розділ 46–47), управління залежностями через Cargo workspace (Розділ 48), unsafe Rust та FFI (Розділ 49), архітектура мультиагентних систем (Розділ 50).
