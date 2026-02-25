# Урок 15 — Распределённые системы: Redis, Kafka, gRPC

## Зачем это нужно

До этого урока `todo-service` — монолит. Всё в одном процессе, всё синхронно, всё через HTTP. В реальных системах это ограничение: HTTP блокирует ожидая ответа, база данных — единственный узел отказа, каждый запрос делает одно и то же снова и снова. Redis убирает лишнюю нагрузку на БД. Kafka позволяет сервисам общаться асинхронно — не ждать друг друга. gRPC заменяет HTTP для внутреннего взаимодействия — быстрее, строже, с контрактами. Это инструменты которые превращают набор сервисов в систему.

---

## Часть 1 — Redis

### Что такое Redis

Redis — in-memory хранилище данных. Работает в оперативной памяти — поэтому в 10-100 раз быстрее PostgreSQL. Данные можно сохранять на диск (persistence), но основное использование — временные данные с коротким временем жизни.

**Когда использовать Redis:**
- Кэш — сохранить результат дорогого запроса к БД
- Сессии — хранить авторизационные токены
- Rate limiting — считать запросы от IP
- Pub/Sub — простая очередь сообщений
- Distributed lock — захватить "замок" на время операции

### Структуры данных Redis

```bash
# Запустить Redis для практики
docker run -d --name redis -p 6379:6379 redis:7-alpine
docker exec -it redis redis-cli

# String — базовая пара ключ/значение
SET user:1:name "Alice"
GET user:1:name                   # "Alice"
SET counter 0
INCR counter                      # 1
INCRBY counter 5                  # 6
EXPIRE user:1:name 3600           # истечёт через 1 час
TTL user:1:name                   # сколько секунд осталось
SETEX session:abc123 3600 "user_id:42"  # SET + EXPIRE за одну команду

# Hash — словарь (как Go map)
HSET user:1 name "Alice" age 30 email "alice@example.com"
HGET user:1 name                  # "Alice"
HGETALL user:1                    # все поля
HDEL user:1 email

# List — двусвязный список (очередь/стек)
LPUSH queue "task1"               # добавить в начало
RPUSH queue "task2" "task3"       # добавить в конец
LRANGE queue 0 -1                 # все элементы
LPOP queue                        # забрать из начала
RPOP queue                        # забрать из конца
BLPOP queue 30                    # блокирующий pop (ждать 30 секунд)

# Set — множество уникальных значений
SADD tags "go" "kubernetes" "docker"
SMEMBERS tags                     # все элементы
SISMEMBER tags "go"               # есть ли элемент
SCARD tags                        # количество элементов

# Sorted Set — множество с числовым рейтингом (скор)
ZADD leaderboard 100 "alice"
ZADD leaderboard 200 "bob"
ZADD leaderboard 150 "charlie"
ZRANGE leaderboard 0 -1 WITHSCORES  # от меньшего к большему
ZREVRANGE leaderboard 0 2           # топ-3
ZINCRBY leaderboard 50 "alice"      # добавить 50 к скору alice
```

### Redis в Go

```bash
go get github.com/redis/go-redis/v9
```

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

var rdb *redis.Client

func initRedis() {
    rdb = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
        PoolSize: 10,
    })

    ctx := context.Background()
    if err := rdb.Ping(ctx).Err(); err != nil {
        panic(fmt.Sprintf("redis ping failed: %v", err))
    }
}

// Кэш для задач
func getCachedTasks(ctx context.Context) ([]Task, error) {
    data, err := rdb.Get(ctx, "tasks:all").Bytes()
    if err == redis.Nil {
        return nil, nil   // кэш пустой
    }
    if err != nil {
        return nil, err
    }

    var tasks []Task
    if err := json.Unmarshal(data, &tasks); err != nil {
        return nil, err
    }
    return tasks, nil
}

func cacheTasks(ctx context.Context, tasks []Task) error {
    data, err := json.Marshal(tasks)
    if err != nil {
        return err
    }
    return rdb.SetEx(ctx, "tasks:all", data, 5*time.Minute).Err()
}

func invalidateTasksCache(ctx context.Context) error {
    return rdb.Del(ctx, "tasks:all").Err()
}

