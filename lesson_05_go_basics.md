# Урок 5 — Go: основы языка

## Зачем именно Go

Go создан в Google в 2009 году с одной целью: писать надёжные серверные программы которые легко читать, быстро компилировать и просто поддерживать. Go намеренно простой — в нём мало концепций, но каждая сделана хорошо. Именно поэтому его выбирают для DevOps инструментов (Docker, Kubernetes, Terraform — всё написано на Go), микросервисов и высоконагруженных систем.

---

## Часть 1 — Установка и первая программа

### Установка Go

```bash
# Скачать и установить Go
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz

# Добавить в PATH (добавь в ~/.bashrc)
export PATH=$PATH:/usr/local/go/bin

# Применить
source ~/.bashrc

# Проверить
go version
```

### Первая программа

```bash
mkdir ~/projects/learn-go
cd ~/projects/learn-go
```

Создай файл `main.go`:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

```bash
go run main.go          # Запустить без компиляции
go build -o hello .     # Скомпилировать в бинарник
./hello                 # Запустить бинарник
```

### Структура Go проекта

```bash
# Инициализировать модуль (делается один раз для проекта)
go mod init github.com/username/todo-service

# Это создаёт go.mod — файл с зависимостями проекта
```

Файл `go.mod`:
```
module github.com/username/todo-service

go 1.22
```

---

## Часть 2 — Типы данных и переменные

### Объявление переменных

```go
// Полная форма
var name string = "Alice"
var age int = 25
var active bool = true

// Короткая форма (только внутри функций)
name := "Alice"
age := 25
active := true

// Несколько переменных
x, y := 10, 20
a, b := b, a        // Поменять местами

// Константы
const Pi = 3.14159
const MaxRetries = 3
```

### Базовые типы

```go
// Целые числа
var i int     = 42       // платформозависимый (32 или 64 бит)
var i8 int8   = 127      // -128 до 127
var i32 int32 = 2147483647
var i64 int64 = 9223372036854775807
var u uint    = 42       // беззнаковый

// Числа с плавающей точкой
var f32 float32 = 3.14
var f64 float64 = 3.141592653589793

// Строки
var s string = "Hello"
s2 := `Многострочная
строка`

// Булев тип
var b bool = true

// Нулевые значения (zero values) — Go инициализирует всё автоматически
var x int     // 0
var s string  // ""
var b bool    // false
var p *int    // nil
```

### Преобразование типов

```go
// Go НЕ делает неявное преобразование типов
var i int = 42
var f float64 = float64(i)    // нужно явно
var u uint = uint(f)

// Строка ↔ число
import "strconv"

n, err := strconv.Atoi("42")        // строка → int
s := strconv.Itoa(42)               // int → строка
f, err := strconv.ParseFloat("3.14", 64)
```

---

## Часть 3 — Управляющие конструкции

### if/else

```go
age := 18

if age >= 18 {
    fmt.Println("Совершеннолетний")
} else if age >= 14 {
    fmt.Println("Подросток")
} else {
    fmt.Println("Ребёнок")
}

// if с инициализацией (часто используется для ошибок)
if err := doSomething(); err != nil {
    fmt.Println("Ошибка:", err)
    return
}
```

### for — единственный цикл в Go

```go
// Классический for
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// Аналог while
n := 0
for n < 10 {
    n++
}

// Бесконечный цикл
for {
    // делай что-то
    if условие {
        break
    }
}

// Перебор слайса
fruits := []string{"apple", "banana", "cherry"}
for i, fruit := range fruits {
    fmt.Printf("%d: %s\n", i, fruit)
}

// Перебор мапы
ages := map[string]int{"Alice": 30, "Bob": 25}
for name, age := range ages {
    fmt.Printf("%s: %d\n", name, age)
}

// Если индекс не нужен
for _, fruit := range fruits {
    fmt.Println(fruit)
}
```

### switch

```go
day := "Monday"

switch day {
case "Monday", "Tuesday", "Wednesday", "Thursday", "Friday":
    fmt.Println("Рабочий день")
case "Saturday", "Sunday":
    fmt.Println("Выходной")
default:
    fmt.Println("Неизвестный день")
}

// Switch без выражения — аналог if/else
x := 42
switch {
case x < 0:
    fmt.Println("Отрицательное")
case x == 0:
    fmt.Println("Ноль")
default:
    fmt.Println("Положительное")
}
```

---

## Часть 4 — Структуры данных

### Массивы и слайсы

```go
// Массив — фиксированная длина
var arr [5]int = [5]int{1, 2, 3, 4, 5}
arr2 := [3]string{"a", "b", "c"}

// Слайс — динамический массив (используется почти всегда)
s := []int{1, 2, 3, 4, 5}
s = append(s, 6)                    // Добавить элемент
s = append(s, 7, 8, 9)             // Добавить несколько

fmt.Println(len(s))                 // Длина
fmt.Println(cap(s))                 // Ёмкость (внутренний размер)

// Срезы (slicing)
s[1:3]    // Элементы с индекса 1 по 2 (не включая 3)
s[:3]     // Первые 3 элемента
s[2:]     // С третьего до конца

// Создать слайс с нужной длиной
zeros := make([]int, 5)             // [0, 0, 0, 0, 0]
buf := make([]byte, 0, 1024)        // длина 0, ёмкость 1024
```

