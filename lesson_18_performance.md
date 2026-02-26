# Урок 18 — Производительность: профилирование, нагрузочное тестирование, оптимизация

## Зачем это нужно

"Работает медленно" — одна из самых частых жалоб в продакшене. Но оптимизировать без данных — это гадание. Профилировщик показывает где именно тратится время и память. Нагрузочный тест показывает как система ведёт себя под реальной нагрузкой до того как это случится в продакшене. После этого урока ты не будешь гадать — ты будешь знать.

**Главное правило производительности:** сначала измерь, потом оптимизируй. Преждевременная оптимизация — корень всех зол (Дональд Кнут).

---

## Часть 1 — Бенчмарки в Go

### Написание бенчмарков

```go
// storage_bench_test.go
package main

import (
    "context"
    "testing"
)

// Имя должно начинаться с Benchmark
func BenchmarkMemoryStorage_Add(b *testing.B) {
    s := NewMemoryStorage()
    ctx := context.Background()

    b.ResetTimer()    // не считать время инициализации

    for i := 0; i < b.N; i++ {
        s.Add(ctx, "task", 1)
    }
}

func BenchmarkMemoryStorage_List(b *testing.B) {
    s := NewMemoryStorage()
    ctx := context.Background()

    // Подготовить данные
    for i := 0; i < 1000; i++ {
        s.Add(ctx, fmt.Sprintf("task %d", i), 1)
    }

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        s.List(ctx)
    }
}

// Бенчмарк с параллельным выполнением
func BenchmarkMemoryStorage_Concurrent(b *testing.B) {
    s := NewMemoryStorage()
    ctx := context.Background()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            s.Add(ctx, "task", 1)
        }
    })
}

// Бенчмарк с sub-тестами для разных размеров
func BenchmarkJSONMarshal(b *testing.B) {
    sizes := []int{1, 10, 100, 1000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            tasks := make([]Task, size)
            for i := range tasks {
                tasks[i] = Task{ID: i, Title: fmt.Sprintf("task %d", i)}
            }

            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                json.Marshal(tasks)
            }
        })
    }
}
```

```bash
# Запустить все бенчмарки
go test -bench=. ./...

# Конкретный бенчмарк
go test -bench=BenchmarkMemoryStorage_Add -benchtime=5s ./...

# С информацией о памяти
go test -bench=. -benchmem ./...

# Вывод:
# BenchmarkMemoryStorage_Add-8   5000000   234 ns/op   48 B/op   2 allocs/op
# Название           горутины  итерации  нс на оп  байт на оп  аллокаций

# Сравнение двух реализаций
go test -bench=. -count=5 ./... > old.txt
# Поменять реализацию
go test -bench=. -count=5 ./... > new.txt
go install golang.org/x/perf/cmd/benchstat@latest
benchstat old.txt new.txt
```

---

## Часть 2 — pprof: профилировщик Go

### Подключение pprof

```go
import _ "net/http/pprof"    // регистрирует эндпоинты автоматически

// Добавить в main.go
go func() {
    // Отдельный порт чтобы не светить в продакшене
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

```bash
# Эндпоинты pprof:
# http://localhost:6060/debug/pprof/          — список всех профилей
# http://localhost:6060/debug/pprof/goroutine — дамп горутин
# http://localhost:6060/debug/pprof/heap      — использование памяти
# http://localhost:6060/debug/pprof/profile   — CPU профиль (30 сек)
# http://localhost:6060/debug/pprof/trace     — трейс выполнения
```

### CPU профилирование

```bash
# Собрать CPU профиль (нагрузи приложение во время сбора)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Или сохранить файл и анализировать позже
curl -o cpu.pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof cpu.pprof

# Внутри интерактивной консоли pprof:
(pprof) top                  # топ функций по времени CPU
(pprof) top -cum             # с учётом вызываемых функций
(pprof) list functionName    # исходный код с аннотациями
(pprof) web                  # открыть граф в браузере (нужен graphviz)
(pprof) png > cpu.png        # сохранить граф как картинку
```

### Heap профилирование (утечки памяти)

```go
// Симуляция утечки памяти — для обучения
var cache = make(map[string][]byte)

func leakyHandler(w http.ResponseWriter, r *http.Request) {
    key := r.URL.Query().Get("key")
    // БАГ: никогда не удаляем из cache
    cache[key] = make([]byte, 1024*1024)    // 1MB
    fmt.Fprintf(w, "stored %s", key)
}
```

```bash
# Собрать heap профиль
go tool pprof http://localhost:6060/debug/pprof/heap

