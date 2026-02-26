# Урок 23 — Go продвинутый: generics, reflection, code generation

## Зачем это нужно

Первые уроки Go дали основы. Теперь — инструменты которые позволяют писать меньше повторяющегося кода, строить гибкие библиотеки и автоматизировать рутину. Generics убирают дублирование когда одна логика работает для разных типов. Reflection позволяет писать код который работает с любыми типами во время выполнения — это основа JSON сериализации, ORM, DI фреймворков. Code generation — генерировать код программно, как делает protoc или mockery. После этого урока ты поймёшь как устроены инструменты которыми уже пользовался.

---

## Часть 1 — Generics

### Проблема без generics

```go
// Без generics — дублирование для каждого типа
func MinInt(a, b int) int {
    if a < b { return a }
    return b
}

func MinFloat(a, b float64) float64 {
    if a < b { return a }
    return b
}

func MinString(a, b string) string {
    if a < b { return a }
    return b
}

// С generics — одна функция для всех типов
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}

Min(3, 5)         // int
Min(3.14, 2.71)   // float64
Min("abc", "xyz") // string
```

### Синтаксис

```go
// Функция с type parameter
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Использование
nums := []int{1, 2, 3, 4}
doubled := Map(nums, func(n int) int { return n * 2 })
// [2, 4, 6, 8]

strs := Map(nums, func(n int) string { return fmt.Sprintf("%d!", n) })
// ["1!", "2!", "3!", "4!"]
```

### Constraints — ограничения на типы

```go
import "golang.org/x/exp/constraints"

// Встроенные constraints
any              // любой тип
comparable       // типы которые можно сравнивать через ==
constraints.Ordered    // типы которые поддерживают < > <= >=
constraints.Integer    // все целые числа
constraints.Float      // все числа с плавающей точкой
constraints.Number     // Integer | Float

// Свой constraint через interface
type Stringer interface {
    String() string
}

func Print[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

// Constraint из нескольких типов
type Number interface {
    int | int8 | int16 | int32 | int64 |
    uint | uint8 | uint16 | uint32 | uint64 |
    float32 | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

### Generic структуры

```go
// Стек для любого типа
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Len() int { return len(s.items) }

// Использование
s := Stack[int]{}
s.Push(1)
s.Push(2)
v, _ := s.Pop()    // 2

// Generic Result тип — замена (T, error)
type Result[T any] struct {
    Value T
    Err   error
}

func (r Result[T]) IsOk() bool { return r.Err == nil }

func (r Result[T]) Unwrap() T {
    if r.Err != nil {
        panic(r.Err)
    }
    return r.Value
}

func GetTask(id int) Result[Task] {
    task, err := storage.Get(context.Background(), id)
    return Result[Task]{Value: task, Err: err}
}
```

### Практические generic утилиты

```go
// Filter
func Filter[T any](slice []T, pred func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if pred(v) { result = append(result, v) }
    }
    return result
}

// Reduce
func Reduce[T, U any](slice []T, initial U, fn func(U, T) U) U {
    acc := initial
    for _, v := range slice {
        acc = fn(acc, v)
    }
    return acc
}

// Contains
func Contains[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item { return true }
    }
    return false
}

// Keys и Values для map
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

// Chunk — разбить на куски
func Chunk[T any](slice []T, size int) [][]T {
    var chunks [][]T
    for size < len(slice) {
        slice, chunks = slice[size:], append(chunks, slice[:size])
    }
    return append(chunks, slice)
}

// Использование
tasks := []Task{{Done: true}, {Done: false}, {Done: true}}
pending := Filter(tasks, func(t Task) bool { return !t.Done })

ids := Map(tasks, func(t Task) int { return t.ID })
total := Reduce(ids, 0, func(acc, id int) int { return acc + id })
```

---

## Часть 2 — Reflection

### Что такое reflection

Reflection позволяет программе исследовать и изменять свою структуру во время выполнения. Это основа:
- `encoding/json` — сериализация любой структуры
- `database/sql` + ORM — маппинг строк БД в структуры
- `testing` — сравнение значений разных типов
- `dependency injection` фреймворки

```go
import "reflect"

// Основные типы reflect
// reflect.Type  — описание типа
// reflect.Value — значение с возможностью изменения

var x float64 = 3.14

t := reflect.TypeOf(x)
v := reflect.ValueOf(x)

