# Урок 21 — Тестирование: TDD, интеграционные тесты, contract testing

## Зачем это нужно

Без тестов ты узнаёшь о сломанном коде от пользователей. С тестами — за секунды при коммите. Но тесты бывают разные: юнит-тест проверяет одну функцию, интеграционный проверяет что сервис и база работают вместе, контрактный — что два сервиса понимают друг друга. Разница между "у нас есть тесты" и "у нас есть правильные тесты" огромная. Этот урок о том как писать тесты которые действительно защищают.

---

## Часть 1 — Пирамида тестирования

```
        /\
       /  \
      / E2E\        — Мало, дорого, медленно
     /──────\
    / Integ  \      — Умеренно
   /──────────\
  /  Unit Tests \   — Много, дёшево, быстро
 /______________\
```

**Unit тесты** — тестируют одну функцию/метод изолированно. Используют моки вместо реальных зависимостей. Работают за миллисекунды.

**Интеграционные тесты** — тестируют взаимодействие компонентов. Используют реальную БД, реальный Redis. Работают за секунды.

**E2E тесты** — тестируют весь путь пользователя. Реальный браузер, реальный бэкенд. Работают за минуты.

**Правило:** если что-то можно проверить юнит-тестом — не пиши интеграционный.

---

## Часть 2 — TDD: Test-Driven Development

### Цикл Red-Green-Refactor

```
1. RED   — написать тест который падает (функции ещё нет)
2. GREEN — написать минимальный код чтобы тест прошёл
3. REFACTOR — улучшить код не ломая тест
```

Не "сначала пишу код, потом тест". TDD — это дизайн через тесты.

### Пример: TDD для Storage

**Шаг 1 — RED: написать тест**

```go
// storage_test.go
func TestStorage_Add_Success(t *testing.T) {
    s := NewMemoryStorage()

    task, err := s.Add(context.Background(), "Learn TDD", 2)

    assert.NoError(t, err)
    assert.Equal(t, 1, task.ID)
    assert.Equal(t, "Learn TDD", task.Title)
    assert.Equal(t, 2, task.Priority)
    assert.False(t, task.Done)
}

func TestStorage_Add_EmptyTitle(t *testing.T) {
    s := NewMemoryStorage()

    _, err := s.Add(context.Background(), "", 1)

    assert.ErrorIs(t, err, ErrInvalidInput)
}

func TestStorage_Add_InvalidPriority(t *testing.T) {
    s := NewMemoryStorage()

    _, err := s.Add(context.Background(), "task", 5)

    assert.ErrorIs(t, err, ErrInvalidInput)
}
```

```bash
go test ./...
# FAIL: NewMemoryStorage undefined
# Тест красный — хорошо, идём дальше
```

**Шаг 2 — GREEN: минимальная реализация**

```go
// storage.go
type MemoryStorage struct {
    tasks  []Task
    nextID int
    mu     sync.Mutex
}

func NewMemoryStorage() *MemoryStorage {
    return &MemoryStorage{nextID: 1}
}

func (s *MemoryStorage) Add(ctx context.Context, title string, priority int) (Task, error) {
    if title == "" {
        return Task{}, ErrInvalidInput
    }
    if priority < 1 || priority > 3 {
        return Task{}, ErrInvalidInput
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    task := Task{
        ID:        s.nextID,
        Title:     title,
        Priority:  priority,
        Done:      false,
        CreatedAt: time.Now(),
    }
    s.tasks = append(s.tasks, task)
    s.nextID++
    return task, nil
}
```

```bash
go test ./...
# ok — тест зелёный
```

**Шаг 3 — REFACTOR: улучшить код**

```go
// Можно добавить валидатор, вынести константы
const (
    MinPriority = 1
    MaxPriority = 3
)

func validateTask(title string, priority int) error {
    if title == "" {
        return fmt.Errorf("%w: title cannot be empty", ErrInvalidInput)
    }
    if priority < MinPriority || priority > MaxPriority {
        return fmt.Errorf("%w: priority must be between %d and %d", ErrInvalidInput, MinPriority, MaxPriority)
    }
    return nil
}
```

