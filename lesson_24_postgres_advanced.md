# Урок 24 — PostgreSQL продвинутый: репликация, партиционирование, full-text search

## Зачем это нужно

Базовый PostgreSQL из урока 7 работает до определённого масштаба. Когда таблица вырастает до 100 миллионов строк — простой индекс уже недостаточно. Когда нужна высокая доступность — один сервер не подходит. Когда пользователи ищут "задачи про kubernetes" — ILIKE не справляется. Это урок про то как PostgreSQL масштабируется до нагрузок на которых работают крупные продукты.

---

## Часть 1 — Репликация

### Зачем репликация

```
Без репликации:          С репликацией:
                         
 ┌──────────┐            ┌──────────┐
 │ Primary  │            │ Primary  │◄── Запись
 │   PostgreSQL  │            │          │    
 └──────────┘            └────┬─────┘
   ↑ читать и писать          │ WAL stream
   ↑ упал = всё упало         ▼
                         ┌──────────┐
                         │ Replica  │◄── Чтение (SELECT)
                         └──────────┘

WAL (Write-Ahead Log) — журнал всех изменений
Replica применяет WAL и держит копию данных
```

**Streaming Replication** — физическая репликация на уровне байт.
- Простая, быстрая
- Реплика точная копия primary
- Нельзя делать write-запросы на реплику

**Logical Replication** — на уровне SQL операций.
- Можно реплицировать только часть таблиц
- Реплика может иметь другие индексы
- Основа для миграций без даунтайма

### Настройка Streaming Replication

```bash
# На primary — postgresql.conf
wal_level = replica         # включить WAL логирование
max_wal_senders = 3         # количество replicas
wal_keep_size = 1GB         # держать WAL на диске

# pg_hba.conf — разрешить реплике подключаться
host replication replicauser 10.0.2.0/24 scram-sha-256
```

```sql
-- Создать пользователя для репликации
CREATE USER replicauser REPLICATION LOGIN PASSWORD 'secret';
```

```bash
# На replica — первоначальная синхронизация
pg_basebackup \
  -h primary-host \
  -U replicauser \
  -D /var/lib/postgresql/data \
  -P -Xs -R
# -R создаёт standby.signal и postgresql.auto.conf автоматически

# postgresql.auto.conf на replica:
primary_conninfo = 'host=primary-host user=replicauser password=secret'
```

```sql
-- Проверить статус репликации (на primary)
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;

-- На replica
SELECT pg_is_in_recovery();    -- true если это replica
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

### Репликация через Docker Compose

```yaml
# docker-compose-replication.yml
services:
  postgres-primary:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: todouser
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: tododb
      POSTGRES_REPLICATION_USER: replicauser
      POSTGRES_REPLICATION_PASSWORD: replsecret
    volumes:
      - ./postgres/primary.conf:/etc/postgresql/postgresql.conf
      - ./postgres/init-primary.sh:/docker-entrypoint-initdb.d/init.sh
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

  postgres-replica:
    image: postgres:16-alpine
    environment:
      PGUSER: replicauser
      PGPASSWORD: replsecret
    depends_on:
      postgres-primary:
        condition: service_healthy
    volumes:
      - ./postgres/init-replica.sh:/docker-entrypoint-initdb.d/init.sh
    command: >
      sh -c "
        pg_basebackup -h postgres-primary -U replicauser -D /var/lib/postgresql/data -P -Xs -R
        postgres
      "
```

### Read/Write разделение в Go

```go
type DBPool struct {
    write *pgxpool.Pool    // primary — только запись
    read  *pgxpool.Pool    // replica — только чтение
}

func NewDBPool(primaryURL, replicaURL string) (*DBPool, error) {
    write, err := pgxpool.New(context.Background(), primaryURL)
    if err != nil { return nil, err }

    read, err := pgxpool.New(context.Background(), replicaURL)
    if err != nil { return nil, err }

    return &DBPool{write: write, read: read}, nil
}

