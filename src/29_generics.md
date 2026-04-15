# Розділ 29. Generics та Trait Bounds

## Анотація

У Розділі 28 ви побачили два підходи до поліморфізму: trait objects (runtime, гнучкий) та коротке згадування generics (compile-time, zero-cost). Цей розділ повністю присвячений другому підходу. Generics дозволяють написати одну функцію, структуру або impl-блок, що працює з будь-яким типом — але компілятор генерує окрему оптимізовану версію для кожного конкретного типу. Результат — код, що одночасно узагальнений (один раз написали — працює з i32, f64, String, Drone) та ефективний (жодного runtime overhead). Trait bounds обмежують параметр типу: "приймаю будь-який T, але лише якщо T реалізує Clone + Debug". Це як контракт: generics дають гнучкість, trait bounds — гарантії.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Написати generic функцію з параметром типу `<T>`.
2. Створити generic struct та impl-блок для неї.
3. Пояснити monomorphization та чому generics — zero-cost.
4. Використати trait bounds: `T: Clone + Debug`, `where` clauses.
5. Використати `impl Trait` у параметрах та поверненні.
6. Пояснити Sized trait та коли потрібен `?Sized`.

---

## Ключові терміни

**Generic (узагальнення)** — параметризація коду типом, що визначається при використанні, а не при оголошенні. `fn max<T>(a: T, b: T) -> T` працює з будь-яким T.

**Type parameter (параметр типу)** — `T`, `U`, `E` у кутових дужках — плейсхолдер для конкретного типу.

**Trait bound (обмеження трейтом)** — `T: Display + Clone` — вимога, що T реалізує вказані traits. Без bound не можна викликати методи trait для T.

**Monomorphization (мономорфізація)** — процес, при якому компілятор створює окрему версію generic-коду для кожного конкретного типу. Generics "зникають" у машинному коді.

**where clause** — альтернативний синтаксис для trait bounds: `where T: Display + Clone`. Зручніший для складних bounds.

---

## Мотиваційний кейс

БПЛА-агент має кілька алгоритмів, що працюють з різними типами даних: пошук найближчого об'єкта (серед координат, серед агентів, серед waypoints), знаходження максимуму (висоти, швидкості, батареї), фільтрація (показань сенсора, подій логу, команд). Без generics кожен алгоритм потрібно написати окремо для кожного типу: `find_nearest_position`, `find_nearest_agent`, `find_nearest_waypoint` — три функції з ідентичною логікою, але різними типами. З generics: одна функція `find_nearest<T: HasPosition>(items: &[T], target: (f64, f64)) -> Option<&T>` — працює з будь-яким типом, що має координати.

---

## 29.1. Проблема дублювання: навіщо generics

Розглянемо задачу: знайти максимальне значення у слайсі. Для i32:

```rust
fn max_i32(data: &[i32]) -> Option<&i32> {
    if data.is_empty() { return None; }
    let mut max = &data[0];
    for item in &data[1..] {
        if item > max { max = item; }
    }
    Some(max)
}
```

Для f64 — ідентичний код, лише тип інший:

```rust
fn max_f64(data: &[f64]) -> Option<&f64> {
    if data.is_empty() { return None; }
    let mut max = &data[0];
    for item in &data[1..] {
        if item > max { max = item; }
    }
    Some(max)
}
```

