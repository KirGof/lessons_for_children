# Урок 20 — Архитектурные паттерны: микросервисы, Event-Driven Architecture, CQRS

## Зачем это нужно

Писать код который работает — одно. Проектировать системы которые работают, масштабируются и остаются понятными через год — другое. Это уровень Senior и выше. Архитектурные паттерны — не академия, это решения реальных проблем которые тысячи команд решали до тебя. Знание паттернов позволяет: быстро понимать чужие системы, объяснять свои решения, выбирать правильный инструмент под задачу.

---

## Часть 1 — Монолит vs Микросервисы

### Когда монолит — правильный выбор

```
Монолит:
┌─────────────────────────────────┐
│           todo-service          │
│  HTTP ─► Auth ─► Tasks ─► DB   │
│          └─► Users ─► Cache     │
└─────────────────────────────────┘

Плюсы:
✅ Просто развернуть
✅ Нет сетевых задержек между компонентами
✅ Транзакции работают из коробки
✅ Легко отлаживать — один процесс
✅ Дёшево на старте

Минусы:
❌ Один упал — всё упало
❌ Масштабируется целиком, не по частям
❌ Большой код — сложно разбираться
❌ Деплой всего ради изменения одной функции
```

Правило: начинай с монолита. Переходи к микросервисам когда монолит реально мешает, а не "потому что так правильно".

### Когда микросервисы оправданы

```
Микросервисы:
┌──────────┐  ┌──────────┐  ┌──────────┐
│   auth   │  │  tasks   │  │   notif  │
│ service  │  │ service  │  │ service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┴──────────────┘
                    │
              ┌─────▼─────┐
              │  API GW   │
              └─────┬─────┘
                    │
                 Client

Плюсы:
✅ Независимый деплой каждого сервиса
✅ Масштабировать только нагруженные части
✅ Разные технологии для разных задач
✅ Команды работают независимо
✅ Изолированные отказы

Минусы:
❌ Сложная инфраструктура
❌ Сетевые вызовы — задержки и отказы
❌ Распределённые транзакции — боль
❌ Тяжело отлаживать
❌ Нужна зрелая DevOps культура
```

### Правила декомпозиции

Как правильно разбить монолит на сервисы:

```
1. По бизнес-возможностям (не по технологиям):
   ✅ auth-service, tasks-service, notification-service
   ❌ database-service, cache-service, http-service

2. Принцип "одной ответственности":
   Каждый сервис — один бизнес-процесс
   Изменение бизнес-логики → изменение одного сервиса

3. Размер команды (закон Конвея):
   Сервис = то что может поддерживать одна команда (2-8 человек)
   "Сервис не должен требовать больше двух пицц чтобы накормить команду"

4. Граница данных:
   Каждый сервис — своя база данных
   Другие сервисы не должны лезть напрямую в чужую БД
```

---

## Часть 2 — API Gateway и Service Mesh

### API Gateway

```
Client  ──► API Gateway ──► auth-service
                        ──► tasks-service
                        ──► notification-service

API Gateway берёт на себя:
- Аутентификацию (проверить JWT один раз)
- Rate limiting
- SSL termination
- Маршрутизацию
- Логирование всех входящих запросов
- Circuit breaker
```

```yaml
# Пример конфигурации Nginx как API Gateway
upstream auth_service {
    server auth-service:8080;
}
upstream tasks_service {
    server tasks-service:8080;
}

server {
    listen 80;

    # Все запросы проверяются через auth
    location /api/ {
        auth_request /auth/validate;
        error_page 401 = @error401;
    }

    location /auth/validate {
        internal;
        proxy_pass http://auth_service/validate;
        proxy_set_header Authorization $http_authorization;
    }

    location /api/tasks/ {
        proxy_pass http://tasks_service/;
    }

    location /api/auth/ {
        auth_request off;    # эти эндпоинты не требуют токена
        proxy_pass http://auth_service/;
    }
}
```

### Service Mesh — Istio/Linkerd

Service Mesh добавляет sidecar proxy к каждому Pod, который перехватывает весь трафик:

```
Pod A:                    Pod B:
┌─────────────────┐       ┌─────────────────┐
│  App Container  │       │  App Container  │
│                 │       │                 │
│  Sidecar Proxy  │──────►│  Sidecar Proxy  │
└─────────────────┘       └─────────────────┘
        │                         │
        └─────────────────────────┘
                Control Plane
             (Istio / Linkerd)

Что получаешь автоматически:
- mTLS между всеми сервисами
- Трейсинг запросов через все сервисы
- Circuit breaker
- Retry logic
- Traffic splitting (canary deployments)
- Метрики без изменения кода
```

