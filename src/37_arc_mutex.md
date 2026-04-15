# Розділ 37. Arc\<T\> та Mutex\<T\>: спільний стан між потоками

## Анотація

У Розділі 36 кожен потік працював з ізольованими даними: move closure переміщувала ownership, і кожен потік мав свою незалежну копію. Це безпечно, але обмежено. Реальний рій БПЛА потребує спільного стану: карта виявлених об'єктів, до якої всі агенти записують знахідки та з якої читають при плануванні маршруту. Лічильник завершених завдань, що оновлюється кожним агентом. Конфігурація місії, що може змінитись від координатора.

Для спільного стану між потоками потрібні два механізми. Arc\<T\> (Atomic Reference Counting) — thread-safe аналог Rc: кілька потоків можуть "володіти" одним значенням, лічильник збільшується/зменшується атомарно. Mutex\<T\> (Mutual Exclusion) — замок, що гарантує ексклюзивний доступ: лише один потік може читати/писати дані в кожен момент часу. Разом `Arc<Mutex<T>>` — стандартний патерн спільного мутабельного стану у Rust.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити, чому Rc не працює для потоків і як Arc це вирішує.
2. Використати `Arc<T>` для спільного read-only доступу між потоками.
3. Використати `Mutex<T>` для ексклюзивного мутабельного доступу.
4. Застосувати патерн `Arc<Mutex<T>>` для спільних мутабельних даних.
5. Використати `RwLock<T>` для сценарію "багато читачів, один писець".
6. Пояснити deadlock та стратегії уникнення.
7. Обрати між clone, Arc та channels для конкретної задачі.

---

## Ключові терміни

**Arc\<T\> (Atomic Reference Counting)** — thread-safe аналог Rc. Лічильник посилань збільшується/зменшується атомарними операціями процесора, безпечними для одночасного доступу з різних потоків.

**Mutex\<T\> (Mutual Exclusion)** — примітив синхронізації, що гарантує: лише один потік має доступ до даних у кожен момент часу. Інші потоки чекають (блокуються), поки замок не звільниться.

**MutexGuard** — "квиток" на доступ до даних всередині Mutex. Поводиться як &mut T. При знищенні (drop) автоматично розблоковує Mutex.

**RwLock\<T\> (Read-Write Lock)** — замок із двома режимами: кілька читачів одночасно (shared lock) АБО один писець (exclusive lock).

**Deadlock (взаємне блокування)** — ситуація, коли два або більше потоків вічно чекають один одного, кожен тримаючи ресурс, потрібний іншому.

**Atomic operation (атомарна операція)** — операція, що виконується повністю або не виконується зовсім, без можливості "частково завершитись". Інший потік бачить або стан до операції, або після — ніколи "посередині".

---

## Мотиваційний кейс

Рій із 10 БПЛА сканує зону пошуку. Кожен дрон знаходить об'єкти та записує їх координати у спільну карту. Координатор читає карту, щоб призначити дронам нові сектори, уникаючи повторного сканування. Без синхронізації: дрон A записує об'єкт у HashMap, одночасно дрон B читає HashMap — data race, пошкоджена структура, undefined behavior. З Mutex: лише один потік має доступ до HashMap у кожен момент — запис та читання ніколи не перекриваються. З RwLock: 9 дронів читають карту одночасно (планування маршруту), поки 1 пише (нове виявлення) — краща пропускна здатність.

---

## 37.1. Від Rc до Arc: атомарний підрахунок

У Розділі 33 ви використовували Rc\<T\> для спільного володіння в однопотоковому коді. Rc не є Send — компілятор заборонить його в потоках. Причина: лічильник посилань Rc — звичайна змінна (usize). Операція "прочитати → збільшити → записати" — три кроки. Якщо два потоки виконують їх одночасно — data race на лічильнику.

