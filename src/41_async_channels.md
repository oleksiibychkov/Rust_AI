# Розділ 41. Async Channels, Streams та синхронізація

## Анотація

У Розділі 40 ви використовували `tokio::sync::mpsc` — один тип каналу для передачі повідомлень. Але реальна система комунікації рою потребує різних патернів: координатор надсилає команду ВСІМ агентам одночасно (broadcast), агент запитує конфігурацію і чекає ОДНУ відповідь (oneshot), координатор публікує "останній відомий стан", який агенти читають коли потрібно (watch). Кожен патерн — окремий тип каналу з оптимізованою семантикою. Крім каналів, Tokio надає Streams — async аналог ітераторів, та Semaphore — примітив для обмеження конкурентності. Цей розділ завершує інструментарій async-комунікації.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Обрати правильний тип каналу: mpsc, broadcast, watch, oneshot.
2. Реалізувати broadcast для "один до багатьох" повідомлень.
3. Використати oneshot для request-response патерну.
4. Використати watch для публікації "останнього стану".
5. Обробляти async потоки даних через Stream та StreamExt.
6. Обмежувати конкурентність через Semaphore.

---

## Ключові терміни

**broadcast** — канал "один до багатьох": кожне повідомлення отримують ВСІ підписники.

**oneshot** — одноразовий канал для рівно одного повідомлення. Ідеальний для request-response.

**watch** — канал "останнє значення": receiver завжди бачить найактуальніше значення, проміжні можуть бути пропущені.

**Stream** — async аналог Iterator. Видає елементи асинхронно, по одному, через poll_next.

**Semaphore** — примітив синхронізації, що обмежує кількість одночасних доступів до ресурсу.

---

## Мотиваційний кейс

Координатор рою комунікує з агентами через кілька патернів одночасно. "Зміна зони пошуку" — потрібно повідомити ВСІХ агентів одночасно (broadcast). "Який твій статус, SCOUT-01?" — запит одному агенту з очікуванням однієї відповіді (oneshot). "Поточна конфігурація місії" — агенти читають коли потрібно, координатор оновлює (watch). Потік телеметрії від кожного агента — безперервний stream показань, що потрібно фільтрувати та агрегувати. Одночасних мережевих з'єднань — не більше 50 (semaphore). Один mpsc не покриває всі ці патерни ефективно.

---

## 41.1. Чотири типи каналів: вибір за патерном

Tokio надає чотири типи каналів, кожен оптимізований для конкретного патерну комунікації. Правильний вибір — ключ до ефективної та зрозумілої архітектури.

| Канал | Producers | Consumers | Буфер | Патерн |
|-------|-----------|-----------|-------|--------|
| mpsc | Багато | Один | Bounded/unbounded | Events, звіти, черга задач |
| broadcast | Один (або багато) | Багато | Bounded (ring buffer) | Команди всім, оновлення |
| watch | Один | Багато | 1 (останнє значення) | Конфігурація, стан |
| oneshot | Один | Один | 1 (одне повідомлення) | Request-response |

mpsc ви вже знаєте з Розділу 40: кілька producers, один consumer. Повідомлення отримує рівно один consumer. Решта — нові.

---

## 41.2. broadcast: повідомлення всім

broadcast канал доставляє кожне повідомлення кожному активному receiver. Якщо координатор надсилає "зміна зони" — всі 10 агентів отримують це повідомлення.

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Канал з буфером на 16 повідомлень
    let (tx, _) = broadcast::channel::<String>(16);
    
    // 3 receiver-и (агенти)
    let mut handles = Vec::new();
    for i in 0..3 {
        let mut rx = tx.subscribe(); // кожен receiver — окремий subscribe
        handles.push(tokio::spawn(async move {
            loop {
                match rx.recv().await {
                    Ok(msg) => println!("  Agent-{}: отримав '{}'", i, msg),
                    Err(broadcast::error::RecvError::Closed) => break,
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        println!("  Agent-{}: пропустив {} повідомлень!", i, n);
                    }
                }
            }
        }));
    }
    
    // Координатор надсилає команди
    sleep(Duration::from_millis(50)).await;
    tx.send(String::from("Зміна зони: сектор B")).unwrap();
    tx.send(String::from("Висота: 200м")).unwrap();
    tx.send(String::from("Завершення місії")).unwrap();
    
    drop(tx); // закриваємо канал
    for h in handles { h.await.unwrap(); }
}
```

Кілька важливих відмінностей від mpsc. Кожен receiver створюється через `tx.subscribe()`, а не через пару з channel(). Можна створити нових receiver-ів у будь-який момент — вони отримуватимуть лише повідомлення, надіслані після subscribe.

Broadcast має фіксований буфер (ring buffer). Якщо повільний receiver не встигає читати, а sender надсилає нові повідомлення — старі витісняються. Повільний receiver отримає `Err(Lagged(n))` — він пропустив n повідомлень. Це свідомий trade-off: sender ніколи не блокується (навіть при повільних receivers), але повільні receivers втрачають повідомлення.

Коли broadcast: команди від координатора до всіх агентів, оновлення конфігурації, події системного рівня ("батарея бази низька", "погода змінилась").

---

## 41.3. oneshot: одна відповідь на один запит

oneshot — канал рівно для одного повідомлення. Sender може відправити лише раз. Receiver може отримати лише раз. Ідеальний для request-response: відправити запит + oneshot Sender, отримати відповідь через oneshot Receiver.

```rust
use tokio::sync::oneshot;
use tokio::time::{sleep, Duration};