(pprof) top                  # топ по использованию памяти
(pprof) list leakyHandler    # посмотреть проблемную функцию

# Сравнить два heap снимка — найти утечку
curl -o heap1.pprof http://localhost:6060/debug/pprof/heap
# ... нагрузить приложение ...
curl -o heap2.pprof http://localhost:6060/debug/pprof/heap
go tool pprof -base heap1.pprof heap2.pprof
# Покажет что добавилось между двумя снимками
```

### Горутины — найти утечку горутин

```go
// Утечка горутины
func badHandler(w http.ResponseWriter, r *http.Request) {
    ch := make(chan string)    // небуферизованный канал

    go func() {
        time.Sleep(time.Hour)
        ch <- "result"    // горутина висит час ожидая читателя
    }()

    // Обработчик завершился — никто не читает ch — горутина зависла навсегда
    fmt.Fprintf(w, "ok")
}
```

```bash
# Посмотреть сколько горутин и что они делают
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Метрика в Prometheus
go_goroutines    # если растёт — утечка горутин
```

### Visualizing с помощью flamegraph

```bash
# Установить Speedscope для красивых flamegraph
npm install -g speedscope

# Собрать профиль в формате для speedscope
curl -o cpu.pprof "http://localhost:6060/debug/pprof/profile?seconds=30"
go tool pprof -proto cpu.pprof | speedscope -

# Или использовать встроенный веб-интерфейс pprof
go tool pprof -http=:8081 cpu.pprof
# Открой http://localhost:8081 — красивые flamegraph
```

---

## Часть 3 — Нагрузочное тестирование

### hey — простой нагрузочный тест

```bash
go install github.com/rakyll/hey@latest

# Базовый тест
hey -n 1000 -c 10 http://localhost:8080/tasks
# -n 1000 — всего запросов
# -c 10   — параллельных запросов

# С авторизацией
hey -n 5000 -c 50 \
    -H "Authorization: Bearer $TOKEN" \
    http://localhost:8080/tasks

# POST запрос
hey -n 1000 -c 20 \
    -m POST \
    -H "Content-Type: application/json" \
    -d '{"title":"test","priority":1}' \
    -H "Authorization: Bearer $TOKEN" \
    http://localhost:8080/tasks

# Вывод:
# Summary:
#   Total:    2.5s
#   Slowest:  0.234s
#   Fastest:  0.001s
#   Average:  0.025s
#   Requests/sec: 2000
#
# Response time histogram:
#   0.001 [100]  |■■■
#   0.025 [800]  |■■■■■■■■■■■■■■■■■■■■■■■
#   0.100 [95]   |■■■
#   0.234 [5]    |
#
# Latency distribution:
#   10% in 0.005s
#   50% in 0.025s
#   75% in 0.040s
#   90% in 0.060s
#   95% in 0.080s
#   99% in 0.150s
```

### k6 — профессиональное нагрузочное тестирование

```bash
# Установить k6
sudo gpg -k
sudo gpg --no-default-keyring \
    --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
    --keyserver hkp://keyserver.ubuntu.com:80 \
    --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" \
    | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6
```

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Кастомные метрики
const errorRate = new Rate('error_rate');
const taskCreationTime = new Trend('task_creation_time');

// Конфигурация нагрузки
export const options = {
    stages: [
        { duration: '1m',  target: 10  },   // разогрев: 10 пользователей
        { duration: '3m',  target: 50  },   // нагрузка: 50 пользователей
        { duration: '1m',  target: 100 },   // пик: 100 пользователей
        { duration: '2m',  target: 50  },   // снижение
        { duration: '1m',  target: 0   },   // охлаждение
    ],
    thresholds: {
        'http_req_duration': ['p(95)<500'],    // 95% запросов < 500ms
        'error_rate': ['rate<0.01'],            // ошибок < 1%
        'http_req_failed': ['rate<0.01'],
    },
};

const BASE_URL = 'http://localhost:8080';

// Авторизоваться один раз перед тестом
export function setup() {
    const res = http.post(`${BASE_URL}/auth/login`, JSON.stringify({
        email: 'test@example.com',
        password: 'testpassword',
    }), {
        headers: { 'Content-Type': 'application/json' },
    });

    return { token: JSON.parse(res.body).token };
}

// Основной сценарий
export default function(data) {
    const headers = {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${data.token}`,
    };

    // 1. Получить список задач
    const listRes = http.get(`${BASE_URL}/tasks`, { headers });
    check(listRes, {
        'list status 200': (r) => r.status === 200,
        'list has tasks': (r) => JSON.parse(r.body).length >= 0,
    });
    errorRate.add(listRes.status !== 200);

    sleep(0.5);

    // 2. Создать задачу
    const createStart = Date.now();
    const createRes = http.post(`${BASE_URL}/tasks`, JSON.stringify({
        title: `Task ${Date.now()}`,
        priority: Math.floor(Math.random() * 3) + 1,
    }), { headers });

    taskCreationTime.add(Date.now() - createStart);

    const taskID = JSON.parse(createRes.body)?.id;
    check(createRes, {
        'create status 201': (r) => r.status === 201,
        'create has id': (r) => taskID !== undefined,
    });

    sleep(1);

    // 3. Выполнить задачу
    if (taskID) {
        const doneRes = http.patch(`${BASE_URL}/tasks/${taskID}/done`, null, { headers });
        check(doneRes, { 'done status 200': (r) => r.status === 200 });
    }

    sleep(1);
}
```

```bash
# Запустить тест
k6 run load-test.js

