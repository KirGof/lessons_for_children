# Урок 17 — Безопасность: JWT, OWASP, HashiCorp Vault

## Зачем это нужно

Приложение без безопасности — это открытая дверь. Утечка паролей пользователей, несанкционированный доступ к API, секреты в git — всё это реальные инциденты которые случаются каждый день. После этого урока `todo-service` будет иметь полноценную аутентификацию, защиту от основных атак и правильное управление секретами. Это не параноя — это профессиональный стандарт.

---

## Часть 1 — Аутентификация и JWT

### Как работает JWT

JWT (JSON Web Token) — способ передавать информацию между сервисами безопасно. Состоит из трёх частей разделённых точкой:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJleHAiOjE3MDAwMDAwMDB9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
       HEADER                                    PAYLOAD                           SIGNATURE
```

- **Header** — алгоритм подписи (`HS256`, `RS256`)
- **Payload** — данные: user_id, роли, время истечения
- **Signature** — подпись которая доказывает что токен не подделан

```
Signature = HMAC_SHA256(base64(header) + "." + base64(payload), SECRET_KEY)
```

Сервер не хранит токены — он проверяет подпись. Если подпись верна и токен не истёк — пользователь аутентифицирован.

### Аутентификация — регистрация и вход

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto
```

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    ID           int       `json:"id"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`    // никогда не отдавать клиенту
    CreatedAt    time.Time `json:"created_at"`
}