// Rate limiting — не более 10 запросов в минуту с IP
func checkRateLimit(ctx context.Context, ip string) (bool, error) {
    key := fmt.Sprintf("ratelimit:%s", ip)

    count, err := rdb.Incr(ctx, key).Result()
    if err != nil {
        return false, err
    }

    if count == 1 {
        // Первый запрос — установить TTL
        rdb.Expire(ctx, key, time.Minute)
    }

    return count <= 10, nil
}

// Distributed lock — только один процесс выполняет операцию
func withLock(ctx context.Context, key string, fn func() error) error {
    lockKey := fmt.Sprintf("lock:%s", key)
    lockValue := fmt.Sprintf("%d", time.Now().UnixNano())

    // Попытаться захватить лок
    ok, err := rdb.SetNX(ctx, lockKey, lockValue, 30*time.Second).Result()
    if err != nil {
        return fmt.Errorf("acquire lock: %w", err)
    }
    if !ok {
        return fmt.Errorf("lock is held by another process")
    }

    defer rdb.Del(ctx, lockKey) // освободить лок

    return fn()
}

// Pub/Sub — публикация событий
func publishTaskCompleted(ctx context.Context, taskID int) error {
    event := map[string]interface{}{
        "type":    "task.completed",
        "task_id": taskID,
        "time":    time.Now().Unix(),
    }
    data, _ := json.Marshal(event)
    return rdb.Publish(ctx, "task-events", data).Err()
}

// Подписка на события
func subscribeToTaskEvents(ctx context.Context) {
    sub := rdb.Subscribe(ctx, "task-events")
    defer sub.Close()

    ch := sub.Channel()
    for msg := range ch {
        fmt.Printf("Received event: %s\n", msg.Payload)
        // обработать событие
    }
}
```

### Cache-Aside паттерн

```go
// Паттерн: сначала смотри в кэш, потом в БД
func (h *Handler) listTasks(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // 1. Попробовать кэш
    tasks, err := getCachedTasks(ctx)
    if err != nil {
        slog.Error("cache get failed", "error", err)
        // Не падаем — идём в БД
    }

    if tasks != nil {
        writeJSON(w, http.StatusOK, tasks)
        return
    }

    // 2. Кэш пустой — идём в БД
    tasks, err = h.storage.List(ctx)
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, errResponse(err))
        return
    }

    // 3. Сохранить в кэш (fire and forget)
    go func() {
        if err := cacheTasks(context.Background(), tasks); err != nil {
            slog.Error("cache set failed", "error", err)
        }
    }()

    writeJSON(w, http.StatusOK, tasks)
}

// При изменении данных — инвалидировать кэш
func (h *Handler) createTask(w http.ResponseWriter, r *http.Request) {
    // ... создать задачу ...

    // Инвалидировать кэш
    go invalidateTasksCache(context.Background())

    writeJSON(w, http.StatusCreated, task)
}
```

---

## Часть 2 — Kafka

### Что такое Kafka и зачем

Представь: `todo-service` должен отправлять email когда задача выполнена, обновлять статистику, писать аудит лог. Если делать это синхронно в HTTP обработчике — запрос тормозит пока все системы не ответят. Если одна упала — всё падает.

Kafka решает это через асинхронность:
1. `todo-service` публикует событие в Kafka (быстро, не ждёт никого)
2. `email-service`, `stats-service`, `audit-service` — каждый читает события независимо
3. Если `email-service` упал — он догонит события когда поднимется

```
                    ┌─────────────┐
                    │   Kafka     │
                    │             │
todo-service ──────►│ topic:      │──────► email-service
(producer)          │ task-events │──────► stats-service
                    │             │──────► audit-service
                    └─────────────┘   (consumers)
```

### Ключевые концепции

- **Topic** — именованный поток событий (как таблица в БД)
- **Partition** — тема разбита на партиции для параллелизма
- **Offset** — позиция сообщения в партиции (число)
- **Consumer Group** — группа потребителей, каждое сообщение читается одним из группы
- **Producer** — публикует сообщения
- **Consumer** — читает сообщения

```
Topic: task-events
Partition 0: [msg0] [msg1] [msg4] [msg6]
Partition 1: [msg2] [msg3] [msg5] [msg7]
                                  ↑
                             current offset
                             consumer group "email-service"
```

### Запуск Kafka

```yaml
# docker-compose.yml — добавить Kafka
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9093:9093"
```

### Kafka в Go

```bash
go get github.com/segmentio/kafka-go
```

```go
package main

