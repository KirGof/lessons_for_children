# Урок 9 — CI/CD: автоматизация сборки и деплоя

## Зачем это нужно

Каждый раз делать одно и то же вручную — источник ошибок и потери времени. CI/CD автоматизирует: запуск тестов при каждом коммите, сборку Docker образа, деплой на сервер. Если тесты упали — код не попадёт в продакшен. Если всё прошло — деплой происходит автоматически. Это стандарт в любой профессиональной команде.

**CI (Continuous Integration)** — автоматически проверять каждый коммит: тесты, линтер, сборка.
**CD (Continuous Delivery/Deployment)** — автоматически доставлять проверенный код на сервер.

---

## Часть 1 — GitHub Actions

GitHub Actions — встроенный CI/CD в GitHub. Бесплатно для публичных репозиториев, 2000 минут в месяц для приватных.

### Основные концепции

```
Workflow (файл .github/workflows/*.yml)
└── Trigger (on: push, pull_request, schedule...)
    └── Job (runs-on: ubuntu-latest)
        └── Step (uses: actions/checkout@v4)
        └── Step (run: go test ./...)
        └── Step (run: docker build ...)
```

- **Workflow** — автоматизированный процесс, описанный в YAML файле
- **Trigger** — событие которое запускает workflow (push, PR, расписание)
- **Job** — набор шагов, выполняется на отдельной машине (runner)
- **Step** — конкретная команда или готовое действие (action)
- **Runner** — виртуальная машина где выполняются шаги

### Первый workflow

Создай файл `.github/workflows/ci.yml`:

```yaml
name: CI

# Когда запускать
on:
  push:
    branches: [ main, dev ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest    # ОС раннера

    steps:
      # Шаг 1: Скачать код репозитория
      - name: Checkout code
        uses: actions/checkout@v4

      # Шаг 2: Установить Go
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true          # кэшировать зависимости

      # Шаг 3: Скачать зависимости
      - name: Download dependencies
        run: go mod download

      # Шаг 4: Запустить тесты
      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...

      # Шаг 5: Показать покрытие
      - name: Show coverage
        run: go tool cover -func=coverage.out
```

---

## Часть 2 — Полный CI pipeline

### Линтер и форматирование

```yaml
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      # golangci-lint — популярный линтер для Go
      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m
```

### Тесты с базой данных

```yaml
  test-with-db:
    name: Test with PostgreSQL
    runs-on: ubuntu-latest

    # Запустить PostgreSQL как сервис
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: todouser
          POSTGRES_PASSWORD: secret123
          POSTGRES_DB: tododb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run migrations
        run: psql $DATABASE_URL -f migrations/001_create_tasks.up.sql
        env:
          DATABASE_URL: postgres://todouser:secret123@localhost:5432/tododb

      - name: Run tests
        run: go test -v ./...
        env:
          DATABASE_URL: postgres://todouser:secret123@localhost:5432/tododb
```

### Сборка Docker образа

```yaml
  build:
    name: Build Docker image
    runs-on: ubuntu-latest
    needs: [test, lint]    # запустить только если test и lint прошли

    steps:
      - uses: actions/checkout@v4

      # Настроить Docker Buildx (для multi-platform builds)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Войти в GitHub Container Registry
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}    # автоматически доступен

      # Собрать и запушить образ
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}    # пушить только из main
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha    # кэш GitHub Actions
          cache-to: type=gha,mode=max
```

---

## Часть 3 — CD: деплой на сервер

### Деплой через SSH

```yaml
  deploy:
    name: Deploy to production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'    # только из main ветки

    environment:
      name: production
      url: https://todo.example.com

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/todo-service
            docker compose pull app
            docker compose up -d --no-deps app
            docker image prune -f
```

### Секреты GitHub

Никогда не храни пароли и ключи прямо в YAML файле. Используй GitHub Secrets:

1. Открой репозиторий на GitHub
2. Settings → Secrets and variables → Actions
3. New repository secret
4. Добавь: `SERVER_HOST`, `SERVER_USER`, `SSH_PRIVATE_KEY`, `DATABASE_URL`

