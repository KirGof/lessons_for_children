# Урок 8 — Docker: контейнеризация приложений

## Зачем это нужно

"На моей машине работает" — классическая проблема разработки. Docker решает её раз и навсегда. Контейнер упаковывает приложение вместе со всеми зависимостями — одинаково запускается на ноутбуке разработчика, на CI сервере и в продакшене. Docker — фундамент современного DevOps. Без понимания Docker невозможно работать с Kubernetes, CI/CD и облаком.

---

## Часть 1 — Как Docker работает изнутри

### Контейнер vs Виртуальная машина

```
Виртуальная машина:           Контейнер:
┌─────────────────────┐       ┌─────────────────────┐
│   App A   │  App B  │       │   App A   │  App B  │
├───────────┼─────────┤       ├───────────┼─────────┤
│  Guest OS │ Guest OS│       │  Libs A   │ Libs B  │
├───────────┴─────────┤       ├─────────────────────┤
│    Hypervisor       │       │   Docker Engine     │
├─────────────────────┤       ├─────────────────────┤
│      Host OS        │       │      Host OS        │
└─────────────────────┘       └─────────────────────┘
```

VM запускает полноценную ОС — тяжело, медленно стартует, гигабайты памяти. Контейнер использует ядро хост-системы через namespaces и cgroups — лёгкий, запускается за секунды, мегабайты памяти.

### Namespaces и cgroups

Docker — это удобная обёртка над двумя механизмами Linux:

**Namespaces** — изоляция ресурсов. Каждый контейнер видит свою файловую систему, свою сеть, свои процессы. Процессы из разных контейнеров не видят друг друга.

**cgroups (Control Groups)** — ограничение ресурсов. Задаёшь контейнеру лимиты CPU и памяти — ядро следит чтобы не превысил.

### Образ и контейнер

- **Образ (Image)** — неизменяемый шаблон. Слои файловой системы. Хранится на диске.
- **Контейнер** — запущенный образ. Имеет своё состояние, записываемый слой сверху образа.

Один образ — много контейнеров. Как класс и объекты в программировании.

```
Image layers (read-only):
┌─────────────────────┐
│   App binary        │  ← твой слой
├─────────────────────┤
│   Config files      │  ← твой слой
├─────────────────────┤
│   Go runtime        │  ← базовый слой
├─────────────────────┤
│   Ubuntu/Alpine     │  ← базовый образ
└─────────────────────┘

Container (+ writable layer on top)
```

---

## Часть 2 — Установка и базовые команды

### Установка Docker

```bash
# Удалить старые версии если есть
sudo apt remove docker docker-engine docker.io containerd runc

# Установить зависимости
sudo apt update
sudo apt install ca-certificates curl gnupg -y

# Добавить официальный репозиторий Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

# Запустить и включить
sudo systemctl start docker
sudo systemctl enable docker

# Добавить пользователя в группу docker (чтобы не писать sudo)
sudo usermod -aG docker $USER
newgrp docker    # применить без перезагрузки

# Проверить
docker version
docker run hello-world
```

### Основные команды

```bash
# Образы
docker images                          # список образов
docker pull nginx                      # скачать образ
docker pull nginx:1.25                 # конкретная версия
docker rmi nginx                       # удалить образ
docker image prune                     # удалить неиспользуемые образы

# Контейнеры
docker run nginx                               # запустить (в foreground)
docker run -d nginx                            # запустить в фоне (detached)
docker run -d -p 8080:80 nginx                 # с маппингом портов
docker run -d -p 8080:80 --name my-nginx nginx # с именем
docker run -it ubuntu bash                     # интерактивно с терминалом

docker ps                              # запущенные контейнеры
docker ps -a                           # все контейнеры включая остановленные

docker stop my-nginx                   # остановить
docker start my-nginx                  # запустить остановленный
docker restart my-nginx                # перезапустить
docker rm my-nginx                     # удалить (должен быть остановлен)
docker rm -f my-nginx                  # принудительно удалить

# Логи и диагностика
docker logs my-nginx                   # логи контейнера
docker logs -f my-nginx                # следить за логами
docker logs --tail 100 my-nginx        # последние 100 строк

docker exec -it my-nginx bash          # войти в запущенный контейнер
docker exec my-nginx cat /etc/nginx/nginx.conf

docker inspect my-nginx                # подробная информация
docker stats                           # использование ресурсов в реальном времени
docker top my-nginx                    # процессы внутри контейнера
```