Arc (Atomic Reference Counting) вирішує це: лічильник зберігається як `AtomicUsize`, і збільшення/зменшення виконується однією атомарною інструкцією процесора. Атомарна інструкція неподільна: інший потік бачить або стан до, або після — ніколи "посередині". Тому Arc безпечний для одночасного clone та drop з різних потоків.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let config = Arc::new(String::from("Місія: патрулювання зони A"));
    
    let mut handles = Vec::new();
    
    for i in 0..4 {
        let config_clone = Arc::clone(&config); // атомарно збільшує лічильник
        let handle = thread::spawn(move || {
            println!("Потік {}: {}", i, config_clone);
            // config_clone знищується при завершенні потоку
            // → атомарно зменшує лічильник
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Лічильник: {}", Arc::strong_count(&config)); // 1 (лише main)
}
```

`Arc::clone(&config)` — як Rc::clone, але атомарно. Дешева операція (одна атомарна інструкція), не глибоке копіювання. Кожен потік отримує свій Arc, що вказує на одні дані. Дані живуть, поки хоча б один Arc існує.

Різниця між Rc та Arc — лише в реалізації лічильника. API ідентичний. Атомарні операції дещо повільніші за звичайні (десятки наносекунд замість одиниць), тому Rc залишається кращим вибором для однопотокового коду. Не використовуйте Arc де достатньо Rc — це зайвий overhead.

Як і Rc, Arc надає лише іммутабельний доступ: `Arc<T>` дає `&T`, а не `&mut T`. Для мутації потрібен другий механізм — Mutex або RwLock.

---

## 37.2. Mutex\<T\>: ексклюзивний доступ

Mutex (Mutual Exclusion) — замок, що гарантує: лише один потік має доступ до даних у кожен момент часу. Потік "блокує" (lock) мьютекс, працює з даними, потім "розблоковує" (unlock). Інші потоки, що намагаються заблокувати — чекають (блокуються), поки замок не звільниться.

```rust
use std::sync::Mutex;

fn main() {
    let counter = Mutex::new(0); // Mutex обгортає значення
    
    {
        let mut guard = counter.lock().unwrap(); // блокуємо → отримуємо MutexGuard
        *guard += 1;                              // працюємо з даними
        println!("Значення: {}", *guard);
    } // guard знищується → Mutex автоматично розблоковується
    
    // Можна заблокувати знову
    {
        let mut guard = counter.lock().unwrap();
        *guard += 1;
    }
    
    println!("Фінал: {}", *counter.lock().unwrap()); // 2
}
```

Механізм детально. `counter.lock()` повертає `Result<MutexGuard<i32>, PoisonError>`. Якщо мьютекс вільний — потік отримує MutexGuard і продовжує. Якщо зайнятий іншим потоком — поточний потік блокується (зупиняється) і чекає, поки інший не звільнить. Це називається contention — конкуренція за ресурс.

`MutexGuard` — розумний об'єкт, що поводиться як `&mut T`. Через Deref trait ви можете працювати з `*guard` як зі звичайною мутабельною змінною. Критично: при drop (знищенні) MutexGuard автоматично розблоковує мьютекс. Тому RAII-патерн працює: блок `{ let guard = ...; ... }` — guard знищується при виході з блоку, мьютекс звільняється. Не потрібно явного unlock — Rust гарантує, що мьютекс завжди розблоковується, навіть при паніці.

`unwrap()` на lock() — чому? Mutex може бути "отруєний" (poisoned): якщо потік панікував, тримаючи guard — мьютекс позначається як poisoned. Наступний lock() поверне Err(PoisonError), бо дані всередині можуть бути у некоректному стані (паніка могла перервати транзакцію посередині). На практиці poisoning рідко зустрічається, і unwrap/expect прийнятні. Для production-коду — `.lock().unwrap_or_else(|e| e.into_inner())` ігнорує poisoning.

### Чому Mutex, а не просто &mut?

Звичайне правило borrowing: один &mut АБО кілька &. Це перевіряється при компіляції. Але компілятор не може відстежити, який потік тримає посилання у кожен конкретний момент — це залежить від runtime (таймінг, завантаженість CPU). Mutex переносить перевірку "один мутабельний доступ" у runtime: lock() гарантує, що лише один потік тримає guard (тобто &mut T) у кожен момент. Це runtime-аналог borrow checker для міжпотокового доступу.

---

## 37.3. Arc\<Mutex\<T\>\>: стандартний патерн

Arc (спільне володіння між потоками) + Mutex (ексклюзивний мутабельний доступ) = спільні мутабельні дані:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0_u32));
    let mut handles = Vec::new();
    
    for _ in 0..10 {
        let counter_clone = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..1000 {
                let mut guard = counter_clone.lock().unwrap();
                *guard += 1;
            } // guard drop → unlock
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Результат: {}", *counter.lock().unwrap());
    // Завжди 10000 — жодних data races
}
```

10 потоків, кожен збільшує лічильник 1000 разів. Без Mutex результат був би непередбачуваним (менше 10000 через data races). З Mutex — завжди рівно 10000. Кожне збільшення захищене lock: прочитати → збільшити → записати → unlock — атомарно з точки зору інших потоків.

Шаблон `Arc<Mutex<T>>` — аналог `Rc<RefCell<T>>` з Розділу 33, але thread-safe:

| Однопотоковий | Багатопотоковий | Призначення |
|---------------|-----------------|-------------|
| Rc\<T\> | Arc\<T\> | Спільне володіння |
| RefCell\<T\> | Mutex\<T\> | Interior mutability |
| Rc\<RefCell\<T\>\> | Arc\<Mutex\<T\>\> | Спільне + мутабельне |
| Cell\<T\> | AtomicU32, AtomicBool | Прості мутабельні значення |

Правило вибору: якщо однопотоковий код — Rc/RefCell (дешевше). Якщо багатопотоковий — Arc/Mutex (thread-safe). Не використовуйте Arc/Mutex де достатньо Rc/RefCell — атомарні операції та блокування мають overhead.

---

## 37.4. RwLock\<T\>: багато читачів або один писець

Mutex блокує для будь-якого доступу — навіть для читання. Якщо 9 потоків лише читають і 1 пише — Mutex змушує всіх чекати по черзі. RwLock (Read-Write Lock) оптимізує це: кілька потоків можуть тримати shared read lock одночасно, але write lock — ексклюзивний.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    let mut handles = Vec::new();
    
    // 5 потоків-читачів
    for i in 0..5 {
        let data_clone = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let read_guard = data_clone.read().unwrap(); // shared lock
            println!("Читач {}: {:?}", i, *read_guard);
            // Кілька read_guard можуть існувати одночасно
        }));
    }
    
    // 1 потік-писець
    {
        let data_clone = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let mut write_guard = data_clone.write().unwrap(); // exclusive lock
            write_guard.push(4);
            println!("Писець: додав елемент");
            // Поки write_guard існує — жоден read не може отримати lock
        }));
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Фінал: {:?}", *data.read().unwrap());
}
```

`.read()` повертає `RwLockReadGuard` — як &T, shared, кілька одночасно. `.write()` повертає `RwLockWriteGuard` — як &mut T, ексклюзивний. Це runtime-версія правила Rust "кілька & АБО один &mut".

Коли RwLock, коли Mutex? RwLock кращий, коли читання значно частіше за запис (наприклад, карта світу: всі агенти читають, рідко один оновлює). Mutex простіший, коли читання та запис приблизно однаково часті, або коли lock тримається дуже коротко (overhead RwLock не окупається). На практиці починайте з Mutex — він простіший і достатній для більшості випадків. Переходьте на RwLock якщо профілювання показує contention на read.

---

## 37.5. Deadlocks: пастка взаємного блокування

Deadlock — найпоширеніша логічна помилка з мьютексами. Rust запобігає data races при компіляції, але deadlocks — ні, бо це логічна помилка порядку операцій.

Класичний сценарій: два мьютекси, два потоки, зворотний порядок блокування:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let resource_a = Arc::new(Mutex::new(String::from("A")));
    let resource_b = Arc::new(Mutex::new(String::from("B")));
    
    let a1 = Arc::clone(&resource_a);
    let b1 = Arc::clone(&resource_b);
    let a2 = Arc::clone(&resource_a);
    let b2 = Arc::clone(&resource_b);
    
    // Потік 1: блокує A, потім B
    let t1 = thread::spawn(move || {
        let _guard_a = a1.lock().unwrap();
        println!("T1: тримаю A, чекаю B...");
        thread::sleep(std::time::Duration::from_millis(100));
        let _guard_b = b1.lock().unwrap(); // Чекає, бо T2 тримає B
        println!("T1: маю обидва");
    });
    
    // Потік 2: блокує B, потім A (зворотний порядок!)
    let t2 = thread::spawn(move || {
        let _guard_b = b2.lock().unwrap();
        println!("T2: тримаю B, чекаю A...");
        thread::sleep(std::time::Duration::from_millis(100));
        let _guard_a = a2.lock().unwrap(); // Чекає, бо T1 тримає A
        println!("T2: маю обидва");
    });
    
    // DEADLOCK: T1 чекає B (тримає T2), T2 чекає A (тримає T1)
    // Програма зависає назавжди
    t1.join().unwrap();
    t2.join().unwrap();
}
```