// Читаем с реплики — снижаем нагрузку на primary
func (db *DBPool) ListTasks(ctx context.Context, userID int) ([]Task, error) {
    rows, err := db.read.Query(ctx,
        "SELECT id, title, priority, done FROM tasks WHERE user_id = $1",
        userID,
    )
    // ...
}

// Пишем только на primary
func (db *DBPool) CreateTask(ctx context.Context, task Task) (Task, error) {
    err := db.write.QueryRow(ctx,
        "INSERT INTO tasks(title, priority, user_id) VALUES($1,$2,$3) RETURNING id",
        task.Title, task.Priority, task.UserID,
    ).Scan(&task.ID)
    // ...
}
```

---

## Часть 2 — Партиционирование

### Зачем партиционирование

```
Таблица tasks: 500 миллионов строк
SELECT * FROM tasks WHERE created_at > '2024-01-01'

Без партиций:
→ Seq Scan 500M строк  — минуты

С партициями по дате:
→ Сканируется только tasks_2024  — секунды
→ Старые партиции можно архивировать или удалять
```

### Range Partitioning по дате

```sql
-- Создать секционированную таблицу
CREATE TABLE tasks (
    id         BIGSERIAL,
    title      TEXT NOT NULL,
    user_id    BIGINT NOT NULL,
    done       BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Создать партиции по месяцам
CREATE TABLE tasks_2024_01 PARTITION OF tasks
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE tasks_2024_02 PARTITION OF tasks
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE tasks_2024_03 PARTITION OF tasks
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Дефолтная партиция для данных вне диапазонов
CREATE TABLE tasks_default PARTITION OF tasks DEFAULT;

-- Индексы создаются на каждой партиции
CREATE INDEX ON tasks_2024_01 (user_id);
CREATE INDEX ON tasks_2024_02 (user_id);

-- Проверить на какой партиции будет запрос
EXPLAIN SELECT * FROM tasks WHERE created_at > '2024-02-01';
-- -> Seq Scan on tasks_2024_02  (только одна партиция!)
-- -> Seq Scan on tasks_2024_03
-- tasks_2024_01 пропущена — partition pruning
```

### Автоматическое создание партиций

```sql
-- Функция которая создаёт партицию для следующего месяца
CREATE OR REPLACE FUNCTION create_next_month_partition()
RETURNS void AS $$
DECLARE
    next_month DATE := date_trunc('month', NOW() + interval '1 month');
    partition_name TEXT := 'tasks_' || to_char(next_month, 'YYYY_MM');
    start_date TEXT := to_char(next_month, 'YYYY-MM-DD');
    end_date TEXT := to_char(next_month + interval '1 month', 'YYYY-MM-DD');
BEGIN
    -- Проверить что партиция ещё не создана
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables WHERE tablename = partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF tasks FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
        EXECUTE format('CREATE INDEX ON %I (user_id)', partition_name);
        RAISE NOTICE 'Created partition: %', partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Вызывать через pg_cron каждый месяц
SELECT cron.schedule('0 0 1 * *', 'SELECT create_next_month_partition()');
```

### Hash Partitioning для равномерного распределения

```sql
-- Когда нет явного временного паттерна
-- Распределить по user_id равномерно
CREATE TABLE tasks (
    id      BIGSERIAL,
    user_id BIGINT NOT NULL,
    title   TEXT NOT NULL
) PARTITION BY HASH (user_id);

-- 4 партиции — каждая примерно 25% данных
CREATE TABLE tasks_p0 PARTITION OF tasks FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE tasks_p1 PARTITION OF tasks FOR VALUES WITH (modulus 4, remainder 1);
CREATE TABLE tasks_p2 PARTITION OF tasks FOR VALUES WITH (modulus 4, remainder 2);
CREATE TABLE tasks_p3 PARTITION OF tasks FOR VALUES WITH (modulus 4, remainder 3);

-- Запрос по user_id автоматически идёт только в одну партицию
SELECT * FROM tasks WHERE user_id = 42;
-- -> Seq Scan on tasks_p2  (42 % 4 = 2)
```

### Архивирование старых партиций

```sql
-- Когда данные за 2022 год больше не нужны в быстром доступе:

-- Отсоединить от основной таблицы (мгновенно)
ALTER TABLE tasks DETACH PARTITION tasks_2022_01;

-- tasks_2022_01 теперь обычная таблица, можно:
-- 1. Скопировать в архив
COPY tasks_2022_01 TO '/backup/tasks_2022_01.csv';

-- 2. Удалить (освободить место)
DROP TABLE tasks_2022_01;

-- 3. Или перенести в дешёвое хранилище (tablespace)
ALTER TABLE tasks_2022_01 SET TABLESPACE slow_disk;
ALTER TABLE tasks ATTACH PARTITION tasks_2022_01
    FOR VALUES FROM ('2022-01-01') TO ('2022-02-01');
```

---

## Часть 3 — Full-Text Search

### Почему не ILIKE

```sql
-- ILIKE — всегда Seq Scan
SELECT * FROM tasks WHERE title ILIKE '%kubernetes%';
-- Execution time: 3200 ms (на 10M строк)

-- Full-Text Search с GIN индексом
SELECT * FROM tasks WHERE to_tsvector('russian', title) @@ plainto_tsquery('russian', 'kubernetes');
-- Execution time: 12 ms
```

### Концепции FTS

```sql
-- tsvector — индексируемое представление текста
SELECT to_tsvector('russian', 'Настройка Kubernetes кластера в продакшене');
-- 'kubernetes':2 'кластер':3 'настройк':1 'продакшен':5

-- Слова нормализованы (лемматизация), стоп-слова убраны
-- Позиции сохранены для ranking

-- tsquery — поисковый запрос
SELECT to_tsquery('russian', 'kubernetes & кластер');
-- 'kubernetes' & 'кластер'

SELECT plainto_tsquery('russian', 'kubernetes кластер');
-- 'kubernetes' & 'кластер'

SELECT websearch_to_tsquery('russian', 'kubernetes OR docker');
-- 'kubernetes' | 'docker'

-- Поиск
SELECT title FROM tasks
WHERE to_tsvector('russian', title) @@ plainto_tsquery('russian', 'kubernetes');
```

### Создание FTS индекса

```sql
-- Способ 1: Вычисляемый столбец с индексом
ALTER TABLE tasks ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        to_tsvector('russian', coalesce(title, '') || ' ' || coalesce(description, ''))
    ) STORED;

