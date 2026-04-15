# Розділ 44. Логування та tracing

## Анотація

Рій із десятків async tasks працює паралельно. Координатор обробляє повідомлення, агенти сканують зони, логер записує події. Щось пішло не так — агент SCOUT-03 не відповідає. Коли він останній раз відправив звіт? Що він робив? Яка була батарея? З println — хаос: десятки рядків від різних tasks, перемішані без контексту. З tracing — структуроване логування з рівнями (info, warn, error), spans (вкладені контексти: "агент SCOUT-03 → крок 42 → сканування зони B") та фільтрацією ("показати лише warn та error від агентів").

Крейт `tracing` — стандарт логування для async Rust. Він розроблений Tokio team і оптимізований для асинхронного коду, де один потік обслуговує багато tasks, і контекст виконання потрібно зберігати явно.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Налаштувати tracing та tracing-subscriber у проєкті.
2. Створювати events різних рівнів: trace, debug, info, warn, error.
3. Використовувати spans для вкладених контекстів.
4. Застосовувати `#[instrument]` для автоматичного tracing функцій.
5. Налаштовувати фільтрацію через EnvFilter.
6. Використовувати структуроване логування (key=value замість текстових рядків).

---

## Ключові терміни

**Event (подія)** — одиничний запис у лозі з рівнем, повідомленням та структурованими полями. Аналог println, але з метаданими.

**Span (проміжок)** — іменований контекст виконання з початком та кінцем. Events всередині span отримують його контекст. Spans вкладаються: "місія → агент → крок".

**Level (рівень)** — серйозність: trace (найдетальніший), debug, info, warn, error (найсерйозніший).

**Subscriber** — "приймач" tracing-даних. Визначає, куди записувати (stdout, файл, мережа) та як форматувати.

**#[instrument]** — атрибут, що автоматично створює span для функції з її параметрами.

---

## Мотиваційний кейс

Під час тестової місії рій із 8 агентів працював 30 хвилин. Один агент загубився — не повернувся на базу. Лог (println): 50 000 рядків тексту, перемішані від 8 задач. Знайти останні повідомлення від загубленого агента — години ручного grep. Лог (tracing): фільтр `agent_id=SCOUT-03 level>=info` — 200 рядків хронологічного логу саме цього агента. Причина знайдена за хвилини: попередження "battery 12%" → помилка "GPS lost" → зникнення зі зв'язку.

---

## 44.1. Налаштування

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

Мінімальна ініціалізація:

```rust
use tracing::{info, warn, error, debug, trace};
use tracing_subscriber;

fn main() {
    // Ініціалізація subscriber (виводить у stdout)
    tracing_subscriber::fmt::init();
    
    info!("Програма запущена");
    debug!("Деталі ініціалізації...");
    warn!("Батарея нижче 30%");
    error!("GPS сигнал втрачено");
}
```

Вивід:

```
2024-01-15T10:30:00.123Z  INFO main: Програма запущена
2024-01-15T10:30:00.124Z DEBUG main: Деталі ініціалізації...
2024-01-15T10:30:00.124Z  WARN main: Батарея нижче 30%
2024-01-15T10:30:00.124Z ERROR main: GPS сигнал втрачено
```

Кожен рядок містить: timestamp, рівень, джерело (модуль), повідомлення. Це вже значно краще за println — timestamp та рівень додаються автоматично.

---

## 44.2. П'ять рівнів логування

Рівні від найдетальнішого до найсерйознішого:

`trace!` — найдетальніший. Кожен виклик функції, кожне значення змінної. Для глибокого дебагу, зазвичай вимкнений у production.

`debug!` — інформація для розробника. Проміжні обчислення, стан змінних, шлях виконання.

`info!` — значущі події для оператора. "Місія розпочата", "Агент SCOUT-01 завершив", "Виявлено 15 об'єктів".

`warn!` — потенційні проблеми, що не зупиняють роботу. "Батарея 25%", "GPS сигнал слабкий", "Затримка відповіді агента".

`error!` — серйозні проблеми. "GPS втрачено", "Сенсор не відповідає", "Неможливо обчислити маршрут".