T1 тримає A і чекає B. T2 тримає B і чекає A. Жоден не може продовжити — вічне очікування. Програма зависає без помилки чи повідомлення.

Стратегії уникнення deadlocks:

Фіксований порядок блокування: всі потоки блокують мьютекси в однаковому порядку (завжди A перед B). Якщо T1 і T2 обидва блокують спершу A, потім B — deadlock неможливий.

Мінімальний scope lock: тримайте guard якомога коротше. Блокуйте → прочитайте/запишіть → розблокуйте. Не виконуйте тривалі операції (мережа, IO, sleep) тримаючи lock.

Один мьютекс замість кількох: якщо дані пов'язані — об'єднайте їх в одну структуру під одним Mutex замість окремих мьютексів.

try_lock: `mutex.try_lock()` повертає Result одразу (не блокує). Якщо мьютекс зайнятий — Err. Потік може "відступити" і спробувати пізніше замість вічного очікування.

---

## 37.6. Практика: спільна карта рою БПЛА

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;
use std::collections::HashMap;

type SharedMap = Arc<Mutex<HashMap<String, (f64, f64)>>>;

fn simulate_agent(id: String, map: SharedMap, objects_to_find: Vec<(String, f64, f64)>) {
    for (obj_id, x, y) in objects_to_find {
        // Симуляція сканування
        thread::sleep(Duration::from_millis(50));
        
        // Записуємо у спільну карту
        {
            let mut guard = map.lock().unwrap();
            guard.insert(obj_id.clone(), (x, y));
            println!("[{}] Виявлено {} на ({:.0}, {:.0}). Карта: {} об'єктів",
                     id, obj_id, x, y, guard.len());
        } // guard drop → unlock
    }
}