CREATE INDEX idx_tasks_search ON tasks USING GIN (search_vector);

-- Теперь запрос использует индекс
SELECT * FROM tasks WHERE search_vector @@ plainto_tsquery('russian', 'kubernetes');

-- Способ 2: Функциональный индекс (если не хочешь менять схему)
CREATE INDEX idx_tasks_fts ON tasks
    USING GIN (to_tsvector('russian', title));

-- Запрос должен точно соответствовать выражению в индексе
SELECT * FROM tasks
WHERE to_tsvector('russian', title) @@ plainto_tsquery('russian', 'kubernetes');
```

### Ранжирование результатов

```sql
-- ts_rank — оценка релевантности
SELECT
    id,
    title,
    ts_rank(search_vector, query) AS rank,
    ts_headline('russian', title, query,
        'StartSel=<mark>, StopSel=</mark>, MaxWords=20') AS highlighted
FROM tasks,
     plainto_tsquery('russian', 'kubernetes кластер') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- Результат:
-- id | title                              | rank  | highlighted
-- ---|------------------------------------|-------|------------------------------------------
-- 42 | Настройка Kubernetes кластера      | 0.847 | Настройка <mark>Kubernetes кластера</mark>
-- 15 | Kubernetes в облаке                | 0.612 | <mark>Kubernetes</mark> в облаке
```

### FTS в Go

```go
// Поиск задач
type SearchParams struct {
    Query    string
    UserID   int
    Language string    // "russian" или "english"
    Limit    int
    Offset   int
}

