# Урок 6 — Go: интерфейсы, горутины и HTTP сервер

## Зачем это нужно

В предыдущем уроке ты научился писать программы. В этом уроке ты научишься писать **серверы**. Интерфейсы позволяют писать гибкий код который легко тестировать и менять. Горутины — это то за что Go любят: простой способ делать тысячи вещей одновременно. HTTP сервер — это основа любого бэкенда. После этого урока у тебя будет работающий REST API.

---

## Часть 1 — Интерфейсы

### Что такое интерфейс

Интерфейс в Go — это набор методов. Если тип реализует все методы интерфейса — он автоматически удовлетворяет интерфейсу. Никакого `implements` не нужно.

```go
// Определяем интерфейс
type Storage interface {
    Add(title string, priority int) (Task, error)
    Get(id int) (Task, error)
    List() []Task
    Delete(id int) error
    Complete(id int) error
}

// Тип реализует интерфейс если у него есть все нужные методы
type MemoryStorage struct {
    tasks  []Task
    nextID int
}

func (s *MemoryStorage) Add(title string, priority int) (Task, error) { ... }
func (s *MemoryStorage) Get(id int) (Task, error) { ... }
func (s *MemoryStorage) List() []Task { ... }
func (s *MemoryStorage) Delete(id int) error { ... }
func (s *MemoryStorage) Complete(id int) error { ... }

// MemoryStorage автоматически удовлетворяет интерфейсу Storage
// Никаких дополнительных объявлений не нужно
```

### Зачем это нужно

```go
// Функция принимает интерфейс — не конкретный тип
type Handler struct {
    storage Storage    // не *MemoryStorage, а интерфейс
}

// Теперь можно подменить реализацию
handler := Handler{storage: &MemoryStorage{}}           // в памяти
handler := Handler{storage: &PostgresStorage{db: db}}   // в базе данных
handler := Handler{storage: &MockStorage{}}             // в тестах

// Код Handler не меняется — он работает с любым Storage
```

### Встроенные интерфейсы

```go
// fmt.Stringer — если реализован, fmt.Println использует его автоматически
type Stringer interface {
    String() string
}

// error — встроенный интерфейс для ошибок
type error interface {
    Error() string
}

// io.Reader и io.Writer — основа всего I/O в Go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Один тип может реализовывать несколько интерфейсов
// Файл реализует и Reader, и Writer — он io.ReadWriter
```

### Пустой интерфейс и type assertion

```go
// any (или interface{}) — принимает любой тип
func printAnything(v any) {
    fmt.Println(v)
}

// Type assertion — получить конкретный тип обратно
var v any = "hello"

s, ok := v.(string)     // безопасно: ok = true, s = "hello"
if !ok {
    fmt.Println("не строка")
}

// Type switch
func describe(v any) {
    switch t := v.(type) {
    case int:
        fmt.Printf("int: %d\n", t)
    case string:
        fmt.Printf("string: %s\n", t)
    case bool:
        fmt.Printf("bool: %t\n", t)
    default:
        fmt.Printf("unknown type: %T\n", t)
    }
}
```

---

## Часть 2 — Горутины и параллелизм

### Горутины

Горутина — это лёгкий поток выполнения управляемый Go runtime. Создать горутину — написать `go` перед вызовом функции.

```go
func sayHello(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    go sayHello("Alice")    // запустить в горутине
    go sayHello("Bob")      // запустить в горутине
    sayHello("Charlie")     // запустить в главной горутине

    time.Sleep(time.Second) // подождать пока горутины завершатся
}
```

Горутины очень дешёвые — можно запускать тысячи и десятки тысяч. Стек горутины начинается с 2KB и растёт по необходимости.

### Каналы

Каналы — способ передавать данные между горутинами безопасно:

```go
// Создание канала
ch := make(chan int)         // небуферизованный
ch := make(chan int, 10)     // буферизованный (вместимость 10)

// Отправка и получение
ch <- 42            // отправить значение (блокирует если канал полный)
v := <-ch           // получить значение (блокирует если канал пустой)
v, ok := <-ch       // ok = false если канал закрыт

// Закрыть канал
close(ch)

// Перебрать все значения из канала
for v := range ch {
    fmt.Println(v)
    // выйдет из цикла когда канал закрыт и пуст
}
```