fn main() {
    let world_map: SharedMap = Arc::new(Mutex::new(HashMap::new()));
    
    let agents_data = vec![
        ("SCOUT-01", vec![("OBJ-1", 10.0, 20.0), ("OBJ-2", 15.0, 25.0)]),
        ("SCOUT-02", vec![("OBJ-3", 50.0, 30.0), ("OBJ-4", 55.0, 35.0)]),
        ("RECON-01", vec![("OBJ-5", 80.0, 80.0)]),
    ];
    
    let handles: Vec<_> = agents_data.into_iter().map(|(id, objects)| {
        let map = Arc::clone(&world_map);
        let objects: Vec<(String, f64, f64)> = objects.into_iter()
            .map(|(oid, x, y)| (String::from(oid), x, y))
            .collect();
        thread::spawn(move || simulate_agent(String::from(id), map, objects))
    }).collect();
    
    for h in handles {
        h.join().unwrap();
    }
    
    // Фінальний звіт
    let map = world_map.lock().unwrap();
    println!("\n=== Карта рою: {} об'єктів ===", map.len());
    for (id, (x, y)) in map.iter() {
        println!("  {} → ({:.0}, {:.0})", id, x, y);
    }
}
```

Три потоки-агенти записують у спільну `SharedMap = Arc<Mutex<HashMap<...>>>`. Кожен запис захищений lock. Блок `{ let mut guard = ...; ... }` мінімізує час утримання lock — одразу після insert guard знищується і мьютекс звільняється для інших.

---

## 37.7. Prompt Engineering: дебаг конкурентного коду

### Промпт-шаблон

```
Мій багатопотоковий код:
[вставити код]

