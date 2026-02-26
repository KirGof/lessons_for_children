# Урок 22 — Облако: деплой в Yandex Cloud с Terraform

## Зачем это нужно

До этого всё работало локально или на условном "сервере". Облако — это стандарт продакшена: managed базы данных, автомасштабирование, балансировщики нагрузки, CDN, managed Kubernetes. Ты платишь только за то что используешь и не занимаешься железом. После этого урока `todo-service` будет работать в реальном облаке — с настоящим доменом, TLS и managed PostgreSQL. Концепции одинаковы для AWS, GCP и Yandex Cloud — различается только синтаксис Terraform провайдера.

---

## Часть 1 — Облачные концепции

### Модели сервисов

```
IaaS — Infrastructure as a Service
  Ты управляешь: ОС, runtime, данными, приложением
  Облако даёт: виртуальные машины, сети, диски
  Пример: Yandex Compute Cloud, AWS EC2

PaaS — Platform as a Service
  Ты управляешь: данными, приложением
  Облако даёт: всё остальное включая ОС и runtime
  Пример: Heroku, Google App Engine, Yandex Serverless

CaaS — Container as a Service
  Ты управляешь: контейнерами и конфигурацией
  Облако даёт: оркестратор контейнеров
  Пример: Yandex Managed Kubernetes, AWS EKS

SaaS — Software as a Service
  Ты используешь готовый продукт
  Пример: GitHub, Slack, Gmail
```

### Yandex Cloud — базовые сущности

```
Organization               — верхний уровень
└── Cloud                  — аккаунт/организация
    └── Folder             — как namespace, группирует ресурсы
        ├── Compute        — виртуальные машины
        ├── VPC            — виртуальные сети
        ├── Managed K8s    — managed Kubernetes кластер
        ├── Managed PG     — managed PostgreSQL
        ├── Object Storage — S3-совместимое хранилище
        ├── Container Registry — Docker образы
        └── DNS            — управление доменами
```

---

## Часть 2 — Подготовка окружения

### Регистрация и настройка

```bash
# 1. Зарегистрироваться на cloud.yandex.ru
# Новые аккаунты получают грант $50 на 2 месяца

# 2. Установить CLI
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
source ~/.bashrc
yc --version

# 3. Авторизоваться
yc init
# Следуй инструкциям: откроет браузер для OAuth

# 4. Проверить
yc config list
yc resource-manager cloud list
yc resource-manager folder list

# 5. Создать service account для Terraform
yc iam service-account create --name terraform-sa
yc resource-manager folder add-access-binding \
  --id $(yc config get folder-id) \
  --role editor \
  --service-account-name terraform-sa

# Создать ключ для Terraform
yc iam key create \
  --service-account-name terraform-sa \
  --output key.json
# key.json — в .gitignore!

# 6. Создать статический ключ для Object Storage (state backend)
yc iam access-key create \
  --service-account-name terraform-sa
# Сохрани access_key и secret_key
```

### Container Registry — хранилище Docker образов

```bash
# Создать реестр
yc container registry create --name todo-registry

# Аутентифицироваться
yc container registry configure-docker

# Тегировать и пушить образ
docker tag todo-service:local \
  cr.yandex/$(yc container registry get todo-registry --format=json | jq -r .id)/todo-service:latest

docker push \
  cr.yandex/$(yc container registry get todo-registry --format=json | jq -r .id)/todo-service:latest
```

---

## Часть 3 — Terraform для Yandex Cloud

### Структура проекта

```
infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars      # в .gitignore
│   └── prod/
│       └── ...
├── modules/
│   ├── network/
│   ├── kubernetes/
│   ├── database/
│   └── registry/
└── .terraform.lock.hcl
```

### Provider и backend

```hcl
# infra/environments/dev/main.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.105"
    }
  }

  backend "s3" {
    endpoint   = "storage.yandexcloud.net"
    bucket     = "todo-terraform-state"
    key        = "dev/terraform.tfstate"
    region     = "ru-central1"
    access_key = "YOUR_ACCESS_KEY"    # или через env YC_STORAGE_ACCESS_KEY
    secret_key = "YOUR_SECRET_KEY"    # или через env YC_STORAGE_SECRET_KEY

    skip_region_validation      = true
    skip_credentials_validation = true
  }
}

provider "yandex" {
  service_account_key_file = var.sa_key_file
  cloud_id                 = var.cloud_id
  folder_id                = var.folder_id
  zone                     = "ru-central1-a"
}
```

### Сеть