```bash
# Установить Istio
curl -L https://istio.io/downloadIstio | sh -
istioctl install --set profile=demo -y

# Включить sidecar injection для namespace
kubectl label namespace production istio-injection=enabled

# Теперь каждый новый Pod получит sidecar автоматически

# Посмотреть трафик между сервисами
istioctl dashboard kiali
```

---

## Часть 3 — Event-Driven Architecture

### Паттерны обмена сообщениями

```
Синхронный (HTTP/gRPC):
A ──── запрос ────► B
A ◄─── ответ ───── B
A ждёт пока B ответит

Асинхронный (Event-Driven):
A ──── событие ───► Kafka ───► B
A не ждёт, продолжает работу
B обрабатывает в своём темпе
```

### Три топологии

**1. Event Notification** — "что-то произошло"

```go
// Просто уведомить другие сервисы о факте
type TaskCompletedEvent struct {
    TaskID    int       `json:"task_id"`
    UserID    int       `json:"user_id"`
    Timestamp time.Time `json:"timestamp"`
}

// Минимальный payload — получатель сам решает что делать
// Notification service пошлёт email
// Stats service обновит счётчик
// Audit service запишет в лог
```

**2. Event-Carried State Transfer** — "что-то произошло + все данные"

```go
// Полный снимок состояния в событии
type TaskUpdatedEvent struct {
    TaskID      int       `json:"task_id"`
    Title       string    `json:"title"`
    Priority    int       `json:"priority"`
    Done        bool      `json:"done"`
    UserID      int       `json:"user_id"`
    UpdatedAt   time.Time `json:"updated_at"`
    // + все поля задачи
}

// Получатель не должен идти в БД за данными — всё в событии
// Минус: большие события, сложнее эволюционировать схему
```

**3. Event Sourcing** — хранить события вместо состояния

```go
// Вместо "текущего состояния" хранить историю событий
type Event struct {
    ID        UUID
    Type      string    // "TaskCreated", "TaskCompleted", "TaskDeleted"
    Payload   []byte    // JSON данные события
    Timestamp time.Time
    Version   int       // порядковый номер
}

// Текущее состояние = воспроизведение всех событий
func RebuildTask(events []Event) Task {
    var task Task
    for _, e := range events {
        switch e.Type {
        case "TaskCreated":
            var p TaskCreatedPayload
            json.Unmarshal(e.Payload, &p)
            task = Task{ID: p.ID, Title: p.Title, Priority: p.Priority}
        case "TaskCompleted":
            task.Done = true
        case "TitleChanged":
            var p TitleChangedPayload
            json.Unmarshal(e.Payload, &p)
            task.Title = p.NewTitle
        }
    }
    return task
}

// Плюсы: полный audit log, можно воспроизвести любой момент прошлого
// Минусы: сложность, медленные запросы текущего состояния (нужны snapshot)
```

### Outbox Pattern — надёжная публикация событий

Проблема:
```go
// КАК НЕЛЬЗЯ — не атомарно
func completeTask(ctx context.Context, id int) error {
    db.Exec("UPDATE tasks SET done=true WHERE id=$1", id)
    // А ВДРУГ ТУТ УПАДЁМ?
    kafka.Publish("task.completed", event)    // событие не отправлено!
    return nil
}
```

Решение — Outbox Pattern:
```go
// Сохранить событие в той же транзакции что и изменение данных
func completeTask(ctx context.Context, id int) error {
    tx, _ := db.Begin(ctx)
    defer tx.Rollback(ctx)

    // 1. Обновить задачу
    tx.Exec(ctx, "UPDATE tasks SET done=true WHERE id=$1", id)

    // 2. Сохранить событие в таблицу outbox (в той же транзакции!)
    tx.Exec(ctx, `
        INSERT INTO outbox (event_type, payload, created_at)
        VALUES ($1, $2, NOW())`,
        "task.completed",
        mustMarshal(TaskCompletedEvent{TaskID: id}),
    )

    tx.Commit(ctx)
    // Если падаем здесь — оба действия отменятся (транзакция)
    return nil
}

// Отдельный процесс (Outbox Worker) читает из outbox и публикует в Kafka
func outboxWorker(ctx context.Context) {
    ticker := time.NewTicker(time.Second)
    for range ticker.C {
        rows, _ := db.Query(ctx,
            "SELECT id, event_type, payload FROM outbox WHERE published=false ORDER BY id LIMIT 100",
        )
        for rows.Next() {
            var id int
            var eventType string
            var payload []byte
            rows.Scan(&id, &eventType, &payload)

            kafka.Publish(eventType, payload)

            db.Exec(ctx, "UPDATE outbox SET published=true WHERE id=$1", id)
        }
    }
}
```