async fn query_agent_status(
    request_tx: tokio::sync::mpsc::Sender<(String, oneshot::Sender<String>)>,
    agent_id: &str,
) -> String {
    let (response_tx, response_rx) = oneshot::channel();
    
    // Надсилаємо запит + канал для відповіді
    request_tx.send((agent_id.to_string(), response_tx)).await.unwrap();
    
    // Чекаємо відповідь
    response_rx.await.unwrap()
}

async fn agent_handler(
    mut rx: tokio::sync::mpsc::Receiver<(String, oneshot::Sender<String>)>,
) {
    while let Some((agent_id, response_tx)) = rx.recv().await {
        // Обробляємо запит
        sleep(Duration::from_millis(50)).await;
        let status = format!("{}: battery=85%, pos=(10,20)", agent_id);
        
        // Відповідаємо через oneshot
        let _ = response_tx.send(status); // може провалитись, якщо caller вже не чекає
    }
}

#[tokio::main]
async fn main() {
    let (request_tx, request_rx) = tokio::sync::mpsc::channel(32);
    
    tokio::spawn(agent_handler(request_rx));
    
    let status = query_agent_status(request_tx.clone(), "SCOUT-01").await;
    println!("Статус: {}", status);
    
    let status = query_agent_status(request_tx, "RECON-02").await;
    println!("Статус: {}", status);
}
```

Патерн: caller створює oneshot, відправляє Sender разом із запитом через mpsc, і чекає Receiver. Handler отримує запит та oneshot Sender, обробляє, відповідає через Sender. Кожен запит — окремий oneshot канал.

Чому не просто mpsc для відповідей? Тому що mpsc — один receiver, і якщо кілька callers чекають відповідей — хто отримає яку? oneshot зв'язує конкретний запит з конкретною відповіддю — один-до-одного.

---

## 41.4. watch: останнє значення

watch — канал, де receiver завжди бачить останнє надіслане значення. Проміжні значення можуть бути пропущені (якщо sender оновлює швидше, ніж receiver читає). Ідеальний для "поточного стану": конфігурація місії, поточна ціль, погодні умови.

```rust
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

#[derive(Debug, Clone)]
struct MissionState {
    target: (f64, f64),
    active: bool,
    phase: String,
}

#[tokio::main]
async fn main() {
    let initial = MissionState {
        target: (100.0, 200.0),
        active: true,
        phase: String::from("Розвідка"),
    };
    
    let (tx, rx) = watch::channel(initial);
    
    // Агент-читач: реагує на зміни
    let mut rx1 = rx.clone();
    tokio::spawn(async move {
        loop {
            // changed() чекає до наступної зміни
            if rx1.changed().await.is_err() { break; }
            let state = rx1.borrow();
            println!("Agent-1 бачить: {:?} → {:?}", state.phase, state.target);
        }
    });
    
    // Координатор оновлює стан
    sleep(Duration::from_millis(100)).await;
    tx.send(MissionState {
        target: (150.0, 250.0),
        active: true,
        phase: String::from("Переслідування"),
    }).unwrap();
    
    sleep(Duration::from_millis(100)).await;
    tx.send(MissionState {
        target: (150.0, 250.0),
        active: false,
        phase: String::from("Завершення"),
    }).unwrap();
    
    sleep(Duration::from_millis(100)).await;
}
```

`rx.changed().await` — блокує task до наступної зміни значення. `rx.borrow()` — іммутабельний доступ до поточного значення (повертає Ref, як RefCell). Receiver можна clone() для кількох читачів.

Відмінність від broadcast: broadcast доставляє кожне повідомлення кожному. watch — лише останнє. Якщо sender оновив значення 5 разів, поки receiver спав — receiver побачить лише останнє п'яте. Для конфігурації це правильна семантика: агенту потрібна актуальна конфігурація, а не історія всіх змін.

---

## 41.5. Stream: async ітератор

Stream — async аналог Iterator. Iterator видає елементи синхронно через `next()`. Stream видає елементи асинхронно через `poll_next()` — кожен елемент може потребувати очікування (мережа, таймер, канал).

Trait Stream визначений у `futures-core`, а корисні адаптери — у `tokio-stream`:

```toml
[dependencies]
tokio-stream = "0.1"
```

```rust
use tokio_stream::StreamExt;
use tokio::time::{interval, Duration};
use tokio_stream::wrappers::IntervalStream;