type SearchResult struct {
    Task
    Rank        float64 `json:"rank"`
    Highlighted string  `json:"highlighted"`
}

func (s *PostgresStorage) Search(ctx context.Context, p SearchParams) ([]SearchResult, error) {
    if p.Language == "" { p.Language = "russian" }
    if p.Limit == 0 { p.Limit = 20 }

    query := `
        SELECT
            t.id, t.title, t.priority, t.done, t.created_at,
            ts_rank(t.search_vector, q.query) AS rank,
            ts_headline(
                $3, t.title, q.query,
                'StartSel=**, StopSel=**, MaxWords=15'
            ) AS highlighted
        FROM tasks t,
             plainto_tsquery($3, $1) AS q(query)
        WHERE t.search_vector @@ q.query
          AND t.user_id = $2
        ORDER BY rank DESC
        LIMIT $4 OFFSET $5
    `

    rows, err := s.pool.Query(ctx, query,
        p.Query, p.UserID, p.Language, p.Limit, p.Offset,
    )
    if err != nil {
        return nil, fmt.Errorf("search: %w", err)
    }
    defer rows.Close()

    var results []SearchResult
    for rows.Next() {
        var r SearchResult
        err := rows.Scan(
            &r.ID, &r.Title, &r.Priority, &r.Done, &r.CreatedAt,
            &r.Rank, &r.Highlighted,
        )
        if err != nil {
            return nil, err
        }
        results = append(results, r)
    }

    return results, nil
}
```

### HTTP эндпоинт поиска

```go
// GET /tasks/search?q=kubernetes&lang=russian&limit=20&page=1
func (h *Handler) searchTasks(w http.ResponseWriter, r *http.Request) {
    claims, _ := userFromContext(r.Context())

    q := r.URL.Query().Get("q")
    if q == "" {
        writeJSON(w, http.StatusBadRequest, errResp("query is required"))
        return
    }

    limit := 20
    if l := r.URL.Query().Get("limit"); l != "" {
        fmt.Sscan(l, &limit)
    }
    page := 1
    if p := r.URL.Query().Get("page"); p != "" {
        fmt.Sscan(p, &page)
    }

    results, err := h.storage.Search(r.Context(), SearchParams{
        Query:    q,
        UserID:   claims.UserID,
        Language: r.URL.Query().Get("lang"),
        Limit:    limit,
        Offset:   (page - 1) * limit,
    })
    if err != nil {
        slog.Error("search failed", "error", err, "query", q)
        writeJSON(w, http.StatusInternalServerError, errResp("search failed"))
        return
    }

    writeJSON(w, http.StatusOK, map[string]interface{}{
        "results": results,
        "query":   q,
        "page":    page,
        "limit":   limit,
    })
}
```

---

## Часть 4 — Продвинутые паттерны

### Materialized Views — кэш для сложных запросов

```sql
-- Дорогой запрос статистики
SELECT
    u.email,
    COUNT(t.id) AS total_tasks,
    COUNT(t.id) FILTER (WHERE t.done) AS done_tasks,
    AVG(t.priority) AS avg_priority,
    MAX(t.created_at) AS last_activity
FROM users u
LEFT JOIN tasks t ON t.user_id = u.id
GROUP BY u.id, u.email;
-- Время: 3 сек на 1M строк

-- Создать materialized view — результат сохраняется на диске
CREATE MATERIALIZED VIEW user_stats AS
SELECT
    u.id AS user_id,
    u.email,
    COUNT(t.id) AS total_tasks,
    COUNT(t.id) FILTER (WHERE t.done) AS done_tasks,
    AVG(t.priority)::NUMERIC(4,2) AS avg_priority,
    MAX(t.created_at) AS last_activity