Для u8, i64, f32 — ще три копії. П'ять функцій з ідентичною логікою. Виправлення бага в одній потребує виправлення в усіх п'яти. Це порушення принципу DRY (Don't Repeat Yourself) — і саме те, що generics вирішують.

---

## 29.2. Generic функції: один код для всіх типів

Generic функція параметризована типом `T`:

```rust
fn max_value<T: PartialOrd>(data: &[T]) -> Option<&T> {
    if data.is_empty() { return None; }
    let mut max = &data[0];
    for item in &data[1..] {
        if item > max { max = item; }
    }
    Some(max)
}

fn main() {
    let ints = vec![3, 7, 1, 9, 4];
    let floats = vec![10.5, 3.2, 22.1, 8.7];
    let names = vec!["Сокіл", "Орел", "Беркут"];
    
    println!("Макс int: {:?}", max_value(&ints));       // Some(9)
    println!("Макс float: {:?}", max_value(&floats));   // Some(22.1)
    println!("Макс name: {:?}", max_value(&names));     // Some("Сокіл") — лексикографічно
}
```

`<T: PartialOrd>` означає: "T — будь-який тип, що реалізує PartialOrd (тобто підтримує `>`). Одна функція замість п'яти. Працює з i32, f64, &str, і будь-яким іншим типом, що реалізує PartialOrd.

Чому потрібен `T: PartialOrd`? Без нього компілятор не знає, що для T можна використовувати `>`. Спробуйте видалити bound:

```rust
fn max_value<T>(data: &[T]) -> Option<&T> {
    // ...
    if item > max { // ПОМИЛКА: binary operation `>` cannot be applied to type `&T`
    // ...
}
```

Компілятор каже: "я не знаю, чи T підтримує порівняння". Trait bound — це контракт: ви обіцяєте обмежити T лише типами з PartialOrd, і компілятор дозволяє використовувати `>`.

Це фундаментальна відмінність від Python та JavaScript, де generic-подібний код просто "пробує" виклик і падає у runtime, якщо метод не існує (duck typing). Rust перевіряє все при компіляції: якщо код компілюється — він гарантовано працює для будь-якого T, що задовольняє bound.

---

## 29.3. Monomorphization: як generics стають zero-cost

Коли компілятор бачить `max_value(&ints)` (де ints — Vec<i32>) та `max_value(&floats)` (де floats — Vec<f64>), він генерує дві окремі версії функції:

```rust
// Компілятор генерує (приблизно):
fn max_value_i32(data: &[i32]) -> Option<&i32> { /* ... */ }
fn max_value_f64(data: &[f64]) -> Option<&f64> { /* ... */ }
```

Це monomorphization (мономорфізація): generic-код "розгортається" у конкретні версії для кожного типу. У бінарному файлі generics не існують — лише конкретні функції. Кожна версія оптимізується незалежно: компілятор може інлайнити виклики, використовувати SIMD-інструкції для конкретного типу, оптимізувати порівняння.

Результат: generic-функція працює рівно так само швидко, як написана вручну для конкретного типу. Нуль runtime overhead. Це і є zero-cost abstraction — абстракція, за яку не платиш продуктивністю.

Порівняйте з trait objects (dyn Trait), де виклик іде через vtable у runtime — один рівень непрямості, неможливість інлайнінгу. Generics уникають цього за рахунок monomorphization. Ціна — більший бінарний файл: якщо max_value використовується з 10 різними типами — 10 копій функції у бінарнику. На практиці це рідко проблема.

---

## 29.4. Generic struct та impl

Структури теж можуть бути generic:

```rust
#[derive(Debug)]
struct Pair<T> {
    first: T,
    second: T,
}

impl<T> Pair<T> {
    fn new(first: T, second: T) -> Self {
        Pair { first, second }
    }
    
    fn into_tuple(self) -> (T, T) {
        (self.first, self.second)
    }
}

// Методи лише для Pair<T> де T: Display
impl<T: std::fmt::Display> Pair<T> {
    fn print(&self) {
        println!("({}, {})", self.first, self.second);
    }
}

// Методи лише для Pair<f64>
impl Pair<f64> {
    fn sum(&self) -> f64 {
        self.first + self.second
    }
}

fn main() {
    let coords = Pair::new(47.5, 35.2);
    coords.print();    // (47.5, 35.2) — Display для f64
    println!("Сума: {}", coords.sum()); // 82.7 — лише для Pair<f64>
    
    let names = Pair::new("Сокіл", "Орел");
    names.print();     // (Сокіл, Орел) — Display для &str
    // names.sum();    // ПОМИЛКА: sum існує лише для Pair<f64>
}
```

Зверніть увагу на три impl-блоки. `impl<T> Pair<T>` — методи для будь-якого T (new, into_tuple). `impl<T: Display> Pair<T>` — методи лише для T з Display (print). `impl Pair<f64>` — методи лише для конкретного типу f64 (sum). Це потужна система: різні набори методів для різних обмежень на T.

Ви вже використовували generic struct, можливо не усвідомлюючи: `Vec<T>`, `HashMap<K, V>`, `Option<T>`, `Result<T, E>` — все це generic types. `Vec<i32>` та `Vec<String>` — це мономорфізовані версії одного generic Vec.

Структура з кількома параметрами типу:

```rust
#[derive(Debug)]
struct KeyValue<K, V> {
    key: K,
    value: V,
}

fn main() {
    let entry = KeyValue { key: String::from("altitude"), value: 150.0_f64 };
    println!("{:?}", entry);
    // KeyValue { key: "altitude", value: 150.0 }
}
```

---

## 29.5. Trait bounds: обмеження параметра типу

Trait bounds — це "фільтр" на T: "приймаю не будь-який тип, а лише той, що реалізує певні traits". Без bounds T — абсолютно невідомий тип, і єдине, що з ним можна зробити — перемістити (move) або повернути (return).

Один bound:

```rust
fn print_all<T: std::fmt::Display>(items: &[T]) {
    for item in items {
        println!("{}", item); // Display потрібен для {}
    }
}
```

Множинні bounds через `+`:

```rust
fn print_sorted<T: std::fmt::Display + Ord>(items: &mut [T]) {
    items.sort();           // Ord потрібен для sort
    for item in items.iter() {
        println!("{}", item); // Display потрібен для {}
    }
}
```

`T: Display + Ord` означає: T повинен реалізувати і Display, і Ord. Кожен `+` додає ще одну вимогу.

Коли bounds стають довгими — `where` clause зручніший:

```rust
// Без where — довгий рядок
fn complex<T: Display + Clone + PartialOrd + Default, U: Debug + Hash>(t: &T, u: &U) {}

// З where — кожен bound на окремому рядку
fn complex<T, U>(t: &T, u: &U)
where
    T: Display + Clone + PartialOrd + Default,
    U: Debug + std::hash::Hash,
{
    // ...
}
```

`where` — це той самий синтаксис, лише перенесений після параметрів. Функціонально ідентичний. Обирайте за читабельністю: для 1–2 простих bounds — inline `<T: Trait>`, для складних — `where`.

---

## 29.6. impl Trait: спрощений синтаксис

`impl Trait` у позиції параметра — синтаксичний цукор для generics:

```rust
// Ці два записи еквівалентні:
fn print_status(agent: &impl Agent) { /* ... */ }
fn print_status<A: Agent>(agent: &A) { /* ... */ }
```

Компілятор обробляє обидва однаково — static dispatch, monomorphization. Різниця лише у читабельності: `impl Agent` простіший для одного параметра. Generics необхідні, коли параметр типу використовується у кількох місцях:

```rust
// Потрібен generic: обидва параметри — один тип T
fn swap<T>(a: &mut T, b: &mut T) {
    std::mem::swap(a, b);
}

// impl Trait не може виразити "обидва — один тип":
// fn swap(a: &mut impl Clone, b: &mut impl Clone) — це РІЗНІ типи!
```

`impl Trait` у позиції повернення — інша семантика. Це opaque type: "я повертаю конкретний тип, що реалізує Trait, але caller не знає який":

```rust
fn create_default_agent() -> impl Agent {
    Drone {
        name: String::from("DEFAULT"),
        x: 0.0, y: 0.0, charge: 100, altitude: 0.0,
    }
}
```

Caller бачить лише `impl Agent` — може викликати методи Agent, але не знає, що це Drone. Це приховування реалізації: зміна повертаного типу з Drone на GroundBot не зламає caller-код.

Обмеження: функція повинна повертати один конкретний тип. Не можна повернути Drone в одній гілці та GroundBot в іншій — для цього потрібен `Box<dyn Agent>`.

---

## 29.7. Sized та ?Sized

За замовчуванням кожен generic параметр T має неявний bound `T: Sized` — тип повинен мати відомий розмір при компіляції. Це логічно: щоб виділити місце на стеку для змінної типу T — потрібно знати розмір T.

Але іноді потрібно працювати з типами невідомого розміру — зокрема `str` (рядковий слайс) та `[T]` (слайс). Для цього використовується `?Sized` — "можливо не Sized":

```rust
// Без ?Sized: приймає лише Sized типи
fn print_info<T: std::fmt::Display>(item: &T) {
    println!("{}", item);
}

// З ?Sized: приймає і Sized, і не-Sized типи (str, [T])
fn print_info_flexible<T: std::fmt::Display + ?Sized>(item: &T) {
    println!("{}", item);
}

fn main() {
    let s: &str = "hello";
    // print_info(s);           // працює: &str → T = str, але str не Sized
    print_info_flexible(s);     // працює: ?Sized дозволяє str
    print_info_flexible(&42);   // працює: i32 — Sized
}
```

На практиці `?Sized` потрібен рідко — переважно у бібліотечному коді, що повинен бути максимально гнучким. Для навчальних проєктів — зазвичай не потрібен. Але знати про нього важливо, бо ви зустрінете `?Sized` у сигнатурах стандартної бібліотеки.

---

## 29.8. Практика: generic алгоритми для БПЛА

Побудуємо набір узагальнених алгоритмів, що працюють з будь-якими типами через trait bounds.

Перший алгоритм — пошук найближчого. Trait HasPosition абстрагує "щось із координатами":

```rust
trait HasPosition {
    fn pos(&self) -> (f64, f64);
}

fn distance(a: (f64, f64), b: (f64, f64)) -> f64 {
    ((b.0 - a.0).powi(2) + (b.1 - a.1).powi(2)).sqrt()
}

fn find_nearest<'a, T: HasPosition>(items: &'a [T], target: (f64, f64)) -> Option<&'a T> {
    items.iter()
        .min_by(|a, b| {
            distance(a.pos(), target)
                .partial_cmp(&distance(b.pos(), target))
                .unwrap()
        })
}

// Реалізуємо HasPosition для різних типів

struct Waypoint { x: f64, y: f64, name: String }
struct DetectedObject { x: f64, y: f64, kind: String }

impl HasPosition for Waypoint {
    fn pos(&self) -> (f64, f64) { (self.x, self.y) }
}

impl HasPosition for DetectedObject {
    fn pos(&self) -> (f64, f64) { (self.x, self.y) }
}

fn main() {
    let waypoints = vec![
        Waypoint { x: 10.0, y: 20.0, name: String::from("Alpha") },
        Waypoint { x: 50.0, y: 50.0, name: String::from("Bravo") },
        Waypoint { x: 5.0, y: 5.0, name: String::from("Charlie") },
    ];
    
    let objects = vec![
        DetectedObject { x: 100.0, y: 100.0, kind: String::from("будівля") },
        DetectedObject { x: 20.0, y: 25.0, kind: String::from("транспорт") },
    ];
    
    let target = (15.0, 15.0);
    
    // Одна функція — різні типи
    if let Some(wp) = find_nearest(&waypoints, target) {
        println!("Найближчий waypoint: {}", wp.name);
    }
    if let Some(obj) = find_nearest(&objects, target) {
        println!("Найближчий об'єкт: {}", obj.kind);
    }
}
```

Одна функція `find_nearest<T: HasPosition>` працює і з waypoints, і з detected objects, і з будь-яким майбутнім типом, що реалізує HasPosition. Monomorphization створить дві оптимізовані версії — для Waypoint та для DetectedObject.

Другий алгоритм — generic статистика:

```rust
fn statistics<T, F>(items: &[T], extract: F) -> (f64, f64, f64)
where
    F: Fn(&T) -> f64,
{
    if items.is_empty() {
        return (0.0, 0.0, 0.0);
    }
    
    let values: Vec<f64> = items.iter().map(|item| extract(item)).collect();
    let sum: f64 = values.iter().sum();
    let count = values.len() as f64;
    let avg = sum / count;
    let min = values.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = values.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    
    (avg, min, max)
}

fn main() {
    let waypoints = vec![
        Waypoint { x: 10.0, y: 20.0, name: String::from("A") },
        Waypoint { x: 50.0, y: 50.0, name: String::from("B") },
        Waypoint { x: 5.0, y: 5.0, name: String::from("C") },
    ];
    
    // Статистика по X-координатах
    let (avg, min, max) = statistics(&waypoints, |wp| wp.x);
    println!("X: avg={:.1}, min={:.1}, max={:.1}", avg, min, max);
    
    // Статистика по відстанях від бази
    let base = (0.0, 0.0);
    let (avg, min, max) = statistics(&waypoints, |wp| distance(wp.pos(), base));
    println!("Відстань: avg={:.1}, min={:.1}, max={:.1}", avg, min, max);
}
```

Функція `statistics` generic по двох параметрах: T (тип елементів) та F (функція витягування числа). `F: Fn(&T) -> f64` — trait bound, що означає "F — замикання, що приймає &T і повертає f64". Це дозволяє обчислити статистику за будь-яким числовим полем будь-якої структури.

## 29.9. Prompt Engineering: узагальнення коду через generics

### Промпт-шаблон

```
Ось три функції, що роблять подібне для різних типів:
[вставити код трьох функцій]

Узагальни їх через одну generic-функцію.
Поясни:
1. Які trait bounds потрібні і чому
2. Чи потрібен where clause чи inline bounds
3. Чи можна зробити ще узагальненіше
```

### Промпт-шаблон: рев'ю generic коду

```
Переглянь мій generic код:
[вставити код]
Перевір:
1. Чи мінімальні trait bounds (не зайві)?
2. Чи не занадто generic (потрібен T де достатньо конкретного типу)?
3. Чи працює monomorphization ефективно?
```

### Вправа з PE

Візьміть три функції з попередніх розділів, що працюють з конкретними типами (наприклад, find_nearest для Position, для DetectedObject, для Waypoint). Узагальніть через generics самостійно. Потім попросіть AI зробити те саме. Порівняйте: чи однакові trait bounds? Хто обрав мінімальніші?

---

## Лабораторна робота No29

### Мета

Замінити дубльований код узагальненими функціями та структурами.

### Завдання базового рівня

1. Generic `fn find_nearest<T: HasPosition>(items: &[T], target: (f64, f64)) -> Option<&T>`.
2. Generic `fn filter_by_radius<T: HasPosition>(items: &[T], center: (f64, f64), radius: f64) -> Vec<&T>`.
3. Generic struct `BoundedBuffer<T> { data: Vec<T>, max_size: usize }` з методами push (видаляє найстаріший при переповненні), iter, len.
4. Тести для кожної функції з двома різними типами, що реалізують HasPosition.

### Варіанти

**A.** Generic priority queue: `PriorityQueue<T: Ord>` з методами push, pop (повертає найбільший), peek, len. Тести з числами та кастомним типом.

**B.** Generic pipeline: `fn process_pipeline<T, F1, F2>(data: Vec<T>, filter: F1, transform: F2) -> Vec<T>` де F1: Fn(&T) -> bool, F2: Fn(T) -> T. Продемонструвати з показаннями сенсорів.

**C.** Порівняння monomorphization: написати одну функцію generic та одну через dyn Trait. Використати `cargo build --release` та порівняти розмір бінарників. Дослідити через `cargo expand`, у що розгортається generic.

### Критерії

| Критерій | Бал |
|----------|-----|
| Generic функції з правильними trait bounds | 25 |
| Generic struct з методами | 20 |
| Мінімальні bounds (не зайві) | 15 |
| Тести з різними типами | 20 |
| Документація (чому обрані саме ці bounds) | 10 |
| Читабельність | 10 |

---

## Troubleshooting

### Помилка 1: `the trait bound T: Display is not satisfied`

Спроба використати метод trait без відповідного bound:

```rust
fn show<T>(item: &T) {
    println!("{}", item); // ПОМИЛКА: T не гарантує Display
}
```

Виправлення: додати bound `fn show<T: std::fmt::Display>(item: &T)`.

### Помилка 2: `the trait PartialOrd is not implemented for T`

Спроба порівнювати generic значення без bound:

```rust
fn max<T>(a: T, b: T) -> T {
    if a > b { a } else { b } // ПОМИЛКА: > потребує PartialOrd
}
```

Виправлення: `fn max<T: PartialOrd>(a: T, b: T) -> T`.

### Помилка 3: `cannot move out of borrowed content` у generic функції

Спроба повернути значення з слайсу без Clone:

```rust
fn first_clone<T>(data: &[T]) -> T {
    data[0] // ПОМИЛКА: не можна перемістити з позиченого слайсу
}
```

Виправлення: додати Clone bound та клонувати: `fn first_clone<T: Clone>(data: &[T]) -> T { data[0].clone() }`. Або повернути посилання: `fn first<T>(data: &[T]) -> &T { &data[0] }`.

### Помилка 4: `impl Trait не можна використати як тип у struct`

```rust
struct Container {
    item: impl Display, // ПОМИЛКА: impl Trait — лише у параметрах та поверненні функцій
}
```

Виправлення: зробити struct generic: `struct Container<T: Display> { item: T }`. Або використати trait object: `struct Container { item: Box<dyn Display> }`.

### Помилка 5: `type annotations needed`

Компілятор не може вивести тип generic параметра:

```rust
let items = Vec::new(); // ПОМИЛКА: тип елементів невідомий
```

Виправлення: або анотація `let items: Vec<i32> = Vec::new()`, або turbofish `Vec::<i32>::new()`, або додати перший елемент одразу — тип виведеться.

---

## Додатково

### Generics vs макроси: два способи кодогенерації

Generics та макроси (`macro_rules!`) обидва генерують код для різних типів. Різниця: generics перевіряються компілятором (type-safe, trait bounds гарантують коректність), а макроси — лише текстова підстановка (помилки з'являються після розгортання, повідомлення незрозуміліші). Generics — для типізованого узагальнення. Макроси — для синтаксичного узагальнення (повторюваний boilerplate, DSL). Якщо задача вирішується generics — використовуйте generics.

### Turbofish: явне задання типу

Іноді компілятор не може вивести generic параметр. "Turbofish" синтаксис `::<Type>` задає його явно:

```rust
let numbers: Vec<i32> = "1 2 3".split_whitespace()
    .map(|s| s.parse::<i32>().unwrap())
    .collect::<Vec<i32>>();
```

`::<i32>` після parse та `::<Vec<i32>>` після collect — це turbofish. Назва неофіційна і походить від візуальної схожості `::<>` з рибою.

### Phantom types та zero-sized types

Іноді generic параметр не використовується у полях struct, а лише для type-safety:

```rust
use std::marker::PhantomData;

struct Meters;
struct Feet;

struct Distance<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl Distance<Meters> {
    fn to_feet(self) -> Distance<Feet> {
        Distance { value: self.value * 3.28084, _unit: PhantomData }
    }
}
```

`Distance<Meters>` та `Distance<Feet>` — різні типи, хоча обидва містять лише f64. Компілятор не дозволить додати `Distance<Meters>` до `Distance<Feet>` — unit mismatch при компіляції. Це phantom type — тип-привид, що існує лише для системи типів і не займає пам'яті.

---

## Контрольні запитання

### Рівень 1

1. Що таке generic параметр типу?

Відповідь: плейсхолдер для конкретного типу в кутових дужках (`<T>`). Конкретний тип визначається при використанні: `max_value::<i32>(...)` або виводиться компілятором.

2. Навіщо trait bounds?

Відповідь: без bounds T — невідомий тип, і для нього не можна викликати жодні методи. Trait bound `T: Display` гарантує, що T реалізує Display, і дозволяє використовувати `{}` для T.

### Рівень 2

3. Що таке monomorphization і чому generics — zero-cost?

Відповідь: компілятор генерує окрему версію generic-коду для кожного конкретного типу (`max_value::<i32>`, `max_value::<f64>`). У бінарному файлі generics не існують — лише конкретні функції з прямими викликами, без vtable чи indirection.

4. Чим `fn f(x: &impl Trait)` відрізняється від `fn f(x: &dyn Trait)`?

Відповідь: `impl Trait` — static dispatch, monomorphization, zero-cost, але один конкретний тип при кожному виклику. `dyn Trait` — dynamic dispatch, vtable, мінімальний overhead, але може приймати різні типи у runtime та зберігати у Vec.

### Рівень 3

5. Напишіть generic функцію `fn summarize<T: Display>(items: &[T]) -> String`, що повертає рядок "item1, item2, item3".

Відповідь:

```rust
fn summarize<T: std::fmt::Display>(items: &[T]) -> String {
    items.iter()
        .map(|item| item.to_string())
        .collect::<Vec<_>>()
        .join(", ")
}
```

### Рівень 4

6. Чому `fn f<T>(x: T) {}` неявно означає `fn f<T: Sized>(x: T) {}`? Коли потрібен `?Sized`?

Відповідь: щоб передати x за значенням (на стеку), компілятор повинен знати розмір T при компіляції. Тому T: Sized за замовчуванням. `?Sized` потрібен для роботи з типами невідомого розміру (`str`, `[T]`) через посилання: `fn f<T: ?Sized>(x: &T)`. Без `?Sized` функція не зможе прийняти `&str` де T = str.

---

## Резюме

Generics параметризують код типом: одна функція/struct працює з багатьма типами. Monomorphization створює окрему версію для кожного конкретного типу — zero-cost abstraction без runtime overhead.

Trait bounds обмежують T: `T: Display + Clone` гарантує, що T реалізує Display та Clone. Без bounds не можна використовувати методи trait для T. `where` clause — альтернативний синтаксис для складних bounds.

`impl Trait` у параметрах — цукор для generics (static dispatch). `impl Trait` у поверненні — opaque type (один конкретний тип, прихований від caller).

Generics vs trait objects: generics — compile-time, zero-cost, один тип. Trait objects — runtime, мінімальний overhead, різні типи. Generics vs макроси: generics type-safe, макроси — текстова підстановка.

---

## Що далі

Частина III завершується Розділом 30 — Практикумом v2.0. Ви об'єднаєте все, що вивчили з Розділів 21–29: динамічні колекції, ітератори, обробку помилок, traits та generics — у рефакторингу агента з v1.0 до v2.0. Агент стане робастною системою: працює з динамічними даними, обробляє помилки gracefully, має абстрактний інтерфейс Agent, і готовий до багатопотоковості у Частині IV.
