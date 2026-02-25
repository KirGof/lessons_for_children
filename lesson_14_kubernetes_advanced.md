# Урок 14 — Kubernetes: Helm, Operators и безопасность

## Зачем это нужно

В предыдущем уроке ты научился деплоить приложение в K8s через сырые манифесты. В реальных командах так никто не делает — слишком много копипасты, нет версионирования конфигов, невозможно управлять несколькими окружениями. Helm решает это. Operators позволяют автоматизировать сложные задачи — например автоматический бэкап базы данных или управление сертификатами. Безопасность — это не опция, это требование: дырявый кластер означает утечку данных или счёт на $50k в облаке.

---

## Часть 1 — Helm: пакетный менеджер для Kubernetes

### Что такое Helm

Helm — это как apt/brew, только для Kubernetes. Вместо того чтобы копировать и редактировать десятки YAML файлов для каждого окружения, ты описываешь шаблон один раз и параметризуешь его.

```
Без Helm:
k8s/dev/deployment.yaml    — 50 строк
k8s/prod/deployment.yaml   — те же 50 строк, но replicas: 3 и image: v2.0
k8s/staging/...            — и ещё раз то же самое

С Helm:
chart/templates/deployment.yaml  — шаблон один раз
values.dev.yaml                  — replicas: 1, image: latest
values.prod.yaml                 — replicas: 3, image: v2.0
```

### Основные концепции

- **Chart** — пакет Helm. Папка с шаблонами и метаданными.
- **Release** — установленный Chart в кластер. Один Chart можно установить несколько раз с разными именами.
- **Values** — параметры которые подставляются в шаблоны.
- **Repository** — хранилище Charts (как npm registry).

### Установка Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Готовые Charts — установка nginx-ingress

```bash
# Добавить репозиторий
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Посмотреть доступные версии
helm search repo ingress-nginx

# Установить
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Посмотреть что установилось
helm list -A
kubectl get all -n ingress-nginx

# Обновить
helm upgrade ingress-nginx ingress-nginx/ingress-nginx

# Удалить
helm uninstall ingress-nginx -n ingress-nginx
```

### Создать свой Chart

```bash
# Создать структуру
helm create todo-service

# Структура:
todo-service/
├── Chart.yaml           # метаданные пакета
├── values.yaml          # значения по умолчанию
├── charts/              # зависимые charts
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── _helpers.tpl     # переиспользуемые шаблоны
    ├── NOTES.txt        # сообщение после установки
    └── tests/
        └── test-connection.yaml
```

### Chart.yaml

```yaml
apiVersion: v2
name: todo-service
description: A TODO service with PostgreSQL backend
type: application
version: 0.1.0          # версия Chart
appVersion: "1.0.0"     # версия приложения
```

### values.yaml

```yaml
replicaCount: 1

image:
  repository: ghcr.io/username/todo-service
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  host: todo.example.com

resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"

env:
  logLevel: info
  port: "8080"

postgresql:
  enabled: true
  host: postgres
  port: 5432
  database: tododb
  username: todouser

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### templates/deployment.yaml — шаблон

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "todo-service.fullname" . }}
  labels:
    {{- include "todo-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "todo-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "todo-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.env.port }}
          env:
            - name: PORT
              value: {{ .Values.env.port | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.env.logLevel }}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "todo-service.fullname" . }}-secrets
                  key: database-url
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          readinessProbe:
            httpGet:
              path: /health
              port: {{ .Values.env.port }}
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: {{ .Values.env.port }}
            initialDelaySeconds: 15
            periodSeconds: 20
```

### templates/_helpers.tpl

```
{{/*
Полное имя релиза
*/}}
{{- define "todo-service.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Общие labels
*/}}
{{- define "todo-service.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "todo-service.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Установка и управление релизами

```bash
# Проверить шаблон перед установкой (dry run)
helm template todo-service ./todo-service

# Lint — проверить синтаксис
helm lint ./todo-service

# Установить
helm install todo-dev ./todo-service \
  --namespace development \
  --create-namespace \
  --set image.tag=v1.2.0 \
  --set replicaCount=1

# Установить с файлом values
helm install todo-prod ./todo-service \
  --namespace production \
  --create-namespace \
  -f values.prod.yaml

# values.prod.yaml
replicaCount: 3
image:
  tag: "v1.2.0"
ingress:
  enabled: true
  host: todo.mycompany.com
autoscaling:
  enabled: true

# Обновить
helm upgrade todo-prod ./todo-service -f values.prod.yaml

