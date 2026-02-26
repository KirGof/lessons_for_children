# Урок 19 — SRE: SLI/SLO, Chaos Engineering, Disaster Recovery

## Зачем это нужно

До этого урока ты строил системы. Теперь научишься отвечать на вопрос: насколько надёжна эта система и что делать когда она сломается? SRE (Site Reliability Engineering) — это дисциплина которая превращает эксплуатацию в инженерную задачу. Вместо "работает — и ладно" появляется точное измерение надёжности, заранее прописанные действия при инцидентах и регулярные тренировки отказов. Именно это отличает Junior-эксплуатацию от Senior-инфраструктуры.

---

## Часть 1 — SLI, SLO, SLA

### Три уровня надёжности

**SLI (Service Level Indicator)** — конкретная измеримая метрика:
- Процент успешных HTTP запросов
- P99 время ответа
- Процент времени доступности

**SLO (Service Level Objective)** — целевое значение SLI которое команда обязуется поддерживать:
- 99.9% запросов завершаются успешно
- P99 latency < 500ms
- Доступность 99.5% за 30 дней

**SLA (Service Level Agreement)** — юридический договор с пользователями/клиентами:
- При нарушении SLA — компенсация
- SLA всегда мягче SLO (нарушить SLO ещё не значит нарушить SLA)

### Error Budget

```
SLO: 99.9% доступность за 30 дней

Всего минут за 30 дней: 30 * 24 * 60 = 43 200 минут
Допустимый даунтайм: 43 200 * 0.001 = 43.2 минуты

Error Budget = 43.2 минуты в месяц

Использование:
- Инцидент 15 минут → осталось 28.2 минуты
- Плановое обновление 10 минут → осталось 18.2 минуты
- Ещё один инцидент 20 минут → БЮДЖЕТ ИСЧЕРПАН
```

Когда error budget исчерпан — стоп новым фичам, всё внимание на надёжность.

### Определение SLI для todo-service

```go
// Правильные SLI для HTTP API:

// 1. Availability — доступность
// SLI = количество успешных запросов / всего запросов * 100
// Успешный = HTTP 2xx или 3xx (не 5xx)

// 2. Latency — задержка
// SLI = процент запросов выполненных быстрее порога
// "99% запросов выполняются за < 200ms"

// 3. Throughput — пропускная способность
// SLI = запросов в секунду
```

### SLO в Prometheus и Grafana

```yaml
# prometheus/rules/slo.yml
groups:
  - name: slo.rules
    interval: 1m
    rules:
      # Availability SLI — процент успешных запросов
      - record: sli:http_availability:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))

      # Latency SLI — процент запросов быстрее 200ms
      - record: sli:http_latency_200ms:ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m]))
          /
          sum(rate(http_request_duration_seconds_count[5m]))

      # Error Budget — сколько осталось (за 30 дней)
      - record: slo:error_budget_remaining:ratio
        expr: |
          1 - (
            (1 - sli:http_availability:ratio_rate5m)
            /
            (1 - 0.999)    # SLO = 99.9%
          )

  - name: slo.alerts
    rules:
      # Алерт: SLO нарушается сейчас
      - alert: SLOAvailabilityBreach
        expr: sli:http_availability:ratio_rate5m < 0.999
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Availability SLO breach"
          description: "Current availability {{ $value | humanizePercentage }}, SLO is 99.9%"

      # Алерт: error budget быстро сгорает
      - alert: ErrorBudgetBurnRateFast
        expr: |
          (1 - sli:http_availability:ratio_rate5m) > 14.4 * (1 - 0.999)
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High error budget burn rate"
          description: "Burning error budget 14.4x faster than allowed"

      # Алерт: медленные ответы
      - alert: SLOLatencyBreach
        expr: sli:http_latency_200ms:ratio_rate5m < 0.99
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Latency SLO breach"
```

### Grafana дашборд для SLO