Пример — параллельная обработка:

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Worker %d обрабатывает задачу %d\n", id, job)
        time.Sleep(time.Second)    // имитация работы
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // Запустить 3 воркера
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Отправить 9 задач
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)

    // Собрать результаты
    for r := 1; r <= 9; r++ {
        fmt.Println(<-results)
    }
}
```

### sync.WaitGroup — ждать завершения горутин

```go
import "sync"

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)           // сказать: "добавляется одна горутина"
        go func(n int) {
            defer wg.Done() // сказать: "эта горутина завершилась"
            fmt.Println("Горутина", n)
        }(i)
    }

    wg.Wait()               // ждать пока все горутины вызовут Done()
    fmt.Println("Все завершились")
}
```

### sync.Mutex — защита общих данных

```go
import "sync"

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()             // захватить мьютекс
    defer c.mu.Unlock()     // освободить при выходе из функции
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

// Без мьютекса при параллельном доступе count будет неверным
counter := &SafeCounter{}
for i := 0; i < 1000; i++ {
    go counter.Increment()
}
```

### select — мультиплексирование каналов

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "один"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "два"
    }()

    // Ждём что придёт первым
    select {
    case msg1 := <-ch1:
        fmt.Println("Получено из ch1:", msg1)
    case msg2 := <-ch2:
        fmt.Println("Получено из ch2:", msg2)
    case <-time.After(3 * time.Second):
        fmt.Println("Таймаут!")
    }
}
```

---

## Часть 3 — HTTP сервер

### Базовый HTTP сервер

```go
package main

import (
    "fmt"
    "net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    http.HandleFunc("/", helloHandler)
    http.ListenAndServe(":8080", nil)
}
```

```bash
go run main.go
curl http://localhost:8080
```

### Request и Response

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Метод запроса
    fmt.Println(r.Method)           // GET, POST, PUT, DELETE...

    // URL и путь
    fmt.Println(r.URL.Path)         // /users/42
    fmt.Println(r.URL.Query())      // ?page=1&limit=10

    // Заголовки
    fmt.Println(r.Header.Get("Content-Type"))
    fmt.Println(r.Header.Get("Authorization"))

    // Тело запроса
    body, err := io.ReadAll(r.Body)
    defer r.Body.Close()

    // Отправить ответ
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)    // 200
    w.Write([]byte(`{"status":"ok"}`))
}
```

### REST API с JSON

```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
    "strings"
)

type Task struct {
    ID       int    `json:"id"`
    Title    string `json:"title"`
    Priority int    `json:"priority"`
    Done     bool   `json:"done"`
}

// Вспомогательные функции для JSON
func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(v)
}

func readJSON(r *http.Request, v any) error {
    return json.NewDecoder(r.Body).Decode(v)
}

// Хранилище (в памяти для простоты)
var tasks []Task
var nextID = 1

// GET /tasks — список всех задач
func listTasks(w http.ResponseWriter, r *http.Request) {
    writeJSON(w, http.StatusOK, tasks)
}

// POST /tasks — создать задачу
func createTask(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Title    string `json:"title"`
        Priority int    `json:"priority"`
    }

    if err := readJSON(r, &req); err != nil {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid json"})
        return
    }

    if req.Title == "" {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "title required"})
        return
    }

    task := Task{ID: nextID, Title: req.Title, Priority: req.Priority}
    nextID++
    tasks = append(tasks, task)

    writeJSON(w, http.StatusCreated, task)
}