---

## Часть 4 — CQRS и Read Models

### Проблема которую решает CQRS

```
Обычный подход:
GET /tasks → SELECT * FROM tasks WHERE user_id=42
POST /tasks → INSERT INTO tasks ...

Проблема при масштабировании:
- Чтений в 100x больше чем записей
- Сложные запросы для чтения (JOIN, агрегации)
- Хочется масштабировать чтение независимо от записи
```

### CQRS — Command Query Responsibility Segregation

```
           Writes                          Reads
           (Commands)                     (Queries)

POST /tasks ──► Command Handler ──► DB (PostgreSQL)
                                         │
                                    Event Published
                                         │
                                         ▼
GET /tasks ──► Query Handler ──► Read Model (Redis/Elasticsearch)
```

```go
// Command — изменить состояние
type CreateTaskCommand struct {
    Title    string
    Priority int
    UserID   int
}

type CommandHandler struct {
    writeDB *pgxpool.Pool
    events  *EventPublisher
}

func (h *CommandHandler) CreateTask(ctx context.Context, cmd CreateTaskCommand) (int, error) {
    var taskID int
    err := h.writeDB.QueryRow(ctx,
        "INSERT INTO tasks(title, priority, user_id) VALUES($1,$2,$3) RETURNING id",
        cmd.Title, cmd.Priority, cmd.UserID,
    ).Scan(&taskID)
    if err != nil {
        return 0, err
    }

    // Публикуем событие для обновления Read Model
    h.events.Publish(ctx, TaskCreatedEvent{
        TaskID:   taskID,
        Title:    cmd.Title,
        Priority: cmd.Priority,
        UserID:   cmd.UserID,
    })

    return taskID, nil
}

// Query — только читать, никаких изменений
type QueryHandler struct {
    readDB *redis.Client    // Read Model в Redis — быстро
}

func (h *QueryHandler) ListTasks(ctx context.Context, userID int) ([]TaskView, error) {
    // Читаем из денормализованного Read Model
    data, err := h.readDB.Get(ctx, fmt.Sprintf("tasks:user:%d", userID)).Bytes()
    if err == redis.Nil {
        return []TaskView{}, nil
    }
    var tasks []TaskView
    json.Unmarshal(data, &tasks)
    return tasks, nil
}

// Projector — обновляет Read Model на основе событий
type TaskProjector struct {
    readDB *redis.Client
}

func (p *TaskProjector) OnTaskCreated(ctx context.Context, event TaskCreatedEvent) error {
    // Обновить Read Model
    key := fmt.Sprintf("tasks:user:%d", event.UserID)

    var tasks []TaskView
    data, err := p.readDB.Get(ctx, key).Bytes()
    if err == nil {
        json.Unmarshal(data, &tasks)
    }

    tasks = append([]TaskView{{
        ID:       event.TaskID,
        Title:    event.Title,
        Priority: event.Priority,
        Done:     false,
    }}, tasks...)

    newData, _ := json.Marshal(tasks)
    p.readDB.Set(ctx, key, newData, 0)
    return nil
}
```

---

## Часть 5 — Saga Pattern для распределённых транзакций

### Проблема

```
Создать заказ = несколько шагов в разных сервисах:
1. Order Service: создать заказ
2. Inventory Service: зарезервировать товар
3. Payment Service: списать деньги
4. Shipping Service: создать доставку

Если шаг 3 упал — нужно откатить шаги 1 и 2
Но нет единой транзакции охватывающей разные БД!
```

### Choreography Saga — через события

```go
// Каждый сервис публикует событие и подписывается на события других

// Order Service
func createOrder(ctx context.Context, order Order) error {
    db.Exec("INSERT INTO orders ...")
    kafka.Publish("order.created", OrderCreatedEvent{...})
    return nil
}

// Inventory Service подписывается на order.created
func onOrderCreated(ctx context.Context, event OrderCreatedEvent) error {
    if !hasStock(event.Items) {
        // Публикуем компенсирующее событие
        kafka.Publish("inventory.reservation.failed", ...)
        return nil
    }
    reserveStock(event.Items)
    kafka.Publish("inventory.reserved", InventoryReservedEvent{...})
    return nil
}

// Payment Service подписывается на inventory.reserved
func onInventoryReserved(ctx context.Context, event InventoryReservedEvent) error {
    if !charge(event.Amount) {
        kafka.Publish("payment.failed", ...)
        return nil
    }
    kafka.Publish("payment.processed", ...)
    return nil
}

// Order Service подписывается на payment.failed — компенсация
func onPaymentFailed(ctx context.Context, event PaymentFailedEvent) error {
    db.Exec("UPDATE orders SET status='cancelled' WHERE id=$1", event.OrderID)
    kafka.Publish("order.cancelled", ...)
    return nil
}
```