Создай дашборд с панелями:

```
┌─────────────────┬─────────────────┬─────────────────┐
│  Availability   │ Latency P99     │  Error Budget   │
│  99.97%  ✅     │  45ms    ✅     │  87% remaining  │
│  SLO: 99.9%     │  SLO: <200ms   │  30-day window  │
├─────────────────┴─────────────────┴─────────────────┤
│              Availability over time                  │
│  1.0 ─────────────────────────────────────          │
│  0.999 ── SLO line ──────────────────────           │
│  0.99 ────────────────────────────────────          │
└──────────────────────────────────────────────────────┘
```

---

## Часть 2 — Runbooks

Runbook — пошаговая инструкция что делать при конкретном инциденте. Пишется заранее, в спокойной обстановке. Используется под давлением, когда нет времени думать.

### Структура Runbook

```markdown
# Runbook: ServiceDown (todo-service недоступен)

## Симптомы
- Алерт ServiceDown в Slack/PagerDuty
- curl http://todo.example.com/health возвращает ошибку
- Пользователи сообщают о недоступности

## Первые 5 минут (Mitigation)

### 1. Оценить масштаб
```bash
kubectl get pods -n production
kubectl get events -n production --sort-by='.lastTimestamp' | tail -20
```

### 2. Проверить логи
```bash
kubectl logs -l app=todo-service -n production --tail=100
kubectl logs -l app=todo-service -n production --previous --tail=50
```

### 3. Быстрые действия
```bash
# Если CrashLoopBackOff — попробовать рестарт
kubectl rollout restart deployment/todo-service -n production

# Если OOMKilled — временно увеличить память
kubectl set resources deployment/todo-service \
  --limits=memory=512Mi -n production

# Если проблема после деплоя — откатить
kubectl rollout undo deployment/todo-service -n production
```

## Диагностика (если быстрые действия не помогли)

### База данных недоступна?
```bash
kubectl exec -it <app-pod> -n production -- \
  psql $DATABASE_URL -c "SELECT 1"
```

### Redis недоступен?
```bash
kubectl exec -it <app-pod> -n production -- \
  redis-cli -u $REDIS_URL ping
```

### Проблема с сетью?
```bash
kubectl exec -it <app-pod> -n production -- \
  curl -v http://postgres:5432
```

## Эскалация
Если не решено за 30 минут:
- Уведомить: tech-lead@company.com
- Канал: #incidents в Slack
- Дежурный: PagerDuty escalation policy

## После решения
1. Написать postmortem в течение 48 часов
2. Создать задачи для устранения root cause
3. Обновить этот runbook если нашли новый сценарий
```

### Каталог Runbooks для todo-service

```
runbooks/
├── service-down.md
├── high-error-rate.md
├── slow-responses.md
├── database-connection-exhausted.md
├── disk-space-full.md
├── memory-leak.md
└── rollback-procedure.md
```

---

## Часть 3 — Chaos Engineering

### Что такое Chaos Engineering

Chaos Engineering — намеренное внесение отказов в систему чтобы проверить как она реагирует. Лучше обнаружить слабое место на нагрузочном тесте, чем в 3 ночи в продакшене.

**Принцип:** систематически проверять гипотезы о поведении системы при отказах.

```
1. Определить "нормальное" состояние системы (steady state)
2. Выдвинуть гипотезу: "если упадёт Redis — сервис продолжит работать"
3. Внести отказ
4. Наблюдать отклонение от steady state
5. Исправить если гипотеза не подтвердилась
```

### Ручные эксперименты