import (
    "context"
    "encoding/json"
    "log/slog"
    "time"

    "github.com/segmentio/kafka-go"
)

// Event — структура события
type TaskEvent struct {
    Type      string    `json:"type"`
    TaskID    int       `json:"task_id"`
    TaskTitle string    `json:"task_title"`
    UserID    int       `json:"user_id,omitempty"`
    Timestamp time.Time `json:"timestamp"`
}

// Producer — публикация событий
type EventPublisher struct {
    writer *kafka.Writer
}

func NewEventPublisher(brokers []string) *EventPublisher {
    return &EventPublisher{
        writer: &kafka.Writer{
            Addr:         kafka.TCP(brokers...),
            Topic:        "task-events",
            Balancer:     &kafka.LeastBytes{},
            RequiredAcks: kafka.RequireOne,
            Async:        false,    // синхронно — ждём подтверждения
        },
    }
}

func (p *EventPublisher) PublishTaskCompleted(ctx context.Context, task Task) error {
    event := TaskEvent{
        Type:      "task.completed",
        TaskID:    task.ID,
        TaskTitle: task.Title,
        Timestamp: time.Now(),
    }

    data, err := json.Marshal(event)
    if err != nil {
        return fmt.Errorf("marshal event: %w", err)
    }

    err = p.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(fmt.Sprintf("task-%d", task.ID)),
        Value: data,
        Headers: []kafka.Header{
            {Key: "event-type", Value: []byte("task.completed")},
            {Key: "service", Value: []byte("todo-service")},
        },
    })
    if err != nil {
        return fmt.Errorf("write message: %w", err)
    }

    slog.Info("published event", "type", event.Type, "task_id", event.TaskID)
    return nil
}

func (p *EventPublisher) Close() error {
    return p.writer.Close()
}

// Consumer — чтение событий
type EventConsumer struct {
    reader *kafka.Reader
}

func NewEventConsumer(brokers []string, groupID string) *EventConsumer {
    return &EventConsumer{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers:        brokers,
            Topic:          "task-events",
            GroupID:        groupID,    // consumer group
            MinBytes:       10e3,       // 10KB минимум для батча
            MaxBytes:       10e6,       // 10MB максимум
            CommitInterval: time.Second,
        }),
    }
}

func (c *EventConsumer) Start(ctx context.Context, handler func(TaskEvent) error) error {
    for {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil {
            if ctx.Err() != nil {
                return nil  // контекст отменён — нормальное завершение
            }
            slog.Error("fetch message failed", "error", err)
            time.Sleep(time.Second)
            continue
        }

        var event TaskEvent
        if err := json.Unmarshal(msg.Value, &event); err != nil {
            slog.Error("unmarshal event failed", "error", err, "offset", msg.Offset)
            // Пропустить невалидное сообщение
            c.reader.CommitMessages(ctx, msg)
            continue
        }

        // Обработать событие
        if err := handler(event); err != nil {
            slog.Error("handle event failed", "error", err, "event_type", event.Type)
            // Не коммитим offset — сообщение будет перечитано
            continue
        }

        // Подтвердить что сообщение обработано
        if err := c.reader.CommitMessages(ctx, msg); err != nil {
            slog.Error("commit failed", "error", err)
        }
    }
}

// Использование в notification-service
func main() {
    consumer := NewEventConsumer([]string{"kafka:9092"}, "notification-service")

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    err := consumer.Start(ctx, func(event TaskEvent) error {
        switch event.Type {
        case "task.completed":
            slog.Info("sending notification", "task_id", event.TaskID)
            // отправить email/push/slack
            return sendNotification(event)
        default:
            slog.Warn("unknown event type", "type", event.Type)
        }
        return nil
    })

    if err != nil {
        slog.Error("consumer error", "error", err)
    }
}
```

### Паттерны Kafka

```go
// Идемпотентность — обработка дублей
// Kafka гарантирует at-least-once (минимум один раз)
// Нужно защититься от повторной обработки