```rust
use tracing::{trace, debug, info, warn, error};

fn process_sensor_reading(value: f64) {
    trace!("Отримано показання: {}", value);
    
    if value < 0.0 {
        warn!(value, "Від'ємне показання — можливий збій сенсора");
        return;
    }
    
    if value > 1000.0 {
        error!(value, "Показання поза межами — ігнорую");
        return;
    }
    
    debug!(value, processed = value * 0.001, "Оброблено");
    info!("Показання прийнято: {:.2}", value);
}
```

Зверніть увагу: `warn!(value, "повідомлення")` — value додається як структуроване поле (key=value), а не через format!. Це структуроване логування: замість текстового рядка "Від'ємне показання: -5.0" — структура `{ value: -5.0, message: "Від'ємне показання" }`. Структуровані дані можна фільтрувати, агрегувати та аналізувати програмно.

Правило вибору рівня: чи потрібна ця інформація оператору в production? info. Чи потрібна розробнику при дебагу? debug. Чи потрібна лише при глибокому аналізі конкретної проблеми? trace. Є проблема, але система продовжує? warn. Система не може виконати операцію? error.

---

## 44.3. Spans: вкладений контекст

Event каже "що сталось". Span каже "в якому контексті". Spans вкладаються — створюючи "дерево" контексту:

```rust
use tracing::{info, warn, span, Level};

fn main() {
    tracing_subscriber::fmt::init();
    
    let mission_span = span!(Level::INFO, "mission", name = "Alpha");
    let _guard = mission_span.enter();
    
    info!("Місія розпочата");
    
    {
        let agent_span = span!(Level::INFO, "agent", id = "SCOUT-01");
        let _guard = agent_span.enter();
        
        info!("Агент активований");
        
        {
            let step_span = span!(Level::DEBUG, "step", number = 1);
            let _guard = step_span.enter();
            
            info!("Сканування зони A");
            warn!("Батарея 28%");
        }
        
        info!("Крок завершено");
    }
    
    info!("Місія завершена");
}
```

Вивід:

```
INFO mission{name="Alpha"}: Місія розпочата
INFO mission{name="Alpha"}:agent{id="SCOUT-01"}: Агент активований
INFO mission{name="Alpha"}:agent{id="SCOUT-01"}:step{number=1}: Сканування зони A
WARN mission{name="Alpha"}:agent{id="SCOUT-01"}:step{number=1}: Батарея 28%
INFO mission{name="Alpha"}:agent{id="SCOUT-01"}: Крок завершено
INFO mission{name="Alpha"}: Місія завершена
```

Кожен event показує повний ланцюжок spans: mission → agent → step. Це вирішує проблему "від якого агента цей лог?" — контекст видимий автоматично. У async коді це особливо цінно: один потік обслуговує десятки tasks, і без spans неможливо зрозуміти, яка task згенерувала event.

`span.enter()` повертає guard — поки guard живий, span активний. При drop guard — span деактивується. RAII — знайомий патерн з Mutex та RefCell.

---

## 44.4. #[instrument]: автоматичний span для функції

Замість ручного створення span для кожної функції — атрибут `#[instrument]`:

```rust
use tracing::{info, warn, instrument};

#[instrument]
fn calculate_route(from: (f64, f64), to: (f64, f64)) -> Vec<(f64, f64)> {
    info!("Обчислення маршруту");
    vec![from, to]
}

#[instrument(skip(readings), fields(count = readings.len()))]
fn analyze_readings(agent_id: &str, readings: &[f64]) -> f64 {
    if readings.is_empty() {
        warn!("Порожні показання");
        return 0.0;
    }
    let avg = readings.iter().sum::<f64>() / readings.len() as f64;
    info!(average = avg, "Аналіз завершено");
    avg
}

#[instrument]
async fn agent_step(id: String, step: u32) {
    info!("Починаю крок");
    tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
    info!("Крок завершено");
}
```

`#[instrument]` автоматично створює span із назвою функції та значеннями параметрів. `skip(readings)` — не включати великий масив у span (зайвий шум). `fields(count = readings.len())` — додати обчислене поле замість сирих даних.

Для async fn `#[instrument]` автоматично зберігає span через .await — events всередині async fn завжди мають правильний контекст, навіть коли task переключається між потоками.