```hcl
# infra/modules/network/main.tf

resource "yandex_vpc_network" "main" {
  name = "${var.name}-network"
}

resource "yandex_vpc_subnet" "public-a" {
  name           = "${var.name}-public-a"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.main.id
  v4_cidr_blocks = ["10.1.0.0/24"]
}

resource "yandex_vpc_subnet" "public-b" {
  name           = "${var.name}-public-b"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.main.id
  v4_cidr_blocks = ["10.2.0.0/24"]
}

resource "yandex_vpc_subnet" "private" {
  name           = "${var.name}-private"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.main.id
  v4_cidr_blocks = ["10.3.0.0/24"]
}

output "network_id"         { value = yandex_vpc_network.main.id }
output "public_subnet_a_id" { value = yandex_vpc_subnet.public-a.id }
output "public_subnet_b_id" { value = yandex_vpc_subnet.public-b.id }
output "private_subnet_id"  { value = yandex_vpc_subnet.private.id }
```

### Managed Kubernetes

```hcl
# infra/modules/kubernetes/main.tf

# Service account для кластера
resource "yandex_iam_service_account" "k8s" {
  name = "${var.name}-k8s-sa"
}

resource "yandex_resourcemanager_folder_iam_member" "k8s-editor" {
  folder_id = var.folder_id
  role      = "editor"
  member    = "serviceAccount:${yandex_iam_service_account.k8s.id}"
}

# Кластер Kubernetes
resource "yandex_kubernetes_cluster" "main" {
  name       = "${var.name}-cluster"
  network_id = var.network_id

  master {
    version   = "1.28"
    zonal {
      zone      = "ru-central1-a"
      subnet_id = var.subnet_id
    }
    public_ip = true    # мастер доступен снаружи для kubectl

    maintenance_policy {
      auto_upgrade = true
      maintenance_window {
        day        = "sunday"
        start_time = "03:00"
        duration   = "3h"
      }
    }
  }

  service_account_id      = yandex_iam_service_account.k8s.id
  node_service_account_id = yandex_iam_service_account.k8s.id

  depends_on = [
    yandex_resourcemanager_folder_iam_member.k8s-editor,
  ]
}

# Node Group — worker nodes
resource "yandex_kubernetes_node_group" "main" {
  cluster_id = yandex_kubernetes_cluster.main.id
  name       = "${var.name}-nodes"
  version    = "1.28"

  instance_template {
    platform_id = "standard-v3"

    resources {
      memory        = var.node_memory
      cores         = var.node_cores
      core_fraction = 50    # burstable CPU (дешевле)
    }

    boot_disk {
      type = "network-ssd"
      size = 64
    }

    network_interface {
      subnet_ids = [var.subnet_id]
      nat        = true
    }

    container_runtime {
      type = "containerd"
    }
  }

  scale_policy {
    auto_scale {
      min     = var.min_nodes
      max     = var.max_nodes
      initial = var.initial_nodes
    }
  }

  allocation_policy {
    location { zone = "ru-central1-a" }
  }

  maintenance_policy {
    auto_upgrade = true
    auto_repair  = true
  }
}

# Вывести команду для подключения kubectl
output "cluster_id" { value = yandex_kubernetes_cluster.main.id }
output "kubeconfig_command" {
  value = "yc managed-kubernetes cluster get-credentials ${yandex_kubernetes_cluster.main.name} --external"
}
```

### Managed PostgreSQL

```hcl
# infra/modules/database/main.tf

resource "yandex_mdb_postgresql_cluster" "main" {
  name        = "${var.name}-postgres"
  environment = var.environment == "prod" ? "PRODUCTION" : "PRESTABLE"
  network_id  = var.network_id

  config {
    version = 16
    resources {
      # s2.micro — 2 vCPU, 8 GB RAM — минимум для продакшена
      resource_preset_id = var.environment == "prod" ? "s2.micro" : "b2.nano"
      disk_type_id       = "network-ssd"
      disk_size          = var.disk_size
    }

    postgresql_config = {
      max_connections                = 100
      shared_buffers_fraction        = 0.25
      effective_cache_size_fraction  = 0.75
    }

    backup_window_start {
      hours   = 2
      minutes = 0
    }

    access {
      web_sql = true    # доступ через веб-консоль
    }
  }

  # Для высокой доступности — несколько хостов
  dynamic "host" {
    for_each = var.hosts
    content {
      zone      = host.value.zone
      subnet_id = host.value.subnet_id
    }
  }

  maintenance_window {
    type = "WEEKLY"
    day  = "SUN"
    hour = 3
  }
}

resource "yandex_mdb_postgresql_database" "app" {
  cluster_id = yandex_mdb_postgresql_cluster.main.id
  name       = var.db_name
  owner      = yandex_mdb_postgresql_user.app.name
}

resource "yandex_mdb_postgresql_user" "app" {
  cluster_id = yandex_mdb_postgresql_cluster.main.id
  name       = var.db_user
  password   = var.db_password

  permission {
    database_name = var.db_name
  }
}

output "host" {
  value = yandex_mdb_postgresql_cluster.main.host[0].fqdn
}

output "connection_string" {
  value     = "postgres://${var.db_user}:${var.db_password}@${yandex_mdb_postgresql_cluster.main.host[0].fqdn}:6432/${var.db_name}?sslmode=verify-full"
  sensitive = true
}
```