### Orchestration Saga — через оркестратор

```go
// Один сервис управляет всей сагой
type OrderSaga struct {
    orderService     OrderService
    inventoryService InventoryService
    paymentService   PaymentService
}

func (s *OrderSaga) Execute(ctx context.Context, order Order) error {
    // Шаг 1: создать заказ
    orderID, err := s.orderService.Create(ctx, order)
    if err != nil {
        return err
    }

    // Шаг 2: зарезервировать товар
    reservationID, err := s.inventoryService.Reserve(ctx, order.Items)
    if err != nil {
        // Компенсация: отменить заказ
        s.orderService.Cancel(ctx, orderID)
        return err
    }

    // Шаг 3: оплатить
    paymentID, err := s.paymentService.Charge(ctx, order.Amount)
    if err != nil {
        // Компенсация: отменить резервацию и заказ
        s.inventoryService.Release(ctx, reservationID)
        s.orderService.Cancel(ctx, orderID)
        return err
    }

    // Успех
    s.orderService.Confirm(ctx, orderID, paymentID)
    return nil
}
```

---

## Практические задания

### Задание 20.1 — Decompose todo-service

Спроектируй разбивку `todo-service` на микросервисы.

Создай документ `ARCHITECTURE.md`:

```markdown
# Architecture Decision Record: Microservices Split

## Context
todo-service вырос до 10k активных пользователей.
Команда разросла до 4 человек.
Notifications нужно масштабировать независимо.

## Decision
Разбить на три сервиса:

### auth-service
- Ответственность: аутентификация, управление пользователями
- API: POST /register, POST /login, POST /refresh, GET /validate
- БД: PostgreSQL (таблицы users, refresh_tokens)
- Команда: 1 человек

### tasks-service
- Ответственность: CRUD задач, теги, поиск
- API: все /tasks/* эндпоинты
- БД: PostgreSQL (таблицы tasks, tags, task_tags)
- Команда: 2 человека

### notification-service
- Ответственность: отправка уведомлений
- Читает: Kafka topic task-events
- Команда: 1 человек

## Граница данных
- auth-service НЕ знает про tasks
- tasks-service НЕ лезет в users таблицу auth-service
- Для получения информации о пользователе — tasks-service
  вызывает auth-service по gRPC

## Коммуникация
- Внешние запросы: API Gateway → HTTP
- Внутренние синхронные: gRPC
- Внутренние асинхронные: Kafka
```

---

### Задание 20.2 — Outbox Pattern

Реализуй Outbox Pattern в `todo-service`:

```sql
CREATE TABLE outbox (
    id          BIGSERIAL PRIMARY KEY,
    event_type  TEXT NOT NULL,
    payload     JSONB NOT NULL,
    published   BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ
);

CREATE INDEX idx_outbox_unpublished ON outbox(id) WHERE published = false;
```

```go
// 1. При выполнении задачи — в той же транзакции писать в outbox
// 2. Outbox Worker читает unpublished записи каждую секунду
// 3. Публикует в Kafka
// 4. Помечает как published
```

**Что нужно сделать:** Убедись что при остановке Kafka события не теряются — они остаются в outbox и будут отправлены когда Kafka вернётся.

---

### Задание 20.3 — Read Model с CQRS

Реализуй простой CQRS для `GET /tasks`:

1. **Write side:** при изменении задачи — обновлять Redis
2. **Read side:** `GET /tasks` читает из Redis, не из PostgreSQL

```go
// После create/complete/delete задачи:
func invalidateAndRebuild(ctx context.Context, userID int) {
    tasks, _ := db.Query(ctx, "SELECT ... FROM tasks WHERE user_id=$1", userID)
    data, _ := json.Marshal(tasks)
    rdb.Set(ctx, fmt.Sprintf("tasks:user:%d", userID), data, 5*time.Minute)
}
```

**Что нужно сделать:** Измерь время ответа `GET /tasks` с Redis vs без. Должно быть в 10+ раз быстрее.