```bash
go test ./...
# ok — рефакторинг не сломал тест
```

### Полезные библиотеки для тестов

```bash
go get github.com/stretchr/testify
go get github.com/stretchr/testify/mock
```

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// assert — тест продолжается после провала
assert.Equal(t, expected, actual)
assert.NoError(t, err)
assert.ErrorIs(t, err, ErrNotFound)
assert.True(t, condition)
assert.Len(t, slice, 3)
assert.Contains(t, slice, element)

// require — тест останавливается при провале (для критичных проверок)
require.NoError(t, err)    // если err != nil — дальше не идём
task, err := storage.Get(ctx, 1)
require.NoError(t, err)    // если упало — зачем проверять task?
assert.Equal(t, "Learn Go", task.Title)
```

---

## Часть 3 — Моки и заглушки

### Когда использовать моки

Мок нужен когда:
- Зависимость медленная (реальная БД)
- Зависимость недетерминированная (время, случайность)
- Зависимость имеет side effects (отправка email)
- Хочешь протестировать поведение при ошибке зависимости

### Ручной мок через интерфейс

```go
// Интерфейс — основа тестируемости
type Storage interface {
    Add(ctx context.Context, title string, priority int) (Task, error)
    Get(ctx context.Context, id int) (Task, error)
    List(ctx context.Context) ([]Task, error)
    Complete(ctx context.Context, id int) error
    Delete(ctx context.Context, id int) error
}

// Ручной мок для тестов
type MockStorage struct {
    tasks map[int]Task
    err   error    // ошибка которую нужно вернуть
}

func NewMockStorage() *MockStorage {
    return &MockStorage{tasks: make(map[int]Task)}
}

func (m *MockStorage) Add(ctx context.Context, title string, priority int) (Task, error) {
    if m.err != nil {
        return Task{}, m.err
    }
    id := len(m.tasks) + 1
    task := Task{ID: id, Title: title, Priority: priority}
    m.tasks[id] = task
    return task, nil
}

// ... остальные методы

// Использование в тесте
func TestHandler_CreateTask_StorageError(t *testing.T) {
    mock := NewMockStorage()
    mock.err = errors.New("database is down")

    handler := NewHandler(mock)

    req := httptest.NewRequest("POST", "/tasks", strings.NewReader(`{"title":"test","priority":1}`))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    handler.createTask(w, req)

    assert.Equal(t, http.StatusInternalServerError, w.Code)
}
```

### testify/mock — автоматические моки

```go
// Генерируемый мок (или написать вручную)
type MockStorage struct {
    mock.Mock
}

func (m *MockStorage) Add(ctx context.Context, title string, priority int) (Task, error) {
    args := m.Called(ctx, title, priority)
    return args.Get(0).(Task), args.Error(1)
}

func (m *MockStorage) Get(ctx context.Context, id int) (Task, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(Task), args.Error(1)
}

// Тест с ожиданиями
func TestHandler_GetTask_Found(t *testing.T) {
    mockStorage := new(MockStorage)
    expectedTask := Task{ID: 1, Title: "Learn Go", Priority: 1}

    // Ожидаем: Get будет вызван с id=1, вернёт task
    mockStorage.On("Get", mock.Anything, 1).Return(expectedTask, nil)

    handler := NewHandler(mockStorage)
    req := httptest.NewRequest("GET", "/tasks/1", nil)
    w := httptest.NewRecorder()

    handler.getTask(w, req)

    assert.Equal(t, http.StatusOK, w.Code)
    mockStorage.AssertExpectations(t)    // проверить что все ожидания выполнились
}