---

## Часть 3 — Dockerfile

### Базовый синтаксис

```dockerfile
# Комментарий

# Базовый образ
FROM ubuntu:22.04

# Метаданные
LABEL maintainer="your@email.com"
LABEL version="1.0"

# Запустить команду при сборке образа
RUN apt update && apt install -y curl

# Установить переменную окружения
ENV APP_PORT=8080
ENV LOG_LEVEL=info

# Скопировать файлы с хоста в образ
COPY . /app
COPY config.yaml /etc/app/config.yaml

# Добавить файлы (умеет распаковывать архивы и скачивать URL)
ADD app.tar.gz /app/

# Рабочая директория (аналог cd)
WORKDIR /app

# Порт который слушает приложение (документация, не открывает порт!)
EXPOSE 8080

# Точка входа — не перезаписывается при docker run
ENTRYPOINT ["./app"]

# Команда по умолчанию — перезаписывается при docker run
CMD ["--port=8080"]
```

### Dockerfile для Go приложения

```dockerfile
# Многоэтапная сборка (multi-stage build)

# Этап 1: Сборка
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Сначала копируем go.mod и go.sum для кэширования зависимостей
# Слои с зависимостями будут кэшироваться если go.mod не изменился
COPY go.mod go.sum ./
RUN go mod download

# Копируем исходный код
COPY . .

# Собираем бинарник
# CGO_ENABLED=0 — статическая сборка без C зависимостей
# -ldflags="-w -s" — уменьшить размер убрав отладочную информацию
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /todo-service .

# Этап 2: Финальный образ
FROM scratch
# или FROM alpine:3.19 — если нужны shell утилиты для отладки

# Копируем бинарник из этапа сборки
COPY --from=builder /todo-service /todo-service

# Копируем нужные файлы
COPY --from=builder /app/migrations /migrations

# Пользователь (не root — безопасность)
# scratch не поддерживает USER, используй alpine если нужен
USER 1000:1000

EXPOSE 8080

ENTRYPOINT ["/todo-service"]
```

```bash
# Собрать образ
docker build -t todo-service .
docker build -t todo-service:1.0 .
docker build -t todo-service:latest -f Dockerfile.prod .

# Запустить
docker run -d \
  -p 8080:8080 \
  -e DATABASE_URL="postgres://todouser:secret@host.docker.internal:5432/tododb" \
  -e PORT=8080 \
  --name todo-service \
  todo-service
```

### .dockerignore

```dockerignore
# Не копировать в образ при COPY . .
.git
.gitignore
*.md
Dockerfile*
docker-compose*
.env
.env.*
tmp/
vendor/
*.test
coverage.out
```

---

## Часть 4 — Docker Compose

Docker Compose позволяет описать и запустить несколько контейнеров как единое приложение.

### docker-compose.yml

```yaml
version: '3.9'

services:
  # База данных
  postgres:
    image: postgres:16-alpine
    container_name: todo-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: todouser
      POSTGRES_PASSWORD: secret123
      POSTGRES_DB: tododb
    ports:
      - "5432:5432"             # хост:контейнер
    volumes:
      - postgres_data:/var/lib/postgresql/data    # сохранять данные
      - ./migrations:/docker-entrypoint-initdb.d  # запустить при первом старте
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todouser -d tododb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis для кэша
  redis:
    image: redis:7-alpine
    container_name: todo-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Наше приложение
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: todo-app
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://todouser:secret123@postgres:5432/tododb
      REDIS_URL: redis://redis:6379
      PORT: 8080
      LOG_LEVEL: info
    depends_on:
      postgres:
        condition: service_healthy    # ждать пока postgres здоров
      redis:
        condition: service_started
    volumes:
      - ./migrations:/migrations    # только если применяем при старте

volumes:
  postgres_data:
  redis_data:
```