---

### Задание 20.4 — Distributed Tracing

Добавь трейсинг для видимости между сервисами:

```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/exporters/jaeger
```

```go
// Инициализация Jaeger exporter
func initTracer() func() {
    exp, _ := jaeger.New(jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint("http://jaeger:14268/api/traces"),
    ))
    tp := tracesdk.NewTracerProvider(
        tracesdk.WithBatcher(exp),
        tracesdk.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String("todo-service"),
        )),
    )
    otel.SetTracerProvider(tp)
    return func() { tp.Shutdown(context.Background()) }
}

// В HTTP middleware
func tracingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        tracer := otel.Tracer("todo-service")
        ctx, span := tracer.Start(r.Context(), r.URL.Path)
        defer span.End()
        next(w, r.WithContext(ctx))
    }
}
```

```yaml
# docker-compose.yml
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"    # UI
      - "14268:14268"    # HTTP collector
```

**Что нужно сделать:** Открой Jaeger UI `http://localhost:16686`. Сделай несколько запросов к API. Найди самый медленный span.

---

### Задание 20.5 — Финальный проект урока

Создай `todo-system` — полная микросервисная архитектура:

```
todo-system/
├── auth-service/         # Go, PostgreSQL
├── tasks-service/        # Go, PostgreSQL, Kafka producer
├── notification-service/ # Go, Kafka consumer
├── api-gateway/          # Nginx конфиг
└── infra/
    ├── docker-compose.yml  # поднять всё локально
    └── k8s/                # деплой в minikube
```

**Требования:**
1. `auth-service` выдаёт JWT
2. `tasks-service` проверяет JWT через gRPC вызов к `auth-service`
3. При выполнении задачи — событие в Kafka через Outbox Pattern
4. `notification-service` читает из Kafka и логирует
5. Все сервисы: структурированные логи, Prometheus метрики, `/health`
6. Distributed tracing через Jaeger — видно путь запроса через все сервисы
7. `docker compose up -d` поднимает всё

**Документация в `ARCHITECTURE.md`:**
- Зачем каждый сервис
- Как сервисы общаются
- Где хранятся данные
- Как масштабировать
- Известные ограничения

Закоммить с тегом `v2.0.0` — это уже не монолит, это система.

---

## Шпаргалка

| Паттерн | Когда использовать |
|---------|-------------------|
| Монолит | Стартап, маленькая команда, неопределённые требования |
| Микросервисы | Большая команда, разные скорости масштабирования |
| API Gateway | Единая точка входа, аутентификация, rate limiting |
| Outbox Pattern | Надёжная публикация событий без потерь |
| CQRS | Разные требования к чтению и записи |
| Event Sourcing | Нужен полный audit log, воспроизведение состояния |
| Choreography Saga | Слабосвязанные сервисы, простые сценарии |
| Orchestration Saga | Сложные сценарии с откатами |
| Service Mesh | Много сервисов, нужен mTLS и observability |

---

## Ресурсы для изучения

- **"Building Microservices"** — Sam Newman — лучшая книга по теме
- **"Designing Data-Intensive Applications"** — Kleppmann — обязательно
- **Martin Fowler:** `https://martinfowler.com/articles/microservices.html`
- **Event Sourcing:** `https://martinfowler.com/eaaDev/EventSourcing.html`
- **CQRS:** `https://martinfowler.com/bliki/CQRS.html`
- **Saga Pattern:** `https://microservices.io/patterns/data/saga.html`
- **Microservices.io** — каталог паттернов: `https://microservices.io/patterns/`

---

## Как понять что урок пройден

- [ ] Могу объяснить когда монолит лучше микросервисов
- [ ] Реализовал Outbox Pattern — события не теряются при отказах
- [ ] CQRS: чтение из Redis, запись в PostgreSQL
- [ ] Распределённый трейсинг показывает путь запроса через сервисы
- [ ] Три сервиса работают вместе через Kafka и gRPC
- [ ] Написал ADR (Architecture Decision Record) объясняющий решения
- [ ] `docker compose up -d` поднимает всю систему

---

## Итог всего курса

Ты прошёл 20 уроков и построил систему которая:

- Работает в Kubernetes с автоскейлингом
- Имеет CI/CD с автоматическими тестами
- Собирает метрики, логи и трейсы
- Защищена JWT, RBAC, TLS
- Восстанавливается из бэкапа
- Масштабируется горизонтально
- Разбита на сервисы с правильными границами

Это не учебный проект. Это production-grade система.

Дальше — только реальные задачи.