fmt.Println(t)          // float64
fmt.Println(t.Kind())   // float64
fmt.Println(v.Float())  // 3.14
```

### Исследование структур

```go
type Task struct {
    ID       int    `json:"id" db:"id"`
    Title    string `json:"title" db:"title"`
    Priority int    `json:"priority,omitempty"`
    Done     bool   `json:"-"`    // не сериализовать
}

func inspectStruct(v interface{}) {
    t := reflect.TypeOf(v)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()    // разыменовать указатель
    }
    if t.Kind() != reflect.Struct {
        fmt.Println("not a struct")
        return
    }

    fmt.Printf("Struct: %s\n", t.Name())

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        jsonTag := field.Tag.Get("json")
        dbTag := field.Tag.Get("db")

        fmt.Printf("  Field: %-12s Type: %-8s json:'%s' db:'%s'\n",
            field.Name, field.Type, jsonTag, dbTag)
    }
}

inspectStruct(Task{})
// Struct: Task
//   Field: ID           Type: int      json:'id'               db:'id'
//   Field: Title        Type: string   json:'title'            db:'title'
//   Field: Priority     Type: int      json:'priority,omitempty' db:''
//   Field: Done         Type: bool     json:'-'                db:''
```

### Простая JSON сериализация через reflection

```go
// Упрощённая версия json.Marshal для понимания
func Marshal(v interface{}) ([]byte, error) {
    val := reflect.ValueOf(v)
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }

    switch val.Kind() {
    case reflect.Struct:
        return marshalStruct(val)
    case reflect.Slice:
        return marshalSlice(val)
    case reflect.String:
        return json.Marshal(val.String())    // используем стандартный для строк
    case reflect.Int, reflect.Int64:
        return []byte(fmt.Sprintf("%d", val.Int())), nil
    case reflect.Bool:
        if val.Bool() { return []byte("true"), nil }
        return []byte("false"), nil
    default:
        return nil, fmt.Errorf("unsupported type: %s", val.Kind())
    }
}

func marshalStruct(val reflect.Value) ([]byte, error) {
    t := val.Type()
    parts := make([]string, 0)

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldVal := val.Field(i)

        // Получить имя из json тега
        tag := field.Tag.Get("json")
        if tag == "-" { continue }    // пропустить
        name := field.Name
        if tag != "" {
            name = strings.Split(tag, ",")[0]
        }

        data, err := Marshal(fieldVal.Interface())
        if err != nil { return nil, err }

        parts = append(parts, fmt.Sprintf("%q:%s", name, data))
    }

    return []byte("{" + strings.Join(parts, ",") + "}"), nil
}
```

### Динамический вызов методов

```go
// Вызвать метод по имени строкой
func callMethod(obj interface{}, method string, args ...interface{}) []reflect.Value {
    v := reflect.ValueOf(obj)
    m := v.MethodByName(method)
    if !m.IsValid() {
        panic(fmt.Sprintf("method %s not found", method))
    }

    in := make([]reflect.Value, len(args))
    for i, arg := range args {
        in[i] = reflect.ValueOf(arg)
    }

    return m.Call(in)
}

// Использование
handler := &Handler{}
callMethod(handler, "listTasks", w, r)

// Dependency injection — автоматически заполнить поля структуры
type Container struct {
    Storage  Storage
    Redis    *redis.Client
    Logger   *slog.Logger
}

func Inject(target interface{}, container Container) {
    v := reflect.ValueOf(target)
    if v.Kind() != reflect.Ptr { panic("need pointer") }
    v = v.Elem()

    cVal := reflect.ValueOf(container)
    cType := cVal.Type()

    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        if !field.CanSet() { continue }

        // Найти совпадение по типу в контейнере
        for j := 0; j < cType.NumField(); j++ {
            if cType.Field(j).Type == field.Type() {
                field.Set(cVal.Field(j))
                break
            }
        }
    }
}
```

---

## Часть 3 — Code Generation

### Зачем генерировать код

```go
// Вместо того чтобы писать вручную:
func (s *PostgresStorage) scanTask(row pgx.Row) (Task, error) {
    var t Task
    return t, row.Scan(&t.ID, &t.Title, &t.Priority, &t.Done, &t.CreatedAt, &t.UpdatedAt)
}