type Claims struct {
    UserID int    `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

var jwtSecret = []byte(os.Getenv("JWT_SECRET"))

// Хэширование пароля — bcrypt автоматически добавляет соль
func hashPassword(password string) (string, error) {
    if len(password) < 8 {
        return "", errors.New("password must be at least 8 characters")
    }
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(hash), err
}

// Проверка пароля
func checkPassword(hash, password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// Создать JWT токен
func createToken(user User) (string, error) {
    claims := Claims{
        UserID: user.ID,
        Email:  user.Email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "todo-service",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// Проверить JWT токен
func validateToken(tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return jwtSecret, nil
    })

    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }

    return claims, nil
}
```

### HTTP обработчики аутентификации

```go
// POST /auth/register
func (h *Handler) register(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    if err := readJSON(r, &req); err != nil {
        writeJSON(w, http.StatusBadRequest, errResp("invalid json"))
        return
    }

    if req.Email == "" || req.Password == "" {
        writeJSON(w, http.StatusBadRequest, errResp("email and password required"))
        return
    }

    // Проверить что email не занят
    _, err := h.userStorage.GetByEmail(r.Context(), req.Email)
    if err == nil {
        writeJSON(w, http.StatusConflict, errResp("email already registered"))
        return
    }

    hash, err := hashPassword(req.Password)
    if err != nil {
        writeJSON(w, http.StatusBadRequest, errResp(err.Error()))
        return
    }

    user, err := h.userStorage.Create(r.Context(), req.Email, hash)
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, errResp("create user failed"))
        return
    }

    token, err := createToken(user)
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, errResp("create token failed"))
        return
    }

    writeJSON(w, http.StatusCreated, map[string]interface{}{
        "user":  user,
        "token": token,
    })
}

// POST /auth/login
func (h *Handler) login(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    if err := readJSON(r, &req); err != nil {
        writeJSON(w, http.StatusBadRequest, errResp("invalid json"))
        return
    }

    user, err := h.userStorage.GetByEmail(r.Context(), req.Email)
    if err != nil {
        // Одинаковое сообщение — не раскрываем существует ли email
        writeJSON(w, http.StatusUnauthorized, errResp("invalid credentials"))
        return
    }

    if !checkPassword(user.PasswordHash, req.Password) {
        writeJSON(w, http.StatusUnauthorized, errResp("invalid credentials"))
        return
    }

    token, err := createToken(user)
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, errResp("create token failed"))
        return
    }

    writeJSON(w, http.StatusOK, map[string]string{"token": token})
}
```

### Auth middleware

```go
type contextKey string
const userContextKey contextKey = "user"

// Middleware — проверить токен и добавить пользователя в контекст
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            writeJSON(w, http.StatusUnauthorized, errResp("authorization header required"))
            return
        }

        // Ожидаем формат: "Bearer <token>"
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            writeJSON(w, http.StatusUnauthorized, errResp("invalid authorization header format"))
            return
        }

        claims, err := validateToken(parts[1])
        if err != nil {
            writeJSON(w, http.StatusUnauthorized, errResp("invalid or expired token"))
            return
        }

        // Добавить информацию о пользователе в контекст
        ctx := context.WithValue(r.Context(), userContextKey, claims)
        next(w, r.WithContext(ctx))
    }
}

// Получить пользователя из контекста
func userFromContext(ctx context.Context) (*Claims, bool) {
    claims, ok := ctx.Value(userContextKey).(*Claims)
    return claims, ok
}

// Использование: задачи принадлежат пользователю
func (h *Handler) listTasks(w http.ResponseWriter, r *http.Request) {
    claims, ok := userFromContext(r.Context())
    if !ok {
        writeJSON(w, http.StatusUnauthorized, errResp("unauthorized"))
        return
    }

    // Возвращать только задачи этого пользователя
    tasks, err := h.storage.ListByUser(r.Context(), claims.UserID)
    // ...
}

// Роутер
http.HandleFunc("/auth/register", h.register)
http.HandleFunc("/auth/login", h.login)
http.HandleFunc("/tasks", authMiddleware(h.listTasks))      // защищённый
http.HandleFunc("/health", h.health)                        // открытый
```

---

## Часть 2 — OWASP Top 10

OWASP (Open Web Application Security Project) ведёт список 10 самых критичных уязвимостей веб-приложений. Каждый должен знать этот список и уметь защититься.

### A01: Broken Access Control

Пользователь может получить доступ к чужим данным.

```go
// УЯЗВИМО: пользователь может запросить любую задачу
func (h *Handler) getTask(w http.ResponseWriter, r *http.Request) {
    id := getIDFromPath(r)
    task, _ := h.storage.Get(r.Context(), id)
    writeJSON(w, http.StatusOK, task)
}

// БЕЗОПАСНО: проверить что задача принадлежит пользователю
func (h *Handler) getTask(w http.ResponseWriter, r *http.Request) {
    claims, _ := userFromContext(r.Context())
    id := getIDFromPath(r)

    task, err := h.storage.Get(r.Context(), id)
    if err != nil {
        writeJSON(w, http.StatusNotFound, errResp("not found"))
        return
    }

    // Проверить владельца
    if task.UserID != claims.UserID {
        writeJSON(w, http.StatusForbidden, errResp("forbidden"))
        return
    }

    writeJSON(w, http.StatusOK, task)
}
```

### A02: Cryptographic Failures

Слабое шифрование или хранение паролей в открытом виде.

```go
// НИКОГДА не делай так:
password_hash = md5(password)         // MD5 — сломан
password_hash = sha256(password)      // без соли — уязвим к rainbow tables
password_hash = sha256(salt+password) // лучше, но всё равно слабо

// ПРАВИЛЬНО: bcrypt, scrypt или argon2
hash, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
// bcrypt автоматически генерирует соль и встраивает её в хэш
// bcrypt намеренно медленный — это защита от брутфорса
```

### A03: SQL Injection

Злоумышленник вставляет SQL код через пользовательский ввод.

```go
// УЯЗВИМО — никогда так не делай:
query := fmt.Sprintf("SELECT * FROM tasks WHERE title = '%s'", userInput)
// userInput = "'; DROP TABLE tasks; --"
// Итоговый запрос: SELECT * FROM tasks WHERE title = ''; DROP TABLE tasks; --'

// БЕЗОПАСНО — параметризованные запросы:
db.QueryRow("SELECT * FROM tasks WHERE title = $1", userInput)
// PostgreSQL обрабатывает $1 как данные, а не как SQL код
```

### A04: Insecure Design

Отсутствие защиты в архитектуре.

```go
// Принципы:
// 1. Defense in depth — несколько уровней защиты
// 2. Fail secure — при ошибке запрещать, не разрешать
// 3. Least privilege — минимум прав для каждого компонента
// 4. Separation of concerns — разные данные — разные хранилища

// Пример: не возвращай лишние данные
type UserResponse struct {
    ID    int    `json:"id"`
    Email string `json:"email"`
    // PasswordHash НЕ включён
}
```

### A05: Security Misconfiguration

Небезопасные настройки по умолчанию.

```go
// Убрать версию из заголовков
w.Header().Del("X-Powered-By")
w.Header().Del("Server")

// Добавить security заголовки
func securityHeadersMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        next(w, r)
    }
}
```

### A07: Identification and Authentication Failures

```go
// Защита от брутфорса: блокировать после N неудачных попыток
func (h *Handler) login(w http.ResponseWriter, r *http.Request) {
    ip := r.RemoteAddr

    // Проверить количество попыток
    attempts, _ := rdb.Get(r.Context(), fmt.Sprintf("login_attempts:%s", ip)).Int()
    if attempts >= 5 {
        writeJSON(w, http.StatusTooManyRequests, errResp("too many failed attempts, try again in 15 minutes"))
        return
    }

    // ... проверить пароль ...

    if !checkPassword(user.PasswordHash, req.Password) {
        // Увеличить счётчик неудач
        rdb.Incr(r.Context(), fmt.Sprintf("login_attempts:%s", ip))
        rdb.Expire(r.Context(), fmt.Sprintf("login_attempts:%s", ip), 15*time.Minute)

        writeJSON(w, http.StatusUnauthorized, errResp("invalid credentials"))
        return
    }

    // Успешный вход — сбросить счётчик
    rdb.Del(r.Context(), fmt.Sprintf("login_attempts:%s", ip))
}

// Refresh токены — короткий access token + долгий refresh token
type TokenPair struct {
    AccessToken  string `json:"access_token"`   // живёт 15 минут
    RefreshToken string `json:"refresh_token"`  // живёт 30 дней
}
```

### A09: Security Logging and Monitoring Failures

```go
// Логировать события безопасности
func logSecurityEvent(ctx context.Context, event string, userID int, details map[string]interface{}) {
    slog.WarnContext(ctx, "security_event",
        "event", event,
        "user_id", userID,
        "ip", ipFromContext(ctx),
        "details", details,
    )
}

// Использование
logSecurityEvent(r.Context(), "login_failed", 0, map[string]interface{}{
    "email": req.Email,
    "reason": "invalid_password",
})
logSecurityEvent(r.Context(), "unauthorized_access", claims.UserID, map[string]interface{}{
    "path": r.URL.Path,
    "attempted_resource_id": id,
})
```

---

## Часть 3 — HashiCorp Vault

### Зачем Vault

Проблема: секреты (пароли, API ключи, TLS сертификаты) где-то должны храниться. В переменных окружения — незашифрованы. В K8s Secrets — base64. В git — катастрофа.

Vault — централизованное хранилище секретов:
- Все секреты зашифрованы
- Каждое обращение к секрету логируется
- Секреты имеют TTL и автоматически ротируются
- Доступ управляется политиками

### Запуск Vault

```bash
# Dev режим — для обучения (не для продакшена)
docker run -d \
  --name vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=myroot \
  hashicorp/vault

# Настроить CLI
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="myroot"

vault status
```

### Базовые операции

```bash
# KV (Key-Value) хранилище секретов
vault kv put secret/todo-service \
  database_url="postgres://user:pass@host:5432/db" \
  jwt_secret="super-secret-key-32-chars-minimum"

vault kv get secret/todo-service
vault kv get -field=database_url secret/todo-service
vault kv list secret/

# Обновить один ключ
vault kv patch secret/todo-service jwt_secret="new-secret"

# История версий
vault kv metadata get secret/todo-service
```

### Политики доступа

```bash
# Создать политику — todo-service может только читать свои секреты
vault policy write todo-service - <<EOF
path "secret/data/todo-service" {
    capabilities = ["read"]
}

path "secret/data/todo-service/*" {
    capabilities = ["read"]
}
EOF

# Создать токен с этой политикой
vault token create -policy=todo-service -ttl=1h
```

### Vault в Go приложении

```bash
go get github.com/hashicorp/vault/api
```

```go
package main

import (
    "context"
    "fmt"

    vault "github.com/hashicorp/vault/api"
    auth "github.com/hashicorp/vault/api/auth/kubernetes"
)

type VaultClient struct {
    client *vault.Client
    path   string
}

func NewVaultClient(addr, token, path string) (*VaultClient, error) {
    config := vault.DefaultConfig()
    config.Address = addr

    client, err := vault.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("vault client: %w", err)
    }

    client.SetToken(token)

    return &VaultClient{client: client, path: path}, nil
}

// В Kubernetes — аутентификация через ServiceAccount
func NewVaultClientK8s(addr, role, path string) (*VaultClient, error) {
    config := vault.DefaultConfig()
    config.Address = addr

    client, err := vault.NewClient(config)
    if err != nil {
        return nil, err
    }

    // Аутентифицироваться через K8s ServiceAccount токен
    k8sAuth, err := auth.NewKubernetesAuth(role)
    if err != nil {
        return nil, err
    }

    authInfo, err := client.Auth().Login(context.Background(), k8sAuth)
    if err != nil {
        return nil, fmt.Errorf("vault login: %w", err)
    }

    _ = authInfo
    return &VaultClient{client: client, path: path}, nil
}

// Получить секрет
func (v *VaultClient) GetSecret(ctx context.Context, key string) (string, error) {
    secret, err := v.client.KVv2(v.path).Get(ctx, "todo-service")
    if err != nil {
        return "", fmt.Errorf("get secret: %w", err)
    }

    value, ok := secret.Data[key].(string)
    if !ok {
        return "", fmt.Errorf("secret key %q not found or not a string", key)
    }

    return value, nil
}

// Использование при старте приложения
func loadConfig() (*Config, error) {
    vaultAddr := os.Getenv("VAULT_ADDR")
    vaultToken := os.Getenv("VAULT_TOKEN")

    if vaultAddr == "" {
        // Fallback на переменные окружения (для локальной разработки)
        return &Config{
            DatabaseURL: os.Getenv("DATABASE_URL"),
            JWTSecret:   os.Getenv("JWT_SECRET"),
        }, nil
    }

    vc, err := NewVaultClient(vaultAddr, vaultToken, "secret")
    if err != nil {
        return nil, err
    }

    ctx := context.Background()
    dbURL, err := vc.GetSecret(ctx, "database_url")
    if err != nil {
        return nil, err
    }

    jwtSecret, err := vc.GetSecret(ctx, "jwt_secret")
    if err != nil {
        return nil, err
    }

    return &Config{
        DatabaseURL: dbURL,
        JWTSecret:   jwtSecret,
    }, nil
}
```

### Dynamic secrets — секреты которые генерируются автоматически

```bash
# Vault может создавать временные учётные данные для PostgreSQL
vault secrets enable database

vault write database/config/todo-postgres \
    plugin_name=postgresql-database-plugin \
    allowed_roles="todo-service-role" \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/tododb" \
    username="vault_admin" \
    password="vault_admin_password"

vault write database/roles/todo-service-role \
    db_name=todo-postgres \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

# Теперь каждый раз получаешь новые временные креды
vault read database/creds/todo-service-role
# Key             Value
# lease_id        database/creds/todo-service-role/abc123
# username        v-todo-xYzAbC
# password        A1B2C3D4E5F6G7H8

# Через час они автоматически инвалидируются
```

---

## Практические задания

### Задание 17.1 — Регистрация и вход

Добавь аутентификацию в `todo-service`:

1. Таблица `users`:
   ```sql
   CREATE TABLE users (
       id            BIGSERIAL PRIMARY KEY,
       email         TEXT UNIQUE NOT NULL,
       password_hash TEXT NOT NULL,
       created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
   );
   ```
2. Эндпоинты: `POST /auth/register` и `POST /auth/login`
3. JWT токен с 24-часовым TTL
4. `authMiddleware` — проверяет токен в заголовке `Authorization: Bearer <token>`
5. Задачи привязаны к пользователю (добавь `user_id` в tasks)
6. Все task эндпоинты за `authMiddleware` — пользователь видит только свои задачи

```bash
# Тест
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"secret123"}'
# Вернёт token

curl http://localhost:8080/tasks \
  -H "Authorization: Bearer <token>"
```

---

### Задание 17.2 — OWASP защита

Проверь и закрой уязвимости в `todo-service`:

1. **SQL Injection** — убедись что все запросы используют `$1, $2` параметры, нет `fmt.Sprintf` в SQL
2. **Broken Access Control** — пользователь не может получить/изменить чужую задачу (напиши тест)
3. **Security Headers** — добавь middleware с заголовками безопасности
4. **Brute Force** — блокировать IP после 5 неудачных попыток входа через Redis
5. **Password strength** — минимум 8 символов, проверка через regexp

Напиши тесты:
```go
func TestBrokenAccessControl(t *testing.T) {
    // Создать двух пользователей
    // Пользователь 1 создаёт задачу
    // Пользователь 2 пытается получить/удалить задачу пользователя 1
    // Ожидаем 403 Forbidden
}
```

---

### Задание 17.3 — Vault для секретов

1. Запусти Vault в dev режиме через Docker Compose
2. Добавь секреты в Vault:
   ```bash
   vault kv put secret/todo-service \
     database_url="postgres://..." \
     jwt_secret="$(openssl rand -hex 32)"
   ```
3. Измени `main.go` — при наличии `VAULT_ADDR` читать секреты из Vault
4. Создай политику которая разрешает приложению только читать `secret/data/todo-service`

```yaml
# docker-compose.yml дополнение
  vault:
    image: hashicorp/vault
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: devtoken
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK
```

---

### Задание 17.4 — Сканирование безопасности

```bash
# gosec — статический анализ безопасности Go кода
go install github.com/securego/gosec/v2/cmd/gosec@latest
gosec ./...

# Изучи каждое предупреждение и исправь реальные проблемы
```

Добавь в CI:
```yaml
- name: Security scan
  run: |
    go install github.com/securego/gosec/v2/cmd/gosec@latest
    gosec -severity medium -confidence medium ./...
```

---

### Задание 17.5 — Финальный проект урока

Полная безопасность `todo-service`:

**Аутентификация:**
- Регистрация/вход с JWT
- Refresh токены (access 15 минут, refresh 30 дней)
- Блокировка после 5 неудачных попыток
- Logout — инвалидация refresh токена в Redis

**Авторизация:**
- Пользователь видит только свои задачи
- Тесты на Broken Access Control

**Секреты:**
- Vault хранит `DATABASE_URL`, `JWT_SECRET`
- При локальной разработке — fallback на `.env`
- Секреты НЕ хранятся в git, в docker-compose, в K8s манифестах

**Защита:**
- Security headers middleware
- Rate limiting на login endpoint
- gosec в CI pipeline
- trivy сканирование Docker образа

**Логирование:**
- Все события безопасности логируются (login, failed login, forbidden access)
- Структурированный JSON формат

Закоммить с тегом `v1.1.0`.

---

## Шпаргалка

| Концепция | Как защититься |
|-----------|---------------|
| SQL Injection | Параметризованные запросы `$1, $2` |
| Broken Access Control | Всегда проверять `task.UserID == claims.UserID` |
| Weak passwords | bcrypt, минимум 8 символов |
| Brute force | Redis счётчик + блокировка |
| Sensitive data exposure | `json:"-"` для полей, `-sensitive` в Terraform |
| Insecure headers | `X-Content-Type-Options`, `X-Frame-Options`, `HSTS` |
| JWT tampering | Всегда проверять подпись, не доверять payload без проверки |

---

## Ресурсы для изучения

- **OWASP Top 10:** `https://owasp.org/www-project-top-ten/`
- **JWT:** `https://jwt.io` — дебаггер токенов
- **golang-jwt:** `https://github.com/golang-jwt/jwt`
- **bcrypt:** `https://pkg.go.dev/golang.org/x/crypto/bcrypt`
- **Vault:** `https://developer.hashicorp.com/vault/docs`
- **gosec:** `https://github.com/securego/gosec`
- **Книга:** "Web Application Security" — Andrew Hoffman

---

## Как понять что урок пройден

- [ ] Регистрация и вход работают, возвращают JWT
- [ ] Защищённые эндпоинты возвращают 401 без токена
- [ ] Пользователь не может получить чужую задачу (403)
- [ ] Пароли хранятся как bcrypt хэш, не в открытом виде
- [ ] SQL запросы используют параметры, нет конкатенации
- [ ] Security headers на всех ответах
- [ ] Vault запущен, приложение читает секреты из него
- [ ] gosec не находит критичных проблем в коде
- [ ] Все события безопасности логируются

---

*Следующий урок: Производительность — профилирование, нагрузочное тестирование, оптимизация*