### Мапы (словари)

```go
// Создание
ages := map[string]int{
    "Alice": 30,
    "Bob":   25,
}

// Или через make
ages := make(map[string]int)

// Операции
ages["Charlie"] = 35            // Добавить/обновить
delete(ages, "Bob")             // Удалить
fmt.Println(ages["Alice"])      // Получить: 30

// Проверить существование ключа
age, ok := ages["Alice"]
if ok {
    fmt.Println("Возраст:", age)
} else {
    fmt.Println("Не найден")
}

// Перебор
for name, age := range ages {
    fmt.Printf("%s: %d\n", name, age)
}
```

### Структуры (structs)

```go
// Определение типа
type Task struct {
    ID          int
    Title       string
    Description string
    Done        bool
    CreatedAt   time.Time
}

// Создание
task := Task{
    ID:    1,
    Title: "Learn Go",
    Done:  false,
}

// Или так
task2 := Task{ID: 2, Title: "Write tests"}

// Доступ к полям
fmt.Println(task.Title)
task.Done = true

// Вложенные структуры
type User struct {
    Name    string
    Age     int
    Address struct {
        City    string
        Country string
    }
}
```

---

## Часть 5 — Функции

```go
// Базовая функция
func add(a int, b int) int {
    return a + b
}

// Несколько параметров одного типа
func add(a, b int) int {
    return a + b
}

// Несколько возвращаемых значений (идиома Go)
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("деление на ноль")
    }
    return a / b, nil
}

// Использование
result, err := divide(10, 3)
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)

// Именованные возвращаемые значения
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, n := range nums {
        if n < min { min = n }
        if n > max { max = n }
    }
    return    // bare return — возвращает min и max
}

// Variadic функции (переменное количество аргументов)
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
sum(1, 2, 3)
sum(1, 2, 3, 4, 5)

nums := []int{1, 2, 3}
sum(nums...)    // Развернуть слайс в аргументы
```

### Методы

```go
type Task struct {
    ID    int
    Title string
    Done  bool
}

// Метод — функция с receiver
func (t Task) String() string {
    status := "[ ]"
    if t.Done {
        status = "[x]"
    }
    return fmt.Sprintf("%s %d: %s", status, t.ID, t.Title)
}

// Pointer receiver — если нужно изменить структуру
func (t *Task) Complete() {
    t.Done = true
}

// Использование
task := Task{ID: 1, Title: "Learn Go"}
fmt.Println(task)       // вызовет String()
task.Complete()         // изменит Done в оригинале
```

### Функции как значения

```go
// Функция как переменная
greet := func(name string) string {
    return "Hello, " + name
}
fmt.Println(greet("Alice"))

// Передача функции как аргумент
func apply(nums []int, f func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = f(n)
    }
    return result
}

doubled := apply([]int{1, 2, 3}, func(n int) int {
    return n * 2
})
```

---

## Часть 6 — Обработка ошибок

В Go нет исключений (exceptions). Ошибки — это обычные значения:

```go
// error — встроенный интерфейс
type error interface {
    Error() string
}

// Функция возвращает ошибку
func readFile(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("readFile: %w", err)    // обернуть ошибку
    }
    return data, nil
}

// Всегда проверяй ошибку
data, err := readFile("config.json")
if err != nil {
    log.Fatal(err)    // или return err
}

// Создать свою ошибку
var ErrNotFound = errors.New("not found")

func findTask(id int) (Task, error) {
    for _, t := range tasks {
        if t.ID == id {
            return t, nil
        }
    }
    return Task{}, fmt.Errorf("findTask: task %d: %w", id, ErrNotFound)
}

// Проверить тип ошибки
if errors.Is(err, ErrNotFound) {
    fmt.Println("Задача не найдена")
}
```

---

## Практические задания

### Задание 5.1 — Основы

Создай файл `~/projects/learn-go/basics.go`:

```go
package main

import "fmt"

func main() {
    // 1. Объяви переменные: имя (строка), возраст (int), рост (float64)
    // 2. Выведи их через fmt.Printf с форматированием
    // 3. Напиши функцию которая принимает имя и возраст и возвращает строку
    //    "Привет, Alice! Тебе 25 лет."
    // 4. Создай слайс из 5 имён и выведи их пронумерованным списком
    // 5. Создай мапу "город → население" для 3 городов и выведи
}
```

**Что нужно сделать:** Заполни функцию `main` по комментариям. Программа должна компилироваться и выводить результат.

---

### Задание 5.2 — Структуры и методы

Создай файл `~/projects/learn-go/task.go`:

Реализуй тип `Task` со следующим поведением:
- Поля: `ID int`, `Title string`, `Priority int` (1-3), `Done bool`
- Метод `Complete()` — отмечает задачу выполненной
- Метод `String() string` — возвращает строку вида `[x] 1: Learn Go (priority: 2)`
- Функция `NewTask(id int, title string, priority int) Task` — создаёт новую задачу
- Функция `FilterDone(tasks []Task) []Task` — возвращает только выполненные задачи
- Функция `FilterByPriority(tasks []Task, priority int) []Task` — фильтр по приоритету

В `main` создай 5 задач, пометь 2 из них выполненными, выведи все задачи, потом только невыполненные с приоритетом 1.

---

### Задание 5.3 — Обработка ошибок

Создай файл `~/projects/learn-go/storage.go`:

```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("task not found")
var ErrDuplicate = errors.New("task already exists")
var ErrInvalidInput = errors.New("invalid input")

type Storage struct {
    tasks  []Task
    nextID int
}

func NewStorage() *Storage {
    return &Storage{nextID: 1}
}

// Реализуй методы:
// Add(title string, priority int) (Task, error)
//   - возвращает ErrInvalidInput если title пустой
//   - возвращает ErrInvalidInput если priority не 1, 2 или 3
//
// Get(id int) (Task, error)
//   - возвращает ErrNotFound если задача не найдена
//
// Delete(id int) error
//   - возвращает ErrNotFound если задача не найдена
//
// Complete(id int) error
//   - возвращает ErrNotFound если задача не найдена
//
// List() []Task
//   - возвращает все задачи
```

В `main` протестируй все методы включая случаи с ошибками.

---

### Задание 5.4 — Файлы и ввод/вывод

Создай программу `~/projects/learn-go/fileio.go` которая:

1. Создаёт список задач в памяти
2. Сохраняет их в файл `tasks.txt` в формате: `ID|Title|Priority|Done`
3. Читает файл обратно и восстанавливает список задач
4. Выводит загруженные задачи

```go
import (
    "bufio"
    "fmt"
    "os"
    "strconv"
    "strings"
)

func saveTasks(tasks []Task, filename string) error {
    // Открыть файл для записи
    // Записать каждую задачу в формате "ID|Title|Priority|Done\n"
}

func loadTasks(filename string) ([]Task, error) {
    // Открыть файл для чтения
    // Прочитать построчно
    // Разбить каждую строку по "|"
    // Создать Task из частей
}
```

---

### Задание 5.5 — Финальный проект урока

Собери всё в одну программу `~/projects/todo-service/main.go`:

Простое CLI (command-line interface) приложение для управления задачами:

```
$ ./todo add "Learn Go" 1
Задача добавлена: [1] Learn Go (priority: 1)

$ ./todo add "Write tests" 2
Задача добавлена: [2] Write tests (priority: 2)

$ ./todo list
[ ] 1: Learn Go (priority: 1)
[ ] 2: Write tests (priority: 2)

$ ./todo done 1
Задача 1 выполнена

$ ./todo list
[x] 1: Learn Go (priority: 1)
[ ] 2: Write tests (priority: 2)

$ ./todo delete 2
Задача 2 удалена

$ ./todo list
[x] 1: Learn Go (priority: 1)
```

Требования:
- Читать команду из `os.Args`
- Задачи сохранять в файл `tasks.json` (используй `encoding/json`)
- При каждом запуске загружать задачи из файла
- Обрабатывать ошибки — неправильные аргументы, несуществующие ID
- Добавить команду `help` которая выводит список команд

Закоммить результат в `todo-service` репозиторий на GitHub.

---

## Шпаргалка

| Концепция | Пример |
|-----------|--------|
| Переменная | `x := 42` |
| Функция | `func add(a, b int) int { return a+b }` |
| Слайс | `s := []int{1, 2, 3}; s = append(s, 4)` |
| Мапа | `m := map[string]int{"a": 1}` |
| Структура | `type T struct { Name string }` |
| Метод | `func (t *T) DoSomething() {}` |
| Ошибка | `val, err := f(); if err != nil { return err }` |
| Цикл | `for i, v := range slice { }` |

---

## Ресурсы для изучения

- **Официальный тур:** `https://go.dev/tour/` — интерактивный, прямо в браузере
- **Документация:** `https://go.dev/doc/` — официальная, на английском
- **Книга на русском:** "Язык программирования Go" — Алан Донован, Брайан Керниган
- **Практика:** `https://exercism.org/tracks/go` — задачи с проверкой
- **Effective Go:** `https://go.dev/doc/effective_go` — как писать идиоматичный Go

---

## Как понять что урок пройден

- [ ] Понимаю разницу между массивом и слайсом
- [ ] Умею создавать структуры и методы к ним
- [ ] Понимаю почему в Go нет исключений и как работает обработка ошибок
- [ ] Умею читать и записывать файлы
- [ ] Написал CLI приложение для задач
- [ ] Закоммитил код в GitHub с осмысленными сообщениями коммитов

---

*Следующий урок: Go — интерфейсы, горутины и HTTP сервер*
