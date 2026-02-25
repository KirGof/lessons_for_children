# Урок 7 — PostgreSQL: реляционные базы данных

## Зачем это нужно

Почти каждое серьёзное приложение хранит данные в базе данных. Файлы на диске не масштабируются, не поддерживают одновременный доступ, не умеют искать по сложным условиям. PostgreSQL — одна из лучших реляционных баз данных в мире: надёжная, быстрая, бесплатная. После этого урока `todo-service` будет хранить данные в настоящей базе — как в продакшене.

---

## Часть 1 — Установка и основы

### Установка PostgreSQL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y

# Запустить и включить автозапуск
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql
```

### Первый вход

```bash
# PostgreSQL создаёт системного пользователя postgres
sudo -u postgres psql          # войти в интерактивную консоль

# Внутри psql:
\l                             # список баз данных
\c dbname                      # подключиться к базе
\dt                            # список таблиц
\d table_name                  # структура таблицы
\du                            # список пользователей
\q                             # выйти
\?                             # помощь по командам psql
\h SELECT                      # помощь по SQL команде
```

### Создание пользователя и базы данных

```sql
-- Войди как postgres: sudo -u postgres psql

-- Создать пользователя
CREATE USER todouser WITH PASSWORD 'secret123';

-- Создать базу данных
CREATE DATABASE tododb OWNER todouser;

-- Дать права
GRANT ALL PRIVILEGES ON DATABASE tododb TO todouser;

-- Выйти
\q
```

```bash
# Подключиться под новым пользователем
psql -U todouser -d tododb -h localhost

# Или через переменную окружения
export DATABASE_URL="postgres://todouser:secret123@localhost:5432/tododb"
psql $DATABASE_URL
```

---

## Часть 2 — SQL основы

### Типы данных PostgreSQL

```sql
-- Числа
INTEGER         -- 32-битное целое (-2.1B до 2.1B)
BIGINT          -- 64-битное целое
SERIAL          -- автоинкремент INTEGER
BIGSERIAL       -- автоинкремент BIGINT
NUMERIC(10,2)   -- точное число (10 цифр, 2 после запятой) — для денег
FLOAT           -- число с плавающей точкой

-- Строки
VARCHAR(255)    -- строка до 255 символов
TEXT            -- строка любой длины (предпочтительно)
CHAR(10)        -- строка фиксированной длины

-- Дата и время
TIMESTAMP       -- дата и время без часового пояса
TIMESTAMPTZ     -- дата и время с часовым поясом (предпочтительно)
DATE            -- только дата
TIME            -- только время