### Сборка всего в environments/dev

```hcl
# infra/environments/dev/main.tf

module "network" {
  source = "../../modules/network"
  name   = "todo-dev"
}

module "database" {
  source      = "../../modules/database"
  name        = "todo-dev"
  environment = "dev"
  network_id  = module.network.network_id
  db_name     = "tododb"
  db_user     = "todouser"
  db_password = var.db_password
  disk_size   = 10

  hosts = [{
    zone      = "ru-central1-a"
    subnet_id = module.network.private_subnet_id
  }]
}

module "kubernetes" {
  source      = "../../modules/kubernetes"
  name        = "todo-dev"
  folder_id   = var.folder_id
  network_id  = module.network.network_id
  subnet_id   = module.network.public_subnet_a_id
  min_nodes   = 1
  max_nodes   = 3
  initial_nodes = 1
  node_memory = 4
  node_cores  = 2
}

output "kubeconfig_command" {
  value = module.kubernetes.kubeconfig_command
}

output "database_connection" {
  value     = module.database.connection_string
  sensitive = true
}
```

---

## Часть 4 — CI/CD в облаке

### GitHub Actions → Yandex Cloud

```yaml
# .github/workflows/deploy-cloud.yml
name: Deploy to Yandex Cloud

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

env:
  REGISTRY: cr.yandex
  IMAGE_NAME: ${{ secrets.YC_REGISTRY_ID }}/todo-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to Yandex Container Registry
        uses: docker/login-action@v3
        with:
          registry: cr.yandex
          username: json_key
          password: ${{ secrets.YC_SA_JSON_KEY }}

      - uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install yc CLI
        run: |
          curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
          echo "$HOME/yandex-cloud/bin" >> $GITHUB_PATH

      - name: Configure kubectl
        run: |
          echo '${{ secrets.YC_SA_JSON_KEY }}' > key.json
          yc config set service-account-key key.json
          yc config set cloud-id ${{ secrets.YC_CLOUD_ID }}
          yc config set folder-id ${{ secrets.YC_FOLDER_ID }}
          yc managed-kubernetes cluster get-credentials \
            ${{ secrets.K8S_CLUSTER_NAME }} --external

      - name: Deploy to Kubernetes
        run: |
          # Обновить образ в Deployment
          kubectl set image deployment/todo-service \
            todo-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production

          # Дождаться завершения rollout
          kubectl rollout status deployment/todo-service -n production --timeout=5m

      - name: Verify deployment
        run: |
          # Подождать чтобы поды поднялись
          sleep 10
          # Проверить health эндпоинт
          EXTERNAL_IP=$(kubectl get svc ingress-nginx-controller \
            -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
          curl -f http://$EXTERNAL_IP/health || exit 1
```

---

## Часть 5 — Prod-ready конфигурация

### Ingress с TLS через cert-manager

```bash
# Установить ingress-nginx через Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Установить cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

```yaml
# k8s/cert-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - todo.yourdomain.com
      secretName: todo-tls
  rules:
    - host: todo.yourdomain.com
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

### Настройка DNS

```hcl
# infra/modules/dns/main.tf

resource "yandex_dns_zone" "main" {
  name        = "todo-zone"
  zone        = "${var.domain}."
  public      = true
}

resource "yandex_dns_recordset" "app" {
  zone_id = yandex_dns_zone.main.id
  name    = "todo.${var.domain}."
  type    = "A"
  ttl     = 300
  data    = [var.ingress_ip]
}
```

### Мониторинг затрат

```bash
# Посмотреть текущий биллинг
yc billing account list
yc billing budget list

# Создать бюджет — алерт при превышении
yc billing budget create \
  --account-id <billing-account-id> \
  --display-name "Monthly limit" \
  --currency RUB \
  --amount 3000 \
  --reset-period monthly \
  --notify-at-percent 50,80,100
```

---

## Практические задания

### Задание 22.1 — Регистрация и первые ресурсы

1. Зарегистрируйся в Yandex Cloud (или используй уже существующий аккаунт)
2. Установи `yc` CLI и авторизуйся
3. Создай service account `terraform-sa` с ролью `editor`
4. Создай Object Storage бакет для Terraform state:
   ```bash
   yc storage bucket create --name todo-terraform-state-$(date +%s)
   ```
5. Напиши минимальный Terraform конфиг который создаёт VPC сеть и подсеть
6. Выполни `terraform init && terraform apply`
7. Убедись что ресурсы появились в консоли Yandex Cloud

---

### Задание 22.2 — Container Registry