func TestHandler_GetTask_NotFound(t *testing.T) {
    mockStorage := new(MockStorage)
    mockStorage.On("Get", mock.Anything, 999).Return(Task{}, ErrNotFound)

    handler := NewHandler(mockStorage)
    req := httptest.NewRequest("GET", "/tasks/999", nil)
    w := httptest.NewRecorder()

    handler.getTask(w, req)

    assert.Equal(t, http.StatusNotFound, w.Code)
}
```

### mockery — автогенерация моков

```bash
go install github.com/vektra/mockery/v2@latest
mockery --name=Storage --dir=. --output=mocks --outpkg=mocks
```

Генерирует полный мок автоматически из интерфейса.

---

## Часть 4 — Интеграционные тесты

### testcontainers-go — реальная БД в тестах

```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

```go
// storage_integration_test.go
//go:build integration

package main

import (
    "context"
    "testing"

    "github.com/stretchr/testify/require"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

func setupTestDB(t *testing.T) (*pgxpool.Pool, func()) {
    ctx := context.Background()

    // Запустить PostgreSQL контейнер
    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("testuser"),
        postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections"),
        ),
    )
    require.NoError(t, err)

    // Получить строку подключения
    connStr, err := container.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    // Подключиться
    pool, err := pgxpool.New(ctx, connStr)
    require.NoError(t, err)

    // Применить миграции
    _, err = pool.Exec(ctx, `
        CREATE TABLE tasks (
            id         BIGSERIAL PRIMARY KEY,
            title      TEXT NOT NULL,
            priority   INTEGER NOT NULL DEFAULT 1,
            done       BOOLEAN NOT NULL DEFAULT false,
            created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
        )
    `)
    require.NoError(t, err)

    // Вернуть cleanup функцию
    cleanup := func() {
        pool.Close()
        container.Terminate(ctx)
    }

    return pool, cleanup
}

func TestPostgresStorage_AddAndGet(t *testing.T) {
    pool, cleanup := setupTestDB(t)
    defer cleanup()

    storage := NewPostgresStorage(pool)
    ctx := context.Background()

    // Создать задачу
    task, err := storage.Add(ctx, "Learn Integration Testing", 2)
    require.NoError(t, err)
    require.NotZero(t, task.ID)

    // Получить задачу
    found, err := storage.Get(ctx, task.ID)
    require.NoError(t, err)
    assert.Equal(t, task.ID, found.ID)
    assert.Equal(t, "Learn Integration Testing", found.Title)
    assert.Equal(t, 2, found.Priority)
    assert.False(t, found.Done)

    // Выполнить задачу
    err = storage.Complete(ctx, task.ID)
    require.NoError(t, err)

    // Проверить что выполнена
    completed, err := storage.Get(ctx, task.ID)
    require.NoError(t, err)
    assert.True(t, completed.Done)
}

func TestPostgresStorage_List_Empty(t *testing.T) {
    pool, cleanup := setupTestDB(t)
    defer cleanup()

    storage := NewPostgresStorage(pool)

    tasks, err := storage.List(context.Background())
    require.NoError(t, err)
    assert.Empty(t, tasks)
}

func TestPostgresStorage_Delete_NotFound(t *testing.T) {
    pool, cleanup := setupTestDB(t)
    defer cleanup()

    storage := NewPostgresStorage(pool)

    err := storage.Delete(context.Background(), 9999)
    assert.ErrorIs(t, err, ErrNotFound)
}
```

```bash
# Запустить только интеграционные тесты
go test -tags=integration ./...

# Запустить все тесты
go test -tags=integration -v ./...

# В CI — отдельный job для интеграционных
```

### HTTP тестирование с httptest

