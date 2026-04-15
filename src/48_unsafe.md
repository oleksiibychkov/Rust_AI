# Розділ 48. Unsafe Rust: межі безпеки

## Анотація

Увесь Rust-код, що ви писали до цього моменту — safe Rust. Компілятор гарантував: жодних data races, dangling references, buffer overflows, use-after-free. Ці гарантії забезпечуються ownership system, borrow checker, Send/Sync traits. Але є операції, які компілятор не може перевірити: виклик C-бібліотеки (FFI), робота з апаратними регістрами, оптимізації, що обходять borrow checker. Для таких випадків Rust має `unsafe` — позначку, що каже: "у цьому блоці компілятор не перевіряє певні правила, відповідальність на програмісті". Unsafe — не "поганий код". Це контрольований механізм для ситуацій, де safe Rust недостатній. Стандартна бібліотека сама містить сотні unsafe блоків — Vec, String, HashMap реалізовані через unsafe під капотом, але надають safe API зовні.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити, що unsafe дозволяє і чого не дозволяє.
2. Перелічити п'ять "суперсил" unsafe.
3. Працювати з raw pointers (`*const T`, `*mut T`).
4. Викликати C-функції через FFI (`extern "C"`).
5. Мінімізувати unsafe surface через safe обгортки.
6. Пояснити, чому unsafe не означає "небезпечний код".

---

## Ключові терміни

**unsafe block** — блок коду `unsafe { ... }`, де доступні додаткові операції, що компілятор не перевіряє.

**Raw pointer** — `*const T` (іммутабельний) та `*mut T` (мутабельний) — вказівники без гарантій ownership та borrowing. Можуть бути null, dangling, або aliased.

**FFI (Foreign Function Interface)** — механізм виклику функцій, написаних іншою мовою (зазвичай C), з Rust-коду.

**Undefined Behavior (UB)** — стан, при якому програма може робити будь-що: працювати "правильно", краш, пошкодження даних. Компілятор має право оптимізувати UB-код довільно.

**Safe abstraction** — unsafe код, обгорнутий у safe API, що гарантує правильне використання.

---

## Мотиваційний кейс

БПЛА використовує C-бібліотеку для управління камерою (драйвер апаратури). Бібліотека надає функції: `camera_init()`, `camera_capture()`, `camera_release()`. Вони написані на C — Rust-компілятор не може перевірити їхні гарантії безпеки. Виклик C-функції з Rust — це unsafe: компілятор не знає, чи camera_capture повертає валідний вказівник, чи camera_release не викликається двічі. Ваша задача — створити safe обгортку (`struct Camera`), що інкапсулює unsafe виклики та надає safe API зовні. Користувач Camera ніколи не бачить unsafe — він працює зі звичайним Rust-типом.

---

## 48.1. Що unsafe дозволяє: п'ять суперсил

unsafe block надає доступ до п'яти операцій, заборонених у safe Rust:

1. Розіменування raw pointer: `*raw_ptr` — доступ до значення за raw pointer. Може бути null, dangling, або вказувати на невалідну пам'ять.

2. Виклик unsafe функції: `unsafe fn dangerous()` — функція, що вимагає дотримання певних інваріантів, які компілятор не може перевірити.

3. Доступ до mutable static: `static mut COUNTER: u32 = 0` — глобальна мутабельна змінна. Data race при доступі з кількох потоків.

4. Реалізація unsafe trait: `unsafe impl Send for MyType` — програміст гарантує, що тип задовольняє контракт trait.

5. Доступ до полів union: union — тип, де кілька полів діляють одну пам'ять (як union у C).

Усе інше — навіть у unsafe блоці — перевіряється як звичайно. Borrow checker працює. Типи перевіряються. Lifetime аналіз діє. unsafe послаблює лише п'ять конкретних перевірок, не "вимикає всю безпеку".

Поширена помилка: "unsafe = можна робити що завгодно". Ні. unsafe = "п'ять додаткових операцій + відповідальність за відсутність UB". Якщо unsafe-код створює data race, розіменовує null, або порушує aliasing правила — це Undefined Behavior, навіть якщо код "працює" на вашій машині.

---

## 48.2. Raw pointers: вказівники без гарантій

Raw pointers — `*const T` та `*mut T` — аналоги вказівників із C. Без ownership, без lifetime, без null-safety:

```rust
fn main() {
    let x = 42;
    let y = &x as *const i32;  // створення raw pointer — safe!
    
    // Розіменування — unsafe
    unsafe {
        println!("Значення: {}", *y); // 42
    }
    
    // Мутабельний raw pointer
    let mut value = 100;
    let ptr = &mut value as *mut i32;
    
    unsafe {
        *ptr = 200;
        println!("Нове значення: {}", *ptr); // 200
    }
}
```

Створення raw pointer — safe (нічого небезпечного у самому вказівнику). Розіменування (`*ptr`) — unsafe, бо компілятор не гарантує, що вказівник валідний.

Raw pointers дозволяють те, що & та &mut забороняють:

```rust
fn main() {
    let mut data = 42;
    
    // Два мутабельних raw pointer на одні дані — safe Rust заборонив би
    let ptr1 = &mut data as *mut i32;
    let ptr2 = &mut data as *mut i32;
    
    unsafe {
        *ptr1 = 100;
        *ptr2 = 200; // перезаписує те, що записав ptr1
        println!("{}", *ptr1); // 200
    }
}
```

У safe Rust два &mut до одних даних — помилка компіляції. З raw pointers — компілюється, але відповідальність на програмісті: якщо два потоки одночасно запишуть через ptr1 та ptr2 — UB.

Коли raw pointers потрібні: FFI (C-функції повертають/приймають вказівники), реалізація структур даних (зв'язний список, B-дерево — потребують складних зв'язків між вузлами), оптимізації (обхід borrow checker де безпечність доведена, але компілятор не може це побачити).

---

## 48.3. FFI: виклик C-коду з Rust

FFI (Foreign Function Interface) — механізм виклику функцій із C-бібліотек. Оголошення зовнішньої функції через `extern "C"`:

```rust
// Оголошення C-функцій
extern "C" {
    fn abs(x: i32) -> i32;         // з libc
    fn sqrt(x: f64) -> f64;       // з libm
    fn strlen(s: *const u8) -> usize; // з libc
}

fn main() {
    unsafe {
        println!("abs(-42) = {}", abs(-42));      // 42
        println!("sqrt(2.0) = {:.4}", sqrt(2.0)); // 1.4142
        
        let s = b"hello\0"; // C-рядок потребує null-terminator
        println!("strlen = {}", strlen(s.as_ptr()));  // 5
    }
}
```

`extern "C"` означає: "ці функції існують у C-бібліотеці, використовують C calling convention". Виклик — unsafe, бо Rust не може перевірити: чи функція існує, чи правильні типи параметрів, чи немає UB всередині.

Для реальних C-бібліотек потрібен лінкер. Cargo.toml:

```toml
[build-dependencies]
cc = "1"  # для компіляції C-коду
```

Або системна бібліотека:

```rust
#[link(name = "m")] // лінкувати з libm
extern "C" {
    fn cbrt(x: f64) -> f64; // cube root
}
```

---

## 48.4. Safe обгортка: мінімізація unsafe surface

Головний принцип: unsafe код повинен бути мінімальним та інкапсульованим у safe API. Користувач вашого коду ніколи не повинен писати unsafe.

```rust
// Unsafe FFI
extern "C" {
    fn c_sensor_init(id: i32) -> *mut std::ffi::c_void;
    fn c_sensor_read(handle: *mut std::ffi::c_void) -> f64;
    fn c_sensor_close(handle: *mut std::ffi::c_void);
}

// Safe обгортка
pub struct Sensor {
    handle: *mut std::ffi::c_void,
}

impl Sensor {
    pub fn new(id: i32) -> Option<Self> {
        let handle = unsafe { c_sensor_init(id) };
        if handle.is_null() {
            None // ініціалізація провалилась
        } else {
            Some(Sensor { handle })
        }
    }
    
    pub fn read(&self) -> f64 {
        unsafe { c_sensor_read(self.handle) }
    }
}

impl Drop for Sensor {
    fn drop(&mut self) {
        unsafe { c_sensor_close(self.handle); }
    }
}

// Користувач бачить лише safe API:
fn main() {
    if let Some(sensor) = Sensor::new(1) {
        println!("Показання: {:.2}", sensor.read());
        // drop автоматично викличе c_sensor_close
    }
}
```

Три unsafe виклики інкапсульовані в Sensor. Зовнішній код працює зі звичайним Rust-типом: Option для обробки помилок ініціалізації, Drop для автоматичного звільнення ресурсу, жоден unsafe не видимий користувачу.

Це RAII-патерн (Resource Acquisition Is Initialization) — той самий, що й для MutexGuard, File, Box. Ресурс захоплюється у конструкторі, звільняється у Drop. Неможливо забути закрити ресурс.

