# Урок 10 — Мониторинг: Prometheus, Grafana и логирование

## Зачем это нужно

Приложение в продакшене — это чёрный ящик. Без мониторинга ты узнаёшь о проблемах от пользователей, а не от системы. Хороший мониторинг отвечает на три вопроса: **что сейчас происходит** (метрики), **что произошло** (логи), **почему это произошло** (трейсинг). Это называется Observability — наблюдаемость системы.

---

## Часть 1 — Три столпа Observability

### Метрики
Числовые измерения собранные с течением времени. Отвечают на вопрос "сколько":
- Сколько запросов в секунду обрабатывает сервер
- Сколько памяти использует приложение
- Какой процент запросов завершается ошибкой
- Как долго в среднем обрабатывается запрос

### Логи
Текстовые записи о событиях. Отвечают на вопрос "что произошло":
- Пользователь авторизовался
- Запрос вернул ошибку 500
- База данных не ответила за 30 секунд

### Трейсинг
Путь запроса через все сервисы системы. Отвечает на вопрос "где тормозит":
- Запрос зашёл в API → пошёл в БД (100ms) → пошёл в Redis (5ms) → вернул ответ
- Видно где именно возникло узкое место

---

## Часть 2 — Prometheus

Prometheus — система мониторинга с собственной базой данных временных рядов. Работает по принципу pull — сам ходит к приложениям и собирает метрики.

### Как работает Prometheus

```
Приложение                 Prometheus              Grafana
┌──────────┐               ┌──────────┐            ┌──────────┐
│ /metrics │ ←─── scrape ──│  scrape  │            │dashboard │
│          │  каждые 15s   │  store   │ ←── query ─│          │
└──────────┘               │  alert   │            └──────────┘
                           └──────────┘
                                │
                           Alertmanager
                                │
                           Email/Slack/PagerDuty
```

### Типы метрик

```
Counter   — монотонно растущий счётчик (только увеличивается)
           Примеры: количество запросов, количество ошибок
           Вопрос: "сколько всего было?"

Gauge     — значение которое может расти и падать
           Примеры: использование памяти, количество горутин, температура
           Вопрос: "сколько сейчас?"

Histogram — распределение значений по bucket-ам
           Примеры: время ответа, размер запроса
           Вопрос: "как распределены значения?"

Summary   — похож на Histogram, считает квантили на стороне приложения
           Используй Histogram — он гибче
```

### Метрики в Go приложении

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promauto
go get github.com/prometheus/client_golang/prometheus/promhttp
```

```go
package main

import (
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // Counter — количество HTTP запросов
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},  // labels
    )

    // Histogram — время обработки запроса
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5},
        },
        []string{"method", "path"},
    )

    // Gauge — количество активных задач
    tasksTotal = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "todo_tasks_total",
            Help: "Total number of tasks",
        },
    )

    // Gauge — количество активных DB соединений
    dbConnectionsActive = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "db_connections_active",
            Help: "Number of active database connections",
        },
    )
)

// Middleware для сбора метрик
func metricsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Обёртка для захвата статус кода
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next(rw, r)

        duration := time.Since(start).Seconds()
        status := http.StatusText(rw.statusCode)

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    }
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func main() {
    // Эндпоинт для Prometheus
    http.Handle("/metrics", promhttp.Handler())

    // Остальные эндпоинты с метриками
    http.HandleFunc("/tasks", metricsMiddleware(listTasks))
    http.HandleFunc("/health", healthHandler)

    http.ListenAndServe(":8080", nil)
}
```

### Формат метрик `/metrics`

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/tasks",status="OK"} 42
http_requests_total{method="POST",path="/tasks",status="Created"} 10

# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",path="/tasks",le="0.005"} 35
http_request_duration_seconds_bucket{method="GET",path="/tasks",le="0.01"} 40
http_request_duration_seconds_bucket{method="GET",path="/tasks",le="+Inf"} 42
http_request_duration_seconds_sum{method="GET",path="/tasks"} 0.187
http_request_duration_seconds_count{method="GET",path="/tasks"} 42
```

### Запуск Prometheus

`prometheus.yml`:
```yaml
global:
  scrape_interval: 15s      # как часто собирать метрики
  evaluation_interval: 15s  # как часто проверять правила алертов

scrape_configs:
  - job_name: 'todo-service'
    static_configs:
      - targets: ['app:8080']    # app — имя сервиса в docker compose

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### PromQL — язык запросов Prometheus

```promql
# Текущее значение метрики
http_requests_total

# Значение с конкретным label
http_requests_total{method="GET"}

# Rate — количество в секунду за последние 5 минут
rate(http_requests_total[5m])

# Сумма по всем методам
sum(rate(http_requests_total[5m]))

# Разбивка по статусу
sum(rate(http_requests_total[5m])) by (status)