#[tokio::main]
async fn main() {
    // Перетворити interval у Stream
    let tick_stream = IntervalStream::new(interval(Duration::from_millis(100)));
    
    // StreamExt дає адаптери: map, filter, take, etc.
    let mut readings = tick_stream
        .enumerate()
        .map(|(i, _)| {
            // Симуляція показання сенсора
            let value = 20.0 + (i as f64 * 0.5).sin() * 5.0;
            (i, value)
        })
        .filter(|(_, v)| futures::future::ready(*v > 18.0))
        .take(5); // лише перших 5
    
    println!("Показання сенсора (filtered):");
    while let Some((idx, value)) = readings.next().await {
        println!("  #{}: {:.1}°C", idx, value);
    }
}
```

`StreamExt` надає адаптери, аналогічні Iterator: `map`, `filter`, `take`, `skip`, `enumerate`, `chain`, `fold`, `collect`. Різниця — кожен `.next().await` може призупинити task.

mpsc Receiver теж можна перетворити у Stream:

```rust
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel::<f64>(100);
    
    // Відправляємо показання
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i as f64 * 2.5).await.unwrap();
        }
    });
    
    // Обробляємо як Stream
    let stream = ReceiverStream::new(rx);
    
    let stats: Vec<f64> = stream
        .filter(|&v| v > 5.0)
        .map(|v| v * 0.001) // перетворення
        .collect()
        .await;
    
    println!("Оброблено: {:?}", stats);
}
```

Stream потужний для data pipeline: сенсор → filter невалідних → map одиниць → average по вікну → collect. Весь pipeline async — кожен крок може чекати наступний елемент без блокування.

---

## 41.6. Semaphore: обмеження конкурентності

Semaphore обмежує кількість одночасних доступів до ресурсу. "Не більше 5 агентів одночасно сканують зону" або "не більше 10 одночасних мережевих з'єднань":

```rust
use tokio::sync::Semaphore;
use tokio::time::{sleep, Duration};
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3)); // макс 3 одночасних
    let mut handles = Vec::new();
    
    for i in 0..8 {
        let sem = Arc::clone(&semaphore);
        handles.push(tokio::spawn(async move {
            let _permit = sem.acquire().await.unwrap(); // чекає вільний слот
            println!("Task {} почала (active: {})", i, 3 - sem.available_permits());
            sleep(Duration::from_millis(200)).await; // "робота"
            println!("Task {} завершила", i);
            // _permit drop → слот звільняється
        }));
    }
    
    for h in handles { h.await.unwrap(); }
}
```

`Semaphore::new(3)` — максимум 3 одночасних permits. `acquire().await` — чекає, поки є вільний permit. Коли permit (SemaphorePermit) знищується (drop) — слот повертається. 8 tasks, але одночасно працюють лише 3 — решта чекають.

Semaphore корисний для: обмеження мережевих з'єднань (rate limiting), обмеження доступу до обмеженого ресурсу (файл, БД), контролю конкурентності для уникнення перевантаження.

---

## 41.7. Практика: комплексна комунікація рою

```rust
use tokio::sync::{broadcast, watch, mpsc, oneshot, Semaphore};
use tokio::time::{sleep, Duration};
use std::sync::Arc;

#[derive(Clone, Debug)]
enum Command {
    ChangeTarget(f64, f64),
    ReturnToBase,
}