---

## 44.5. Фільтрація через EnvFilter

У production не потрібні trace та debug для всіх модулів. EnvFilter дозволяє налаштувати рівень для кожного модуля:

```rust
use tracing_subscriber::EnvFilter;

fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .init();
}
```

Тепер фільтр налаштовується через змінну середовища `RUST_LOG`:

```bash
# Лише info та вище
RUST_LOG=info cargo run

# Debug для модуля agent, info для решти
RUST_LOG=info,agent=debug cargo run

# Trace для конкретного модуля
RUST_LOG=warn,my_crate::sensor=trace cargo run

# Все вимкнено, лише error
RUST_LOG=error cargo run
```

Це дозволяє змінювати рівень логування без перекомпіляції — ідеально для production. За замовчуванням — info (значущі події). При розслідуванні проблеми — debug або trace для конкретного модуля.

---

## 44.6. Практика: tracing для рою БПЛА

```rust
use tracing::{info, warn, error, instrument, span, Level};
use tracing_subscriber::EnvFilter;
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[derive(Debug)]
struct AgentReport {
    agent_id: String,
    objects: u32,
    battery: u8,
}

#[instrument(skip(tx))]
async fn simulate_agent(id: String, tx: mpsc::Sender<AgentReport>) {
    info!("Агент запущено");
    
    let mut battery = 100_u8;
    let mut found = 0_u32;
    
    for step in 0..5 {
        let step_span = span!(Level::DEBUG, "step", number = step);
        let _guard = step_span.enter();
        
        sleep(Duration::from_millis(80)).await;
        battery = battery.saturating_sub(15);
        
        if step % 2 == 0 {
            found += 1;
            info!(object_id = found, "Об'єкт виявлено");
        }
        
        if battery < 30 {
            warn!(battery, "Низька батарея");
        }
    }
    
    info!(total_objects = found, final_battery = battery, "Місія завершена");
    let _ = tx.send(AgentReport { agent_id: id, objects: found, battery }).await;
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .init();
    
    info!("Запуск рою");
    
    let (tx, mut rx) = mpsc::channel(64);
    
    for i in 0..3 {
        let tx = tx.clone();
        tokio::spawn(simulate_agent(format!("AGENT-{}", i), tx));
    }
    drop(tx);
    
    let mut total = 0;
    while let Some(report) = rx.recv().await {
        info!(
            agent = %report.agent_id,
            objects = report.objects,
            battery = report.battery,
            "Звіт отримано"
        );
        total += report.objects;
    }
    
    info!(total_objects = total, "Місія рою завершена");
}
```

`%report.agent_id` — Display форматування для поля (замість Debug). `#[instrument(skip(tx))]` — span з параметрами функції, але без tx (Sender не потрібен у лозі).

---

## 44.7. Prompt Engineering: додавання tracing

### Промпт-шаблон

```
Ось мій async код:
[вставити код]

Додай tracing:
1. #[instrument] для кожної async fn
2. info! для значущих подій
3. warn! для потенційних проблем
4. error! для серйозних помилок
5. span для вкладених контекстів
Поясни вибір рівня для кожного event.
```

---

## Лабораторна робота No44

### Мета

Додати структуроване логування до рою БПЛА.

### Завдання базового рівня

1. tracing-subscriber з EnvFilter.
2. `#[instrument]` для 3+ async fn.
3. Events з різними рівнями (info, warn, error) — мінімум 10 місць.
4. Spans для вкладених контекстів (місія → агент → крок).
5. Запуск з різними RUST_LOG та порівняння виводу.

### Варіанти

**A.** Файловий subscriber: логувати у файл через `tracing-appender`. JSON формат для machine-readable логів.

**B.** Метрики: підрахувати кількість events кожного рівня за місію. Фінальний звіт: "42 info, 7 warn, 1 error".

**C.** Distributed tracing: кожен агент додає свій id до кожного span. Координатор корелює events від різних агентів за timestamp.

### Критерії

| Критерій | Бал |
|----------|-----|
| Ініціалізація subscriber з EnvFilter | 15 |
| #[instrument] для async fn | 20 |
| Events з правильними рівнями | 25 |
| Spans для контексту | 20 |
| Структуровані поля (key=value) | 10 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: tracing events не виводяться

