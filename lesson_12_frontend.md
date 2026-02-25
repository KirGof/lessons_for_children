# Урок 12 — Frontend: HTML, CSS, JavaScript и React

## Зачем это нужно

Инженер-оркестр должен понимать фронтенд достаточно чтобы: сверстать рабочий интерфейс, написать компонент на React, подключить его к своему API, понять что делает фронтенд разработчик. Не нужно становиться верстальщиком — нужно перестать бояться браузера. После этого урока у `todo-service` появится полноценный веб-интерфейс.

---

## Часть 1 — Как работает браузер

### Критический путь рендеринга

Когда браузер получает HTML страницу:

```
1. HTML парсинг → DOM дерево
2. CSS парсинг  → CSSOM дерево
3. DOM + CSSOM  → Render Tree
4. Layout       → вычислить размеры и позиции
5. Paint        → нарисовать пиксели
6. Composite    → склеить слои
```

Это важно знать потому что:
- `<script>` в `<head>` блокирует парсинг HTML → ставь в конец `<body>` или используй `defer`
- Большой CSS замедляет первую отрисовку
- Изменение DOM → перерасчёт Layout → дорого (reflow)

### DevTools

Главный инструмент фронтенд разработчика — DevTools в браузере (F12):

- **Elements** — HTML и CSS в реальном времени, можно редактировать
- **Console** — JavaScript консоль, ошибки
- **Network** — все HTTP запросы, время загрузки
- **Application** — localStorage, cookies, Service Workers
- **Performance** — профилировщик, найти тормоза

---

## Часть 2 — HTML

### Структура страницы

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Todo Service</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <header>
        <h1>Мои задачи</h1>
    </header>

    <main>
        <section id="task-form">
            <h2>Добавить задачу</h2>
            <!-- форма -->
        </section>

        <section id="task-list">
            <h2>Список задач</h2>
            <!-- список -->
        </section>
    </main>

    <footer>
        <p>Todo Service v1.0</p>
    </footer>

    <script src="app.js" defer></script>
</body>
</html>
```

### Семантические теги

```html
<!-- Плохо — нет смысла -->
<div class="header">
<div class="nav">
<div class="article">

<!-- Хорошо — семантично -->
<header>          <!-- шапка страницы или секции -->
<nav>             <!-- навигация -->
<main>            <!-- основной контент -->
<article>         <!-- самодостаточный контент -->
<section>         <!-- секция с заголовком -->
<aside>           <!-- боковая панель -->
<footer>          <!-- подвал -->
```

### Формы и ввод

```html
<form id="task-form">
    <div class="form-group">
        <label for="title">Название задачи</label>
        <input
            type="text"
            id="title"
            name="title"
            placeholder="Что нужно сделать?"
            required
            minlength="1"
            maxlength="200"
        >
    </div>

    <div class="form-group">
        <label for="priority">Приоритет</label>
        <select id="priority" name="priority">
            <option value="1">Низкий</option>
            <option value="2" selected>Средний</option>
            <option value="3">Высокий</option>
        </select>
    </div>

    <button type="submit">Добавить</button>
    <button type="reset">Очистить</button>
</form>
```

---

## Часть 3 — CSS

### Основы

```css
/* Селекторы */
h1 { }              /* тег */
.class-name { }     /* класс */
#id-name { }        /* идентификатор */
a:hover { }         /* псевдокласс */
p::first-line { }   /* псевдоэлемент */
div > p { }         /* прямой потомок */
div p { }           /* любой потомок */

/* Блочная модель (Box Model) */
.element {
    width: 300px;
    height: 100px;
    padding: 16px;          /* внутренний отступ */
    border: 1px solid #ccc; /* граница */
    margin: 8px;            /* внешний отступ */

    /* По умолчанию width не включает padding и border */
    /* box-sizing: border-box — включить всё в width */
    box-sizing: border-box;
}