-- Прочее
BOOLEAN         -- true / false
UUID            -- универсальный уникальный идентификатор
JSONB           -- JSON с индексацией (быстрее чем JSON)
```

### DDL — создание структуры

```sql
-- Создать таблицу
CREATE TABLE tasks (
    id          BIGSERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    description TEXT,
    priority    INTEGER NOT NULL DEFAULT 1 CHECK (priority BETWEEN 1 AND 3),
    done        BOOLEAN NOT NULL DEFAULT false,
    user_id     BIGINT REFERENCES users(id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Создать индекс
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_done ON tasks(done);
CREATE INDEX idx_tasks_created_at ON tasks(created_at DESC);

-- Изменить таблицу
ALTER TABLE tasks ADD COLUMN tags TEXT[];
ALTER TABLE tasks ALTER COLUMN priority SET DEFAULT 2;
ALTER TABLE tasks DROP COLUMN description;

-- Удалить таблицу
DROP TABLE tasks;
DROP TABLE IF EXISTS tasks;       -- не ошибается если нет
DROP TABLE tasks CASCADE;         -- удалить со всеми зависимостями
```

### DML — работа с данными

```sql
-- INSERT
INSERT INTO tasks (title, priority)
VALUES ('Learn PostgreSQL', 1);

INSERT INTO tasks (title, priority, done)
VALUES
    ('Write tests', 2, false),
    ('Deploy to production', 3, false),
    ('Review PR', 1, true);

-- INSERT с возвратом созданной записи
INSERT INTO tasks (title, priority)
VALUES ('New task', 1)
RETURNING *;                      -- вернуть всю запись
RETURNING id, created_at;         -- вернуть только нужные поля

-- SELECT
SELECT * FROM tasks;
SELECT id, title, done FROM tasks;
SELECT * FROM tasks WHERE done = false;
SELECT * FROM tasks WHERE priority = 1 AND done = false;
SELECT * FROM tasks WHERE title LIKE '%Go%';   -- поиск по подстроке
SELECT * FROM tasks ORDER BY created_at DESC;
SELECT * FROM tasks LIMIT 10 OFFSET 20;        -- пагинация

-- UPDATE
UPDATE tasks SET done = true WHERE id = 1;
UPDATE tasks SET done = true, updated_at = NOW() WHERE id = 1;
UPDATE tasks SET priority = priority + 1 WHERE done = false;

-- UPDATE с возвратом
UPDATE tasks SET done = true WHERE id = 1 RETURNING *;

-- DELETE
DELETE FROM tasks WHERE id = 1;
DELETE FROM tasks WHERE done = true AND created_at < NOW() - INTERVAL '30 days';
DELETE FROM tasks WHERE id = 1 RETURNING *;
```

---

## Часть 3 — Продвинутый SQL

### JOIN — объединение таблиц

```sql
-- Пример: таблица users и tasks
CREATE TABLE users (
    id       BIGSERIAL PRIMARY KEY,
    name     TEXT NOT NULL,
    email    TEXT UNIQUE NOT NULL
);

-- INNER JOIN — только записи где есть совпадение в обеих таблицах
SELECT u.name, t.title, t.done
FROM users u
INNER JOIN tasks t ON t.user_id = u.id
WHERE t.done = false;

-- LEFT JOIN — все записи из левой таблицы + совпадения из правой
SELECT u.name, COUNT(t.id) as task_count
FROM users u
LEFT JOIN tasks t ON t.user_id = u.id
GROUP BY u.id, u.name;

-- Можно сделать вывод понятнее
SELECT
    u.name          AS user_name,
    u.email         AS user_email,
    t.title         AS task_title,
    t.priority      AS task_priority,
    t.created_at    AS created
FROM users u
LEFT JOIN tasks t ON t.user_id = u.id
ORDER BY u.name, t.created_at DESC;
```

### Агрегатные функции

```sql
-- COUNT, SUM, AVG, MIN, MAX
SELECT COUNT(*) FROM tasks;
SELECT COUNT(*) FROM tasks WHERE done = true;
SELECT AVG(priority) FROM tasks;
SELECT MIN(created_at), MAX(created_at) FROM tasks;

-- GROUP BY
SELECT priority, COUNT(*) as count
FROM tasks
GROUP BY priority
ORDER BY priority;

-- HAVING — фильтр после GROUP BY
SELECT user_id, COUNT(*) as task_count
FROM tasks
GROUP BY user_id
HAVING COUNT(*) > 5;

-- Подзапросы
SELECT * FROM tasks
WHERE user_id IN (
    SELECT id FROM users WHERE email LIKE '%@gmail.com'
);
```

### Транзакции

```sql
-- Транзакция гарантирует что либо все операции выполнятся, либо ни одна

BEGIN;

INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO tasks (title, priority, user_id)
VALUES ('First task', 1, currval('users_id_seq'));

COMMIT;    -- применить всё

-- Если что-то пошло не так:
ROLLBACK;  -- отменить всё

-- Пример с обработкой ошибки
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Если второй UPDATE упал — ROLLBACK автоматически
COMMIT;
```

### EXPLAIN ANALYZE — анализ запросов

```sql
-- Посмотреть план выполнения запроса
EXPLAIN SELECT * FROM tasks WHERE user_id = 1;

-- С реальным временем выполнения
EXPLAIN ANALYZE SELECT * FROM tasks WHERE user_id = 1;

-- Вывод показывает:
-- Seq Scan    — последовательное сканирование (медленно для больших таблиц)
-- Index Scan  — использование индекса (быстро)
-- cost=0.00..8.27 rows=1 width=100
--   первое число — стоимость до первой строки
--   второе       — полная стоимость
-- actual time=0.015..0.016 rows=1 loops=1
```

---

## Часть 4 — Go + PostgreSQL

### Подключение через pgx

```bash
cd ~/projects/todo-service
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
```

### Пул соединений

```go
package main

import (
    "context"
    "fmt"
    "os"

    "github.com/jackc/pgx/v5/pgxpool"
)

func connectDB() (*pgxpool.Pool, error) {
    databaseURL := os.Getenv("DATABASE_URL")
    if databaseURL == "" {
        databaseURL = "postgres://todouser:secret123@localhost:5432/tododb"
    }

    config, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }

    config.MaxConns = 10    // максимум соединений в пуле

    pool, err := pgxpool.NewWithConfig(context.Background(), config)
    if err != nil {
        return nil, fmt.Errorf("create pool: %w", err)
    }

    // Проверить соединение
    if err := pool.Ping(context.Background()); err != nil {
        return nil, fmt.Errorf("ping: %w", err)
    }

    return pool, nil
}
```

### PostgresStorage

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

type Task struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Priority  int       `json:"priority"`
    Done      bool      `json:"done"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type PostgresStorage struct {
    db *pgxpool.Pool
}

func NewPostgresStorage(db *pgxpool.Pool) *PostgresStorage {
    return &PostgresStorage{db: db}
}

func (s *PostgresStorage) Add(ctx context.Context, title string, priority int) (Task, error) {
    if title == "" {
        return Task{}, ErrInvalidInput
    }
    if priority < 1 || priority > 3 {
        return Task{}, ErrInvalidInput
    }

    var task Task
    err := s.db.QueryRow(ctx,
        `INSERT INTO tasks (title, priority)
         VALUES ($1, $2)
         RETURNING id, title, priority, done, created_at, updated_at`,
        title, priority,
    ).Scan(&task.ID, &task.Title, &task.Priority, &task.Done, &task.CreatedAt, &task.UpdatedAt)

    if err != nil {
        return Task{}, fmt.Errorf("add task: %w", err)
    }

    return task, nil
}

func (s *PostgresStorage) Get(ctx context.Context, id int) (Task, error) {
    var task Task
    err := s.db.QueryRow(ctx,
        `SELECT id, title, priority, done, created_at, updated_at
         FROM tasks WHERE id = $1`,
        id,
    ).Scan(&task.ID, &task.Title, &task.Priority, &task.Done, &task.CreatedAt, &task.UpdatedAt)

    if errors.Is(err, pgx.ErrNoRows) {
        return Task{}, ErrNotFound
    }
    if err != nil {
        return Task{}, fmt.Errorf("get task %d: %w", id, err)
    }

    return task, nil
}

func (s *PostgresStorage) List(ctx context.Context) ([]Task, error) {
    rows, err := s.db.Query(ctx,
        `SELECT id, title, priority, done, created_at, updated_at
         FROM tasks ORDER BY created_at DESC`,
    )
    if err != nil {
        return nil, fmt.Errorf("list tasks: %w", err)
    }
    defer rows.Close()

    var tasks []Task
    for rows.Next() {
        var task Task
        if err := rows.Scan(
            &task.ID, &task.Title, &task.Priority,
            &task.Done, &task.CreatedAt, &task.UpdatedAt,
        ); err != nil {
            return nil, fmt.Errorf("scan task: %w", err)
        }
        tasks = append(tasks, task)
    }

    return tasks, rows.Err()
}

func (s *PostgresStorage) Complete(ctx context.Context, id int) error {
    result, err := s.db.Exec(ctx,
        `UPDATE tasks SET done = true, updated_at = NOW() WHERE id = $1`,
        id,
    )
    if err != nil {
        return fmt.Errorf("complete task %d: %w", id, err)
    }
    if result.RowsAffected() == 0 {
        return ErrNotFound
    }
    return nil
}

func (s *PostgresStorage) Delete(ctx context.Context, id int) error {
    result, err := s.db.Exec(ctx,
        `DELETE FROM tasks WHERE id = $1`,
        id,
    )
    if err != nil {
        return fmt.Errorf("delete task %d: %w", id, err)
    }
    if result.RowsAffected() == 0 {
        return ErrNotFound
    }
    return nil
}
```

### Миграции

Миграции — это SQL файлы которые описывают изменения структуры базы данных. Нужны чтобы изменения базы были воспроизводимы и версионированы в git.

```bash
# Структура миграций
migrations/
    001_create_tasks.up.sql
    001_create_tasks.down.sql
    002_add_users.up.sql
    002_add_users.down.sql
```

`migrations/001_create_tasks.up.sql`:
```sql
CREATE TABLE tasks (
    id          BIGSERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    priority    INTEGER NOT NULL DEFAULT 1 CHECK (priority BETWEEN 1 AND 3),
    done        BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_done ON tasks(done);
CREATE INDEX idx_tasks_priority ON tasks(priority);
```

`migrations/001_create_tasks.down.sql`:
```sql
DROP TABLE IF EXISTS tasks;
```

```bash
# Применить миграции вручную
psql $DATABASE_URL -f migrations/001_create_tasks.up.sql

# Или через инструмент migrate
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

migrate -path migrations -database $DATABASE_URL up
migrate -path migrations -database $DATABASE_URL down 1
migrate -path migrations -database $DATABASE_URL version
```

---

## Практические задания

### Задание 7.1 — Создание базы данных

```bash
sudo -u postgres psql
```

Создай:
1. Пользователя `todouser` с паролем
2. Базу данных `tododb`
3. Таблицу `tasks` с полями: id, title, priority, done, created_at, updated_at
4. Индексы на `done` и `priority`

Проверь через `\dt` и `\d tasks` что всё создано правильно.

---

### Задание 7.2 — SQL запросы

Заполни таблицу тестовыми данными:

```sql
INSERT INTO tasks (title, priority, done) VALUES
    ('Learn SQL', 1, true),
    ('Learn PostgreSQL', 1, false),
    ('Write migrations', 2, false),
    ('Add indexes', 2, false),
    ('Deploy to server', 3, false),
    ('Write documentation', 2, true),
    ('Code review', 1, false);
```

Напиши SQL запросы которые:
1. Выводят все невыполненные задачи отсортированные по приоритету
2. Считают сколько задач каждого приоритета
3. Находят задачи со словом "SQL" в названии
4. Обновляют все задачи с приоритетом 3 — ставят им `done = true`
5. Удаляют все выполненные задачи и возвращают количество удалённых строк
6. Выводят 3 самые новые невыполненные задачи

Для каждого запроса напиши `EXPLAIN ANALYZE` и посмотри использует ли он индекс.

---

### Задание 7.3 — Go + PostgreSQL

Замени `MemoryStorage` в `todo-service` на `PostgresStorage`:

1. Напиши `PostgresStorage` реализующий интерфейс `Storage`
2. Все методы должны принимать `context.Context` первым аргументом
3. Обработай все ошибки включая `pgx.ErrNoRows`
4. Создай файл миграции `migrations/001_create_tasks.up.sql`
5. Добавь в `main.go`:
   - Подключение к базе через `DATABASE_URL`
   - Применение миграций при старте (можно вручную через psql)
   - Graceful shutdown закрывает пул соединений

Проверь что все эндпоинты работают и данные сохраняются между перезапусками.

---

### Задание 7.4 — Транзакция

Добавь новый эндпоинт `POST /tasks/bulk-complete` который:
- Принимает `{"ids": [1, 2, 3]}`
- Отмечает все указанные задачи выполненными **в одной транзакции**
- Если хотя бы один ID не существует — откатывает всё и возвращает 404
- Возвращает список обновлённых задач

```go
func (s *PostgresStorage) BulkComplete(ctx context.Context, ids []int) ([]Task, error) {
    tx, err := s.db.Begin(ctx)
    if err != nil {
        return nil, err
    }
    defer tx.Rollback(ctx)    // откатить если не было Commit

    // ... обновить каждый id в транзакции ...

    if err := tx.Commit(ctx); err != nil {
        return nil, err
    }
    return tasks, nil
}
```

---

### Задание 7.5 — Финальный проект урока

1. Добавь в `todo-service` таблицу `tags`:
   ```sql
   CREATE TABLE tags (
       id      BIGSERIAL PRIMARY KEY,
       name    TEXT UNIQUE NOT NULL
   );

   CREATE TABLE task_tags (
       task_id BIGINT REFERENCES tasks(id) ON DELETE CASCADE,
       tag_id  BIGINT REFERENCES tags(id) ON DELETE CASCADE,
       PRIMARY KEY (task_id, tag_id)
   );
   ```

2. Добавь эндпоинты:
   - `POST /tasks/{id}/tags` — добавить тег к задаче `{"tag": "go"}`
   - `DELETE /tasks/{id}/tags/{tag}` — удалить тег
   - `GET /tasks?tag=go` — фильтр задач по тегу

3. Напиши SQL запрос который возвращает задачи вместе со списком их тегов

4. Закоммить с тегом `v0.2.0`

---

## Шпаргалка

| Команда / SQL | Что делает |
|---------------|-----------|
| `\l` | Список баз данных |
| `\dt` | Список таблиц |
| `\d table` | Структура таблицы |
| `SELECT * FROM t WHERE ...` | Выборка с условием |
| `INSERT INTO t (...) VALUES (...) RETURNING *` | Вставка с возвратом |
| `UPDATE t SET ... WHERE ... RETURNING *` | Обновление с возвратом |
| `DELETE FROM t WHERE ... ` | Удаление |
| `EXPLAIN ANALYZE query` | Анализ запроса |
| `BEGIN; ...; COMMIT;` | Транзакция |
| `pgx.ErrNoRows` | Запись не найдена |

---

## Ресурсы для изучения

- **Документация PostgreSQL:** `https://www.postgresql.org/docs/` — лучшая документация в мире
- **Интерактивный SQL:** `https://pgexercises.com` — задачи по SQL с проверкой
- **pgx документация:** `https://pkg.go.dev/github.com/jackc/pgx/v5`
- **Визуализация JOIN:** `https://joins.spathon.com`
- **Русский ресурс:** `https://postgrespro.ru/docs/postgresql` — документация на русском

---

## Как понять что урок пройден

- [ ] Умею создавать таблицы, индексы, пользователей в PostgreSQL
- [ ] Пишу SELECT, INSERT, UPDATE, DELETE уверенно
- [ ] Понимаю JOIN и умею объединять таблицы
- [ ] Знаю что такое транзакция и когда она нужна
- [ ] Подключил Go сервис к PostgreSQL через pgx
- [ ] Обрабатываю ошибку `pgx.ErrNoRows` правильно
- [ ] Создал файлы миграций
- [ ] Данные сохраняются между перезапусками сервиса

---

*Следующий урок: Docker — контейнеризация приложений*