// Роутер — разбираем путь вручную
func tasksRouter(w http.ResponseWriter, r *http.Request) {
    path := strings.TrimPrefix(r.URL.Path, "/tasks")

    // /tasks или /tasks/
    if path == "" || path == "/" {
        switch r.Method {
        case http.MethodGet:
            listTasks(w, r)
        case http.MethodPost:
            createTask(w, r)
        default:
            w.WriteHeader(http.StatusMethodNotAllowed)
        }
        return
    }

    // /tasks/42
    idStr := strings.TrimPrefix(path, "/")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid id"})
        return
    }

    switch r.Method {
    case http.MethodGet:
        getTask(w, r, id)
    case http.MethodDelete:
        deleteTask(w, r, id)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
    }
}

func main() {
    http.HandleFunc("/tasks", tasksRouter)
    http.HandleFunc("/tasks/", tasksRouter)

    fmt.Println("Сервер запущен на :8080")
    http.ListenAndServe(":8080", nil)
}
```

### Middleware

Middleware — функция которая оборачивает обработчик и добавляет поведение до или после него:

```go
// Тип для middleware
type Middleware func(http.HandlerFunc) http.HandlerFunc

// Логирование запросов
func logging(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next(w, r)
        fmt.Printf("%s %s %v\n", r.Method, r.URL.Path, time.Since(start))
    }
}

// Проверка авторизации
func auth(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token != "Bearer secret-token" {
            writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "unauthorized"})
            return
        }
        next(w, r)
    }
}

// Применение middleware
http.HandleFunc("/tasks", logging(auth(listTasks)))
```

### Context — отмена запросов

```go
func slowHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()    // контекст запроса, отменяется когда клиент отключается

    select {
    case <-time.After(5 * time.Second):
        // Работа завершена
        writeJSON(w, http.StatusOK, map[string]string{"status": "done"})
    case <-ctx.Done():
        // Клиент отключился
        fmt.Println("Клиент отключился:", ctx.Err())
    }
}

// Контекст с таймаутом
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.example.com", nil)
resp, err := http.DefaultClient.Do(req)
if err != nil {
    // err будет context.DeadlineExceeded если таймаут
}
```

---

## Практические задания

### Задание 6.1 — Интерфейсы

Создай `~/projects/todo-service/storage.go`:

```go
// Определи интерфейс Storage с методами:
// Add, Get, List, Delete, Complete, ListByPriority

// Реализуй MemoryStorage которая хранит задачи в []Task

// Реализуй FileStorage которая сохраняет задачи в JSON файл
// при каждом изменении

// Напиши функцию:
// func NewStorage(storageType string) (Storage, error)
// которая возвращает нужную реализацию по строке "memory" или "file"
```

**Что нужно сделать:** Убедись что обе реализации взаимозаменяемы — код который использует `Storage` интерфейс не должен знать с чем работает.

---

### Задание 6.2 — Горутины

Создай `~/projects/learn-go/concurrent.go`:

Реализуй функцию `fetchAll(urls []string) []Result` которая:
- Принимает список URL
- Делает HTTP GET запрос к каждому **параллельно**
- Собирает результаты (статус код, время ответа)
- Возвращает все результаты когда все запросы завершились

```go
type Result struct {
    URL        string
    StatusCode int
    Duration   time.Duration
    Error      string
}
```

Протестируй на 5-10 URL. Выведи результаты в таблицу. Сравни время выполнения с последовательным вариантом.

---

### Задание 6.3 — HTTP сервер

Преобразуй `todo-service` в HTTP API сервер `~/projects/todo-service/main.go`:

Реализуй следующие эндпоинты:

```
GET    /tasks              — список всех задач
POST   /tasks              — создать задачу
GET    /tasks/{id}         — получить задачу по ID
DELETE /tasks/{id}         — удалить задачу
PATCH  /tasks/{id}/done    — отметить выполненной
GET    /health             — {"status":"ok"} (для проверки что сервер жив)
```

Требования:
- Все ответы в JSON
- Правильные HTTP статус коды (200, 201, 204, 400, 404, 405)
- Логирование каждого запроса: метод, путь, время выполнения
- Использовать интерфейс `Storage` из задания 6.1
- Graceful shutdown — при получении SIGTERM сервер завершает текущие запросы и выходит

```go
// Graceful shutdown
srv := &http.Server{Addr: ":8080", Handler: mux}