FROM users u
LEFT JOIN tasks t ON t.user_id = u.id
GROUP BY u.id, u.email;

CREATE UNIQUE INDEX ON user_stats (user_id);

-- Теперь запрос мгновенный
SELECT * FROM user_stats WHERE user_id = 42;
-- Время: 1 ms

-- Обновлять materialized view по расписанию
-- CONCURRENTLY — без блокировки, нужен UNIQUE INDEX
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;

-- Через pg_cron каждые 5 минут
SELECT cron.schedule('*/5 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats');
```

### Advisory Locks — пользовательские блокировки

```sql
-- Получить эксклюзивный лок по числовому ключу
-- Полезно для distributed lock без Redis
SELECT pg_try_advisory_lock(42);    -- возвращает true/false (не блокирует)
SELECT pg_advisory_lock(42);        -- блокирует до получения лока

-- Освободить
SELECT pg_advisory_unlock(42);

-- В Go
func withAdvisoryLock(ctx context.Context, db *pgxpool.Pool, key int64, fn func() error) error {
    conn, err := db.Acquire(ctx)
    if err != nil { return err }
    defer conn.Release()

    var locked bool
    err = conn.QueryRow(ctx, "SELECT pg_try_advisory_lock($1)", key).Scan(&locked)
    if err != nil { return err }
    if !locked { return fmt.Errorf("lock is held by another process") }

    defer conn.Exec(ctx, "SELECT pg_advisory_unlock($1)", key)

    return fn()
}
```

### LISTEN/NOTIFY — события из PostgreSQL

```go
// PostgreSQL может отправлять события
// Приложение подписывается и получает их в реальном времени