```go
// handler_test.go — тест HTTP хэндлеров
func TestHandler_ListTasks_Empty(t *testing.T) {
    mockStorage := new(MockStorage)
    mockStorage.On("List", mock.Anything).Return([]Task{}, nil)

    handler := NewHandler(mockStorage)

    // Создать тестовый HTTP запрос
    req := httptest.NewRequest(http.MethodGet, "/tasks", nil)
    req = addAuthHeader(req, createTestToken(t, 1))    // добавить JWT
    w := httptest.NewRecorder()

    handler.listTasks(w, req)

    assert.Equal(t, http.StatusOK, w.Code)
    assert.Equal(t, "application/json", w.Header().Get("Content-Type"))

    var tasks []Task
    err := json.NewDecoder(w.Body).Decode(&tasks)
    require.NoError(t, err)
    assert.Empty(t, tasks)

    mockStorage.AssertExpectations(t)
}

func TestHandler_CreateTask_InvalidJSON(t *testing.T) {
    handler := NewHandler(new(MockStorage))

    req := httptest.NewRequest(http.MethodPost, "/tasks",
        strings.NewReader("not json"))
    req = addAuthHeader(req, createTestToken(t, 1))
    w := httptest.NewRecorder()

    handler.createTask(w, req)

    assert.Equal(t, http.StatusBadRequest, w.Code)
}

// Таблично-управляемые тесты — для однотипных случаев
func TestHandler_CreateTask(t *testing.T) {
    tests := []struct {
        name           string
        body           string
        mockSetup      func(*MockStorage)
        expectedStatus int
    }{
        {
            name: "success",
            body: `{"title":"Learn Go","priority":1}`,
            mockSetup: func(m *MockStorage) {
                m.On("Add", mock.Anything, "Learn Go", 1).
                    Return(Task{ID: 1, Title: "Learn Go", Priority: 1}, nil)
            },
            expectedStatus: http.StatusCreated,
        },
        {
            name:           "empty title",
            body:           `{"title":"","priority":1}`,
            mockSetup:      func(m *MockStorage) {},
            expectedStatus: http.StatusBadRequest,
        },
        {
            name:           "invalid priority",
            body:           `{"title":"task","priority":5}`,
            mockSetup:      func(m *MockStorage) {},
            expectedStatus: http.StatusBadRequest,
        },
        {
            name: "storage error",
            body: `{"title":"task","priority":1}`,
            mockSetup: func(m *MockStorage) {
                m.On("Add", mock.Anything, "task", 1).
                    Return(Task{}, errors.New("db down"))
            },
            expectedStatus: http.StatusInternalServerError,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockStorage := new(MockStorage)
            tt.mockSetup(mockStorage)

            handler := NewHandler(mockStorage)
            req := httptest.NewRequest(http.MethodPost, "/tasks",
                strings.NewReader(tt.body))
            req.Header.Set("Content-Type", "application/json")
            req = addAuthHeader(req, createTestToken(t, 1))
            w := httptest.NewRecorder()

            handler.createTask(w, req)

            assert.Equal(t, tt.expectedStatus, w.Code)
            mockStorage.AssertExpectations(t)
        })
    }
}
```

---

## Часть 5 — Contract Testing

### Проблема интеграции микросервисов

```
tasks-service ──HTTP──► notification-service

Тест tasks-service проходит.
Тест notification-service проходит.
Но они сломаны вместе — tasks-service изменил формат события.
```

Contract Testing решает это: каждая сторона описывает контракт (что ожидает), и оба тестируются против этого контракта.

### Pact — Consumer-Driven Contract Testing

```bash
go get github.com/pact-foundation/pact-go/v2
```

**Consumer (tasks-service) — описывает что ожидает от API notification-service:**