go srv.ListenAndServe()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

---

### Задание 6.4 — Тестирование

Создай `~/projects/todo-service/storage_test.go`:

```go
package main

import "testing"

func TestMemoryStorage_Add(t *testing.T) {
    s := NewMemoryStorage()

    task, err := s.Add("Learn Go", 1)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if task.Title != "Learn Go" {
        t.Errorf("expected title 'Learn Go', got '%s'", task.Title)
    }
    if task.ID != 1 {
        t.Errorf("expected ID 1, got %d", task.ID)
    }
}

// Напиши тесты для:
// - Add с пустым title (должен вернуть ошибку)
// - Add с неверным priority (должен вернуть ошибку)
// - Get существующей задачи
// - Get несуществующей задачи (должен вернуть ErrNotFound)
// - Complete задачи
// - Delete задачи
// - List возвращает все задачи
```

```bash
go test ./...           # Запустить все тесты
go test -v ./...        # С подробным выводом
go test -cover ./...    # С покрытием кода
```

---

### Задание 6.5 — Финальный проект урока

Добавь в `todo-service` следующее:

1. **Middleware для логирования** — каждый запрос логируется в формате:
   ```
   2024-01-15 10:30:00 POST /tasks 201 Created 1.2ms
   ```

2. **Rate limiting** — не более 10 запросов в секунду с одного IP (используй `sync.Map` и горутину для сброса счётчика каждую секунду)

3. **Graceful shutdown** — при Ctrl+C сервер корректно завершает работу

4. **Конфигурация через переменные окружения:**
   ```
   PORT=8080
   STORAGE_TYPE=memory    # или file
   LOG_LEVEL=info
   ```

5. Запусти сервер и протестируй все эндпоинты через curl:
   ```bash
   # Создать задачу
   curl -X POST http://localhost:8080/tasks \
     -H "Content-Type: application/json" \
     -d '{"title":"Learn Go","priority":1}'

   # Получить список
   curl http://localhost:8080/tasks

   # Отметить выполненной
   curl -X PATCH http://localhost:8080/tasks/1/done

   # Удалить
   curl -X DELETE http://localhost:8080/tasks/1
   ```

6. Закоммить всё в GitHub с тегом `v0.1.0`:
   ```bash
   git tag v0.1.0
   git push origin v0.1.0
   ```

---

## Шпаргалка

| Концепция | Пример |
|-----------|--------|
| Интерфейс | `type Storage interface { Get(id int) (Task, error) }` |
| Горутина | `go func() { ... }()` |
| Канал | `ch := make(chan int); ch <- 1; v := <-ch` |
| WaitGroup | `wg.Add(1); go func() { defer wg.Done() }(); wg.Wait()` |
| Mutex | `mu.Lock(); defer mu.Unlock()` |
| HTTP хэндлер | `func(w http.ResponseWriter, r *http.Request)` |
| JSON encode | `json.NewEncoder(w).Encode(v)` |
| JSON decode | `json.NewDecoder(r.Body).Decode(&v)` |
| Тест | `func TestXxx(t *testing.T) { ... }` |

---

## Ресурсы для изучения

- **Горутины визуально:** `https://gobyexample.com` — примеры по всем темам
- **Конкурентность:** `https://go.dev/blog/pipelines` — официальный блог Go
- **HTTP:** `https://pkg.go.dev/net/http` — документация стандартной библиотеки
- **Тестирование:** `https://go.dev/doc/tutorial/add-a-test`

---

## Как понять что урок пройден

- [ ] Понимаю что такое интерфейс и зачем он нужен
- [ ] Могу запустить горутины и синхронизировать их через WaitGroup
- [ ] Понимаю как работают каналы и не путаю буферизованные и небуферизованные
- [ ] Написал HTTP сервер с несколькими эндпоинтами
- [ ] Все эндпоинты возвращают правильные статус коды и JSON
- [ ] Написал тесты и они проходят
- [ ] Сервер корректно завершает работу при Ctrl+C

---

*Следующий урок: PostgreSQL — реляционные базы данных*
