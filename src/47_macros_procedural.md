# Розділ 47. Макроси: процедурні

## Анотація

У Розділі 46 macro_rules! генерував код за патернами токенів. Це потужно для DSL та простої кодогенерації, але обмежено: macro_rules! не може "зрозуміти" структуру Rust-коду — парсити поля struct, аналізувати типи, генерувати impl на основі визначення типу. Процедурні макроси знімають це обмеження: вони отримують TokenStream (потік токенів Rust-коду), можуть парсити його через бібліотеку `syn`, аналізувати структуру, і генерувати новий код через `quote`. Саме так працюють `#[derive(Debug)]`, `#[derive(Serialize)]`, `#[tokio::main]`, `#[instrument]` — все, що ви використовували через derive та атрибути.

---

## Цілі навчання

Після опрацювання цього розділу студент зможе:

1. Пояснити три типи процедурних макросів: derive, attribute, function-like.
2. Створити крейт для процедурного макросу.
3. Використати syn для парсингу TokenStream.
4. Використати quote для генерації коду.
5. Написати derive-макрос для автоматичної генерації impl.
6. Пояснити, чому proc macros живуть в окремому крейті.

---

## Ключові терміни

**Procedural macro (процедурний макрос)** — функція, що приймає TokenStream і повертає TokenStream. Повноцінна програма на Rust, що трансформує код.

**Derive macro** — процедурний макрос, що додає реалізацію trait через `#[derive(MyTrait)]`.

**Attribute macro** — процедурний макрос, що трансформує елемент із атрибутом: `#[my_attr] fn foo() {}`.

**syn** — бібліотека для парсингу TokenStream у AST (Abstract Syntax Tree) Rust-коду.

**quote** — бібліотека для генерації TokenStream з шаблонів: `quote! { impl Display for #name { ... } }`.

**TokenStream** — послідовність токенів Rust (ідентифікатори, літерали, розділювачі). Вхід та вихід процедурного макросу.

---

## Мотиваційний кейс

У Розділі 42 кожен актор потребує impl з run(), Handle з new() та send(). У Розділі 27 кожна struct потребує impl Display вручну. `#[derive(Agent)]` автоматично згенерує базовий impl Agent для будь-якої struct, що має поля id, position, battery — парсючи поля struct та генеруючи відповідний код. Як thiserror генерує Display для ваших помилок, як serde генерує Serialize — ви зможете створювати свої derive-макроси.

---

## 47.1. Три типи процедурних макросів

Derive macros — найпоширеніший тип. Додають impl для типу:

```rust
#[derive(Debug, Clone, MyCustomTrait)]
struct Agent { id: String, battery: u8 }
```

Компілятор бачить `#[derive(MyCustomTrait)]`, знаходить процедурний макрос MyCustomTrait, передає йому TokenStream визначення struct Agent, і додає згенерований код (impl MyCustomTrait for Agent) до програми.

Attribute macros — трансформують елемент із атрибутом:

```rust
#[tokio::main]  // attribute macro — трансформує async fn main
async fn main() { }

#[instrument]    // attribute macro — обгортає fn у span
async fn step() { }
```

Function-like macros — виглядають як виклик макросу, але з повним доступом до TokenStream:

```rust
sql!(SELECT * FROM agents WHERE battery > 50)
html!(<div class="agent">{name}</div>)
```

У цьому розділі зосередимось на derive macros — найпрактичнішому типі для нашого проєкту.

---

## 47.2. Архітектура: окремий крейт

Процедурні макроси повинні жити в окремому крейті з `proc-macro = true`. Це технічне обмеження: proc macros виконуються під час компіляції основного крейту, тому повинні бути скомпільовані раніше.

Структура проєкту:

```
my_project/
├── Cargo.toml              # основний крейт
├── src/
│   └── main.rs
└── my_macros/              # крейт для proc macros
    ├── Cargo.toml
    └── src/
        └── lib.rs
```

my_macros/Cargo.toml:

```toml
[package]
name = "my_macros"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true          # ключовий рядок!

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

Основний Cargo.toml:

```toml
[dependencies]
my_macros = { path = "./my_macros" }
```

`proc-macro = true` позначає крейт як процедурний макрос. `syn` парсить Rust-код. `quote` генерує код. `proc-macro2` — утиліти для роботи з TokenStream.

---

## 47.3. Перший derive-макрос: Describe

Створимо derive-макрос, що генерує метод `describe() -> String` для будь-якої struct:

my_macros/src/lib.rs:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    // Парсимо TokenStream у структурований AST
    let input = parse_macro_input!(input as DeriveInput);
    
    // Ім'я типу (наприклад, "Agent")
    let name = &input.ident;
    let name_str = name.to_string();
    
    // Генеруємо код
    let expanded = quote! {
        impl #name {
            pub fn describe(&self) -> String {
                format!("{}({})", #name_str, std::any::type_name::<Self>())
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

Використання у main.rs:

```rust
use my_macros::Describe;

