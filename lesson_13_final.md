# Урок 13 — Финал: собираем всё вместе

## Куда ты пришёл

Если ты дошёл до этого урока и выполнил все задания — у тебя есть:

- Уверенная работа в Linux терминале
- Понимание сети и как работает интернет
- Git и профессиональный workflow с ветками и PR
- Go бэкенд с REST API, тестами и обработкой ошибок
- PostgreSQL — схема, миграции, индексы, транзакции
- Docker — контейнеризация, многоэтапная сборка, Compose
- CI/CD — автоматические тесты и деплой через GitHub Actions
- Мониторинг — Prometheus, Grafana, структурированные логи
- Kubernetes — деплой без даунтайма, масштабирование
- React фронтенд подключённый к API

Это и есть инженер-оркестр. Этот урок — не новая тема. Это финальная практика: реальные сценарии которые встречаются каждый день в работе.

---

## Часть 1 — Итоговая архитектура todo-service

К концу всех уроков проект должен выглядеть так:

```
todo-service/
├── .github/
│   └── workflows/
│       ├── ci-cd.yml
│       └── release.yml
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   └── components/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── package.json
├── k8s/
│   ├── namespace.yaml
│   ├── postgres/
│   └── app/
├── migrations/
│   ├── 001_create_tasks.up.sql
│   └── 001_create_tasks.down.sql
├── monitoring/
│   ├── prometheus.yml
│   ├── alerts.yml
│   └── grafana/
├── main.go
├── handler.go
├── storage.go
├── storage_test.go
├── Dockerfile
├── docker-compose.yml
├── Makefile
└── README.md
```

---

## Часть 2 — Реальные сценарии

Это задачи которые реально встречаются в работе. Они не имеют одного правильного ответа — важен процесс думать и отлаживать.

---

### Сценарий 1: «Сервис упал в продакшене»

Ты получаешь алерт: `ServiceDown` — сервис недоступен.

**Что делаешь:**

```bash
# 1. Смотришь статус подов
kubectl get pods -n production
# todo-app-xyz   0/1   CrashLoopBackOff   5   3m

# 2. Смотришь логи
kubectl logs todo-app-xyz -n production
kubectl logs todo-app-xyz -n production --previous

# 3. Смотришь события
kubectl describe pod todo-app-xyz -n production
# В Events видишь: OOMKilled

# 4. Временное решение: увеличить limits
kubectl set resources deployment todo-service \
  --limits=memory=512Mi -n production

# 5. Долгосрочное: найти утечку памяти в коде
# go tool pprof http://localhost:8080/debug/pprof/heap
```

**Задание:** Намеренно вызови OOMKilled — запусти контейнер с `--memory=10m` и нагрузочный тест. Научись читать события и находить причину.

---

### Сценарий 2: «Медленные запросы к базе данных»

Алерт: `SlowResponses` — P95 latency > 1 секунда.

```bash
# 1. Смотришь Grafana — медленно только POST /tasks

# 2. Подключаешься к БД и ищешь медленные запросы
kubectl exec -it postgres-pod -n production -- psql -U todouser -d tododb

-- Найти медленные запросы
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Смотришь план запроса
EXPLAIN ANALYZE SELECT * FROM tasks WHERE title ILIKE '%go%';
-- Seq Scan! Нужен индекс.

-- Создаёшь индекс без блокировки таблицы
CREATE INDEX CONCURRENTLY idx_tasks_title ON tasks(title);
```

**Задание:** Добавь в `todo-service` эндпоинт `GET /tasks?search=query`. Реализуй поиск через PostgreSQL. Измерь время до и после индекса через `EXPLAIN ANALYZE`.

---

### Сценарий 3: «Миграция без даунтайма»

Нужно добавить поле `tags TEXT[]` в таблицу `tasks`.

```sql
-- НЕЛЬЗЯ — блокирует таблицу
ALTER TABLE tasks ADD COLUMN tags TEXT[] NOT NULL DEFAULT '{}';

-- ПРАВИЛЬНО — три шага:

-- Шаг 1: добавить nullable (мгновенно, без блокировки)
ALTER TABLE tasks ADD COLUMN tags TEXT[];

-- Шаг 2: задеплоить код который умеет читать NULL tags
-- (пишет пустой массив, читает с fallback на [])

-- Шаг 3: заполнить NULL и добавить constraint
UPDATE tasks SET tags = '{}' WHERE tags IS NULL;
ALTER TABLE tasks ALTER COLUMN tags SET NOT NULL;
ALTER TABLE tasks ALTER COLUMN tags SET DEFAULT '{}';
```

**Задание:** Создай `migrations/002_add_tags.up.sql` и `down.sql`. Реализуй `POST /tasks/{id}/tags` и `GET /tasks?tag=go`.

---

### Сценарий 4: «Масштабирование под нагрузку»

```bash
# Нагрузочный тест
hey -n 5000 -c 50 http://localhost:8080/tasks

# Смотришь Grafana — CPU растёт, P95 > 500ms
# Масштабируешь
kubectl scale deployment todo-service --replicas=5 -n production
```

**HorizontalPodAutoscaler — автомасштабирование:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: todo-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Задание:** Настрой HPA. Запусти нагрузочный тест и наблюдай как K8s автоматически масштабирует приложение.

---

### Сценарий 5: «Инцидент — нужно откатиться»