/* Единицы измерения */
px      /* пиксели — абсолютные */
%       /* процент от родителя */
rem     /* относительно root font-size (16px по умолчанию) */
em      /* относительно font-size текущего элемента */
vw, vh  /* процент от ширины/высоты viewport */
```

### Flexbox — одномерные layouts

```css
.container {
    display: flex;
    flex-direction: row;         /* row | column */
    justify-content: space-between; /* выравнивание по главной оси */
    align-items: center;         /* выравнивание по поперечной оси */
    gap: 16px;                   /* отступы между элементами */
    flex-wrap: wrap;             /* перенос на новую строку */
}

.item {
    flex: 1;                     /* растянуть равномерно */
    flex: 0 0 200px;             /* фиксированная ширина 200px */
}
```

### Grid — двумерные layouts

```css
.grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);   /* 3 равные колонки */
    grid-template-columns: 200px 1fr 1fr;    /* первая фиксирована */
    grid-template-rows: auto;
    gap: 16px;
}

.item-wide {
    grid-column: 1 / 3;     /* занять 2 колонки */
}
```

### Адаптивный дизайн — Media queries

```css
/* Mobile first — базовые стили для мобильных */
.container {
    padding: 8px;
}

/* Таблетки и шире */
@media (min-width: 768px) {
    .container {
        padding: 16px;
        max-width: 960px;
        margin: 0 auto;
    }
}

/* Десктоп */
@media (min-width: 1024px) {
    .container {
        max-width: 1200px;
    }
}
```

### Переменные CSS

```css
:root {
    --color-primary: #3b82f6;
    --color-success: #22c55e;
    --color-danger: #ef4444;
    --color-text: #1f2937;
    --color-bg: #f9fafb;
    --border-radius: 8px;
    --shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.button {
    background-color: var(--color-primary);
    border-radius: var(--border-radius);
}
```

---

## Часть 4 — JavaScript

### Основы

```javascript
// Переменные
const name = 'Alice';       // константа
let count = 0;              // переменная
// var — устарел, не использовать

// Типы
typeof 42           // "number"
typeof "hello"      // "string"
typeof true         // "boolean"
typeof null         // "object" (баг языка)
typeof undefined    // "undefined"
typeof {}           // "object"
typeof []           // "object"
typeof function(){} // "function"

// Сравнение
== // нестрогое (с приведением типов) — не использовать
=== // строгое — всегда использовать

// Деструктуризация
const { name, age } = user;
const [first, second, ...rest] = array;

// Spread / Rest
const newArr = [...arr1, ...arr2];
const newObj = { ...obj1, newField: 'value' };

// Optional chaining
const city = user?.address?.city;       // не упадёт если address = null
const len = arr?.length ?? 0;           // ?? — значение по умолчанию
```

### Функции

```javascript
// Обычная функция
function add(a, b) {
    return a + b;
}

// Стрелочная функция
const add = (a, b) => a + b;
const double = n => n * 2;
const greet = () => 'Hello!';

// Параметры по умолчанию
const connect = (host = 'localhost', port = 8080) => `${host}:${port}`;

// Замыкание
function counter() {
    let count = 0;
    return {
        increment: () => ++count,
        decrement: () => --count,
        value: () => count,
    };
}
const c = counter();
c.increment(); // 1
c.increment(); // 2
```

### Массивы — главные методы

```javascript
const tasks = [
    { id: 1, title: 'Learn JS', done: false },
    { id: 2, title: 'Write tests', done: true },
    { id: 3, title: 'Deploy app', done: false },
];

// map — трансформировать каждый элемент
const titles = tasks.map(t => t.title);
// ['Learn JS', 'Write tests', 'Deploy app']

// filter — отфильтровать
const pending = tasks.filter(t => !t.done);
// [{id:1,...}, {id:3,...}]

// find — найти первый подходящий
const task = tasks.find(t => t.id === 2);
// {id: 2, title: 'Write tests', done: true}

// some / every
const hasUrgent = tasks.some(t => t.priority === 3);
const allDone = tasks.every(t => t.done);

// reduce — свести к одному значению
const doneCount = tasks.reduce((acc, t) => acc + (t.done ? 1 : 0), 0);

// sort
const sorted = [...tasks].sort((a, b) => a.id - b.id);
// spread чтобы не мутировать оригинал
```

### Промисы и async/await

```javascript
// Промис — асинхронная операция
const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve('данные');     // успех
        // reject(new Error('ошибка')); // ошибка
    }, 1000);
});