### Команды Docker Compose

```bash
# Запустить все сервисы
docker compose up -d

# Запустить и пересобрать образы
docker compose up -d --build

# Остановить
docker compose down

# Остановить и удалить данные (volumes)
docker compose down -v

# Посмотреть статус
docker compose ps

# Логи всех сервисов
docker compose logs -f

# Логи конкретного сервиса
docker compose logs -f app

# Выполнить команду в сервисе
docker compose exec postgres psql -U todouser -d tododb
docker compose exec app sh

# Перезапустить один сервис
docker compose restart app

# Пересобрать и перезапустить только app
docker compose up -d --build app
```

---

## Часть 5 — Сети и тома

### Сети

```bash
# Список сетей
docker network ls

# Создать сеть
docker network create my-network

# Запустить контейнер в сети
docker run -d --network my-network --name app1 nginx
docker run -d --network my-network --name app2 nginx

# Внутри одной сети контейнеры видят друг друга по имени
# app1 может обращаться к app2 как http://app2:80

# Подключить существующий контейнер к сети
docker network connect my-network existing-container

# Посмотреть детали сети
docker network inspect my-network
```

### Тома (Volumes)

```bash
# Создать том
docker volume create my-data

# Список томов
docker volume ls

# Смонтировать том в контейнер
docker run -d -v my-data:/app/data nginx

# Bind mount — монтировать директорию с хоста
docker run -d -v /home/user/data:/app/data nginx
docker run -d -v $(pwd):/app nginx    # текущая директория

# Разница:
# Volume — управляется Docker, хранится в /var/lib/docker/volumes/
# Bind mount — конкретная директория на хосте, удобна для разработки

# Удалить том
docker volume rm my-data
docker volume prune    # удалить неиспользуемые
```

---

## Часть 6 — Полезные паттерны

### Переменные окружения и секреты

```bash
# Передать переменные при запуске
docker run -e DB_PASSWORD=secret app

# Передать файл с переменными
docker run --env-file .env app

# В docker-compose — из файла
services:
  app:
    env_file:
      - .env
      - .env.production
```

### Healthcheck

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### Ограничение ресурсов

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'      # максимум 0.5 ядра
          memory: 256M     # максимум 256MB памяти
        reservations:
          cpus: '0.1'
          memory: 64M
```

---

## Практические задания

### Задание 8.1 — Первые шаги

```bash
# Запусти nginx и проверь что работает
docker run -d -p 8080:80 --name my-nginx nginx
curl http://localhost:8080

# Посмотри логи
docker logs my-nginx

# Войди внутрь контейнера
docker exec -it my-nginx bash
# Внутри: посмотри где лежит конфиг nginx, выйди exit

# Остановить и удалить
docker stop my-nginx && docker rm my-nginx
```

**Что нужно сделать:** Запусти PostgreSQL в Docker:
```bash
docker run -d \
  --name todo-postgres \
  -e POSTGRES_USER=todouser \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=tododb \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16-alpine
```
Подключись к нему через `psql` и создай таблицу `tasks`. Убедись что данные сохраняются после `docker restart todo-postgres`.

---

### Задание 8.2 — Dockerfile для todo-service

Напиши `Dockerfile` для `todo-service`:
1. Используй многоэтапную сборку
2. Финальный образ должен быть на `alpine` (не `scratch`) — понадобится для отладки
3. Бинарник должен запускаться как пользователь `1000` (не root)
4. Добавь `HEALTHCHECK` который проверяет `/health` эндпоинт

```bash
# Собери образ
docker build -t todo-service:local .

# Посмотри размер
docker images todo-service

# Запусти
docker run -d \
  -p 8080:8080 \
  -e DATABASE_URL="postgres://todouser:secret123@host.docker.internal:5432/tododb" \
  --name todo-service \
  todo-service:local