```bash
# Задеплоил новую версию — рост ошибок в Grafana

# Немедленный откат
kubectl rollout undo deployment/todo-service -n production

# Убеждаешься что откатилось
kubectl rollout status deployment/todo-service -n production
```

**Шаблон постмортема:**

```markdown
## Инцидент: [дата] Рост ошибок после деплоя v0.8.0

### Хронология
- 14:00 Задеплоена версия v0.8.0
- 14:02 Алерт HighErrorRate — Error rate 15%
- 14:05 Принято решение откатиться
- 14:06 Выполнен rollback на v0.7.0
- 14:07 Error rate вернулся в норму

### Причина
Новый endpoint не обрабатывал пустой массив IDs — падал с panic.

### Что сделали
Добавили проверку на пустой массив и тест на этот кейс.

### Что изменим
- Добавить recover() middleware для перехвата panic
- Запускать smoke tests после деплоя
```

---

## Часть 3 — Финальный чеклист проекта

### Код и качество
- [ ] `go vet ./...` — нет ошибок
- [ ] `gofmt -l .` — нет неотформатированных файлов
- [ ] Покрытие тестами > 60%
- [ ] Все ошибки обрабатываются и логируются
- [ ] Нет `fmt.Println` в production коде — только `slog`
- [ ] Нет секретов в коде и git истории

### API
- [ ] Все эндпоинты возвращают правильные HTTP статусы
- [ ] Все ответы в JSON с полем `error` при ошибках
- [ ] CORS настроен
- [ ] `/health` и `/metrics` работают

### База данных
- [ ] Есть файлы миграций (up и down)
- [ ] Индексы на часто используемых колонках
- [ ] Используется connection pool
- [ ] Транзакции там где нужно

### Docker
- [ ] Многоэтапная сборка, образ < 30MB
- [ ] Приложение не запускается от root
- [ ] `HEALTHCHECK` настроен
- [ ] `docker compose up -d` поднимает всё с нуля

### CI/CD
- [ ] Тесты запускаются при каждом push
- [ ] Docker образ собирается и пушится автоматически
- [ ] Badge в README показывает статус CI
- [ ] Релиз создаётся автоматически по тегу `v*`

### Мониторинг
- [ ] Метрики: RPS, Error Rate, P95 Latency, Memory
- [ ] Grafana дашборд с этими метриками
- [ ] Логи структурированные (JSON)
- [ ] Минимум 3 правила алертов

### Kubernetes
- [ ] Все манифесты в `k8s/`
- [ ] Liveness и Readiness probes
- [ ] Resource limits на всех контейнерах
- [ ] Данные БД в PersistentVolumeClaim

### Фронтенд
- [ ] Список задач загружается из API
- [ ] CRUD операции работают
- [ ] Обработка ошибок и loading состояния
- [ ] Адаптивный дизайн (работает на телефоне)
- [ ] Собирается в Docker образ

---

## Часть 4 — Что дальше

### Следующие 3-6 месяцев

**Go — углубление:**
- Профилировщик `pprof` — найти утечку памяти
- Бенчмарки — `go test -bench`
- Middleware архитектура

**PostgreSQL — продвинутый уровень:**
- EXPLAIN ANALYZE — читать и понимать планы запросов
- Партиционирование таблиц
- Репликация (primary → replica)

**Kubernetes — глубже:**
- Helm — пакетный менеджер для K8s
- Network Policies — сетевая изоляция
- RBAC — разграничение прав
- Написать своего Operator

**Безопасность:**
- JWT аутентификация
- OWASP Top 10 — изучить и закрыть в своём проекте
- Сканирование образов (trivy)
- HashiCorp Vault для секретов

### 6-12 месяцев

**Распределённые системы:**
- Redis — кэш, сессии, pub/sub, distributed lock
- Kafka — очередь сообщений для асинхронного взаимодействия
- gRPC — альтернатива REST для внутренних сервисов

**Инфраструктура как код:**
- Terraform — создавать облачные ресурсы кодом
- Ansible — конфигурация серверов
- Облако — AWS/GCP/Yandex Cloud

**SRE практики:**
- SLI/SLO/SLA — договориться о надёжности
- Chaos Engineering — намеренно ломать систему
- Disaster Recovery — восстановление после катастрофы

### Читать каждую неделю

- `https://cloudflare.com/blog` — сеть, безопасность, производительность
- `https://discord.com/blog` — высокие нагрузки, Go
- `https://netflixtechblog.com` — Netflix Engineering
- `https://github.blog` — GitHub Engineering

### Книги

- **"Designing Data-Intensive Applications"** — Kleppmann — must-read
- **"Site Reliability Engineering"** — Google — бесплатно на `sre.google`
- **"The Go Programming Language"** — Donovan, Kernighan

---

## Часть 5 — Финальное задание

Это итоговая работа. Нет подсказок — только требования.

**Задача:** Добавить в `todo-service` систему уведомлений.

**Требования:**
1. Новый сервис `notification-service` на Go (отдельный репозиторий)
2. Когда задача помечается выполненной — `todo-service` отправляет событие
3. `notification-service` получает событие и логирует уведомление
4. Связь между сервисами через HTTP
5. Оба сервиса задеплоены в K8s в одном namespace
6. Оба сервиса в Grafana дашборде
7. CI/CD для обоих сервисов

**Формат сдачи:**
- Два репозитория на GitHub с README
- `docker compose up -d` поднимает оба сервиса
- Скриншот Grafana дашборда где видны оба сервиса

---