```bash
# Эксперимент 1: Что случится если Redis упадёт?
docker compose stop redis
# Проверить: отвечает ли API?
curl http://localhost:8080/tasks
# Ожидаем: работает (деградировано — без кэша, но работает)
# Проверить: не теряем ли запросы?
hey -n 100 http://localhost:8080/tasks
docker compose start redis
# Ожидаем: восстановление без перезапуска приложения

# Эксперимент 2: Что случится если PostgreSQL недоступен?
docker compose stop postgres
curl http://localhost:8080/tasks
# Ожидаем: 503 Service Unavailable с понятным сообщением
# НЕ ожидаем: panic, висячий запрос без timeout
docker compose start postgres
# Ожидаем: автоматическое восстановление connection pool

# Эксперимент 3: OOM — что если памяти не хватает?
docker update --memory=50m todo-service
hey -n 10000 -c 100 http://localhost:8080/tasks
# Ожидаем: контейнер упадёт, Kubernetes перезапустит
# НЕ ожидаем: зависание, неотвеченные запросы без таймаута

# Эксперимент 4: Медленная сеть
# Добавить задержку к исходящим соединениям
kubectl exec -it <app-pod> -- tc qdisc add dev eth0 root netem delay 200ms
# Проверить что timeout'ы срабатывают правильно
```

### Chaos Monkey в Kubernetes

```bash
# kube-monkey — случайно убивает Pod в рабочее время
kubectl apply -f https://raw.githubusercontent.com/asobti/kube-monkey/master/examples/example_km_config.yaml

# Конфигурация какие Deployment атаковать
# (нужно добавить labels к Deployment)
labels:
  kube-monkey/enabled: "enabled"
  kube-monkey/mtbf: "2"           # среднее время между убийствами (часы)
  kube-monkey/kill-mode: "fixed"
  kube-monkey/kill-value: "1"     # убивать 1 Pod за раз
```

### Litmus Chaos — фреймворк для K8s

```bash
# Установить Litmus
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v3.0.0.yaml

# Эксперимент: убить pod
cat <<EOF | kubectl apply -f -
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: todo-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=todo-service"
    appkind: deployment
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"    # 60 секунд хаоса
            - name: CHAOS_INTERVAL
              value: "10"    # убивать каждые 10 секунд
            - name: FORCE
              value: "false"
EOF

# Смотреть что происходит
kubectl get pods -n production -w
# Grafana должна показывать что SLO не нарушается
```

### Что проверять во время Chaos экспериментов

```
✅ Liveness probe перезапускает упавший Pod
✅ Readiness probe не пропускает трафик к нездоровому Pod
✅ Rolling update не вызывает даунтайма
✅ При падении Redis — API работает в деградированном режиме
✅ При падении одной реплики — остальные принимают трафик
✅ Timeouts срабатывают (не висим вечно)
✅ Circuit breaker срабатывает при cascade failures
✅ Метрики показывают аномалии
✅ Алерты срабатывают в нужный момент
```

---

## Часть 4 — Disaster Recovery

### RPO и RTO

**RPO (Recovery Point Objective)** — максимально допустимая потеря данных.
- RPO = 0 — ноль потерь данных (очень дорого)
- RPO = 1 час — допускаем потерю до 1 часа данных

**RTO (Recovery Time Objective)** — максимально допустимое время восстановления.
- RTO = 15 минут — должны восстановиться за 15 минут
- RTO = 4 часа — допускаем 4 часа даунтайма

```
Стоимость ↑         RPO/RTO ↓ (лучше)
─────────────────────────────────────────
$$$$ — Active-Active: нет даунтайма, нет потерь данных
$$$  — Active-Passive: минуты RTO, секунды RPO
$$   — Backup + Restore: часы RTO, часы RPO
$    — Ничего: неизвестно RTO и RPO
```

### Бэкапы PostgreSQL

```bash
# pg_dump — логический бэкап
pg_dump -U todouser -d tododb > backup_$(date +%Y%m%d_%H%M%S).sql

# Сжатый
pg_dump -U todouser -d tododb | gzip > backup_$(date +%Y%m%d).sql.gz

# Только схема (без данных)
pg_dump -U todouser -d tododb --schema-only > schema.sql

# Только данные (без схемы)
pg_dump -U todouser -d tododb --data-only > data.sql

# Восстановление
psql -U todouser -d tododb < backup.sql
gunzip -c backup.sql.gz | psql -U todouser -d tododb

# pg_basebackup — физический бэкап (быстрее для больших БД)
pg_basebackup -U replicauser -h localhost -D /backup/base -P -Xs -R
```