#[derive(Debug)]
struct AgentReport {
    agent_id: String,
    objects_found: u32,
}

async fn agent(
    id: String,
    mut commands: broadcast::Receiver<Command>,
    report_tx: mpsc::Sender<AgentReport>,
    scan_semaphore: Arc<Semaphore>,
) {
    let mut found = 0_u32;
    
    for step in 0..5 {
        tokio::select! {
            cmd = commands.recv() => {
                match cmd {
                    Ok(Command::ReturnToBase) => {
                        println!("[{}] Отримано: повернення", id);
                        break;
                    }
                    Ok(Command::ChangeTarget(x, y)) => {
                        println!("[{}] Нова ціль: ({:.0}, {:.0})", id, x, y);
                    }
                    Err(_) => break,
                }
            }
            _ = async {
                // Обмежений доступ до сканування
                let _permit = scan_semaphore.acquire().await.unwrap();
                sleep(Duration::from_millis(100)).await;
                found += 1;
            } => {
                println!("[{}] Крок {}, знайдено: {}", id, step, found);
            }
        }
    }
    
    let _ = report_tx.send(AgentReport { agent_id: id, objects_found: found }).await;
}

#[tokio::main]
async fn main() {
    let (cmd_tx, _) = broadcast::channel::<Command>(16);
    let (report_tx, mut report_rx) = mpsc::channel::<AgentReport>(32);
    let scan_sem = Arc::new(Semaphore::new(2)); // макс 2 одночасних скани
    
    for i in 0..4 {
        let cmd_rx = cmd_tx.subscribe();
        let rep_tx = report_tx.clone();
        let sem = Arc::clone(&scan_sem);
        tokio::spawn(agent(format!("AGENT-{}", i), cmd_rx, rep_tx, sem));
    }
    drop(report_tx);
    
    // Координатор: через 250мс змінити ціль, через 500мс — повернення
    tokio::spawn(async move {
        sleep(Duration::from_millis(250)).await;
        let _ = cmd_tx.send(Command::ChangeTarget(50.0, 80.0));
        sleep(Duration::from_millis(250)).await;
        let _ = cmd_tx.send(Command::ReturnToBase);
    });
    
    // Збираємо звіти
    let mut total = 0;
    while let Some(report) = report_rx.recv().await {
        println!("[REPORT] {}: {} об'єктів", report.agent_id, report.objects_found);
        total += report.objects_found;
    }
    println!("Всього виявлено: {}", total);
}
```

Чотири примітиви в одній системі: broadcast для команд (один координатор → всі агенти), mpsc для звітів (всі агенти → координатор), Semaphore для обмеження одночасного сканування (2 з 4 агентів), select! для обробки команд та роботи конкурентно.

---

## 41.8. Prompt Engineering: вибір каналу

### Промпт-шаблон

```
Система рою БПЛА з такими потоками повідомлень:
1. Координатор надсилає команди всім агентам
2. Агент запитує конфігурацію і чекає відповідь
3. Координатор публікує поточний стан місії
4. Агенти надсилають телеметрію координатору
5. Не більше 5 одночасних мережевих запитів