# Откатить
helm rollback todo-prod 1        # откат к revision 1
helm history todo-prod            # история релизов

# Посмотреть текущие values
helm get values todo-prod

# Удалить
helm uninstall todo-prod -n production
```

---

## Часть 2 — Operators: автоматизация сложных задач

### Что такое Operator

Kubernetes управляет stateless приложениями хорошо — Deployment сам перезапускает упавшие Pod. Но stateful приложения (БД, очереди) требуют человеческой логики: как правильно сделать бэкап, как добавить реплику, как обновить без потери данных.

**Operator** — это программа которая работает внутри K8s и управляет сложными приложениями используя те же механизмы что и K8s сам. Operator расширяет API кластера через **Custom Resource Definitions (CRD)**.

```
Обычный K8s:              С Operator:
kubectl get pods          kubectl get postgresclusters
kubectl get deployments   kubectl get redisfailover
                          kubectl get certificates
```

### Готовые Operators — примеры

```bash
# cert-manager — автоматические TLS сертификаты
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# После установки можно делать так:
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: todo-tls
spec:
  secretName: todo-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - todo.example.com
EOF
# cert-manager сам получит сертификат от Let's Encrypt и обновит его

# CloudNativePG — Operator для PostgreSQL
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.22.0.yaml

# Создать кластер PostgreSQL с репликой одной командой:
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: todo-postgres
spec:
  instances: 3          # 1 primary + 2 replicas
  storage:
    size: 10Gi
  bootstrap:
    initdb:
      database: tododb
      owner: todouser
EOF
```

### Написать свой простой Operator

Operator — это просто программа которая:
1. Подписывается на события K8s (create/update/delete ресурсов)
2. Реагирует — создаёт/изменяет/удаляет другие ресурсы

Для Go используется **controller-runtime**:

```bash
# Установить kubebuilder — фреймворк для написания Operators
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/linux/amd64
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/

# Создать проект
mkdir todo-operator && cd todo-operator
kubebuilder init --domain example.com --repo github.com/username/todo-operator

# Создать API (CRD + Controller)
kubebuilder create api --group apps --version v1alpha1 --kind TodoApp
```

Это создаст:
```
todo-operator/
├── api/v1alpha1/
│   └── todoapp_types.go    # определение нового ресурса
├── internal/controller/
│   └── todoapp_controller.go  # логика reconciliation
└── config/
    └── crd/                # манифест CRD
```

`api/v1alpha1/todoapp_types.go`:
```go
type TodoAppSpec struct {
    Replicas  int32  `json:"replicas"`
    Image     string `json:"image"`
    DBSecret  string `json:"dbSecret"`
}

type TodoAppStatus struct {
    AvailableReplicas int32  `json:"availableReplicas"`
    Phase             string `json:"phase"` // Pending, Running, Failed
}
```

`internal/controller/todoapp_controller.go`:
```go
func (r *TodoAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // 1. Получить ресурс
    var app appsv1alpha1.TodoApp
    if err := r.Get(ctx, req.NamespacedName, &app); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Убедиться что Deployment существует с нужными параметрами
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: app.Name, Namespace: app.Namespace}, deployment)
    if errors.IsNotFound(err) {
        // Создать Deployment
        dep := r.buildDeployment(&app)
        if err := r.Create(ctx, dep); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("Created Deployment", "name", app.Name)
    }

    // 3. Обновить статус
    app.Status.AvailableReplicas = deployment.Status.AvailableReplicas
    app.Status.Phase = "Running"
    r.Status().Update(ctx, &app)

    return ctrl.Result{RequeueAfter: time.Minute}, nil
}
```

После деплоя Operator можно делать:
```yaml
apiVersion: apps.example.com/v1alpha1
kind: TodoApp
metadata:
  name: my-todo
spec:
  replicas: 3
  image: ghcr.io/username/todo-service:v1.0
  dbSecret: todo-db-secret
```

---

## Часть 3 — Безопасность кластера

### RBAC — Role-Based Access Control

По умолчанию каждый Pod имеет ServiceAccount с минимальными правами. Но если кто-то получит доступ к Pod — он не должен иметь возможность управлять всем кластером.

```yaml
# Создать ServiceAccount для приложения
apiVersion: v1
kind: ServiceAccount
metadata:
  name: todo-service-sa
  namespace: production

---
# Role — что можно делать (в рамках namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: todo-service-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]        # только читать ConfigMaps
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["todo-secrets"]  # только конкретный Secret
    verbs: ["get"]

---
# RoleBinding — привязать Role к ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: todo-service-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: todo-service-sa
    namespace: production