# Проверь
curl http://localhost:8080/health
curl http://localhost:8080/tasks
```

**Что нужно сделать:** Сравни размер образа с `FROM golang:1.22` и с многоэтапной сборкой. Объясни почему разница такая большая.

---

### Задание 8.3 — Docker Compose

Создай `docker-compose.yml` в корне `todo-service`:

1. Три сервиса: `postgres`, `app`, и `adminer` (веб-интерфейс для БД: `image: adminer`)
2. Приложение должно стартовать **только после** того как postgres здоров
3. Данные postgres должны сохраняться в volume
4. Все переменные окружения приложения должны читаться из `.env` файла

```bash
# Запустить всё
docker compose up -d --build

# Проверить
docker compose ps
docker compose logs -f app

# Открой http://localhost:8080/tasks
# Открой http://localhost:8888 — веб-интерфейс Adminer
```

**Что нужно сделать:** Добавь задачу через API, потом выполни `docker compose down` и снова `docker compose up -d`. Убедись что данные сохранились.

---

### Задание 8.4 — Отладка

Научись отлаживать контейнеры:

```bash
# Запусти контейнер который упадёт
docker run --name broken alpine sh -c "exit 1"

# Посмотри код выхода
docker inspect broken --format='{{.State.ExitCode}}'

# Запусти контейнер с ограничениями
docker run -d --name limited \
  --memory 50m \
  --cpus 0.1 \
  nginx

# Посмотри ресурсы
docker stats limited --no-stream
```

**Что нужно сделать:**
1. Намеренно сломай `todo-service` — передай неверный `DATABASE_URL`
2. Запусти контейнер и посмотри в логах почему он упал
3. Войди внутрь упавшего контейнера для диагностики:
   ```bash
   docker run -it --entrypoint sh todo-service:local
   ```
4. Исправь ошибку и убедись что сервис запускается

---

### Задание 8.5 — Финальный проект урока

Подготовь `todo-service` к "продакшену":

1. **Многоэтапный Dockerfile** — итоговый образ меньше 20MB
2. **docker-compose.yml** с сервисами: postgres, app
3. **Makefile** с командами:
   ```makefile
   build:      docker build -t todo-service:local .
   up:         docker compose up -d --build
   down:       docker compose down
   logs:       docker compose logs -f app
   test:       go test ./...
   migrate:    docker compose exec app ./migrate up
   ```
4. **README.md** с инструкцией:
   - Как запустить локально через Docker Compose
   - Переменные окружения
   - Доступные эндпоинты

5. Закоммить всё с тегом `v0.3.0`:
   ```bash
   git add .
   git commit -m "Add Docker support"
   git tag v0.3.0
   git push origin main --tags
   ```

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `docker build -t name .` | Собрать образ |
| `docker run -d -p 8080:80 name` | Запустить в фоне с портом |
| `docker ps` | Запущенные контейнеры |
| `docker logs -f name` | Следить за логами |
| `docker exec -it name sh` | Войти в контейнер |
| `docker stop name` | Остановить контейнер |
| `docker rm name` | Удалить контейнер |
| `docker compose up -d` | Запустить все сервисы |
| `docker compose down` | Остановить все сервисы |
| `docker compose logs -f` | Логи всех сервисов |
| `docker system prune` | Очистить всё неиспользуемое |

---

## Ресурсы для изучения

- **Официальная документация:** `https://docs.docker.com`
- **Play with Docker:** `https://labs.play-with-docker.com` — браузерная песочница
- **Docker Compose:** `https://docs.docker.com/compose/`
- **Best practices Dockerfile:** `https://docs.docker.com/develop/develop-images/dockerfile_best-practices/`
- **Русский курс:** Хекслет — "Введение в Docker"

---

## Как понять что урок пройден

- [ ] Понимаю разницу между образом и контейнером
- [ ] Понимаю зачем нужна многоэтапная сборка и использую её
- [ ] Написал Dockerfile для Go приложения
- [ ] Настроил Docker Compose с несколькими сервисами
- [ ] Данные PostgreSQL сохраняются в volume между перезапусками
- [ ] Умею читать логи и отлаживать упавшие контейнеры
- [ ] todo-service запускается одной командой: `docker compose up -d`

---

*Следующий урок: CI/CD — автоматизация сборки и деплоя*