Перевір:
1. Чи є потенційні deadlocks (кілька lock у різному порядку)?
2. Чи мінімальний scope lock (не тримаємо guard зайво)?
3. Чи коректний вибір Arc/Mutex vs Arc/RwLock?
4. Чи є зайві clone де достатньо Arc::clone?
```

### Вправа з PE

Напишіть код із навмисним deadlock (два мьютекси, зворотний порядок). Дайте AI цей код і попросіть знайти проблему та запропонувати виправлення. Чи знайшов AI deadlock? Чи запропонував фіксований порядок блокування?

---

## Лабораторна робота No37

### Мета

Реалізувати спільний стан рою через Arc та Mutex.

### Завдання базового рівня

1. `Arc<Mutex<HashMap<String, AgentReport>>>` — спільний реєстр звітів.
2. 4+ потоки-агенти, кожен записує свій звіт.
3. Main thread читає реєстр після join та виводить підсумки.
4. Мінімальний scope lock у кожному потоці.

### Варіанти

**A.** RwLock: реалізувати сценарій "5 читачів, 1 писець" з Arc<RwLock<Vec>>. Порівняти час з Arc<Mutex<Vec>> при 100000 операціях.

**B.** Спільний лічильник завдань: `Arc<Mutex<TaskQueue>>` де потоки беруть завдання з черги та виконують. Main чекає поки черга порожня.

**C.** Deadlock detection: написати код з навмисним deadlock. Додати timeout через try_lock та логування. Продемонструвати, що deadlock виявляється.

### Критерії

| Критерій | Бал |
|----------|-----|
| Arc\<Mutex\<T\>\> з коректним lock/unlock | 25 |
| Мінімальний scope lock | 15 |
| Кілька потоків працюють паралельно | 20 |
| Join та збір результатів | 10 |
| Тести | 20 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `Mutex<T> cannot be shared between threads safely` без Arc

```rust
let m = Mutex::new(42);
let m2 = m; // move — другий потік не отримає
```

Виправлення: обгорнути в Arc: `let m = Arc::new(Mutex::new(42));`.

### Помилка 2: deadlock через lock у тому ж потоці

```rust
let m = Mutex::new(42);
let g1 = m.lock().unwrap();
let g2 = m.lock().unwrap(); // DEADLOCK: потік чекає сам себе
```

Стандартний Mutex у Rust — не рекурсивний. Другий lock у тому ж потоці — deadlock. Виправлення: не блокувати двічі, або використати `parking_lot::ReentrantMutex`.

### Помилка 3: lock тримається занадто довго

```rust
let guard = data.lock().unwrap();
// ... тривала операція (мережевий запит, 500мс) ...
// Всі інші потоки заблоковані 500мс!
```

Виправлення: скопіювати потрібні дані під lock, звільнити lock, потім працювати з копією:

```rust
let snapshot = {
    let guard = data.lock().unwrap();
    guard.clone() // копіюємо під lock
}; // guard drop → unlock
// Працюємо з snapshot без lock
process(snapshot); // тривала операція — інші потоки не заблоковані
```

### Помилка 4: PoisonError

Потік панікував тримаючи lock — мьютекс "отруєний":

```rust
let result = mutex.lock(); // Err(PoisonError) якщо попередній потік панікував
```

Виправлення: `mutex.lock().unwrap_or_else(|e| e.into_inner())` — ігнорувати poisoning і працювати з даними (якщо ви впевнені, що дані у коректному стані).

---

## Додатково

### Atomic types: без Mutex для простих даних

Для простих лічильників та прапорців існують атомарні типи — вони не потребують Mutex:

```rust
use std::sync::atomic::{AtomicU32, AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU32::new(0));
    let running = Arc::new(AtomicBool::new(true));
    
    let c = Arc::clone(&counter);
    let r = Arc::clone(&running);
    let handle = thread::spawn(move || {
        while r.load(Ordering::Relaxed) {
            c.fetch_add(1, Ordering::Relaxed);
            thread::sleep(std::time::Duration::from_millis(10));
        }
    });
    
    thread::sleep(std::time::Duration::from_millis(100));
    running.store(false, Ordering::Relaxed); // сигнал зупинки
    handle.join().unwrap();
    
    println!("Лічильник: {}", counter.load(Ordering::Relaxed));
}
```

AtomicU32, AtomicBool, AtomicUsize — thread-safe без Mutex. Операції (load, store, fetch_add) — одна атомарна інструкція процесора. Значно швидше за Mutex для простих операцій. Ordering::Relaxed — найпростіший режим пам'яті, достатній для лічильників.

### Вибір між підходами до спільного стану

| Підхід | Коли використовувати |
|--------|---------------------|
| clone перед spawn | Дані малі, кожен потік потребує свою копію |
| Arc\<T\> (read-only) | Спільне читання без модифікації |
| Arc\<Mutex\<T\>\> | Спільне читання + запис, lock короткий |
| Arc\<RwLock\<T\>\> | Багато читачів, мало писців |
| Atomic types | Прості лічильники та прапорці |
| Channels (Розділ 38) | Обмін повідомленнями замість спільного стану |

---

## Контрольні запитання

### Рівень 1

1. Чим Arc відрізняється від Rc?

Відповідь: Arc використовує атомарний лічильник посилань (thread-safe), Rc — звичайний (не thread-safe). Arc можна ділити між потоками (Send + Sync), Rc — ні. Arc дещо повільніший через атомарні операції.

2. Що робить `mutex.lock()`?

Відповідь: якщо мьютекс вільний — блокує його і повертає MutexGuard для доступу до даних. Якщо зайнятий — потік блокується і чекає звільнення. MutexGuard розблоковує мьютекс при drop.

### Рівень 2

3. Чому `Arc<Mutex<T>>`, а не просто `Mutex<T>` між потоками?

Відповідь: Mutex потребує одного власника (ownership). Для кількох потоків потрібне спільне володіння → Arc. Arc дає спільний read-only доступ до Mutex. Mutex дає ексклюзивний мутабельний доступ до T. Разом: спільне мутабельне — кілька потоків можуть по черзі модифікувати T.

4. Чим RwLock кращий за Mutex?

Відповідь: RwLock дозволяє кільком читачам одночасно (shared read lock), тоді як Mutex блокує для будь-якого доступу (навіть читання). Якщо 9 потоків читають і 1 пише — RwLock дозволяє 9 одночасних read, Mutex — лише 1. Але RwLock складніший і може мати writer starvation (писець вічно чекає, бо завжди є читачі).

### Рівень 3

5. Опишіть сценарій deadlock з двома мьютексами та як його уникнути.

Відповідь: потік A блокує mutex1, потім mutex2. Потік B блокує mutex2, потім mutex1. Якщо A заблокував mutex1, а B — mutex2, обидва чекають один одного вічно. Уникнення: фіксований порядок — обидва потоки блокують спершу mutex1, потім mutex2.

### Рівень 4

6. Порівняйте `Arc<Mutex<Vec<T>>>` та channels (`mpsc::channel`) для передачі даних від потоків-агентів до координатора.

Відповідь: Arc<Mutex<Vec>> — спільний мутабельний вектор, кожен потік блокує Mutex для push. Простий, але contention при частих записах від багатьох потоків. Channel — кожен потік відправляє повідомлення через Sender, координатор отримує через Receiver. Без lock contention — кожен send/recv незалежний. Channel зазвичай кращий для producer-consumer патерну. Arc<Mutex> — для даних, що потребують читання з будь-якого потоку (не лише координатора).

---

## Резюме

Arc\<T\> — thread-safe Rc з атомарним лічильником. Спільне іммутабельне володіння між потоками. Arc::clone — дешева атомарна операція.

Mutex\<T\> — ексклюзивний доступ через lock(). MutexGuard поводиться як &mut T та автоматично розблоковує при drop. Мінімізуйте scope lock для зменшення contention.

Arc\<Mutex\<T\>\> — стандартний патерн спільного мутабельного стану. Аналог Rc\<RefCell\<T\>\> для потоків.

RwLock\<T\> — оптимізація для "багато читачів, мало писців". read() — shared, write() — exclusive.

Deadlocks: Rust не запобігає їм автоматично. Стратегії: фіксований порядок блокування, мінімальний scope, один мьютекс замість кількох, try_lock.

---

## Що далі

Arc та Mutex дали спільний мутабельний стан, але з lock contention: потоки чекають один одного. Альтернативний підхід — не ділити стан, а обмінюватись повідомленнями: "share memory by communicating, don't communicate by sharing memory" (Go proverb). У Розділі 38 ви вивчите channels — mpsc (multiple producer, single consumer), sync_channel, crossbeam-channel. Агенти надсилатимуть звіти координатору через Sender, координатор отримуватиме через Receiver — без жодного Mutex, без contention, без ризику deadlock.