roleRef:
  kind: Role
  name: todo-service-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Проверить права
kubectl auth can-i get secrets --as=system:serviceaccount:production:todo-service-sa -n production
# yes

kubectl auth can-i delete pods --as=system:serviceaccount:production:todo-service-sa -n production
# no

# Привязать ServiceAccount к Deployment
spec:
  template:
    spec:
      serviceAccountName: todo-service-sa
```

### Network Policies — сетевая изоляция

По умолчанию в K8s все Pod могут общаться друг с другом. Network Policy ограничивает это.

```yaml
# Запретить весь входящий трафик к postgres, кроме app
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: todo-service    # только от todo-service
      ports:
        - protocol: TCP
          port: 5432
  egress: []    # postgres не инициирует исходящие соединения

---
# Разрешить todo-service только к postgres и наружу (но не к другим Pod)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: todo-service-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: todo-service
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:
        - namespaceSelector: {}   # DNS resolution
      ports:
        - port: 53
```

### Pod Security — запрет опасных конфигураций

```yaml
# PodSecurity Admission — на уровне namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted   # строгий режим
    pod-security.kubernetes.io/warn: restricted

---
# SecurityContext на уровне Pod
spec:
  securityContext:
    runAsNonRoot: true          # запретить запуск от root
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault      # ограничить системные вызовы

  containers:
    - name: todo-service
      securityContext:
        allowPrivilegeEscalation: false    # запретить повышение привилегий
        readOnlyRootFilesystem: true       # файловая система read-only
        capabilities:
          drop:
            - ALL                          # убрать все Linux capabilities
```

### Secrets Management — правильное хранение секретов

Обычные K8s Secrets хранятся в etcd в base64 — это не безопасно. Есть несколько подходов:

**Sealed Secrets — зашифрованные секреты в git:**
```bash
# Установить
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Зашифровать секрет
kubectl create secret generic todo-secrets \
  --from-literal=database-url="postgres://..." \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# sealed-secret.yaml можно коммитить в git — он зашифрован
# Только kube-system/sealed-secrets сможет его расшифровать
```

**External Secrets Operator — секреты из Vault/AWS SSM:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: todo-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: todo-secrets
  data:
    - secretKey: database-url
      remoteRef:
        key: secret/todo-service
        property: database_url
```

### Сканирование образов

```bash
# trivy — сканер уязвимостей
apt install trivy

# Сканировать образ
trivy image ghcr.io/username/todo-service:latest

# Добавить в CI/CD
- name: Scan Docker image
  run: trivy image --exit-code 1 --severity CRITICAL ghcr.io/${{ github.repository }}:${{ github.sha }}
  # --exit-code 1 — упасть если найдены CRITICAL уязвимости
```

---

## Практические задания

### Задание 14.1 — Первый Helm Chart

Преобразуй манифесты `todo-service` из `k8s/` в Helm Chart:

```bash
helm create todo-service-chart
```

1. Перенеси Deployment, Service, ConfigMap, Secret в `templates/`
2. Вынеси все переменные параметры в `values.yaml`:
   - `replicaCount`
   - `image.repository` и `image.tag`
   - `env.logLevel`
   - `ingress.enabled` и `ingress.host`
3. Используй `_helpers.tpl` для имён ресурсов

```bash
# Проверь
helm lint ./todo-service-chart
helm template ./todo-service-chart

# Установи
helm install todo-dev ./todo-service-chart -n development --create-namespace
helm install todo-prod ./todo-service-chart -n production --create-namespace \
  -f values.prod.yaml
```

**Что нужно сделать:** Убедись что два независимых релиза работают одновременно в разных namespace.

---

### Задание 14.2 — Зависимости в Helm

Добавь PostgreSQL как зависимость Chart:

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

```bash
helm dependency update ./todo-service-chart
```

В `values.yaml`:
```yaml
postgresql:
  enabled: true
  auth:
    username: todouser
    password: secret123
    database: tododb
  primary:
    persistence:
      size: 1Gi
```

**Что нужно сделать:** Установи Chart с встроенным PostgreSQL. Убедись что `todo-service` подключается к нему. Попробуй установить с `postgresql.enabled=false` — должна использоваться внешняя БД.

---

### Задание 14.3 — RBAC

Создай минимальные права для `todo-service`:

1. `ServiceAccount` — `todo-service-sa`
2. `Role` — разрешить только читать свой ConfigMap и Secret
3. `RoleBinding` — привязать к ServiceAccount
4. Обнови Deployment чтобы использовал этот ServiceAccount