```go
// tasks_service_pact_test.go
func TestTasksService_NotifiesOnComplete(t *testing.T) {
    // Pact создаёт mock сервер
    mockProvider, err := consumer.NewV2Pact(consumer.MockHTTPProviderConfig{
        Consumer: "tasks-service",
        Provider: "notification-service",
    })
    require.NoError(t, err)

    // Описать ожидаемое взаимодействие
    mockProvider.
        AddInteraction().
        Given("notification service is available").
        UponReceiving("a task completed notification").
        WithRequest(dsl.Request{
            Method: "POST",
            Path:   dsl.String("/notifications"),
            Headers: dsl.MapMatcher{
                "Content-Type": dsl.String("application/json"),
            },
            Body: dsl.Like(map[string]interface{}{
                "event_type": "task.completed",
                "task_id":    dsl.Like(1),
                "task_title": dsl.Like("Learn Go"),
            }),
        }).
        WillRespondWith(dsl.Response{
            Status: 200,
            Body:   dsl.Like(map[string]interface{}{"status": "sent"}),
        })

    // Запустить тест против mock сервера
    err = mockProvider.ExecuteTest(t, func(config consumer.MockServerConfig) error {
        // tasks-service вызывает notification-service
        client := NewNotificationClient(fmt.Sprintf("http://%s:%d", config.Host, config.Port))
        err := client.NotifyTaskCompleted(context.Background(), Task{ID: 1, Title: "Learn Go"})
        return err
    })
    require.NoError(t, err)

    // Сохранить контракт в файл (pact file)
    // Этот файл потом проверит notification-service
}
```

**Provider (notification-service) — проверяет что соответствует контракту:**

```go
// notification_service_pact_test.go
func TestNotificationService_SatisfiesContract(t *testing.T) {
    // Запустить реальный notification-service
    server := startNotificationService()
    defer server.Close()

    // Загрузить контракт из pact файла
    verifier := provider.NewVerifier()
    err := verifier.VerifyProvider(t, provider.VerifyRequest{
        ProviderBaseURL: server.URL,
        PactFiles:       []string{"./pacts/tasks-service-notification-service.json"},
        StateHandlers: provider.StateHandlers{
            "notification service is available": func(setup bool, s provider.ProviderState) (provider.ProviderStateResponse, error) {
                // Настроить состояние для теста
                return provider.ProviderStateResponse{}, nil
            },
        },
    })
    require.NoError(t, err)
}
```

---

## Практические задания

### Задание 21.1 — TDD для новой фичи

Реализуй через TDD метод `BulkComplete(ctx, ids []int) ([]Task, error)`:

1. Сначала напиши тесты (они должны падать):
   - Успешное выполнение нескольких задач
   - Пустой список IDs — вернуть ошибку
   - Один из IDs не существует — вернуть ErrNotFound, ничего не изменять
   - Дубликаты в списке — обработать корректно

2. Только потом напиши реализацию

3. Убедись что все тесты зелёные

4. Сделай рефакторинг не ломая тесты

```bash
# Начать с
go test -run TestStorage_BulkComplete ./...
# FAIL: method undefined — правильно!
```

---

### Задание 21.2 — Таблично-управляемые тесты

Перепиши тесты хэндлеров в таблично-управляемый формат:

```go
func TestHandler_CompleteTask(t *testing.T) {
    tests := []struct {
        name           string
        taskID         string
        userID         int
        mockSetup      func(*MockStorage)
        expectedStatus int
        expectedBody   string
    }{
        // заполни таблицу: success, not found, forbidden (чужая задача), invalid id
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // ...
        })
    }
}
```

Покрытие должно быть > 80% для пакета `handler`:
```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out    # визуально
go tool cover -func=coverage.out    # в тексте
```

---

### Задание 21.3 — Интеграционные тесты с testcontainers

Напиши интеграционные тесты для `PostgresStorage`:

Все тесты в `storage_integration_test.go` с тегом `//go:build integration`:

1. `TestPostgresStorage_Add` — создать задачу, проверить ID, поля
2. `TestPostgresStorage_Get_NotFound` — ошибка ErrNotFound
3. `TestPostgresStorage_List` — 0 задач, 1 задача, 10 задач
4. `TestPostgresStorage_Complete` — задача помечается выполненной
5. `TestPostgresStorage_Delete` — задача удаляется из БД
6. `TestPostgresStorage_Concurrent` — 10 горутин одновременно добавляют задачи