Правило unsafe surface: кількість рядків unsafe повинна бути мінімальною. Чим менше unsafe — тим менше місць, де може бути UB. Ідеально: unsafe лише у найнижчому рівні (FFI виклики), все зовнішнє — safe.

---

## 48.5. unsafe fn: контракт з caller-ом

Функція може бути позначена як unsafe — це означає "caller повинен гарантувати певні умови":

```rust
/// Читає N байтів з буфера.
/// # Safety
/// - `ptr` повинен вказувати на валідний буфер мінімум `len` байтів
/// - Буфер не повинен модифікуватись іншим кодом під час виклику
unsafe fn read_buffer(ptr: *const u8, len: usize) -> Vec<u8> {
    let mut result = Vec::with_capacity(len);
    for i in 0..len {
        result.push(*ptr.add(i));
    }
    result
}

fn main() {
    let data = vec![1_u8, 2, 3, 4, 5];
    
    // Caller гарантує: ptr валідний, len коректний
    let copy = unsafe { read_buffer(data.as_ptr(), data.len()) };
    println!("{:?}", copy);
}
```

Документація `# Safety` — обов'язкова конвенція для unsafe fn. Описує, що caller повинен гарантувати. Без цієї документації інший програміст не знає, які інваріанти дотримувати — і може створити UB.

---

## 48.6. static mut: глобальний мутабельний стан

`static mut` — глобальна мутабельна змінна. Доступ — unsafe, бо будь-який потік може записати одночасно (data race):

```rust
static mut REQUEST_COUNT: u32 = 0;

fn increment() {
    unsafe {
        REQUEST_COUNT += 1; // НЕБЕЗПЕЧНО: data race якщо кілька потоків
    }
}

fn main() {
    increment();
    increment();
    unsafe {
        println!("Запитів: {}", REQUEST_COUNT);
    }
}
```

static mut — майже завжди поганий вибір. Альтернативи: AtomicU32 (thread-safe без Mutex), lazy_static!/OnceLock (ініціалізація один раз), Arc<Mutex> (спільний стан). Використовуйте static mut лише коли жоден safe варіант не працює (наприклад, FFI callback, де C-код очікує глобальну змінну).

---

## 48.7. Prompt Engineering: перевірка unsafe коду

### Промпт-шаблон

```
Ось мій unsafe блок:
[код]

Перевір на:
1. Undefined Behavior (null deref, dangling, data race, aliasing violation)
2. Порушення інваріантів (Safety документація)
3. Чи можна замінити на safe код?
4. Чи мінімальний unsafe surface?
```

---

## Лабораторна робота No48

### Мета

Створити safe обгортку для unsafe операцій.

### Завдання базового рівня

1. Симуляція FFI: extern "C" функції (можна фейкові), safe struct-обгортка з Drop.
2. Raw pointer: функція, що працює з &[T] через raw pointers, safe обгортка.
3. Документація # Safety для кожної unsafe fn.
4. Тести для safe API.

### Варіанти

**A.** FFI з реальною бібліотекою: обгортка для libc функцій (getpid, gethostname).

**B.** Unsafe оптимізація: реалізувати split_at_mut через raw pointers (як стандартна бібліотека), safe обгортка.

**C.** AtomicU32 замість static mut: переписати код зі static mut на atomic, порівняти безпеку.

### Критерії

| Критерій | Бал |
|----------|-----|
| unsafe блоки з конкретною причиною | 20 |
| Safe обгортка (жоден unsafe видимий зовні) | 25 |
| # Safety документація | 15 |
| Drop для автоматичного звільнення | 15 |
| Тести safe API | 15 |
| Мінімальний unsafe surface | 10 |

---

## Troubleshooting

### Помилка 1: `dereference of raw pointer is unsafe and requires unsafe block`

```rust
let ptr: *const i32 = &42;
println!("{}", *ptr); // ПОМИЛКА: потрібен unsafe
```

Виправлення: `unsafe { println!("{}", *ptr); }`.

### Помилка 2: `cannot assign to immutable static item`

```rust
static COUNT: u32 = 0;
COUNT += 1; // ПОМИЛКА: static (не static mut) — іммутабельний
```

Виправлення: `static mut COUNT: u32 = 0;` та unsafe для доступу. Або краще: `static COUNT: AtomicU32 = AtomicU32::new(0);`.

### Помилка 3: segfault при FFI

C-функція отримала невалідний аргумент (null, неправильний тип, некоректний розмір буфера). Rust не перевіряє FFI-аргументи — відповідальність на програмісті.