```yaml
# Использование в workflow
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}

# Или в шаге
- name: Deploy
  env:
    DB_PASS: ${{ secrets.DB_PASSWORD }}
  run: ./deploy.sh
```

---

## Часть 4 — Полный workflow файл

`.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, dev ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ──────────────────────────────
  # JOB 1: Проверка кода
  # ──────────────────────────────
  check:
    name: Check code
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true

      - name: Verify go.mod is tidy
        run: |
          go mod tidy
          git diff --exit-code go.mod go.sum

      - name: Check formatting
        run: |
          if [ -n "$(gofmt -l .)" ]; then
            echo "Code is not formatted. Run: gofmt -w ."
            gofmt -l .
            exit 1
          fi

      - name: Run vet
        run: go vet ./...

  # ──────────────────────────────
  # JOB 2: Тесты
  # ──────────────────────────────
  test:
    name: Run tests
    runs-on: ubuntu-latest
    needs: check

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: todouser
          POSTGRES_PASSWORD: secret123
          POSTGRES_DB: tododb_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true

      - name: Run migrations
        run: psql $DATABASE_URL -f migrations/001_create_tasks.up.sql
        env:
          DATABASE_URL: postgres://todouser:secret123@localhost:5432/tododb_test

      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...
        env:
          DATABASE_URL: postgres://todouser:secret123@localhost:5432/tododb_test

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.out

  # ──────────────────────────────
  # JOB 3: Сборка образа
  # ──────────────────────────────
  build:
    name: Build and push image
    runs-on: ubuntu-latest
    needs: test
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ──────────────────────────────
  # JOB 4: Деплой (только main)
  # ──────────────────────────────
  deploy:
    name: Deploy to production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    environment:
      name: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            set -e
            cd /opt/todo-service

            # Обновить docker-compose.yml
            git pull origin main

            # Скачать новый образ и перезапустить
            export IMAGE_TAG=sha-${{ github.sha }}
            docker compose pull app
            docker compose up -d --no-deps app

            # Проверить что сервис поднялся
            sleep 5
            curl -f http://localhost:8080/health || exit 1

            # Очистить старые образы
            docker image prune -f

            echo "Deploy successful!"
```

---

## Часть 5 — Makefile для локальной разработки

```makefile
.PHONY: build test lint run up down logs migrate clean

# Переменные
BINARY=todo-service
IMAGE=todo-service:local
COMPOSE=docker compose

## Сборка
build:
	go build -ldflags="-w -s" -o $(BINARY) .

build-docker:
	docker build -t $(IMAGE) .

## Тесты
test:
	go test -v -race ./...

test-cover:
	go test -v -race -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html
	@echo "Open coverage.html in browser"

## Качество кода
lint:
	golangci-lint run ./...

fmt:
	gofmt -w .
	goimports -w .

vet:
	go vet ./...

## Docker Compose
up:
	$(COMPOSE) up -d --build

down:
	$(COMPOSE) down

logs:
	$(COMPOSE) logs -f app

restart:
	$(COMPOSE) restart app

## База данных
migrate-up:
	$(COMPOSE) exec app psql $$DATABASE_URL -f migrations/001_create_tasks.up.sql

migrate-down:
	$(COMPOSE) exec postgres psql -U todouser -d tododb \
		-f /migrations/001_create_tasks.down.sql

## Утилиты
clean:
	rm -f $(BINARY) coverage.out coverage.html
	$(COMPOSE) down -v
	docker image prune -f

help:
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'
```

---

## Практические задания

### Задание 9.1 — Первый CI pipeline

В репозитории `todo-service` создай `.github/workflows/ci.yml`:

1. Запускается при push в любую ветку и при PR в main
2. Шаги: checkout → setup-go → go mod download → go vet → go test
3. Запушь в GitHub и убедись что workflow запустился
4. Найди результат в разделе "Actions" репозитория

**Что нужно сделать:** Намеренно сломай тест (измени ожидаемое значение) и запушь. Убедись что CI становится красным. Исправь и запушь снова.