// Генератор создаёт это автоматически из аннотации к структуре
// При изменении Task — перегенерировать одной командой
```

### go generate

```go
// Добавить директиву в файл
//go:generate mockery --name=Storage --dir=. --output=mocks

// Запустить генерацию
go generate ./...
```

### Написать свой генератор

Создадим генератор который создаёт метод `Validate()` для структур с тегами `validate`.

```go
// tools/gen-validator/main.go
package main

import (
    "bytes"
    "go/ast"
    "go/format"
    "go/parser"
    "go/token"
    "os"
    "strings"
    "text/template"
)

const validatorTemplate = `
// Code generated by gen-validator. DO NOT EDIT.
package {{ .Package }}

import "fmt"

{{ range .Structs }}
func (v {{ .Name }}) Validate() error {
    {{ range .Fields }}
    {{ if .Required }}
    if v.{{ .Name }} == {{ .ZeroValue }} {
        return fmt.Errorf("{{ .JSONName }} is required")
    }
    {{ end }}
    {{ if .Min }}
    if v.{{ .Name }} < {{ .Min }} {
        return fmt.Errorf("{{ .JSONName }} must be >= {{ .Min }}")
    }
    {{ end }}
    {{ if .Max }}
    if v.{{ .Name }} > {{ .Max }} {
        return fmt.Errorf("{{ .JSONName }} must be <= {{ .Max }}")
    }
    {{ end }}
    {{ end }}
    return nil
}
{{ end }}
`

type FieldInfo struct {
    Name      string
    JSONName  string
    Required  bool
    Min, Max  string
    ZeroValue string
}

type StructInfo struct {
    Name   string
    Fields []FieldInfo
}

type TemplateData struct {
    Package string
    Structs []StructInfo
}

func main() {
    // Парсим Go файл
    fset := token.NewFileSet()
    f, err := parser.ParseFile(fset, os.Args[1], nil, parser.ParseComments)
    if err != nil {
        panic(err)
    }

    data := TemplateData{Package: f.Name.Name}

    // Обходим AST
    ast.Inspect(f, func(n ast.Node) bool {
        typeSpec, ok := n.(*ast.TypeSpec)
        if !ok { return true }

        structType, ok := typeSpec.Type.(*ast.StructType)
        if !ok { return true }

        si := StructInfo{Name: typeSpec.Name.Name}

        for _, field := range structType.Fields.List {
            if field.Tag == nil { continue }

            tag := strings.Trim(field.Tag.Value, "`")
            validateTag := parseTag(tag, "validate")
            if validateTag == "" { continue }

            jsonTag := parseTag(tag, "json")
            jsonName := field.Names[0].Name
            if jsonTag != "" {
                jsonName = strings.Split(jsonTag, ",")[0]
            }

            fi := FieldInfo{
                Name:     field.Names[0].Name,
                JSONName: jsonName,
            }

            // Парсим validate:"required,min=1,max=100"
            for _, rule := range strings.Split(validateTag, ",") {
                if rule == "required" {
                    fi.Required = true
                    // Определить zero value по типу
                    switch fmt.Sprint(field.Type) {
                    case "string": fi.ZeroValue = `""`
                    default:       fi.ZeroValue = "0"
                    }
                }
                if strings.HasPrefix(rule, "min=") {
                    fi.Min = strings.TrimPrefix(rule, "min=")
                }
                if strings.HasPrefix(rule, "max=") {
                    fi.Max = strings.TrimPrefix(rule, "max=")
                }
            }

            si.Fields = append(si.Fields, fi)
        }

        if len(si.Fields) > 0 {
            data.Structs = append(data.Structs, si)
        }

        return true
    })

    // Генерируем код
    tmpl := template.Must(template.New("validator").Parse(validatorTemplate))
    var buf bytes.Buffer
    tmpl.Execute(&buf, data)

    // Форматируем через gofmt
    formatted, err := format.Source(buf.Bytes())
    if err != nil {
        // Если форматирование упало — записать неформатированный для отладки
        os.WriteFile("validator_gen.go", buf.Bytes(), 0644)
        panic(fmt.Sprintf("format error: %v", err))
    }

    os.WriteFile("validator_gen.go", formatted, 0644)
}