func handleEvent(ctx context.Context, event TaskEvent) error {
    // Проверить не обработали ли уже
    key := fmt.Sprintf("processed:%s:%d", event.Type, event.TaskID)
    ok, err := rdb.SetNX(ctx, key, "1", 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !ok {
        slog.Info("duplicate event, skipping", "key", key)
        return nil  // уже обработано
    }

    // Обработать
    return processEvent(event)
}

// Dead Letter Queue — куда слать невалидные сообщения
dlqWriter := &kafka.Writer{
    Addr:  kafka.TCP("kafka:9092"),
    Topic: "task-events-dlq",
}

if err := handler(event); err != nil {
    // Отправить в DLQ для ручного анализа
    dlqWriter.WriteMessages(ctx, kafka.Message{
        Value: msg.Value,
        Headers: []kafka.Header{
            {Key: "error", Value: []byte(err.Error())},
            {Key: "original-topic", Value: []byte("task-events")},
        },
    })
}
```

---

## Часть 3 — gRPC

### HTTP vs gRPC

```
HTTP/JSON (внешний API):          gRPC (внутренний API):
- Человекочитаемый JSON           - Бинарный Protocol Buffers
- Большой размер сообщений        - Маленький размер (3-10x меньше)
- Медленная сериализация          - Быстрая сериализация
- Нет строгого контракта          - Строгий контракт (.proto файл)
- Нет кодогенерации               - Автогенерация клиентов на любом языке
- HTTP/1.1                        - HTTP/2 (мультиплексирование)
```

gRPC хорош для взаимодействия между сервисами внутри кластера — быстрее, строже, меньше.

### Protocol Buffers — определение контракта

```bash
# Установить protoc компилятор
apt install protobuf-compiler

# Go плагины
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

Создай файл `proto/task.proto`:

```protobuf
syntax = "proto3";

package task;
option go_package = "github.com/username/todo-service/gen/task";

import "google/protobuf/timestamp.proto";

// Определение сервиса
service TaskService {
    rpc GetTask(GetTaskRequest) returns (GetTaskResponse);
    rpc ListTasks(ListTasksRequest) returns (ListTasksResponse);
    rpc CreateTask(CreateTaskRequest) returns (CreateTaskResponse);
    rpc CompleteTask(CompleteTaskRequest) returns (CompleteTaskResponse);

    // Server streaming — сервер отправляет поток обновлений
    rpc WatchTasks(WatchTasksRequest) returns (stream TaskEvent);
}

message Task {
    int64  id       = 1;
    string title    = 2;
    int32  priority = 3;
    bool   done     = 4;
    google.protobuf.Timestamp created_at = 5;
}

message GetTaskRequest  { int64 id = 1; }
message GetTaskResponse { Task task = 1; }

message ListTasksRequest {
    bool   done_only = 1;
    int32  limit     = 2;
    int32  offset    = 3;
}
message ListTasksResponse {
    repeated Task tasks = 1;
    int32 total         = 2;
}

message CreateTaskRequest {
    string title    = 1;
    int32  priority = 2;
}
message CreateTaskResponse { Task task = 1; }

message CompleteTaskRequest  { int64 id = 1; }
message CompleteTaskResponse { Task task = 1; }

message WatchTasksRequest  {}
message TaskEvent {
    string event_type = 1;
    Task   task       = 2;
}
```

```bash
# Сгенерировать Go код
protoc \
    --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    proto/task.proto
```

### gRPC сервер на Go

```go
package main

import (
    "context"
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"

    pb "github.com/username/todo-service/gen/task"
)

type TaskGRPCServer struct {
    pb.UnimplementedTaskServiceServer
    storage Storage
}

func (s *TaskGRPCServer) GetTask(ctx context.Context, req *pb.GetTaskRequest) (*pb.GetTaskResponse, error) {
    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "id must be positive")
    }

    task, err := s.storage.Get(ctx, int(req.Id))
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, status.Errorf(codes.NotFound, "task %d not found", req.Id)
        }
        return nil, status.Errorf(codes.Internal, "get task: %v", err)
    }

    return &pb.GetTaskResponse{
        Task: toProtoTask(task),
    }, nil
}

func (s *TaskGRPCServer) CreateTask(ctx context.Context, req *pb.CreateTaskRequest) (*pb.CreateTaskResponse, error) {
    if req.Title == "" {
        return nil, status.Error(codes.InvalidArgument, "title is required")
    }

    task, err := s.storage.Add(ctx, req.Title, int(req.Priority))
    if err != nil {
        return nil, status.Errorf(codes.Internal, "create task: %v", err)
    }

    return &pb.CreateTaskResponse{Task: toProtoTask(task)}, nil
}

// Server streaming — отправлять обновления в реальном времени
func (s *TaskGRPCServer) WatchTasks(req *pb.WatchTasksRequest, stream pb.TaskService_WatchTasksServer) error {
    // Подписаться на Redis pub/sub
    sub := rdb.Subscribe(stream.Context(), "task-events")
    defer sub.Close()

    ch := sub.Channel()
    for {
        select {
        case <-stream.Context().Done():
            return nil  // клиент отключился
        case msg := <-ch:
            var event TaskEvent
            json.Unmarshal([]byte(msg.Payload), &event)

            if err := stream.Send(&pb.TaskEvent{
                EventType: event.Type,
                Task:      toProtoTask(event.Task),
            }); err != nil {
                return err
            }
        }
    }
}

func toProtoTask(t Task) *pb.Task {
    return &pb.Task{
        Id:       int64(t.ID),
        Title:    t.Title,
        Priority: int32(t.Priority),
        Done:     t.Done,
    }
}

func startGRPCServer(storage Storage) error {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        return err
    }

    srv := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            loggingInterceptor,
            recoveryInterceptor,
        ),
    )

    pb.RegisterTaskServiceServer(srv, &TaskGRPCServer{storage: storage})

    slog.Info("gRPC server started", "addr", ":50051")
    return srv.Serve(lis)
}

// Interceptor — аналог HTTP middleware для gRPC
func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    slog.Info("grpc request",
        "method", info.FullMethod,
        "duration_ms", time.Since(start).Milliseconds(),
        "error", err,
    )
    return resp, err
}
```

### gRPC клиент

```go
package main

import (
    "context"
    "log"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "github.com/username/todo-service/gen/task"
)

func main() {
    // Подключиться к серверу
    conn, err := grpc.Dial("todo-service:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    client := pb.NewTaskServiceClient(conn)
    ctx := context.Background()

    // Создать задачу
    resp, err := client.CreateTask(ctx, &pb.CreateTaskRequest{
        Title:    "Learn gRPC",
        Priority: 1,
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Created: %+v\n", resp.Task)

    // Получить список
    list, err := client.ListTasks(ctx, &pb.ListTasksRequest{Limit: 10})
    if err != nil {
        log.Fatal(err)
    }
    for _, task := range list.Tasks {
        fmt.Printf("%d: %s (done: %v)\n", task.Id, task.Title, task.Done)
    }

    // Стриминг — получать обновления в реальном времени
    stream, err := client.WatchTasks(ctx, &pb.WatchTasksRequest{})
    if err != nil {
        log.Fatal(err)
    }

    for {
        event, err := stream.Recv()
        if err != nil {
            break
        }
        fmt.Printf("Event: %s - Task: %s\n", event.EventType, event.Task.Title)
    }
}
```

---

## Практические задания

### Задание 15.1 — Redis кэш

Добавь Redis в `todo-service`:

1. Добавь Redis в `docker-compose.yml`
2. Реализуй Cache-Aside для `GET /tasks`:
   - Первый запрос: идёт в PostgreSQL, результат сохраняется в Redis на 5 минут
   - Последующие: отдаются из Redis
   - При создании/удалении/выполнении задачи: инвалидировать кэш
3. Добавь метрику `cache_hits_total` и `cache_misses_total` в Prometheus
4. Проверь через Grafana что кэш работает — второй запрос должен быть значительно быстрее

```bash
# Проверка вручную
redis-cli KEYS "*"           # посмотреть ключи
redis-cli GET "tasks:all"    # посмотреть кэш
redis-cli TTL "tasks:all"    # сколько жить
```

---

### Задание 15.2 — Rate Limiting через Redis

Замени текущий rate limiter (если есть) на Redis-based:

```go
// Скользящее окно — не более 60 запросов в минуту с IP
func rateLimiter(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        allowed, err := checkRateLimit(r.Context(), ip)
        if err != nil {
            slog.Error("rate limit check failed", "error", err)
            // При ошибке Redis — пропускаем (fail open)
        } else if !allowed {
            writeJSON(w, http.StatusTooManyRequests, map[string]string{
                "error": "too many requests",
            })
            return
        }
        next(w, r)
    }
}
```

**Что нужно сделать:** Протестируй что при > 10 запросов в минуту с одного IP сервер возвращает 429. Убедись что счётчик сбрасывается через минуту.

---

### Задание 15.3 — Kafka события

Создай новый сервис `notification-service` (отдельная папка или репозиторий):

В `todo-service`:
- При выполнении задачи публиковать событие в topic `task-events`
- Событие: `{type: "task.completed", task_id: X, task_title: "...", timestamp: "..."}`

В `notification-service`:
- Читать события из topic `task-events`
- Логировать: `"Notification: Task 'Learn Go' has been completed!"`
- Реализовать идемпотентность через Redis

```bash
# Проверка
docker compose up -d
curl -X PATCH http://localhost:8080/tasks/1/done
# В логах notification-service должно появиться сообщение
docker compose logs notification-service
```

---

### Задание 15.4 — gRPC сервер

Добавь gRPC интерфейс к `todo-service` параллельно с HTTP:

1. Напиши `proto/task.proto` с методами `GetTask`, `ListTasks`, `CreateTask`, `CompleteTask`
2. Сгенерируй Go код через `protoc`
3. Реализуй gRPC сервер на порту `:50051`
4. Запускай оба сервера параллельно:

```go
func main() {
    // HTTP на :8080
    go func() {
        log.Fatal(http.ListenAndServe(":8080", httpMux))
    }()

    // gRPC на :50051
    log.Fatal(startGRPCServer(storage))
}
```

5. Напиши простой клиент `cmd/grpc-client/main.go` который:
   - Создаёт задачу через gRPC
   - Получает список задач
   - Выводит их в терминал

---

### Задание 15.5 — Финальный проект урока

Собери полную систему из трёх сервисов:

```
todo-service          — HTTP + gRPC API, пишет в Kafka
notification-service  — читает Kafka, логирует уведомления
stats-service         — читает Kafka, считает статистику в Redis
```

`stats-service` должен:
- Считать общее количество выполненных задач (`INCR stats:tasks:completed`)
- Считать выполненные задачи по дням (`INCR stats:tasks:2024-01-15`)
- Отдавать статистику на `GET /stats`

В `docker-compose.yml`:
```yaml
services:
  redis:        # кэш + rate limiting + stats
  kafka:        # очередь событий
  zookeeper:    # нужен для kafka
  postgres:     # основная БД
  todo-service:         # :8080 HTTP, :50051 gRPC
  notification-service: # читает kafka
  stats-service:        # читает kafka, :8081 HTTP
```

Добавь в Grafana дашборд:
- Панель "Задач выполнено сегодня" — из Redis через `/stats`
- Cache Hit Rate — из Prometheus метрик

Закоммить с тегом `v0.9.0`.

---

## Шпаргалка

| Инструмент | Когда использовать |
|-----------|-------------------|
| Redis String + TTL | Кэш запросов |
| Redis INCR | Счётчики, rate limiting |
| Redis SetNX | Distributed lock |
| Redis Pub/Sub | Простые события (не критичные) |
| Kafka | Надёжная доставка событий, replay |
| gRPC | Внутреннее взаимодействие сервисов |

| Redis команда | Что делает |
|---------------|-----------|
| `SET key val EX 60` | Установить с TTL 60 сек |
| `GET key` | Получить |
| `DEL key` | Удалить |
| `INCR key` | Атомарно +1 |
| `EXPIRE key 60` | Установить TTL |
| `SETNX key val` | Установить если не существует |

---

## Ресурсы для изучения

- **Redis документация:** `https://redis.io/docs/`
- **go-redis:** `https://redis.uptrace.dev/`
- **Kafka документация:** `https://kafka.apache.org/documentation/`
- **kafka-go:** `https://github.com/segmentio/kafka-go`
- **gRPC Go:** `https://grpc.io/docs/languages/go/`
- **Protocol Buffers:** `https://protobuf.dev/`
- **Книга:** "Designing Data-Intensive Applications" — глава про очереди сообщений

---

## Как понять что урок пройден

- [ ] Redis кэш работает — повторные запросы быстрее
- [ ] Rate limiting через Redis ограничивает запросы
- [ ] `todo-service` публикует события в Kafka при выполнении задачи
- [ ] `notification-service` читает события и логирует уведомления
- [ ] Идемпотентность: повторная доставка события не вызывает дубль
- [ ] gRPC сервер работает параллельно с HTTP
- [ ] Написан gRPC клиент который читает задачи
- [ ] Три сервиса поднимаются через `docker compose up -d`

---

*Следующий урок: Terraform и инфраструктура как код*
