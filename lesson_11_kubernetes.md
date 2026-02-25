# Урок 11 — Kubernetes: оркестрация контейнеров

## Зачем это нужно

Docker Compose хорош для одного сервера. Но что если нужно: запустить приложение на 10 серверах, автоматически перезапустить его если оно упало, обновить без даунтайма, масштабировать под нагрузку? Для этого существует Kubernetes (K8s). Это стандарт индустрии для запуска контейнеров в продакшене. Docker, Terraform, Prometheus — всё это инструменты. Kubernetes — это платформа которая их объединяет.

---

## Часть 1 — Архитектура Kubernetes

### Кластер

```
┌─────────────────────────────────────────────────────┐
│                    K8s Cluster                      │
│                                                     │
│  ┌──────────────────┐    ┌──────────────────────┐   │
│  │   Control Plane  │    │     Worker Nodes      │   │
│  │                  │    │                       │   │
│  │  ┌────────────┐  │    │  ┌────────┐           │   │
│  │  │ API Server │  │    │  │  Node 1│           │   │
│  │  ├────────────┤  │    │  │ ┌─Pod─┐│           │   │
│  │  │   etcd     │  │    │  │ │ App ││           │   │
│  │  ├────────────┤  │    │  │ └─────┘│           │   │
│  │  │ Scheduler  │  │    │  └────────┘           │   │
│  │  ├────────────┤  │    │  ┌────────┐           │   │
│  │  │ Controller │  │    │  │  Node 2│           │   │
│  │  │  Manager   │  │    │  │ ┌─Pod─┐│           │   │
│  │  └────────────┘  │    │  │ │ App ││           │   │
│  └──────────────────┘    │  │ └─────┘│           │   │
│                           │  └────────┘           │   │
└─────────────────────────────────────────────────────┘
```

**Control Plane** — мозг кластера:
- **API Server** — единственная точка входа, все команды идут через него
- **etcd** — база данных кластера, хранит всё состояние
- **Scheduler** — решает на какой node запустить Pod
- **Controller Manager** — следит чтобы реальное состояние = желаемому

**Worker Node** — рабочие лошади:
- **kubelet** — агент на каждой node, запускает контейнеры
- **kube-proxy** — сетевые правила, маршрутизация трафика
- **Container Runtime** — Docker или containerd

### Основные объекты

```
Pod           — минимальная единица. Один или несколько контейнеров.
Deployment    — управляет набором одинаковых Pod. Обновления, откаты.
Service       — стабильный IP/DNS для набора Pod. Балансировка нагрузки.
ConfigMap     — конфигурация (не секреты) для Pod.
Secret        — секреты (пароли, ключи) для Pod.
Ingress       — HTTP маршрутизация снаружи кластера.
PersistentVolume — хранилище данных.
Namespace     — логическое разделение кластера.
```

---

## Часть 2 — Установка локального кластера

### minikube — для обучения

```bash
# Установить minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Запустить кластер
minikube start --driver=docker
minikube start --cpus=2 --memory=4096

# Статус
minikube status

# Остановить
minikube stop

# Удалить
minikube delete
```

### kubectl — CLI для Kubernetes

```bash
# Установить kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# Настроить автодополнение (добавить в ~/.bashrc)
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k

# Базовые команды
kubectl version
kubectl cluster-info
kubectl get nodes
kubectl get all
```

---

## Часть 3 — Основные объекты

### Pod

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: todo-pod
  labels:
    app: todo-service
spec:
  containers:
    - name: todo-service
      image: ghcr.io/username/todo-service:latest
      ports:
        - containerPort: 8080
      env:
        - name: PORT
          value: "8080"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: todo-secrets
              key: database-url
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 20
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod todo-pod
kubectl logs todo-pod
kubectl logs -f todo-pod          # следить за логами
kubectl exec -it todo-pod -- sh   # войти в контейнер
kubectl delete pod todo-pod
```

### Deployment

Deployment управляет ReplicaSet который управляет Pod. Обеспечивает нужное количество запущенных Pod.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-service
  labels:
    app: todo-service
spec:
  replicas: 3                     # сколько Pod держать запущенными
  selector:
    matchLabels:
      app: todo-service
  strategy:
    type: RollingUpdate           # обновлять постепенно
    rollingUpdate:
      maxSurge: 1                 # максимум +1 pod при обновлении
      maxUnavailable: 0           # минимум 0 недоступных
  template:                       # шаблон для Pod
    metadata:
      labels:
        app: todo-service
    spec:
      containers:
        - name: todo-service
          image: ghcr.io/username/todo-service:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: todo-config
            - secretRef:
                name: todo-secrets
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

```bash
kubectl apply -f deployment.yaml

# Посмотреть статус
kubectl get deployments
kubectl get pods -l app=todo-service

# Масштабировать
kubectl scale deployment todo-service --replicas=5