1. Создай Container Registry через Terraform:
   ```hcl
   resource "yandex_container_registry" "main" {
     name = "todo-registry"
   }
   ```
2. Настрой Docker для работы с реестром:
   ```bash
   yc container registry configure-docker
   ```
3. Собери и запушь образ `todo-service` в реестр
4. Добавь в CI/CD пуш в Yandex Container Registry

---

### Задание 22.3 — Managed PostgreSQL

Создай managed PostgreSQL через Terraform:

1. Один хост в `ru-central1-a` для dev
2. База данных `tododb`, пользователь `todouser`
3. Пароль храни в переменной Terraform (никогда не хардкодь)

```bash
terraform apply -var="db_password=$(openssl rand -hex 16)"
terraform output database_connection    # подключиться и проверить
```

**Что нужно сделать:** Подключись к managed PostgreSQL, примени миграции, убедись что таблица `tasks` создана.

---

### Задание 22.4 — Managed Kubernetes

1. Создай Kubernetes кластер через Terraform (минимальная конфигурация: 1 node, 2 vCPU, 4 GB)
2. Получи credentials:
   ```bash
   yc managed-kubernetes cluster get-credentials todo-dev-cluster --external
   kubectl get nodes    # должна появиться нода
   ```
3. Задеплой `todo-service` в кластер:
   ```bash
   kubectl apply -f k8s/ -n production
   ```
4. Установи ingress-nginx через Helm
5. Приложение должно быть доступно по внешнему IP load balancer

**Что нужно сделать:** Выполни `curl http://<external-ip>/health` — должен вернуть `{"status":"ok"}`.

---

### Задание 22.5 — Финальный проект урока

Полный prod-ready деплой в Yandex Cloud:

**Инфраструктура через Terraform:**
```
infra/environments/dev/
├── main.tf      — подключает все модули
└── terraform.tfvars.example

Создаёт:
- VPC сеть с публичными и приватными подсетями
- Managed Kubernetes (1-3 nodes, autoscaling)
- Managed PostgreSQL (1 хост, 10 GB)
- Container Registry
```

**Приложение в Kubernetes:**
```
- Deployment: 2 реплики
- Service + Ingress с TLS (если есть домен)
- ConfigMap + Secret (DB пароль из Vault или K8s Secret)
- HorizontalPodAutoscaler (min: 2, max: 5)
- PodDisruptionBudget (минимум 1 реплика всегда доступна)
```

**CI/CD:**
```yaml
Push to main →
  1. Тесты
  2. Сборка образа → push в cr.yandex
  3. kubectl set image
  4. Проверка health
```

**Мониторинг:**
- Prometheus + Grafana задеплоены в кластер (через Helm)
- Дашборд показывает RPS, latency, error rate

**Документация:**
```markdown
# Deployment Guide

## Prerequisites
- yc CLI настроен
- kubectl настроен
- Terraform >= 1.6

## Deploy infrastructure
cd infra/environments/dev
terraform init
terraform apply

## Deploy application
kubectl apply -f k8s/ -n production

## Verify
curl https://todo.yourdomain.com/health
```

Закоммить с тегом `v2.2.0`.

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `yc config list` | Текущая конфигурация CLI |
| `yc compute instance list` | Список ВМ |
| `yc managed-kubernetes cluster list` | K8s кластеры |
| `yc managed-postgresql cluster list` | PostgreSQL кластеры |
| `yc container registry list` | Container Registry |
| `yc container registry configure-docker` | Настроить Docker |
| `yc managed-kubernetes cluster get-credentials NAME --external` | Получить kubeconfig |
| `yc iam service-account list` | Service accounts |
| `yc billing budget list` | Бюджеты |

---

## Ресурсы для изучения

- **Yandex Cloud документация:** `https://cloud.yandex.ru/docs`
- **Terraform Yandex provider:** `https://registry.terraform.io/providers/yandex-cloud/yandex`
- **Yandex Container Registry:** `https://cloud.yandex.ru/docs/container-registry/`
- **Managed Kubernetes:** `https://cloud.yandex.ru/docs/managed-kubernetes/`
- **Цены:** `https://cloud.yandex.ru/prices`

---

## Как понять что урок пройден

- [ ] Terraform создаёт VPC, Kubernetes, PostgreSQL в Yandex Cloud
- [ ] State хранится в Object Storage (не локально)
- [ ] Docker образ пушится в Yandex Container Registry из CI
- [ ] Приложение задеплоено в managed Kubernetes кластер
- [ ] Managed PostgreSQL доступен из кластера
- [ ] Есть LoadBalancer — приложение доступно по внешнему IP
- [ ] `terraform destroy` чисто удаляет всё (не платим за неиспользуемое)
- [ ] Бюджетный алерт настроен — не получим неожиданный счёт

---

*Следующий урок: Go продвинутый — generics, reflection, code generation*