// .then/.catch — старый способ
promise
    .then(data => console.log(data))
    .catch(err => console.error(err));

// async/await — современный способ
async function loadData() {
    try {
        const data = await promise;
        console.log(data);
    } catch (err) {
        console.error(err);
    }
}

// fetch API — HTTP запросы в браузере
async function getTasks() {
    const response = await fetch('http://localhost:8080/tasks');

    if (!response.ok) {
        throw new Error(`HTTP error: ${response.status}`);
    }

    const tasks = await response.json();
    return tasks;
}

async function createTask(title, priority) {
    const response = await fetch('http://localhost:8080/tasks', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ title, priority }),
    });

    if (!response.ok) {
        throw new Error(`Failed to create task: ${response.status}`);
    }

    return response.json();
}
```

### DOM — работа с HTML

```javascript
// Найти элементы
const el = document.getElementById('task-list');
const els = document.querySelectorAll('.task-item');
const first = document.querySelector('.task-item');

// Изменить содержимое
el.textContent = 'Текст';               // только текст
el.innerHTML = '<strong>HTML</strong>'; // HTML (осторожно с XSS!)

// Атрибуты и классы
el.setAttribute('data-id', '42');
el.getAttribute('data-id');
el.classList.add('active');
el.classList.remove('active');
el.classList.toggle('active');
el.classList.contains('active');       // true/false

// Создать и добавить элемент
const li = document.createElement('li');
li.className = 'task-item';
li.textContent = 'Новая задача';
document.getElementById('task-list').appendChild(li);

// Удалить элемент
el.remove();

// События
el.addEventListener('click', (event) => {
    event.preventDefault();    // отменить действие по умолчанию
    event.stopPropagation();   // не всплывать вверх
    console.log('clicked', event.target);
});

// Делегирование событий — один обработчик для всех дочерних
document.getElementById('task-list').addEventListener('click', (event) => {
    const taskItem = event.target.closest('.task-item');
    if (taskItem) {
        const id = taskItem.dataset.id;
        handleTaskClick(id);
    }
});
```

---

## Часть 5 — React

### Что такое React

React — библиотека для построения UI из компонентов. Вместо ручного обновления DOM ты описываешь как должен выглядеть интерфейс, а React сам обновляет только изменившиеся части.

```bash
# Создать React проект
npx create-react-app todo-frontend
# или современный способ
npm create vite@latest todo-frontend -- --template react
cd todo-frontend
npm install
npm run dev
```

### Компоненты и JSX

```jsx
// Компонент — функция которая возвращает JSX
function TaskItem({ task, onComplete, onDelete }) {
    return (
        <li className={`task-item ${task.done ? 'done' : ''}`}>
            <span>{task.title}</span>
            <span className="priority">P{task.priority}</span>
            <div className="actions">
                {!task.done && (
                    <button onClick={() => onComplete(task.id)}>
                        ✓ Выполнено
                    </button>
                )}
                <button
                    className="danger"
                    onClick={() => onDelete(task.id)}
                >
                    Удалить
                </button>
            </div>
        </li>
    );
}
```

### Хуки — useState и useEffect

```jsx
import { useState, useEffect } from 'react';

const API_URL = 'http://localhost:8080';