#[derive(Describe)]
struct Agent {
    id: String,
    battery: u8,
}

#[derive(Describe)]
struct Position {
    x: f64,
    y: f64,
}

fn main() {
    let a = Agent { id: String::from("S-01"), battery: 85 };
    println!("{}", a.describe()); // Agent(my_project::Agent)
}
```

Розберемо покроково. `#[proc_macro_derive(Describe)]` реєструє функцію як derive-макрос з ім'ям Describe. Коли компілятор бачить `#[derive(Describe)]` — він викликає цю функцію. `parse_macro_input!(input as DeriveInput)` — syn парсить TokenStream у DeriveInput — структуру, що містить ім'я типу, generics, поля, атрибути. `input.ident` — ім'я типу (Agent, Position). `quote! { ... }` — генерує TokenStream. `#name` всередині quote замінюється на значення змінної name (ідентифікатор Agent). Результат — TokenStream із impl Agent { pub fn describe() { ... } }.

---

## 47.4. Доступ до полів struct

Реальна потужність derive — генерація коду на основі полів struct:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields};

#[proc_macro_derive(FieldNames)]
pub fn derive_field_names(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    
    // Витягуємо імена полів
    let field_names: Vec<String> = match &input.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(fields) => {
                fields.named.iter()
                    .map(|f| f.ident.as_ref().unwrap().to_string())
                    .collect()
            }
            _ => vec![],
        },
        _ => panic!("FieldNames працює лише для struct з іменованими полями"),
    };
    
    let expanded = quote! {
        impl #name {
            pub fn field_names() -> &'static [&'static str] {
                &[#(#field_names),*]
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

```rust
#[derive(FieldNames)]
struct Agent {
    id: String,
    battery: u8,
    position: (f64, f64),
}

fn main() {
    println!("Поля Agent: {:?}", Agent::field_names());
    // Поля Agent: ["id", "battery", "position"]
}
```

`input.data` — Data::Struct, Data::Enum, або Data::Union. Для struct з іменованими полями — Fields::Named. Кожне поле має ident (ім'я) та ty (тип). `#(#field_names),*` у quote — повторення, як `$(...)*` у macro_rules, але для quote.

---

## 47.5. Генерація impl Trait

Головний use case — derive для власного trait:

```rust
// У основному крейті визначаємо trait
pub trait AgentInfo {
    fn agent_id(&self) -> &str;
    fn battery_level(&self) -> u8;
    fn summary(&self) -> String;
}

// derive-макрос генерує impl AgentInfo
// Шукає поля з іменами "id" (String) та "battery" (u8)
```

Proc macro крейт:

```rust
#[proc_macro_derive(AgentInfo)]
pub fn derive_agent_info(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    
    let expanded = quote! {
        impl AgentInfo for #name {
            fn agent_id(&self) -> &str {
                &self.id
            }
            fn battery_level(&self) -> u8 {
                self.battery
            }
            fn summary(&self) -> String {
                format!("[{}] battery={}%", self.id, self.battery)
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

```rust
use my_macros::AgentInfo;

trait AgentInfo {
    fn agent_id(&self) -> &str;
    fn battery_level(&self) -> u8;
    fn summary(&self) -> String;
}

#[derive(AgentInfo)]
struct Drone {
    id: String,
    battery: u8,
    altitude: f64,
}

#[derive(AgentInfo)]
struct GroundBot {
    id: String,
    battery: u8,
    speed: f64,
}

fn main() {
    let d = Drone { id: String::from("D-01"), battery: 85, altitude: 150.0 };
    let g = GroundBot { id: String::from("G-01"), battery: 60, speed: 30.0 };
    
    println!("{}", d.summary()); // [D-01] battery=85%
    println!("{}", g.summary()); // [G-01] battery=60%
}
```

Один `#[derive(AgentInfo)]` замість ручного impl для кожного типу. Додаєте новий тип агента — додаєте derive, не пишете impl.

---

## 47.6. Prompt Engineering: створення derive macros

### Промпт-шаблон

```
Я хочу derive-макрос #[derive(MyTrait)] для:
trait MyTrait {
    fn method1(&self) -> ReturnType;
    fn method2(&self) -> ReturnType;
}

Макрос повинен:
- Працювати для struct з полями [список полів]
- Генерувати impl на основі [логіка]

Згенеруй:
1. proc-macro крейт (Cargo.toml + lib.rs)
2. Приклад використання
3. Результат cargo expand
```

---

## Лабораторна робота No47

### Мета

Створити derive-макрос для проєкту рою.

### Завдання базового рівня

1. Окремий крейт my_macros з proc-macro = true.
2. Derive-макрос #[derive(Describe)] — генерує fn describe() з назвою типу та полями.
3. Використання для 3+ різних struct.
4. Перевірка через cargo expand.

### Варіанти

**A.** #[derive(AgentInfo)] — генерує impl trait з методами agent_id(), battery_level(), summary() для struct з полями id та battery.

**B.** #[derive(Builder)] — генерує struct AgentBuilder з методами .id("...").battery(85).build() для довільної struct.

**C.** Attribute macro: #[log_calls] перед fn — обгортає функцію у tracing span автоматично (спрощений #[instrument]).

### Критерії

| Критерій | Бал |
|----------|-----|
| Окремий proc-macro крейт | 15 |
| syn для парсингу DeriveInput | 20 |
| quote для генерації коду | 20 |
| Доступ до полів struct | 15 |
| cargo expand перевірка | 15 |
| Тести | 15 |

---

## Troubleshooting

### Помилка 1: `proc-macro crate types cannot export any items other than functions tagged with #[proc_macro]`

Proc-macro крейт не може експортувати struct, enum, trait — лише proc-macro функції.

Виправлення: trait та типи визначайте в окремому крейті (наприклад, my_core), а proc-macro крейт лише генерує impl.

### Помилка 2: `can't use derive macro from the same crate`

Proc-macro не може використовуватись у тому ж крейті, де визначений.

Виправлення: proc-macros завжди в окремому крейті. Типова структура: my_core (trait), my_macros (derive), my_project (використання обох).

### Помилка 3: `expected struct, found enum` у парсингу

Derive-макрос очікує struct, але отримав enum.

Виправлення: обробити Data::Enum у match або повернути compile_error!:

```rust
match &input.data {
    Data::Struct(data) => { /* ... */ },
    _ => return quote! { compile_error!("MyDerive працює лише для struct"); }.into(),
}
```

---

## Додатково

### Як працює #[derive(Debug)]

Коли ви пишете `#[derive(Debug)]`, компілятор:
1. Парсить struct у DeriveInput.
2. Для кожного поля генерує виклик `field.fmt(f)?` у impl Debug.
3. Обгортає у `fn fmt(&self, f: &mut Formatter) -> fmt::Result`.
4. Додає згенерований impl до програми.

Ваші derive-макроси працюють за тим самим механізмом. Різниця лише в тому, який код генерується.

### syn та quote: глибше

syn парсить TokenStream у типізований AST: DeriveInput, ItemFn, Expr, Type. Кожен вузол AST — Rust-тип із полями. Ви працюєте з Rust-кодом як з даними — ітеруєте по полях struct, перевіряєте типи, генеруєте відповідний код.

quote перетворює шаблон із змінними назад у TokenStream. `#name` замінюється на значення змінної. `#(#items)*` — повторення. quote — це "зворотній" syn: syn парсить код → дані, quote генерує дані → код.

---

## Контрольні запитання

### Рівень 1

1. Чим derive-макрос відрізняється від macro_rules?

Відповідь: macro_rules працює з патернами токенів — не "розуміє" Rust-код. Derive-макрос отримує повний AST (поля struct, типи, generics) через syn і може генерувати код на основі структури типу. macro_rules — текстова підстановка, derive — програмна трансформація.

2. Чому proc-macros живуть в окремому крейті?

Відповідь: proc-macros виконуються під час компіляції основного крейту. Вони повинні бути скомпільовані раніше — тому окремий крейт, що компілюється першим.

### Рівень 2

3. Що роблять syn та quote?

Відповідь: syn парсить TokenStream у типізований AST (DeriveInput з полями, типами, generics). quote генерує TokenStream з шаблону (#name підставляє змінні). syn: код → дані. quote: дані → код.

4. Що означає `#(#fields),*` у quote?

Відповідь: повторення — генерує код для кожного елемента вектора fields, розділених комами. Аналог `$(...),*` у macro_rules, але для quote.

### Рівень 4

5. Порівняйте macro_rules!, derive macros та attribute macros. Коли який?

Відповідь: macro_rules — для простої підстановки, DSL, змінної кількості аргументів. Derive — для автоматичної генерації impl trait на основі визначення типу. Attribute — для трансформації елементу (обгортання fn у span, трансформація async fn). macro_rules найпростіший, derive найпоширеніший, attribute найгнучкіший.

---

## Резюме

Процедурні макроси — функції, що трансформують TokenStream. Три типи: derive (генерація impl), attribute (трансформація елемента), function-like (довільний синтаксис).

syn парсить TokenStream у AST. quote генерує TokenStream із шаблонів. Разом — повноцінне метапрограмування: аналіз struct → генерація impl.

Proc-macros живуть в окремому крейті з `proc-macro = true`. Типова структура: core (traits), macros (derive), project (використання).

derive-макроси — найпотужніший інструмент для усунення boilerplate: один #[derive(MyTrait)] замість десятків рядків ручного impl.

---

## Що далі

Макроси дали кодогенерацію. Розділ 48 досліджує межі безпеки Rust: unsafe блоки, raw pointers, FFI (Foreign Function Interface). Коли ownership та borrow checker не достатні — або занадто обмежувальні — unsafe дозволяє "вийти за межі" з повною відповідальністю програміста. Це не "поганий код" — це контрольований escape hatch для ситуацій, де компілятор не може перевірити безпеку (системний код, FFI з C-бібліотеками, оптимізації).