# С выводом в реальном времени
k6 run --out json=results.json load-test.js

# Smoke test — минимальная нагрузка чтобы проверить что всё работает
k6 run --vus 1 --duration 30s load-test.js

# Stress test — найти точку поломки
k6 run --vus 500 --duration 5m load-test.js
```

---

## Часть 4 — Оптимизация Go кода

### Избегай лишних аллокаций

```go
// МЕДЛЕННО — новая строка на каждой итерации
func buildQuery(filters []string) string {
    result := ""
    for _, f := range filters {
        result += f + " AND "    // каждый += создаёт новую строку
    }
    return result
}

// БЫСТРО — strings.Builder использует буфер
func buildQuery(filters []string) string {
    var sb strings.Builder
    for i, f := range filters {
        if i > 0 {
            sb.WriteString(" AND ")
        }
        sb.WriteString(f)
    }
    return sb.String()
}

// МЕДЛЕННО — аллокация на каждый вызов
func processItems(items []Item) []Result {
    results := []Result{}    // пустой слайс — будет расти
    for _, item := range items {
        results = append(results, process(item))
    }
    return results
}

// БЫСТРО — предаллоцировать нужный размер
func processItems(items []Item) []Result {
    results := make([]Result, 0, len(items))    // capacity известен
    for _, item := range items {
        results = append(results, process(item))
    }
    return results
}
```

### sync.Pool — переиспользование объектов

```go
// Пример: переиспользование буферов для JSON сериализации
var bufPool = sync.Pool{
    New: func() interface{} {
        return &bytes.Buffer{}
    },
}

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return
    }
    w.Write(buf.Bytes())
}
```

### Горутины: осторожно с количеством

```go
// ПЛОХО — запускать горутину на каждый элемент без ограничений
func processBatch(items []Item) {
    for _, item := range items {
        go processItem(item)    // при 10000 items — 10000 горутин
    }
}

// ХОРОШО — worker pool с ограниченным числом горутин
func processBatch(items []Item, workerCount int) {
    jobs := make(chan Item, len(items))
    var wg sync.WaitGroup

    // Запустить workers
    for w := 0; w < workerCount; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {
                processItem(item)
            }
        }()
    }

    // Отправить задачи
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    wg.Wait()
}
```

### Кэширование на уровне приложения

```go
// Простой in-memory кэш с TTL
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]cacheItem[V]
}

type cacheItem[V any] struct {
    value     V
    expiresAt time.Time
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    item, ok := c.items[key]
    if !ok || time.Now().After(item.expiresAt) {
        var zero V
        return zero, false
    }
    return item.value, true
}

func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.items[key] = cacheItem[V]{
        value:     value,
        expiresAt: time.Now().Add(ttl),
    }
}
```

---

## Часть 5 — Оптимизация PostgreSQL

### EXPLAIN ANALYZE — читать планы запросов

```sql
-- Медленный запрос
EXPLAIN ANALYZE
SELECT t.*, u.email
FROM tasks t
JOIN users u ON u.id = t.user_id
WHERE t.done = false
  AND t.user_id = 42
ORDER BY t.created_at DESC
LIMIT 20;

-- Вывод:
-- Limit  (cost=0.43..8.45 rows=20) (actual time=0.123..0.456 rows=20 loops=1)
--   ->  Index Scan Backward using idx_tasks_user_created on tasks t
--         (cost=0.43..412.89 rows=1000) (actual time=0.120..0.440 rows=20)
--         Index Cond: (user_id = 42)
--         Filter: (done = false)
--         Rows Removed by Filter: 180
--
-- Planning Time: 0.5 ms
-- Execution Time: 0.8 ms