# Обновить образ
kubectl set image deployment/todo-service todo-service=ghcr.io/username/todo-service:v2.0

# История обновлений
kubectl rollout history deployment/todo-service

# Откатить
kubectl rollout undo deployment/todo-service
kubectl rollout undo deployment/todo-service --to-revision=2

# Статус обновления
kubectl rollout status deployment/todo-service
```

### Service

Service даёт стабильный IP и DNS для набора Pod. Pod могут создаваться и удаляться, Service всегда знает где они.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-service
spec:
  selector:
    app: todo-service             # находить Pod с этим label
  ports:
    - port: 80                    # порт Service
      targetPort: 8080            # порт контейнера
  type: ClusterIP                 # только внутри кластера
```

Типы Service:
```yaml
type: ClusterIP    # только внутри кластера (по умолчанию)
type: NodePort     # открыть порт на каждой node (для разработки)
type: LoadBalancer # облачный балансировщик (AWS/GCP/Azure)
```

```bash
kubectl apply -f service.yaml
kubectl get services
kubectl describe service todo-service

# Временно открыть сервис локально (для разработки)
kubectl port-forward service/todo-service 8080:80
kubectl port-forward pod/todo-pod-xxx 8080:8080
```

### ConfigMap и Secret

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-config
data:
  PORT: "8080"
  LOG_LEVEL: "info"
  STORAGE_TYPE: "postgres"
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: todo-secrets
type: Opaque
stringData:                       # автоматически base64 кодируется
  database-url: "postgres://todouser:secret123@postgres:5432/tododb"
  secret-key: "my-super-secret-key"
```

> ⚠️ Secrets в K8s хранятся в etcd в base64 — это не шифрование. Для настоящей безопасности используй Sealed Secrets или HashiCorp Vault.

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

kubectl get configmaps
kubectl get secrets
kubectl describe configmap todo-config

# Создать secret из командной строки (не коммить в git!)
kubectl create secret generic todo-secrets \
  --from-literal=database-url="postgres://..." \
  --from-literal=secret-key="my-secret"
```

---

## Часть 4 — Ingress и постоянное хранилище

### Ingress — маршрутизация HTTP трафика

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: todo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: todo-service
                port:
                  number: 80
```

```bash
# Включить ingress в minikube
minikube addons enable ingress

kubectl apply -f ingress.yaml
kubectl get ingress
```

### Persistent Volume для PostgreSQL

```yaml
# postgres.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: POSTGRES_DB
              value: tododb
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          ports:
            - containerPort: 5432
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

---

## Часть 5 — Namespaces и структура проекта

### Namespaces

```bash
# Создать namespace
kubectl create namespace production
kubectl create namespace staging

# Работать в конкретном namespace
kubectl get pods -n production
kubectl apply -f deployment.yaml -n production

# Установить namespace по умолчанию
kubectl config set-context --current --namespace=production

# Посмотреть ресурсы во всех namespace
kubectl get pods --all-namespaces
kubectl get pods -A    # сокращение
```

### Структура K8s манифестов в проекте

```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml          # в .gitignore! или используй sealed secrets
├── postgres/
│   ├── pvc.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── app/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── monitoring/
    ├── prometheus/
    └── grafana/
```

```bash
# Применить все манифесты из папки
kubectl apply -f k8s/
kubectl apply -f k8s/ --recursive    # включая подпапки

# Удалить все
kubectl delete -f k8s/
```

### Полезные команды для отладки

```bash
# Описание объекта с событиями — первое место смотреть при проблемах
kubectl describe pod todo-pod-xxx

# Логи
kubectl logs todo-pod-xxx
kubectl logs todo-pod-xxx --previous    # логи предыдущего запуска (если упал)
kubectl logs -l app=todo-service        # логи всех pod с label

# Войти в pod
kubectl exec -it todo-pod-xxx -- sh
kubectl exec -it todo-pod-xxx -- /bin/bash

# Смотреть события кластера
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n production

# Смотреть ресурсы в реальном времени
kubectl get pods -w                    # watch
watch kubectl get pods                 # через watch

# Проверить resolving DNS внутри кластера
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup todo-service

# Проверить HTTP изнутри кластера
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://todo-service/health
```

---

## Практические задания

### Задание 11.1 — Первый деплой

```bash
# Запусти minikube
minikube start --driver=docker

# Настрой Docker чтобы использовать registry minikube
eval $(minikube docker-env)

# Собери образ прямо в minikube
docker build -t todo-service:local .
```

Создай `k8s/app/deployment.yaml` и `k8s/app/service.yaml`.

```bash
kubectl apply -f k8s/app/

# Проверь
kubectl get pods
kubectl get services

# Открой локально
kubectl port-forward service/todo-service 8080:80
curl http://localhost:8080/health
```