// В БД — триггер на изменение задачи
CREATE OR REPLACE FUNCTION notify_task_change()
RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify(
        'task_changes',
        json_build_object(
            'id', NEW.id,
            'operation', TG_OP,
            'user_id', NEW.user_id
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER task_change_trigger
    AFTER INSERT OR UPDATE OR DELETE ON tasks
    FOR EACH ROW EXECUTE FUNCTION notify_task_change();

// В Go — подписаться на уведомления
func listenForChanges(ctx context.Context, connStr string) {
    conn, err := pgx.Connect(ctx, connStr)
    if err != nil { panic(err) }
    defer conn.Close(ctx)

    _, err = conn.Exec(ctx, "LISTEN task_changes")
    if err != nil { panic(err) }

    for {
        notification, err := conn.WaitForNotification(ctx)
        if err != nil {
            if ctx.Err() != nil { return }
            slog.Error("notification error", "error", err)
            continue
        }

        slog.Info("task changed",
            "channel", notification.Channel,
            "payload", notification.Payload,
        )
        // Инвалидировать кэш, отправить WebSocket событие и т.д.
    }
}
```

---

## Практические задания

### Задание 24.1 — Репликация локально

Настрой streaming replication через Docker Compose:

1. Primary на порту 5432
2. Replica на порту 5433
3. Проверить:
   ```sql
   -- На primary
   INSERT INTO tasks(title, user_id) VALUES('test', 1);

   -- На replica (через порт 5433)
   SELECT * FROM tasks WHERE title = 'test';
   -- Должна появиться та же строка
   ```
4. Добавь в Go приложение поддержку read/write splitting

---

### Задание 24.2 — Партиционирование

Мигрируй таблицу `tasks` на партиционированную версию по месяцам:

```sql
-- 1. Создать новую секционированную таблицу
-- 2. Перенести данные
-- 3. Переименовать
-- 4. Создать партицию для текущего месяца и следующего
-- 5. Написать функцию для автосоздания партиций
```

Проверить через `EXPLAIN ANALYZE` что запрос за конкретный месяц использует только одну партицию.

---

### Задание 24.3 — Full-Text Search

Добавь поиск по задачам:

1. Миграция: добавить `search_vector` как GENERATED STORED столбец
2. Создать GIN индекс
3. Метод `PostgresStorage.Search(ctx, params)` с ранжированием
4. Эндпоинт `GET /tasks/search?q=...`
5. Выделять совпадения через `ts_headline`

```bash
# Тест
curl "http://localhost:8080/tasks/search?q=kubernetes+кластер" \
  -H "Authorization: Bearer $TOKEN"
```

Сравни время ответа ILIKE vs FTS через `EXPLAIN ANALYZE`.

---

### Задание 24.4 — Materialized View для статистики

Создай materialized view `user_stats`:

```sql
-- total_tasks, done_tasks, pending_tasks, avg_priority, streak_days
```

```go
// Эндпоинт GET /users/me/stats
// Читает из materialized view — быстро
```

Настрой обновление через `pg_cron` каждые 5 минут (или через Go фоновый воркер).

---

### Задание 24.5 — Финальный проект урока

Полная production-ready схема БД:

**Партиционирование:**
- Таблица `tasks` секционирована по месяцам
- Автосоздание партиций через pg_cron
- Процедура архивирования партиций старше 2 лет

**Репликация:**
- Primary + 1 replica в docker-compose
- Go приложение использует replica для SELECT
- Метрика `replica_lag_seconds` в Prometheus

**Full-Text Search:**
- Поиск по title + description (если есть)
- Поддержка русского и английского языков
- Ранжирование и подсветка совпадений

**Мониторинг БД:**
- Запросы к `pg_stat_statements` для медленных запросов
- Размер таблиц и индексов
- Количество активных соединений
- Lag репликации

Закоммить с тегом `v2.4.0`.

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `pg_stat_replication` | Статус репликации |
| `pg_is_in_recovery()` | True если это replica |
| `PARTITION BY RANGE (col)` | Секционировать по диапазону |
| `PARTITION BY HASH (col)` | Секционировать по хэшу |
| `EXPLAIN` + `partition pruning` | Проверить что лишние секции не сканируются |
| `to_tsvector('lang', text)` | Преобразовать текст для FTS |
| `plainto_tsquery('lang', text)` | Поисковый запрос |
| `ts_rank(vector, query)` | Оценка релевантности |
| `ts_headline(...)` | Подсветить совпадения |
| `REFRESH MATERIALIZED VIEW CONCURRENTLY` | Обновить без блокировки |
| `pg_notify(channel, payload)` | Отправить уведомление |
| `LISTEN channel` | Подписаться на уведомления |

---

## Ресурсы для изучения

- **Streaming Replication:** `https://www.postgresql.org/docs/current/warm-standby.html`
- **Partitioning:** `https://www.postgresql.org/docs/current/ddl-partitioning.html`
- **Full-Text Search:** `https://www.postgresql.org/docs/current/textsearch.html`
- **pg_cron:** `https://github.com/citusdata/pg_cron`
- **Use The Index Luke:** `https://use-the-index-luke.com` — глубоко про индексы
- **Книга:** "PostgreSQL: Up and Running" — Regina Obe, Leo Hsu

---

## Как понять что урок пройден

- [ ] Streaming replication работает: данные на primary появляются на replica
- [ ] Go приложение читает с replica, пишет на primary
- [ ] Таблица `tasks` партиционирована, EXPLAIN показывает partition pruning
- [ ] Full-text search возвращает результаты с ранжированием и подсветкой
- [ ] FTS в 100+ раз быстрее ILIKE на больших данных (проверил через EXPLAIN ANALYZE)
- [ ] Materialized view обновляется по расписанию
- [ ] Lag репликации виден в Prometheus метриках

---

*Следующий урок: Карьера — как устроиться на работу, System Design интервью, портфолио*