Забули ініціалізувати subscriber:

```rust
fn main() {
    info!("Нічого не виведеться"); // немає subscriber
}
```

Виправлення: `tracing_subscriber::fmt::init();` на початку main.

### Помилка 2: занадто багато логів

Рівень trace виводить тисячі рядків на секунду.

Виправлення: змінити RUST_LOG на info або вище: `RUST_LOG=info cargo run`. Або фільтрувати по модулю: `RUST_LOG=warn,my_module=debug`.

### Помилка 3: span не відображається в async

Async fn з ручним span::enter() — guard знищується при .await (task переключається):

```rust
async fn bad() {
    let span = span!(Level::INFO, "my_span");
    let _guard = span.enter(); // guard drop при await!
    some_async().await; // events тут не мають span
}
```

Виправлення: `#[instrument]` замість ручного span, або `.instrument(span)` від `tracing::Instrument` trait.

---

## Додатково

### tracing vs log

Крейт `log` — старший стандарт логування у Rust (макроси info!, warn!). `tracing` — наступне покоління: додає spans, структуровані поля, оптимізований для async. tracing сумісний з log через `tracing-log` bridge — можна використовувати обидва одночасно. Для нових проєктів — tracing.

### Формати виводу

tracing-subscriber підтримує кілька форматів: `fmt()` — human-readable (за замовчуванням), `.json()` — JSON (для machine-readable логів, ELK stack), `.pretty()` — кольоровий з відступами (для розробки). Вибір при ініціалізації:

```rust
tracing_subscriber::fmt()
    .json()  // JSON формат
    .with_env_filter(EnvFilter::new("info"))
    .init();
```

---

## Контрольні запитання

### Рівень 1

1. Чим tracing відрізняється від println?

Відповідь: tracing додає рівні (info/warn/error), timestamp, spans (контекст), структуровані поля (key=value), фільтрацію (RUST_LOG). println — просто текст без метаданих.

2. Які п'ять рівнів логування?

Відповідь: trace (найдетальніший), debug, info, warn, error (найсерйозніший).

### Рівень 2

3. Чим span відрізняється від event?

Відповідь: event — одиничний запис ("об'єкт виявлено"). Span — іменований контекст з початком та кінцем ("агент SCOUT-01, крок 42"). Events всередині span автоматично отримують його контекст.

4. Що робить `#[instrument]`?

Відповідь: автоматично створює span для функції з її назвою та параметрами. Для async fn зберігає span через .await. Еквівалент ручного span!(Level::INFO, "fn_name", param1, param2) + enter().

### Рівень 3

5. Як налаштувати різні рівні для різних модулів?

Відповідь: `RUST_LOG=warn,my_crate::agent=debug,my_crate::sensor=trace`. Глобальний рівень warn, модуль agent — debug, модуль sensor — trace. Без перекомпіляції.

### Рівень 4

6. Чому span.enter() не працює через .await в async fn?

Відповідь: enter() повертає guard, що деактивує span при drop. В async fn .await може переключити task на інший потік — guard знищується, span деактивується. Наступні events після await не мають span. Рішення: #[instrument] або .instrument(span) — вони зберігають span через await.

---

## Резюме

tracing — структуроване логування для async Rust. Events (info!, warn!, error!) із структурованими полями (key=value). Spans — вкладені контексти (місія → агент → крок).

`#[instrument]` — автоматичний span для функцій. Зберігає контекст через .await в async fn.

EnvFilter — фільтрація через RUST_LOG без перекомпіляції. Різні рівні для різних модулів.

П'ять рівнів: trace (все), debug (розробник), info (оператор), warn (проблеми), error (серйозне).

---

## Що далі

Частина V завершується Розділом 45 — Практикумом v3.0. Ви об'єднаєте все: Tokio runtime, actors, channels (mpsc/broadcast/watch), Rayon для CPU-bound, tracing для логування — у повноцінний асинхронний рій з 10+ агентів-акторів, координатором, логером та паралельними обчисленнями. Це кульмінація підручника перед переходом до архітектурних тем Частини VI.