### Автоматические бэкапы через CronJob в K8s

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"    # каждый день в 2:00
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16-alpine
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  FILENAME="backup_$(date +%Y%m%d_%H%M%S).sql.gz"
                  echo "Starting backup: $FILENAME"

                  pg_dump -U todouser -h postgres tododb | \
                    gzip > /tmp/$FILENAME

                  # Загрузить в S3/Object Storage
                  # aws s3 cp /tmp/$FILENAME s3://my-backups/postgres/
                  # или через curl в Yandex Object Storage

                  echo "Backup completed: $FILENAME"
                  ls -lh /tmp/$FILENAME
```

### Тестирование восстановления

```bash
# Тест DR — делай это регулярно (раз в месяц)

# 1. Остановить prod-подобное окружение
kubectl scale deployment todo-service --replicas=0 -n staging
kubectl exec -it postgres-pod -n staging -- \
  psql -U todouser -c "DROP DATABASE tododb;"

# 2. Восстановить из бэкапа
kubectl exec -it postgres-pod -n staging -- \
  psql -U todouser -c "CREATE DATABASE tododb;"

# Скачать последний бэкап
kubectl exec -it postgres-pod -n staging -- \
  sh -c "gunzip -c /backups/latest.sql.gz | psql -U todouser tododb"

# 3. Запустить приложение
kubectl scale deployment todo-service --replicas=2 -n staging

# 4. Проверить что всё работает
curl http://staging.todo.example.com/health
curl http://staging.todo.example.com/tasks

# 5. Записать результат: сколько времени заняло
echo "DR Test: $(date), восстановление за 23 минуты" >> dr-log.txt
```

---

## Практические задания

### Задание 19.1 — Определить SLO

Для `todo-service` определи и задокументируй SLO:

Создай файл `SLO.md`:
```markdown
# Service Level Objectives — todo-service

## Availability SLO
- Target: 99.5% за скользящие 30 дней
- SLI: (успешные запросы / все запросы) * 100
- Успешный запрос: HTTP 2xx или 4xx (не 5xx)
- Error Budget: ... минут в месяц

## Latency SLO
- Target: 99% запросов за < 300ms
- Измеряется на P99

## Error Budget Policy
- Если >50% бюджета использовано — review всех деплоев
- Если бюджет исчерпан — мораторий на новые фичи
```

Реализуй в Prometheus:
1. Запись SLI через `record` rules
2. Алерт при нарушении SLO
3. Алерт при burn rate > 5x

---

### Задание 19.2 — Runbook

Напиши два Runbook в `runbooks/`:

**`runbooks/high-error-rate.md`:**
- Симптомы
- Первые действия (< 5 минут)
- Диагностика (что проверить)
- Частые причины и их решения
- Эскалация

**`runbooks/slow-responses.md`:**
- Как понять что проблема в БД vs приложение vs Redis
- SQL запросы для диагностики PostgreSQL
- Команды для проверки Redis
- Когда масштабировать горизонтально

---

### Задание 19.3 — Chaos Experiments

Проведи 4 эксперимента и задокументируй результаты:

```markdown
# Chaos Experiment: Redis Failure

## Гипотеза
При падении Redis API продолжит работать без кэша.
Время восстановления < 30 секунд после старта Redis.

## Методология
1. Запустить hey -n 1000 -c 10 в фоне (следить за error rate)
2. docker compose stop redis
3. Продолжать нагрузку 60 секунд
4. docker compose start redis
5. Продолжать нагрузку ещё 60 секунд