**Что нужно сделать:** Убедись что 3 реплики запущены. Удали один pod вручную (`kubectl delete pod <name>`) и посмотри как Deployment автоматически создаёт новый.

---

### Задание 11.2 — ConfigMap и Secret

1. Создай `k8s/configmap.yaml` с конфигурацией приложения
2. Создай `k8s/secret.yaml` с `DATABASE_URL` и `SECRET_KEY`
3. Подключи их в Deployment через `envFrom`
4. Примени и убедись что переменные доступны внутри pod:
   ```bash
   kubectl exec -it <pod-name> -- env | grep DATABASE
   ```

---

### Задание 11.3 — PostgreSQL в K8s

Создай манифесты для PostgreSQL:
1. `Secret` с `POSTGRES_USER`, `POSTGRES_PASSWORD`
2. `PersistentVolumeClaim` на 1Gi
3. `Deployment` с одной репликой
4. `Service` типа `ClusterIP`

```bash
kubectl apply -f k8s/postgres/

# Проверь что postgres доступен из приложения
kubectl exec -it <app-pod> -- sh
# внутри: psql $DATABASE_URL -c "\l"
```

Примени миграции:
```bash
kubectl exec -it <postgres-pod> -- psql -U todouser -d tododb \
  -c "CREATE TABLE tasks (...);"
```

---

### Задание 11.4 — Rolling Update

1. Сделай изменение в коде (например измени сообщение в `/health`)
2. Собери новый образ с тегом `v2`:
   ```bash
   eval $(minikube docker-env)
   docker build -t todo-service:v2 .
   ```
3. Обнови Deployment:
   ```bash
   kubectl set image deployment/todo-service todo-service=todo-service:v2
   ```
4. Наблюдай за обновлением:
   ```bash
   kubectl rollout status deployment/todo-service
   kubectl get pods -w
   ```
5. Откати обновление:
   ```bash
   kubectl rollout undo deployment/todo-service
   ```

**Что нужно сделать:** Объясни почему при `maxUnavailable: 0` не было даунтайма во время обновления.

---

### Задание 11.5 — Финальный проект урока

Полный деплой `todo-service` в Kubernetes:

**Структура:**
```
k8s/
├── namespace.yaml        # namespace: todo-production
├── postgres/
│   ├── secret.yaml
│   ├── pvc.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── app/
    ├── configmap.yaml
    ├── secret.yaml
    ├── deployment.yaml   # 2 реплики, liveness/readiness probes
    ├── service.yaml
    └── ingress.yaml
```

**Требования:**
1. Все ресурсы в namespace `todo-production`
2. Liveness probe и Readiness probe для приложения
3. Ограничения ресурсов (requests/limits) для всех контейнеров
4. `kubectl apply -f k8s/ --recursive` поднимает всё с нуля

**Проверка:**
```bash
# Всё запущено
kubectl get all -n todo-production

# Приложение отвечает
kubectl port-forward -n todo-production service/todo-service 8080:80
curl http://localhost:8080/tasks
```

Добавь в `.github/workflows/ci-cd.yml` деплой в minikube (через `kubectl apply`) при push в main. Закоммить с тегом `v0.6.0`.

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `kubectl apply -f file.yaml` | Создать/обновить объект |
| `kubectl get pods` | Список pod |
| `kubectl describe pod name` | Подробности + события |
| `kubectl logs pod-name` | Логи pod |
| `kubectl exec -it pod -- sh` | Войти в pod |
| `kubectl delete -f file.yaml` | Удалить объект |
| `kubectl scale deploy name --replicas=3` | Масштабировать |
| `kubectl rollout undo deploy name` | Откатить обновление |
| `kubectl port-forward svc/name 8080:80` | Пробросить порт |
| `kubectl get events` | События кластера |

---

## Ресурсы для изучения

- **Интерактивный курс:** `https://kubernetes.io/docs/tutorials/` — официальные туториалы
- **Игровая площадка:** `https://killercoda.com/kubernetes` — браузерный K8s
- **Визуализация:** `https://k8s.af` — визуальное объяснение концепций
- **Книга:** "Kubernetes in Action" — Marko Luksa (лучшая книга по K8s)
- **kubectl cheatsheet:** `https://kubernetes.io/docs/reference/kubectl/cheatsheet/`

---

## Как понять что урок пройден

- [ ] Понимаю архитектуру K8s: control plane, worker nodes, основные объекты
- [ ] Умею создавать Deployment, Service, ConfigMap, Secret
- [ ] Приложение запущено с 2+ репликами и Readiness probe
- [ ] Rolling update работает без даунтайма
- [ ] Умею откатить обновление через `kubectl rollout undo`
- [ ] PostgreSQL запущен с PersistentVolumeClaim
- [ ] Умею отлаживать проблемы через `describe`, `logs`, `exec`

---

*Следующий урок: Frontend — HTML, CSS, JavaScript и React*