function App() {
    // Состояние
    const [tasks, setTasks] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [newTitle, setNewTitle] = useState('');
    const [newPriority, setNewPriority] = useState(2);

    // Загрузить задачи при монтировании
    useEffect(() => {
        loadTasks();
    }, []);    // [] — запустить один раз

    async function loadTasks() {
        try {
            setLoading(true);
            const response = await fetch(`${API_URL}/tasks`);
            if (!response.ok) throw new Error('Failed to load tasks');
            const data = await response.json();
            setTasks(data || []);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }

    async function handleCreate(e) {
        e.preventDefault();
        if (!newTitle.trim()) return;

        try {
            const response = await fetch(`${API_URL}/tasks`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ title: newTitle, priority: newPriority }),
            });
            if (!response.ok) throw new Error('Failed to create task');
            const task = await response.json();
            setTasks(prev => [task, ...prev]);
            setNewTitle('');
        } catch (err) {
            setError(err.message);
        }
    }

    async function handleComplete(id) {
        try {
            await fetch(`${API_URL}/tasks/${id}/done`, { method: 'PATCH' });
            setTasks(prev =>
                prev.map(t => t.id === id ? { ...t, done: true } : t)
            );
        } catch (err) {
            setError(err.message);
        }
    }

    async function handleDelete(id) {
        try {
            await fetch(`${API_URL}/tasks/${id}`, { method: 'DELETE' });
            setTasks(prev => prev.filter(t => t.id !== id));
        } catch (err) {
            setError(err.message);
        }
    }

    if (loading) return <div className="loading">Загрузка...</div>;

    return (
        <div className="app">
            <header>
                <h1>Мои задачи</h1>
            </header>

            {error && (
                <div className="error" onClick={() => setError(null)}>
                    {error} ✕
                </div>
            )}

            <form onSubmit={handleCreate} className="task-form">
                <input
                    type="text"
                    value={newTitle}
                    onChange={e => setNewTitle(e.target.value)}
                    placeholder="Новая задача..."
                    required
                />
                <select
                    value={newPriority}
                    onChange={e => setNewPriority(Number(e.target.value))}
                >
                    <option value={1}>Низкий</option>
                    <option value={2}>Средний</option>
                    <option value={3}>Высокий</option>
                </select>
                <button type="submit">Добавить</button>
            </form>

            <ul className="task-list">
                {tasks.length === 0 && (
                    <li className="empty">Задач пока нет</li>
                )}
                {tasks.map(task => (
                    <TaskItem
                        key={task.id}
                        task={task}
                        onComplete={handleComplete}
                        onDelete={handleDelete}
                    />
                ))}
            </ul>

            <footer>
                <span>{tasks.filter(t => !t.done).length} осталось</span>
                <span>{tasks.filter(t => t.done).length} выполнено</span>
            </footer>
        </div>
    );
}

export default App;
```

### CORS — Cross-Origin Resource Sharing

Браузер блокирует запросы к другому домену/порту. Нужно разрешить на сервере:

```go
// Go сервер — добавить CORS middleware
func corsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next(w, r)
    }
}
```

---

## Практические задания

### Задание 12.1 — Статическая страница

Создай папку `frontend/` в репозитории `todo-service`.

Создай `frontend/index.html` — статическую страницу со списком задач:
- Заголовок страницы
- Форма добавления (поле ввода + выбор приоритета + кнопка)
- Список задач (пока пустой)
- Простые стили в `frontend/styles.css`

**Что нужно сделать:** Открой в браузере. Убедись что страница выглядит аккуратно. Проверь через DevTools что нет ошибок в Console.

---

### Задание 12.2 — Vanilla JavaScript

Добавь в `frontend/app.js` логику без React:

1. Загрузка задач с API при старте (`fetch`)
2. Отображение задач в списке
3. Добавление новой задачи через форму
4. Отметка выполненной (кнопка на каждой задаче)
5. Удаление задачи

```javascript
// Структура:
const API_URL = 'http://localhost:8080';

async function loadTasks() { ... }
async function createTask(title, priority) { ... }
async function completeTask(id) { ... }
async function deleteTask(id) { ... }
function renderTasks(tasks) { ... }