func parseTag(tag, key string) string {
    for _, part := range strings.Fields(tag) {
        if strings.HasPrefix(part, key+":") {
            return strings.Trim(strings.TrimPrefix(part, key+":"), `"`)
        }
    }
    return ""
}
```

Использование:
```go
// task.go
type CreateTaskRequest struct {
    Title    string `json:"title"    validate:"required"`
    Priority int    `json:"priority" validate:"required,min=1,max=3"`
}

//go:generate go run tools/gen-validator/main.go task.go

// После go generate ./... появится validator_gen.go:
func (v CreateTaskRequest) Validate() error {
    if v.Title == "" {
        return fmt.Errorf("title is required")
    }
    if v.Priority == 0 {
        return fmt.Errorf("priority is required")
    }
    if v.Priority < 1 {
        return fmt.Errorf("priority must be >= 1")
    }
    if v.Priority > 3 {
        return fmt.Errorf("priority must be <= 3")
    }
    return nil
}
```

### Использование ast/template для генерации

```go
// Популярные инструменты кодогенерации в Go
// jennifer — программное создание Go кода
go get github.com/dave/jennifer/jen

// Пример: генерировать CRUD методы для любой структуры
func generateCRUD(structName, tableName string) *jen.File {
    f := jen.NewFile("main")

    // Генерируем func GetByID
    f.Func().
        Params(jen.Id("s").Op("*").Id(structName+"Storage")).
        Id("GetByID").
        Params(
            jen.Id("ctx").Qual("context", "Context"),
            jen.Id("id").Int(),
        ).
        Params(jen.Id(structName), jen.Error()).
        Block(
            jen.Var().Id("result").Id(structName),
            jen.Err().Op(":=").Id("s").Dot("db").Dot("QueryRow").Call(
                jen.Id("ctx"),
                jen.Lit(fmt.Sprintf("SELECT * FROM %s WHERE id = $1", tableName)),
                jen.Id("id"),
            ).Dot("Scan").Call(jen.Op("&").Id("result")),
            jen.If(jen.Err().Op("!=").Nil()).Block(
                jen.Return(jen.Id(structName).Values(), jen.Err()),
            ),
            jen.Return(jen.Id("result"), jen.Nil()),
        )

    return f
}
```

---

## Практические задания

### Задание 23.1 — Generic утилиты

Создай файл `pkg/slices/slices.go` с generic функциями:

```go
func Map[T, U any](s []T, f func(T) U) []U
func Filter[T any](s []T, f func(T) bool) []T
func Reduce[T, U any](s []T, init U, f func(U, T) U) U
func Contains[T comparable](s []T, v T) bool
func Unique[T comparable](s []T) []T
func GroupBy[T any, K comparable](s []T, key func(T) K) map[K][]T
func Partition[T any](s []T, f func(T) bool) (yes, no []T)
```

Напиши тесты для каждой функции. Используй эти утилиты в `todo-service` где есть ручные циклы.

**Что нужно сделать:** Найди в `todo-service` хотя бы 3 места где можно заменить циклы на Map/Filter/Reduce. Сделай замену.

---

### Задание 23.2 — Generic кэш с TTL

Реализуй generic кэш:

```go
type Cache[K comparable, V any] struct { /* ... */ }

func NewCache[K comparable, V any](defaultTTL time.Duration) *Cache[K, V]

func (c *Cache[K, V]) Set(key K, value V)
func (c *Cache[K, V]) SetWithTTL(key K, value V, ttl time.Duration)
func (c *Cache[K, V]) Get(key K) (V, bool)
func (c *Cache[K, V]) Delete(key K)
func (c *Cache[K, V]) Len() int

// Вариация: GetOrSet — вернуть из кэша или вычислить и сохранить
func (c *Cache[K, V]) GetOrSet(key K, fn func() (V, error)) (V, error)
```

Требования:
- Thread-safe (sync.RWMutex)
- Фоновая очистка истёкших записей каждую минуту
- Тесты на concurrent доступ (`go test -race`)

Используй этот кэш вместо прямых обращений к Redis для часто запрашиваемых данных.

---

### Задание 23.3 — Reflection для маппинга структур

Напиши функцию `MapStruct(src, dst interface{}) error` которая копирует поля из одной структуры в другую по имени поля (или по `mapstruct` тегу):

```go
type TaskRow struct {    // DTO из БД
    ID         int
    Title      string
    Priority   int
    IsDone     bool    `mapstruct:"done"`
    CreatedAt  time.Time
}