# Процент ошибок
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# 95-й перцентиль времени ответа
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Использование памяти
process_resident_memory_bytes / 1024 / 1024  # в MB
```

---

## Часть 3 — Grafana

Grafana — инструмент для визуализации метрик. Подключается к Prometheus и строит дашборды.

### Запуск Grafana

Добавь в `docker-compose.yml`:

```yaml
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    ports:
      - "9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin123
      GF_USERS_ALLOW_SIGN_UP: false
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

### Автоматическая настройка Grafana

`monitoring/grafana/provisioning/datasources/prometheus.yml`:
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### Основные дашборды для приложения

Создай дашборд в Grafana с такими панелями:

1. **RPS (Requests Per Second)** — запросов в секунду
   ```promql
   sum(rate(http_requests_total[1m]))
   ```

2. **Error Rate** — процент ошибок
   ```promql
   sum(rate(http_requests_total{status=~"5.."}[1m]))
   / sum(rate(http_requests_total[1m])) * 100
   ```

3. **P95 Latency** — время ответа 95-го перцентиля
   ```promql
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[1m])) by (le))
   ```

4. **Memory Usage** — использование памяти
   ```promql
   process_resident_memory_bytes / 1024 / 1024
   ```

5. **Goroutines** — количество горутин
   ```promql
   go_goroutines
   ```

---

## Часть 4 — Логирование

### Структурированные логи

Плохо — текстовые логи:
```
2024-01-15 10:30:00 ERROR failed to connect to database: connection refused
```

Хорошо — структурированные JSON логи:
```json
{
  "time": "2024-01-15T10:30:00Z",
  "level": "error",
  "msg": "failed to connect to database",
  "error": "connection refused",
  "host": "localhost",
  "port": 5432,
  "attempt": 3,
  "service": "todo-service",
  "version": "1.2.3"
}
```

Структурированные логи легко парсить, фильтровать и искать по полям.

### slog — стандартная библиотека Go

```go
package main

import (
    "context"
    "log/slog"
    "os"
)

func main() {
    // JSON логгер
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,    // минимальный уровень
    }))

    // Установить как глобальный
    slog.SetDefault(logger)

    // Использование
    slog.Info("server started", "port", 8080)
    slog.Error("database connection failed",
        "error", err,
        "host", "localhost",
        "port", 5432,
    )
    slog.Debug("processing request", "method", "GET", "path", "/tasks")

    // С контекстом (для трейсинга)
    ctx := context.Background()
    slog.InfoContext(ctx, "task created", "task_id", 42, "title", "Learn Go")
}
```

### Middleware для логирования запросов

```go
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next(rw, r)

        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "status", rw.statusCode,
            "duration_ms", time.Since(start).Milliseconds(),
            "remote_addr", r.RemoteAddr,
            "user_agent", r.UserAgent(),
        )
    }
}
```

Вывод:
```json
{"time":"2024-01-15T10:30:00.123Z","level":"INFO","msg":"request","method":"GET","path":"/tasks","status":200,"duration_ms":3,"remote_addr":"172.17.0.1:54321","user_agent":"curl/7.88.1"}
```

### Уровни логов

```go
slog.Debug(...)  // отладочная информация, не нужна в продакшене
slog.Info(...)   // нормальная работа системы
slog.Warn(...)   // что-то подозрительное, но не критичное
slog.Error(...)  // ошибка, требует внимания
```

Управляй уровнем через переменную окружения:
```go
levelStr := os.Getenv("LOG_LEVEL")
level := slog.LevelInfo
switch levelStr {
case "debug": level = slog.LevelDebug
case "warn":  level = slog.LevelWarn
case "error": level = slog.LevelError
}
```

### Loki — агрегация логов

Loki — система хранения логов от создателей Grafana. Работает в паре с Promtail (агент сбора логов).

```yaml
# docker-compose.yml дополнение
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./monitoring/promtail.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
    restart: unless-stopped
```

`monitoring/promtail.yml`:
```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: container
```

---

## Часть 5 — Алертинг

### Правила алертов в Prometheus

`monitoring/alerts.yml`:
```yaml
groups:
  - name: todo-service
    rules:
      # Алерт если сервис недоступен
      - alert: ServiceDown
        expr: up{job="todo-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Todo service is down"
          description: "Todo service has been down for more than 1 minute"

      # Алерт если много ошибок
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # Алерт если медленные ответы
      - alert: SlowResponses
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow API responses"
          description: "95th percentile response time is {{ $value }}s"

      # Алерт если много памяти
      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes > 500 * 1024 * 1024
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanize1024 }}B"
```

Добавь в `prometheus.yml`:
```yaml
rule_files:
  - /etc/prometheus/alerts.yml
```

---

## Практические задания