-- Что искать:
-- "Seq Scan" на большой таблице — нужен индекс
-- "Rows Removed by Filter" — много строк отфильтровано — нужен более специфичный индекс
-- "Nested Loop" с большим loops= — возможно N+1 запрос
-- "Hash" вместо "Index" — нет индекса для JOIN
```

### Частые проблемы и решения

```sql
-- Проблема 1: Нет индекса для часто используемого фильтра
SELECT * FROM tasks WHERE user_id = 42 AND done = false;
-- Seq Scan на миллионе строк

-- Решение: составной индекс
CREATE INDEX idx_tasks_user_done ON tasks(user_id, done)
    WHERE done = false;    -- partial index — только незакрытые задачи

-- Проблема 2: N+1 запросы
-- Плохо: в Go коде
for _, task := range tasks {
    user, _ := db.GetUser(task.UserID)    // N запросов к БД
}

-- Хорошо: один JOIN запрос
SELECT t.*, u.email FROM tasks t JOIN users u ON u.id = t.user_id;

-- Проблема 3: SELECT * — тянуть лишние данные
SELECT * FROM tasks;    -- тянем все колонки включая большие TEXT

-- Решение: выбирать только нужные поля
SELECT id, title, done, created_at FROM tasks;

-- Проблема 4: ILIKE без индекса для полнотекстового поиска
SELECT * FROM tasks WHERE title ILIKE '%go%';    -- Seq Scan всегда

-- Решение: GIN индекс для полнотекстового поиска
CREATE INDEX idx_tasks_title_fts ON tasks USING gin(to_tsvector('russian', title));
SELECT * FROM tasks WHERE to_tsvector('russian', title) @@ plainto_tsquery('russian', 'go');

-- Проблема 5: Пагинация через OFFSET при больших значениях
SELECT * FROM tasks ORDER BY id LIMIT 20 OFFSET 10000;
-- PostgreSQL сканирует 10020 строк и выбрасывает 10000

-- Решение: keyset pagination
SELECT * FROM tasks WHERE id > 10000 ORDER BY id LIMIT 20;
```

### Connection Pooling

```go
// pgxpool — правильные настройки пула соединений
config, _ := pgxpool.ParseConfig(databaseURL)
config.MaxConns = 20               // максимум соединений
config.MinConns = 5                // минимум (держать готовыми)
config.MaxConnLifetime = time.Hour // пересоздавать соединение через час
config.MaxConnIdleTime = 30 * time.Minute
config.HealthCheckPeriod = time.Minute

pool, _ := pgxpool.NewWithConfig(ctx, config)

// Мониторинг пула
stats := pool.Stat()
slog.Info("pool stats",
    "total", stats.TotalConns(),
    "idle", stats.IdleConns(),
    "acquired", stats.AcquiredConns(),
)
```

---

## Практические задания

### Задание 18.1 — Бенчмарки

Напиши бенчмарки для `todo-service`:

1. `BenchmarkStorageAdd` — добавление задачи (MemoryStorage и PostgresStorage)
2. `BenchmarkStorageList` — список при 100, 1000, 10000 задачах
3. `BenchmarkJSONMarshal` — сериализация списка задач разного размера
4. `BenchmarkAuthMiddleware` — проверка JWT токена

```bash
go test -bench=. -benchmem -benchtime=3s ./...
```

**Что нужно сделать:** Найди самую медленную операцию. Объясни почему она медленная.

---

### Задание 18.2 — pprof под нагрузкой

1. Добавь pprof эндпоинты в `todo-service` (на отдельном порту `:6060`)
2. Запусти приложение
3. В фоне запусти нагрузочный тест:
   ```bash
   hey -n 10000 -c 50 -H "Authorization: Bearer $TOKEN" http://localhost:8080/tasks
   ```
4. Во время теста собери CPU профиль:
   ```bash
   go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=30
   ```
5. Открой `http://localhost:8081` — найди топ функций по CPU

**Что нужно сделать:** Сделай скриншот flamegraph. Найди что занимает больше всего CPU и объясни почему.

---

### Задание 18.3 — Нагрузочный тест с k6

Напиши `tests/load-test.js` для `todo-service`:

1. Setup: создать тестового пользователя и получить токен
2. Сценарий (80% get, 15% create, 5% complete):
   ```
   Stages:
   - 1 min: разогрев до 10 пользователей
   - 3 min: нагрузка 50 пользователей
   - 1 min: пик 100 пользователей
   - 1 min: охлаждение
   ```
3. Thresholds:
   - p95 < 200ms
   - error rate < 1%