type Task struct {       // бизнес-объект
    ID        int
    Title     string
    Priority  int
    Done      bool
    CreatedAt time.Time
}

var row TaskRow
// ... scan из БД ...

var task Task
MapStruct(row, &task)
// task.Done = row.IsDone (через тег mapstruct)
// остальные поля по совпадению имени
```

Напиши тесты:
- Поля совпадают по имени — копируются
- Поля совпадают по тегу — копируются
- Поле есть в src но нет в dst — пропускается (не ошибка)
- Несовместимые типы — возвращает ошибку

---

### Задание 23.4 — Генератор моков

Напиши `tools/gen-mock/main.go` — простой генератор моков для интерфейсов.

Входные данные:
```go
// storage.go
type Storage interface {
    Add(ctx context.Context, title string, priority int) (Task, error)
    Get(ctx context.Context, id int) (Task, error)
    List(ctx context.Context) ([]Task, error)
}
```

Генерирует:
```go
// mocks/storage_mock.go (сгенерировано)
type MockStorage struct {
    AddFunc  func(ctx context.Context, title string, priority int) (Task, error)
    GetFunc  func(ctx context.Context, id int) (Task, error)
    ListFunc func(ctx context.Context) ([]Task, error)
}

func (m *MockStorage) Add(ctx context.Context, title string, priority int) (Task, error) {
    return m.AddFunc(ctx, title, priority)
}
// ... и т.д.
```

Используй `go/parser` + `go/ast` для парсинга интерфейса и `text/template` для генерации кода.

---

### Задание 23.5 — Финальный проект урока

Создай библиотеку `pkg/` для `todo-service`:

**`pkg/slices/`** — generic утилиты для слайсов (Map, Filter, Reduce, ...)

**`pkg/cache/`** — generic TTL кэш

**`pkg/validate/`** — валидация через reflection и теги:
```go
type CreateTaskRequest struct {
    Title    string `validate:"required,max=200"`
    Priority int    `validate:"required,min=1,max=3"`
}

err := validate.Struct(req)
// err: "title: required" или nil
```

**`pkg/mapper/`** — маппинг структур через reflection

**`tools/gen-mock/`** — генератор моков из интерфейсов

**Добавь в `Makefile`:**
```makefile
generate:
    go generate ./...    # запускает все //go:generate директивы
```

Весь код в `pkg/` должен иметь тесты с покрытием > 85%.

Закоммить с тегом `v2.3.0`.

---

## Шпаргалка

| Концепция | Когда использовать |
|-----------|-------------------|
| Generics | Одна логика для разных типов (коллекции, утилиты) |
| `constraints.Ordered` | Типы поддерживающие сравнение < > |
| `comparable` | Типы которые можно сравнивать через == |
| `reflect.TypeOf(v)` | Получить тип значения |
| `reflect.ValueOf(v)` | Получить значение для изменения |
| `t.NumField()` | Количество полей структуры |
| `t.Field(i).Tag.Get("json")` | Получить тег поля |
| `go:generate` | Директива для запуска генерации |
| `go/ast` | Парсинг Go кода в AST |
| `text/template` | Шаблоны для генерации кода |

---

## Ресурсы для изучения

- **Generics tutorial:** `https://go.dev/doc/tutorial/generics`
- **Generics proposal:** `https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md`
- **golang.org/x/exp/constraints:** `https://pkg.go.dev/golang.org/x/exp/constraints`
- **reflect package:** `https://pkg.go.dev/reflect`
- **Laws of Reflection:** `https://go.dev/blog/laws-of-reflection` — обязательно прочитать
- **go/ast:** `https://pkg.go.dev/go/ast`
- **jennifer:** `https://github.com/dave/jennifer`

---

## Как понять что урок пройден

- [ ] Написал generic функции Map, Filter, Reduce и использую их в коде
- [ ] Generic кэш работает корректно под конкурентной нагрузкой (`-race`)
- [ ] Могу через reflection прочитать поля структуры и теги
- [ ] Написал простой маппер структур через reflection
- [ ] `go generate ./...` генерирует код без ошибок
- [ ] Написал простой генератор моков используя `go/ast`
- [ ] Весь код в `pkg/` покрыт тестами > 85%

---

*Следующий урок: PostgreSQL продвинутый — репликация, партиционирование, full-text search*