```bash
# Проверь что права работают правильно
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:production:todo-service-sa -n production
# yes

kubectl auth can-i delete deployments \
  --as=system:serviceaccount:production:todo-service-sa -n production
# no
```

**Что нужно сделать:** Попробуй из Pod получить список всех секретов:
```bash
kubectl exec -it <todo-pod> -- sh
# внутри:
curl -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api/v1/namespaces/production/secrets \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
# Должен вернуть 403 Forbidden
```

---

### Задание 14.4 — Network Policy

Реализуй сетевую изоляцию для `todo-service`:

1. Запрети весь входящий трафик к postgres кроме как от `todo-service`
2. Запрети `todo-service` обращаться к чему-либо кроме postgres (и DNS)
3. Разреши входящий трафик к `todo-service` только от ingress-controller

```bash
# Проверь что изоляция работает
# Запусти тестовый pod и попробуй подключиться к postgres напрямую
kubectl run -it --rm test --image=postgres:16-alpine -n production -- \
  psql postgres://todouser:secret@postgres:5432/tododb
# Должен зависнуть и не подключиться

# Но todo-service должен работать
curl http://todo-service/tasks
# Должен вернуть задачи
```

---

### Задание 14.5 — Финальный проект урока

Собери production-ready деплой `todo-service`:

**1. Helm Chart с поддержкой окружений:**
```
charts/
└── todo-service/
    ├── Chart.yaml
    ├── values.yaml           # defaults
    ├── values.dev.yaml       # dev: replicas=1, no ingress
    └── values.prod.yaml      # prod: replicas=3, ingress, autoscaling
```

**2. SecurityContext на Pod:**
- `runAsNonRoot: true`
- `readOnlyRootFilesystem: true`
- `allowPrivilegeEscalation: false`
- Убедись что приложение запускается (возможно нужно добавить `tmpfs` для временных файлов)

**3. Sealed Secrets:**
- Установи `sealed-secrets` controller
- Зашифруй `DATABASE_URL` через `kubeseal`
- Закоммить `sealed-secret.yaml` в репозиторий

**4. Сканирование в CI:**
```yaml
- name: Scan for vulnerabilities
  run: trivy image --exit-code 1 --severity CRITICAL $IMAGE
```

**5. Добавь в `Makefile`:**
```makefile
helm-install-dev:
    helm upgrade --install todo-dev ./charts/todo-service \
      -f charts/todo-service/values.dev.yaml \
      -n development --create-namespace

helm-install-prod:
    helm upgrade --install todo-prod ./charts/todo-service \
      -f charts/todo-service/values.prod.yaml \
      -n production --create-namespace

helm-diff:
    helm diff upgrade todo-prod ./charts/todo-service \
      -f charts/todo-service/values.prod.yaml
```

Закоммить с тегом `v0.8.0`.

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `helm install name ./chart` | Установить Chart |
| `helm upgrade name ./chart` | Обновить релиз |
| `helm rollback name 1` | Откатить к ревизии 1 |
| `helm template ./chart` | Рендер шаблонов без установки |
| `helm lint ./chart` | Проверить синтаксис |
| `helm get values name` | Текущие values релиза |
| `helm history name` | История релизов |
| `kubectl auth can-i VERB RESOURCE` | Проверить права |
| `trivy image name:tag` | Сканировать образ |

---

## Ресурсы для изучения

- **Helm документация:** `https://helm.sh/docs/`
- **Helm Hub:** `https://artifacthub.io` — поиск готовых Charts
- **RBAC:** `https://kubernetes.io/docs/reference/access-authn-authz/rbac/`
- **Network Policies:** `https://networkpolicy.io` — визуальный редактор политик
- **Sealed Secrets:** `https://github.com/bitnami-labs/sealed-secrets`
- **trivy:** `https://github.com/aquasecurity/trivy`
- **Книга:** "Kubernetes Security" — Liz Rice

---

## Как понять что урок пройден

- [ ] Написал Helm Chart для `todo-service` с параметризованными values
- [ ] Два окружения (dev/prod) разворачиваются одним Chart с разными values
- [ ] Зависимость PostgreSQL работает через `helm dependency`
- [ ] RBAC настроен — Pod не может делать лишнего в кластере
- [ ] Network Policy изолирует postgres от посторонних Pod
- [ ] SecurityContext: приложение работает не от root с read-only FS
- [ ] trivy сканирование встроено в CI pipeline
- [ ] Sealed Secrets — зашифрованные секреты хранятся в git

---

*Следующий урок: Распределённые системы — Redis, Kafka, gRPC*
