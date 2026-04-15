# Основи програмування мовою Rust з AI-асистентом

Підручник з програмування на Rust з інтегрованим курсом Prompt Engineering для першокурсників. Наскрізний проєкт: симуляція інтелектуального автономного рою БПЛА.

## Структура

- **Частина I** (Розділи 1–12): Перші кроки — синтаксис, типи, функції
- **Частина II** (Розділи 13–20): Ownership та безпека пам'яті
- **Частина III** (Розділи 21–30): Робастність — колекції, ітератори, помилки, traits
- **Частина IV** (Розділи 31–38): Smart Pointers та спільна пам'ять
- **Частина V** (Розділи 39–45): Асинхронність та масштабування
- **Частина VI** (Розділи 46–50): Архітектура, макроси та екосистема
- **Частина VII** (Розділи 51–52): Фінальний проєкт

## Збірка книги

### Вимоги

- [mdBook](https://rust-lang.github.io/mdBook/) v0.4+

```bash
cargo install mdbook
```

### Локальна збірка

```bash
mdbook build      # збірка → ./book/
mdbook serve      # локальний сервер на http://localhost:3000
```

### Деплой на GitHub Pages

Автоматичний через GitHub Actions при push у main. Налаштуйте:

1. Settings → Pages → Source: GitHub Actions
2. Push у main → workflow автоматично побудує та задеплоїть

## Ліцензія

CC BY-NC-SA 4.0


git init
git add .
git commit -m "Initial commit: Rust AI Textbook"
git remote add origin https://github.com/YOUR_USERNAME/rust-ai-textbook.git
git push -u origin main