```bash
k6 run tests/load-test.js
```

**Что нужно сделать:** Найди при каком количестве пользователей p95 latency превышает 200ms. Это точка насыщения системы.

---

### Задание 18.4 — Найти и исправить узкое место

После нагрузочного теста наверняка найдётся узкое место. Типичные проблемы:

**Сценарий A: медленные запросы к БД**
```bash
# Включить логирование медленных запросов
ALTER SYSTEM SET log_min_duration_statement = '50';
SELECT pg_reload_conf();

# Смотреть логи PostgreSQL
docker compose logs postgres | grep duration
```

**Сценарий B: много аллокаций памяти**
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
(pprof) top -cum
```

**Сценарий C: N+1 запросы**
```go
// Добавить логирование количества запросов к БД
// Если на один HTTP запрос идёт N+1 SQL — переписать на JOIN
```

**Что нужно сделать:**
1. Запусти нагрузочный тест
2. Собери CPU + heap профиль
3. Найди узкое место
4. Исправь
5. Повтори тест
6. Сравни результаты через `benchstat`

---

### Задание 18.5 — Финальный проект урока

Полная оптимизация `todo-service`:

**Измерения до оптимизации:**
```bash
k6 run tests/load-test.js > before.txt
```

**Оптимизации которые нужно применить:**

1. **Индексы в PostgreSQL:**
   - Составной индекс на `(user_id, done)`
   - Индекс на `created_at DESC` для сортировки
   - Проверь через `EXPLAIN ANALYZE` что индексы используются

2. **Connection pool:**
   - Настрой правильные `MaxConns`, `MinConns`
   - Добавь метрику использования пула в Prometheus

3. **Redis кэш:**
   - Кэшировать `GET /tasks` на 30 секунд
   - Измерить cache hit rate

4. **JSON оптимизация:**
   - Использовать `sync.Pool` для буферов JSON
   - Или заменить `encoding/json` на `github.com/bytedance/sonic`

5. **Бенчмарки до и после каждой оптимизации**

**Измерения после:**
```bash
k6 run tests/load-test.js > after.txt
```

Создай файл `PERFORMANCE.md` с результатами:
```markdown
## Performance Report

### Before optimization
- P50: 45ms
- P95: 230ms
- RPS: 850

### After optimization
- P50: 8ms
- P95: 42ms
- RPS: 3200

### Changes made
1. Added composite index on (user_id, done) — P95 улучшился на 40%
2. Redis cache for GET /tasks — RPS вырос в 3.7x
3. Connection pool tuning — устранили connection timeout под нагрузкой
```

Закоммить с тегом `v1.2.0`.

---

## Шпаргалка

| Инструмент | Что измеряет |
|-----------|-------------|
| `go test -bench=.` | Скорость конкретной функции |
| `go tool pprof` + CPU | Где тратится время CPU |
| `go tool pprof` + heap | Где аллоцируется память |
| `hey` | Простая HTTP нагрузка |
| `k6` | Сложные сценарии нагрузки |
| `EXPLAIN ANALYZE` | План SQL запроса |

| Симптом | Диагноз | Лечение |
|---------|---------|---------|
| Высокое CPU | pprof CPU профиль | Найти горячую функцию |
| Растёт память | pprof heap профиль | Найти утечку |
| Медленные запросы | `EXPLAIN ANALYZE` | Добавить индекс |
| Много соединений к БД | `pool.Stat()` | Настроить MaxConns |
| N+1 запросы | Логи SQL | Переписать на JOIN |

---

## Ресурсы для изучения

- **pprof:** `https://pkg.go.dev/runtime/pprof`
- **Profiling Go:** `https://go.dev/blog/pprof`
- **k6:** `https://k6.io/docs/`
- **hey:** `https://github.com/rakyll/hey`
- **benchstat:** `https://pkg.go.dev/golang.org/x/perf/cmd/benchstat`
- **Use The Index Luke:** `https://use-the-index-luke.com` — про индексы БД
- **Книга:** "Systems Performance" — Brendan Gregg

---

## Как понять что урок пройден

- [ ] Написал бенчмарки для основных операций
- [ ] Умею запустить pprof и читать flamegraph
- [ ] Нашёл узкое место через профилировщик (не угадал)
- [ ] Написал k6 тест с реальными сценариями и thresholds
- [ ] Добавил правильные индексы в PostgreSQL
- [ ] Сравнил производительность до и после через benchstat/k6
- [ ] Создал `PERFORMANCE.md` с измерениями

---

*Следующий урок: SRE практики — SLI/SLO, Chaos Engineering, Disaster Recovery*