## Наблюдения
- Error rate во время падения: 0% (работает с деградацией)
- Latency выросла с 8ms до 45ms (нет кэша)
- Время восстановления: 5 секунд

## Результат: ✅ Гипотеза подтверждена

## Action Items
- Добавить метрику redis_errors_total
- Алерт если Redis недоступен > 1 минуты
```

---

### Задание 19.4 — Бэкапы

Настрой автоматические бэкапы:

1. Напиши скрипт `scripts/backup.sh`:
   ```bash
   #!/bin/bash
   FILENAME="backup_$(date +%Y%m%d_%H%M%S).sql.gz"
   pg_dump -U todouser -h localhost tododb | gzip > /backups/$FILENAME
   echo "Backup: $FILENAME ($(du -h /backups/$FILENAME | cut -f1))"
   # Удалить бэкапы старше 7 дней
   find /backups -name "*.sql.gz" -mtime +7 -delete
   ```

2. Добавь CronJob в K8s который запускает бэкап каждые 6 часов
3. Проверь что бэкап создаётся: `kubectl logs -l job=postgres-backup`
4. Проведи тест восстановления:
   - Добавь несколько задач
   - Сделай бэкап
   - Удали все задачи из БД напрямую
   - Восстанови из бэкапа
   - Убедись что задачи вернулись

---

### Задание 19.5 — Финальный проект урока

Полный SRE пакет для `todo-service`:

**`SLO.md`** — задокументированные SLO с error budget policy

**`runbooks/`** — 5 runbooks для основных инцидентов

**Prometheus rules:**
- SLI recording rules
- Error budget burn rate alerts
- Multi-window multi-burn-rate alerts (Google SRE book подход)

**Chaos testing:**
- Скрипт `chaos/run-experiments.sh` который проводит базовые эксперименты
- Документация результатов каждого эксперимента

**Disaster Recovery:**
- CronJob для бэкапов каждые 6 часов
- Скрипт `scripts/restore.sh` для восстановления
- Задокументированный DR план с RTO/RPO целями

**`DR-TEST-LOG.md`** — результаты тестирования восстановления с временными метками

Закоммить с тегом `v1.3.0`.

---

## Шпаргалка

| Термин | Определение |
|--------|------------|
| SLI | Измеримая метрика качества (% успешных запросов) |
| SLO | Целевое значение SLI (99.9%) |
| SLA | Договор с пользователями |
| Error Budget | Сколько можно "нарушать" SLO |
| RPO | Максимальная потеря данных |
| RTO | Максимальное время восстановления |
| Chaos Engineering | Намеренные отказы для поиска слабых мест |
| Runbook | Инструкция для инцидента |
| Postmortem | Анализ инцидента после устранения |

---

## Ресурсы для изучения

- **Google SRE Book:** `https://sre.google/sre-book/` — бесплатно, это библия SRE
- **SRE Workbook:** `https://sre.google/workbook/` — практические примеры
- **Chaos Engineering:** `https://principlesofchaos.org/`
- **Litmus Chaos:** `https://litmuschaos.io/`
- **Runbook шаблоны:** `https://github.com/nicholasjackson/consul-vault-monitoring`
- **Error Budget Policy:** `https://sre.google/workbook/error-budget-policy/`

---

## Как понять что урок пройден

- [ ] Понимаю разницу между SLI, SLO, SLA и Error Budget
- [ ] Определил SLO для `todo-service` с конкретными числами
- [ ] Prometheus считает SLI и Error Budget в реальном времени
- [ ] Написал минимум 2 Runbook для реальных сценариев
- [ ] Провёл и задокументировал Chaos эксперимент
- [ ] Настроен автоматический бэкап PostgreSQL
- [ ] Провёл тест восстановления из бэкапа
- [ ] Знаю RTO и RPO своей системы

---

*Следующий урок: Архитектурные паттерны — микросервисы, Event-Driven Architecture, CQRS*