// Запустить при загрузке страницы
document.addEventListener('DOMContentLoaded', () => {
    loadTasks();
    document.getElementById('task-form').addEventListener('submit', handleSubmit);
});
```

**Что нужно сделать:** Добавь CORS middleware в Go сервер. Запусти оба сервера и проверь что всё работает.

---

### Задание 12.3 — React приложение

Создай React приложение:

```bash
cd todo-service
npm create vite@latest frontend -- --template react
cd frontend
npm install
npm run dev
```

Реализуй компоненты:
- `App` — главный компонент, хранит состояние
- `TaskForm` — форма добавления задачи
- `TaskList` — список задач
- `TaskItem` — один элемент списка

Требования:
- Загружать задачи из API при монтировании
- Все CRUD операции через API
- Показывать loader при загрузке
- Показывать ошибки пользователю
- Счётчик выполненных/невыполненных

---

### Задание 12.4 — Стили

Добавь CSS в React приложение `frontend/src/App.css`:

1. Карточки задач с тенью и скруглёнными углами
2. Цветовая индикация приоритета (зелёный/жёлтый/красный)
3. Выполненные задачи — серые с зачёркнутым текстом
4. Адаптивный дизайн — работает на телефоне
5. Hover-эффекты на кнопках

---

### Задание 12.5 — Финальный проект урока

Полноценный фронтенд для `todo-service`:

1. **Фильтрация** — кнопки "Все", "Активные", "Выполненные"
2. **Поиск** — поле поиска фильтрует задачи по названию в реальном времени
3. **Сортировка** — по приоритету, по дате создания
4. **Оптимистичные обновления** — сразу обновляй UI не ожидая ответа сервера
5. **Dockerfile для фронтенда:**
   ```dockerfile
   FROM node:20-alpine AS builder
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build

   FROM nginx:alpine
   COPY --from=builder /app/dist /usr/share/nginx/html
   COPY nginx.conf /etc/nginx/nginx.conf
   EXPOSE 80
   ```
6. Добавь `frontend` сервис в `docker-compose.yml`

`nginx.conf`:
```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://app:8080/;
        proxy_set_header Host $host;
    }
}
```

Закоммит с тегом `v0.7.0`. Теперь `docker compose up -d` поднимает полноценное приложение с фронтендом, бэкендом и базой данных.

---

## Шпаргалка

| Концепция | Пример |
|-----------|--------|
| Fetch GET | `const r = await fetch(url); const data = await r.json()` |
| Fetch POST | `fetch(url, {method:'POST', headers:{...}, body: JSON.stringify(data)})` |
| useState | `const [value, setValue] = useState(initial)` |
| useEffect | `useEffect(() => { loadData() }, [])` |
| map в JSX | `{items.map(item => <li key={item.id}>{item.name}</li>)}` |
| Условный рендер | `{condition && <Component />}` |
| Форма | `<form onSubmit={e => { e.preventDefault(); ... }}>` |
| CORS | `w.Header().Set("Access-Control-Allow-Origin", "*")` |

---

## Ресурсы для изучения

- **HTML/CSS:** `https://developer.mozilla.org/ru/` — лучшая документация, есть на русском
- **JavaScript:** `https://learn.javascript.ru` — отличный учебник на русском
- **React:** `https://react.dev` — официальная документация, интерактивные примеры
- **Flexbox:** `https://flexboxfroggy.com` — игра для изучения Flexbox
- **Grid:** `https://cssgridgarden.com` — игра для изучения CSS Grid
- **DevTools:** `https://developer.chrome.com/docs/devtools/`

---

## Как понять что урок пройден

- [ ] Понимаю как браузер рендерит страницу
- [ ] Умею делать HTML разметку с семантическими тегами
- [ ] Умею верстать на Flexbox и Grid
- [ ] Делаю HTTP запросы из браузера через fetch/async/await
- [ ] Написал React приложение с несколькими компонентами
- [ ] Понимаю useState и useEffect
- [ ] Настроил CORS на Go сервере
- [ ] Фронтенд собирается в Docker образ и работает через nginx

---

*Следующий урок: Итог — собираем всё вместе и разбираем реальные сценарии*