### Задание 10.1 — Метрики в приложении

Добавь в `todo-service` метрики Prometheus:

1. `http_requests_total` с labels: method, path, status
2. `http_request_duration_seconds` — гистограмма времени ответа
3. `todo_tasks_total` — текущее количество задач (обновляй при create/delete)
4. `db_query_duration_seconds` — время выполнения запросов к БД

```bash
# Проверь что метрики доступны
curl http://localhost:8080/metrics
```

Найди в выводе свои метрики.

---

### Задание 10.2 — Prometheus + Grafana

Добавь в `docker-compose.yml` сервисы Prometheus и Grafana:

1. Создай `monitoring/prometheus.yml` с настройкой scrape для `app`
2. Запусти стек: `docker compose up -d`
3. Открой Prometheus UI: `http://localhost:9090`
   - Убедись что `todo-service` в статусе UP (Status → Targets)
   - Выполни запрос: `rate(http_requests_total[1m])`
4. Открой Grafana: `http://localhost:3000` (admin/admin123)
   - Подключи Prometheus как datasource
   - Создай дашборд с 4 панелями: RPS, Error Rate, P95 Latency, Memory

---

### Задание 10.3 — Структурированное логирование

Замени все `fmt.Println` и `log.Printf` в `todo-service` на `slog`:

1. Логируй каждый HTTP запрос (method, path, status, duration)
2. Логируй ошибки с контекстом (что делали, какая ошибка)
3. Логируй старт и остановку сервера
4. Логируй подключение к БД
5. Управляй уровнем через `LOG_LEVEL` env

```bash
# Запусти и посмотри логи
docker compose up -d
docker compose logs -f app | python3 -m json.tool
# Должны быть красивые JSON логи
```

---

### Задание 10.4 — Loki + Grafana

Добавь Loki и Promtail в `docker-compose.yml`:

1. Настрой Promtail для сбора логов Docker контейнеров
2. В Grafana добавь Loki как datasource
3. Создай запросы в Grafana Explore:
   - Все логи `todo-service`: `{container="todo-app"}`
   - Только ошибки: `{container="todo-app"} | json | level="ERROR"`
   - Медленные запросы: `{container="todo-app"} | json | duration_ms > 100`

---

### Задание 10.5 — Финальный проект урока

Собери полный стек наблюдаемости для `todo-service`:

**Структура проекта:**
```
monitoring/
├── prometheus.yml
├── alerts.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   ├── prometheus.yml
        │   └── loki.yml
        └── dashboards/
            ├── dashboard.yml
            └── todo-service.json    # экспортированный дашборд
```

**Что должно работать:**
1. `docker compose up -d` поднимает: app, postgres, prometheus, grafana, loki, promtail
2. Grafana дашборд показывает: RPS, Error Rate, P95 Latency, Memory, активные задачи
3. Логи всех контейнеров видны в Grafana Explore
4. Три правила алертов настроены в Prometheus

**Нагрузочный тест:**
```bash
# Установи hey
go install github.com/rakyll/hey@latest

# Создай нагрузку
hey -n 1000 -c 10 http://localhost:8080/tasks

# Посмотри как меняются метрики в Grafana в реальном времени
```

Закоммить с тегом `v0.5.0`.

---

## Шпаргалка

| Инструмент | Назначение | Порт |
|-----------|-----------|------|
| Prometheus | Сбор и хранение метрик | 9090 |
| Grafana | Визуализация | 3000 |
| Loki | Хранение логов | 3100 |
| Promtail | Агент сбора логов | 9080 |

| PromQL | Что считает |
|--------|------------|
| `rate(counter[5m])` | Скорость роста счётчика |
| `sum(...) by (label)` | Сумма с группировкой |
| `histogram_quantile(0.95, ...)` | 95-й перцентиль |
| `increase(counter[1h])` | Прирост за час |

---

## Ресурсы для изучения

- **Prometheus:** `https://prometheus.io/docs/`
- **PromQL:** `https://promlabs.com/promql-cheat-sheet/`
- **Grafana:** `https://grafana.com/docs/`
- **slog:** `https://pkg.go.dev/log/slog`
- **Готовые дашборды:** `https://grafana.com/grafana/dashboards/` — ищи Go Application

---

## Как понять что урок пройден

- [ ] Понимаю разницу между метриками, логами и трейсингом
- [ ] Знаю 4 типа метрик Prometheus и когда какой использовать
- [ ] Приложение отдаёт метрики на `/metrics`
- [ ] Prometheus собирает метрики и они видны в UI
- [ ] Grafana дашборд показывает RPS, ошибки и латентность
- [ ] Логи структурированные (JSON) с нужными полями
- [ ] Настроены правила алертов

---

*Следующий урок: Kubernetes — оркестрация контейнеров*