```bash
go test -tags=integration -v ./...
```

**Что нужно сделать:** Убедись что тесты изолированы — каждый тест запускается на чистой БД (использовать `t.Cleanup` или создавать контейнер на каждый тест).

---

### Задание 21.4 — HTTP интеграционный тест

Напиши полный HTTP интеграционный тест:

```go
//go:build integration

func TestAPI_TaskLifecycle(t *testing.T) {
    // 1. Запустить полноценный сервер с реальной БД
    pool, cleanup := setupTestDB(t)
    defer cleanup()

    server := httptest.NewServer(NewRouter(NewPostgresStorage(pool)))
    defer server.Close()

    client := server.Client()
    baseURL := server.URL

    // 2. Зарегистрироваться
    // 3. Войти, получить токен
    // 4. Создать задачу
    // 5. Получить список — должна быть одна задача
    // 6. Выполнить задачу
    // 7. Получить список — задача помечена выполненной
    // 8. Удалить задачу
    // 9. Получить список — пусто
}
```

---

### Задание 21.5 — Финальный проект урока

Полное тестовое покрытие `todo-service`:

**Юнит-тесты** (без БД, без HTTP):
- `storage_unit_test.go` — MemoryStorage все методы
- `handler_unit_test.go` — все HTTP хэндлеры с mock storage
- Покрытие: > 80%

**Интеграционные тесты** (с реальной БД через testcontainers):
- `storage_integration_test.go` — PostgresStorage все методы
- `api_integration_test.go` — полный HTTP lifecycle

**В CI (`.github/workflows/ci-cd.yml`):**
```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - run: go test ./...    # без тега — только юнит

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - run: go test -tags=integration ./...    # testcontainers сами поднимут PostgreSQL
```

**Makefile:**
```makefile
test:           go test ./...
test-integration: go test -tags=integration -v ./...
test-cover:     go test -coverprofile=coverage.out ./... && go tool cover -func=coverage.out
test-all:       make test && make test-integration
```

Закоммить с тегом `v2.1.0`.

---

## Шпаргалка

| Инструмент | Назначение |
|-----------|-----------|
| `go test ./...` | Запустить все тесты |
| `go test -run TestName` | Запустить конкретный тест |
| `go test -v` | Подробный вывод |
| `go test -race` | Детектор гонок данных |
| `go test -cover` | Покрытие кода |
| `testify/assert` | Удобные проверки |
| `testify/mock` | Моки с ожиданиями |
| `testcontainers-go` | Реальная БД в тестах |
| `httptest.NewRecorder()` | Фиктивный HTTP ответ |
| `httptest.NewRequest()` | Фиктивный HTTP запрос |

---

## Ресурсы для изучения

- **Go Testing:** `https://pkg.go.dev/testing`
- **testify:** `https://github.com/stretchr/testify`
- **testcontainers-go:** `https://golang.testcontainers.org/`
- **mockery:** `https://github.com/vektra/mockery`
- **Pact:** `https://docs.pact.io/`
- **Статья:** "The Practical Test Pyramid" — Martin Fowler
- **Книга:** "Test-Driven Development" — Kent Beck

---

## Как понять что урок пройден

- [ ] Написал тесты ПЕРЕД кодом (TDD) хотя бы для одной фичи
- [ ] Таблично-управляемые тесты для хэндлеров
- [ ] Покрытие кода > 80%
- [ ] Интеграционные тесты работают через testcontainers (без docker compose)
- [ ] Моки изолируют тесты от реальных зависимостей
- [ ] Юнит и интеграционные тесты разделены по build тегам
- [ ] CI запускает юнит-тесты быстро, интеграционные — в отдельном job

---

*Следующий урок: Облако — деплой в Yandex Cloud / AWS с Terraform*