Для кожного: обери тип каналу (mpsc/broadcast/watch/oneshot/semaphore) 
та поясни чому саме він.
```

---

## Лабораторна робота No41

### Мета

Реалізувати комплексну async комунікацію рою з різними типами каналів.

### Завдання базового рівня

1. broadcast для команд координатора (3+ варіанти enum Command).
2. mpsc для звітів агентів.
3. watch для поточного стану місії (MissionState struct).
4. Semaphore для обмеження одночасних операцій.

### Варіанти

**A.** Request-response через oneshot: координатор запитує конкретного агента "який твій статус?" і чекає відповідь з timeout.

**B.** Stream pipeline: потік показань сенсора → filter невалідних → map одиниць → moving average → collect.

**C.** Повна система: broadcast (команди) + watch (конфігурація) + mpsc (звіти) + oneshot (запити) + semaphore (обмеження) в одному проєкті.

### Критерії

| Критерій | Бал |
|----------|-----|
| 3+ типи каналів у одній системі | 25 |
| Обґрунтований вибір каналу для кожного потоку | 20 |
| Stream з адаптерами | 15 |
| Semaphore для обмеження | 10 |
| Тести | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `Lagged` у broadcast

Повільний receiver не встиг прочитати — повідомлення витіснене з ring buffer:

Виправлення: збільшити буфер `broadcast::channel(256)`, або обробляти Lagged (логувати та продовжити).

### Помилка 2: `watch::Receiver::changed()` пропускає проміжні

watch зберігає лише останнє значення. 5 оновлень між двома changed() — receiver побачить лише 5-те.

Це не помилка — це семантика watch. Якщо потрібні всі повідомлення — використовуйте mpsc або broadcast.

### Помилка 3: oneshot Sender використаний двічі

```rust
let (tx, rx) = oneshot::channel();
tx.send(1).unwrap();
tx.send(2).unwrap(); // ПОМИЛКА: tx уже спожитий (moved)
```

oneshot — рівно одне повідомлення. send() споживає Sender (move). Для кількох повідомлень — mpsc.

### Помилка 4: Stream "зависає" — забули закрити sender

Як з mpsc: ReceiverStream завершується, коли всі Sender-и знищені. Не забути drop(tx).

---

## Додатково

### Таблиця порівняння каналів

| Канал | Пропускна здатність | Гарантія доставки | Блокування sender | Типовий use case |
|-------|--------------------|--------------------|-------------------|------------------|
| mpsc | Висока | Кожне одному consumer | bounded: так | Events, звіти |
| broadcast | Середня | Кожне всім (або Lagged) | Ні (ring buffer) | Команди, broadcast |
| watch | Низька | Лише останнє | Ні | Стан, конфігурація |
| oneshot | N/A (одноразовий) | Одне одному | Ні | Request-response |

---

## Контрольні запитання

### Рівень 1

1. Чим broadcast відрізняється від mpsc?

Відповідь: mpsc — повідомлення отримує один consumer. broadcast — кожне повідомлення отримують всі subscribers. mpsc — для збору звітів, broadcast — для розсилки команд.

2. Що таке oneshot канал?

Відповідь: канал для рівно одного повідомлення. Sender може відправити лише раз (consumed після send). Ідеальний для request-response.

### Рівень 2

3. Коли watch кращий за broadcast?

Відповідь: коли потрібне лише актуальне значення, а не вся історія. Конфігурація: агенту потрібна поточна ціль, а не список усіх минулих цілей. watch зберігає лише останнє, broadcast — всі (в межах буфера). watch дешевший для "стану", broadcast — для "подій".

4. Чим Stream відрізняється від Iterator?

Відповідь: Iterator видає елементи синхронно — next() повертає одразу. Stream видає асинхронно — next().await може призупинити task до появи наступного елемента. Stream — для обробки даних, що надходять з IO (мережа, канал, таймер).

### Рівень 3

5. Спроєктуйте комунікацію: координатор + 10 агентів + логер. Які канали?

Відповідь: координатор → агенти: broadcast (команди всім). Агенти → координатор: mpsc (звіти). Координатор → логер: mpsc (логи). Координатор → всі (стан місії): watch. Координатор ← конкретний агент (запит статусу): mpsc запит + oneshot відповідь.

### Рівень 4

6. Чому Semaphore, а не просто лічильник AtomicU32?

Відповідь: Semaphore — async: acquire().await призупиняє task до звільнення permit. AtomicU32 — sync: потрібен busy-wait (loop з spin) або std::thread::sleep (блокує потік). Semaphore інтегрований з Tokio runtime — task ефективно "спить", поки permit не доступний, потік обслуговує інші tasks. AtomicU32 або блокує потік (погано для async), або витрачає CPU на spin (неефективно).

---

## Резюме

Чотири типи каналів для різних патернів: mpsc (many→one, events), broadcast (one→many, commands), watch (last value, config), oneshot (one-time, request-response).

Stream — async ітератор з адаптерами (map, filter, take). ReceiverStream перетворює mpsc::Receiver у Stream. Pipeline: source → transform → consume — все async.

Semaphore обмежує конкурентність: `acquire().await` чекає permit, drop — звільняє. Для rate limiting, обмеження ресурсів.

Вибір: mpsc для збору даних, broadcast для розсилки, watch для стану, oneshot для запит-відповідь, Semaphore для обмежень.

---

## Що далі

Канали та streams дали інструменти комунікації. Розділ 42 об'єднає їх у архітектурний патерн: Actor Model. Кожен агент стане "актором" — async task із власним mailbox (mpsc channel) та handle (Sender). Координатор буде актором. Логер буде актором. Кожен актор — ізольований, комунікує лише через повідомлення. Це дасть чітку архітектуру для складних систем із десятками компонентів.