Виправлення: перевіряйте аргументи перед unsafe викликом. Документуйте вимоги через # Safety.

---

## Додатково

### Як стандартна бібліотека використовує unsafe

Vec<T> внутрішньо використовує raw pointer на heap-буфер та unsafe для доступу. String — обгортка навколо Vec<u8> з unsafe для UTF-8 перевірок. HashMap — складна хеш-таблиця з unsafe для bucket management. Усі надають safe API. Це модель: unsafe всередині → safe зовні.

Кількість unsafe у std: сотні блоків. Кожен — ретельно перевірений, задокументований, протестований. Це нормально — unsafe необхідний для системного коду. Ненормально — unsafe у прикладному коді без вагомої причини.

### unsafe не вимикає borrow checker

Поширене хибне уявлення: "unsafe дозволяє обійти borrow checker". Частково правда — через raw pointers можна мати два мутабельних вказівника. Але якщо ви створите дві &mut через unsafe і використаєте одночасно — це UB, навіть якщо компілюється. unsafe дозволяє операцію, але не гарантує безпеку. Відповідальність за відсутність UB — на програмісті.

---

## Контрольні запитання

### Рівень 1

1. Що unsafe дозволяє?

Відповідь: п'ять операцій: розіменування raw pointer, виклик unsafe fn, доступ до static mut, реалізація unsafe trait, доступ до полів union. Все інше перевіряється як звичайно.

2. Чому unsafe не означає "небезпечний код"?

Відповідь: unsafe означає "компілятор не перевіряє ці п'ять операцій, відповідальність на програмісті". Правильно написаний unsafe код — безпечний. "Небезпечний" — unsafe код із UB (null deref, data race, dangling pointer). unsafe — інструмент, не вирок.

### Рівень 2

3. Чим raw pointer відрізняється від &T?

Відповідь: &T — перевіряється компілятором (lifetime, aliasing, null-safety). *const T — не перевіряється: може бути null, dangling, aliased. Створення raw pointer — safe, розіменування — unsafe. & завжди валідне, * — не обов'язково.

4. Що таке "safe обгортка" і навіщо?

Відповідь: struct з safe API, що інкапсулює unsafe всередині. Користувач не бачить unsafe, працює зі звичайним Rust-типом. Мінімізує кількість коду, де можливий UB. Приклад: Vec — safe API (push, pop, get), unsafe — лише внутрішня реалізація буфера.

### Рівень 3

5. Чому static mut — поганий вибір і які альтернативи?

Відповідь: static mut — data race при доступі з кількох потоків (UB). Кожен доступ — unsafe, легко забути. Альтернативи: AtomicU32/AtomicBool (thread-safe, lock-free), OnceLock (ініціалізація один раз), Arc<Mutex> (спільний захищений стан). Всі — safe, thread-safe.

### Рівень 4

6. Якщо std використовує сотні unsafe — чому ми кажемо, що Rust "безпечний"?

Відповідь: безпека Rust — у моделі "unsafe всередині, safe зовні". Vec, String, HashMap містять unsafe, але надають safe API. Прикладний код використовує safe API і не може створити UB. unsafe у std ретельно перевірений (fuzzing, Miri, code review). Гарантія Rust: safe код не може створити UB, навіть якщо бібліотеки всередині використовують unsafe. Це layered safety — безпека шарами.

---

## Резюме

unsafe — контрольований механізм для п'яти операцій, які компілятор не може перевірити: raw pointer deref, unsafe fn call, static mut, unsafe trait impl, union fields. Все інше перевіряється нормально.

Raw pointers (*const T, *mut T) — вказівники без гарантій. Створення — safe, розіменування — unsafe. Потрібні для FFI та низькорівневого коду.

FFI (extern "C") — виклик C-функцій з Rust. Unsafe, бо Rust не може перевірити C-код.

Safe обгортка — головний патерн: unsafe інкапсульований у struct із safe API. Drop для автоматичного звільнення. Мінімальний unsafe surface.

static mut — уникайте. AtomicU32, OnceLock, Arc<Mutex> — safe альтернативи.

---

## Що далі

Розділ 49 повністю присвячений просунутому Prompt Engineering: chain-of-thought, few-shot, системні промпти, ітеративний промптинг, робота з великим контекстом, AI для code review та рефакторингу, обмеження та галюцинації. Це завершальний PE-розділ, що узагальнює весь досвід курсу у професійний workflow з AI-асистентом.