---

### Задание 9.2 — Тесты с PostgreSQL

Расширь workflow:

1. Добавь сервис PostgreSQL
2. Примени миграции перед тестами
3. Передай `DATABASE_URL` в шаг тестов
4. Добавь шаг `go test -race` — детектор гонок данных
5. Загружай `coverage.out` как артефакт

Напиши хотя бы один интеграционный тест который работает с реальной базой:

```go
// storage_integration_test.go
//go:build integration

package main

import (
    "context"
    "os"
    "testing"
)

func TestPostgresStorage_AddAndGet(t *testing.T) {
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        t.Skip("DATABASE_URL not set")
    }

    // ... тест с реальной БД
}
```

---

### Задание 9.3 — Сборка Docker образа

Добавь в workflow job для сборки образа:

1. Настрой вход в `ghcr.io` (GitHub Container Registry)
2. Собирай образ при каждом push в main
3. Тегируй образ: `latest` + SHA коммита
4. Используй кэш GitHub Actions для ускорения сборки

После успешного пуша найди образ в разделе "Packages" репозитория на GitHub.

---

### Задание 9.4 — Environments и защита

1. Создай environment `production` в Settings → Environments
2. Добавь правило: требовать ручного подтверждения перед деплоем
3. Добавь секреты в environment (не в репозиторий)
4. Обнови workflow чтобы deploy job использовал этот environment

```yaml
deploy:
  environment:
    name: production
    url: http://your-server-ip:8080
```

---

### Задание 9.5 — Финальный проект урока

Собери полный CI/CD pipeline для `todo-service`:

**CI (при каждом push/PR):**
- Проверка форматирования кода (`gofmt`)
- `go vet`
- Тесты с PostgreSQL
- Покрытие тестами > 60% (иначе падать)

**CD (только при push в main):**
- Сборка Docker образа
- Push в `ghcr.io`
- Деплой на сервер (можно имитировать — просто echo команды если нет сервера)

**Дополнительно:**
- Badge в README показывающий статус CI:
  ```markdown
  ![CI](https://github.com/username/todo-service/actions/workflows/ci-cd.yml/badge.svg)
  ```
- Workflow для автоматического создания релиза при теге `v*`:

```yaml
name: Release

on:
  push:
    tags: [ 'v*' ]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
```

Закоммить с тегом `v0.4.0` и убедись что создался автоматический релиз на GitHub.

---

## Шпаргалка

| Концепция | Описание |
|-----------|----------|
| `on: push` | Запускать при push |
| `on: pull_request` | Запускать при PR |
| `runs-on: ubuntu-latest` | ОС раннера |
| `needs: [job1, job2]` | Зависимость от других jobs |
| `if: github.ref == 'refs/heads/main'` | Условие выполнения |
| `uses: actions/checkout@v4` | Готовое действие |
| `run: go test ./...` | Произвольная команда |
| `${{ secrets.MY_SECRET }}` | Использовать секрет |
| `${{ github.sha }}` | SHA текущего коммита |
| `services:` | Запустить сопутствующие контейнеры |

---

## Ресурсы для изучения

- **Документация:** `https://docs.github.com/en/actions`
- **Готовые actions:** `https://github.com/marketplace?type=actions`
- **Примеры для Go:** `https://github.com/actions/setup-go`
- **golangci-lint:** `https://golangci-lint.run`
- **Контекст и переменные:** `https://docs.github.com/en/actions/learn-github-actions/contexts`

---

## Как понять что урок пройден

- [ ] Понимаю разницу между CI и CD
- [ ] Написал workflow который запускает тесты при каждом push
- [ ] Тесты используют реальный PostgreSQL через services
- [ ] Docker образ собирается и пушится в registry автоматически
- [ ] Секреты хранятся в GitHub Secrets, не в коде
- [ ] README содержит badge со статусом CI
- [ ] При теге `v*` автоматически создаётся релиз

---

*Следующий урок: Мониторинг — Prometheus, Grafana и логирование*
