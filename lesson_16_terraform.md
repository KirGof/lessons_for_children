# Урок 16 — Terraform: инфраструктура как код

## Зачем это нужно

До этого урока ты запускал всё локально или вручную кликал в консоли облака. В реальной работе инфраструктура — это тоже код: версионируется в git, проверяется на CI, разворачивается одной командой. Terraform позволяет описать сервер, базу данных, DNS запись, сертификат — всё что угодно — в виде `.tf` файлов. И воспроизвести это окружение точь-в-точь в любой момент. Это разница между "я помню как это настраивал" и "это записано и воспроизводимо".

---

## Часть 1 — Как работает Terraform

### Концепции

```
Provider    — плагин для работы с конкретным облаком/сервисом (AWS, GCP, K8s, GitHub...)
Resource    — инфраструктурный объект (сервер, БД, DNS запись, S3 бакет...)
Data source — читать существующие ресурсы (не создавать)
Variable    — входные параметры
Output      — выходные значения
State       — файл с текущим состоянием (что уже создано)
Module      — переиспользуемый блок конфигурации
```

### Жизненный цикл

```
terraform init     — скачать провайдеры и модули
terraform plan     — показать что изменится (не применяет!)
terraform apply    — применить изменения
terraform destroy  — удалить всю инфраструктуру
```

### Модель работы

```
.tf файлы          Terraform          Облако
(желаемое) ──────► сравнить с ──────► применить
                   state файлом       изменения
                   (текущее)
```

Terraform всегда знает что уже создано благодаря `terraform.tfstate`. Если ресурс существует — обновит. Если нет — создаст. Если удалён из конфига — удалит.

---

## Часть 2 — Установка и первые шаги

### Установка

```bash
# Через официальный репозиторий
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform -y

terraform version
```

### Первый конфиг — локальный файл

Создай папку `infra/` в репозитории:

```hcl
# infra/main.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# Создать локальный файл — для понимания синтаксиса
resource "local_file" "readme" {
  filename = "${path.module}/output/README.md"
  content  = <<-EOT
    # Todo Service Infrastructure

    Environment: ${var.environment}
    Created at:  ${timestamp()}
  EOT
}

variable "environment" {
  description = "Environment name (dev/staging/prod)"
  type        = string
  default     = "dev"
}

output "file_path" {
  description = "Path to created README"
  value       = local_file.readme.filename
}
```

```bash
cd infra/
terraform init       # скачать провайдер local
terraform plan       # показать что создастся
terraform apply      # создать файл
terraform apply -var="environment=prod"  # с переменной
terraform destroy    # удалить
```

---

## Часть 3 — Реальная инфраструктура

Для практики будем использовать **Yandex Cloud** — есть бесплатный грант для новых аккаунтов. Концепции одинаковы для любого облака.

### Настройка провайдера

```bash
# Установить Yandex Cloud CLI
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
yc init    # авторизоваться

# Получить токен
yc iam create-token
```

```hcl
# infra/provider.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.100"
    }
  }

  # Remote state — хранить state не локально, а в облаке
  backend "s3" {
    endpoint = "storage.yandexcloud.net"
    bucket   = "my-terraform-state"
    key      = "todo-service/terraform.tfstate"
    region   = "ru-central1"

    skip_region_validation      = true
    skip_credentials_validation = true
  }
}

provider "yandex" {
  token     = var.yc_token
  cloud_id  = var.yc_cloud_id
  folder_id = var.yc_folder_id
  zone      = "ru-central1-a"
}
```

### Переменные

```hcl
# infra/variables.tf

variable "yc_token" {
  description = "Yandex Cloud IAM token"
  type        = string
  sensitive   = true    # не выводить в логах
}

variable "yc_cloud_id" {
  description = "Yandex Cloud ID"
  type        = string
}

variable "yc_folder_id" {
  description = "Yandex Cloud Folder ID"
  type        = string
}

variable "environment" {
  description = "Environment: dev, staging, prod"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "app_replicas" {
  description = "Number of application replicas"
  type        = number
  default     = 1
}
```

```hcl
# infra/terraform.tfvars  — в .gitignore!
yc_token     = "my-iam-token"
yc_cloud_id  = "b1g..."
yc_folder_id = "b1g..."
environment  = "dev"
```

### Сеть и подсети

```hcl
# infra/network.tf

resource "yandex_vpc_network" "main" {
  name = "todo-${var.environment}-network"
}

resource "yandex_vpc_subnet" "public" {
  name           = "todo-${var.environment}-public"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.main.id
  v4_cidr_blocks = ["10.0.1.0/24"]
}

resource "yandex_vpc_subnet" "private" {
  name           = "todo-${var.environment}-private"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.main.id
  v4_cidr_blocks = ["10.0.2.0/24"]
}
```

### Виртуальная машина

```hcl
# infra/compute.tf

# Данные об образе Ubuntu
data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2204-lts"
}

resource "yandex_compute_instance" "app" {
  count = var.app_replicas

  name        = "todo-app-${var.environment}-${count.index}"
  platform_id = "standard-v3"
  zone        = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2    # GB
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.id
      size     = 20    # GB
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    nat       = true    # внешний IP
  }

  metadata = {
    # cloud-init скрипт — выполнится при первом запуске
    user-data = templatefile("${path.module}/templates/cloud-init.yaml", {
      ssh_key     = file("~/.ssh/id_ed25519.pub")
      environment = var.environment
    })
  }

  lifecycle {
    create_before_destroy = true    # создать новый, потом удалить старый
  }

  labels = {
    environment = var.environment
    managed-by  = "terraform"
    service     = "todo-app"
  }
}
```

`templates/cloud-init.yaml`:
```yaml
#cloud-config
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - ${ssh_key}

packages:
  - docker.io
  - docker-compose-plugin

runcmd:
  - systemctl start docker
  - systemctl enable docker
  - usermod -aG docker ubuntu
  - mkdir -p /opt/todo-service
  - echo "ENVIRONMENT=${environment}" > /opt/todo-service/.env
```

### Managed PostgreSQL

```hcl
# infra/database.tf

resource "yandex_mdb_postgresql_cluster" "main" {
  name        = "todo-${var.environment}-postgres"
  environment = var.environment == "prod" ? "PRODUCTION" : "PRESTABLE"
  network_id  = yandex_vpc_network.main.id

  config {
    version = 16
    resources {
      resource_preset_id = "s2.micro"    # 2 vCPU, 8 GB RAM
      disk_type_id       = "network-ssd"
      disk_size          = 10            # GB
    }
  }

  host {
    zone      = "ru-central1-a"
    subnet_id = yandex_vpc_subnet.private.id
  }
}

resource "yandex_mdb_postgresql_database" "tododb" {
  cluster_id = yandex_mdb_postgresql_cluster.main.id
  name       = "tododb"
  owner      = yandex_mdb_postgresql_user.todouser.name
}

resource "yandex_mdb_postgresql_user" "todouser" {
  cluster_id = yandex_mdb_postgresql_cluster.main.id
  name       = "todouser"
  password   = var.db_password
}

variable "db_password" {
  description = "PostgreSQL password"
  type        = string
  sensitive   = true
}
```

### Outputs — что вернуть после apply

```hcl
# infra/outputs.tf

output "app_external_ips" {
  description = "External IP addresses of app servers"
  value       = yandex_compute_instance.app[*].network_interface[0].nat_ip_address
}

output "db_host" {
  description = "PostgreSQL cluster host"
  value       = yandex_mdb_postgresql_cluster.main.host[0].fqdn
  sensitive   = true    # не показывать в CLI output
}

output "database_url" {
  description = "Full DATABASE_URL"
  value       = "postgres://todouser:${var.db_password}@${yandex_mdb_postgresql_cluster.main.host[0].fqdn}:6432/tododb"
  sensitive   = true
}
```

```bash
# Посмотреть outputs
terraform output
terraform output app_external_ips
terraform output -json database_url    # показать sensitive значение
```

---

## Часть 4 — Модули

Модули — способ переиспользовать конфигурацию. Как функции в программировании.

### Структура с модулями

```
infra/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    ├── network/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── database/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```hcl
# infra/modules/compute/main.tf

variable "name" { type = string }
variable "environment" { type = string }
variable "subnet_id" { type = string }
variable "instance_count" {
  type    = number
  default = 1
}

resource "yandex_compute_instance" "app" {
  count = var.instance_count
  name  = "${var.name}-${var.environment}-${count.index}"
  # ...
}

output "external_ips" {
  value = yandex_compute_instance.app[*].network_interface[0].nat_ip_address
}
```

```hcl
# infra/main.tf — использование модуля

module "network" {
  source      = "./modules/network"
  environment = var.environment
}

module "app" {
  source         = "./modules/compute"
  name           = "todo-app"
  environment    = var.environment
  subnet_id      = module.network.public_subnet_id
  instance_count = var.app_replicas
}

module "database" {
  source      = "./modules/database"
  environment = var.environment
  network_id  = module.network.network_id
  subnet_id   = module.network.private_subnet_id
  db_password = var.db_password
}

# Использовать output одного модуля как input другого
output "database_url" {
  value     = "postgres://todouser:${var.db_password}@${module.database.host}:6432/tododb"
  sensitive = true
}
```

---

## Часть 5 — Terraform в CI/CD

### Структура окружений

```
infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf         # ссылается на корневые модули
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
└── modules/
    └── ...
```

### GitHub Actions для Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [ main ]
    paths: [ 'infra/**' ]
  pull_request:
    paths: [ 'infra/**' ]

jobs:
  terraform:
    name: Terraform Plan/Apply
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/environments/prod

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.YC_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.YC_SECRET_KEY }}

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        env:
          TF_VAR_yc_token: ${{ secrets.YC_TOKEN }}
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
        continue-on-error: true

      # Показать план в комментарии к PR
      - name: Comment PR with Plan
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const output = `#### Terraform Plan 📝
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      # Apply только в main ветке
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
        env:
          TF_VAR_yc_token: ${{ secrets.YC_TOKEN }}
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
```

### Лучшие практики

```hcl
# 1. Всегда используй remote state
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    # State locking через DynamoDB
    dynamodb_table = "terraform-state-lock"
  }
}

# 2. Никогда не коммить terraform.tfvars с секретами
# .gitignore:
# terraform.tfvars
# *.tfstate
# *.tfstate.backup
# .terraform/

# 3. Используй locals для повторяющихся значений
locals {
  common_tags = {
    environment = var.environment
    managed-by  = "terraform"
    service     = "todo-service"
  }
}

resource "yandex_compute_instance" "app" {
  labels = local.common_tags
}

# 4. Блокируй версии провайдеров
required_providers {
  yandex = {
    source  = "yandex-cloud/yandex"
    version = "= 0.105.0"    # точная версия в проде
  }
}

# 5. Используй lifecycle для защиты критичных ресурсов
resource "yandex_mdb_postgresql_cluster" "main" {
  lifecycle {
    prevent_destroy = true    # terraform destroy вернёт ошибку
  }
}
```

---

## Практические задания

### Задание 16.1 — Первый Terraform конфиг

Создай папку `infra/` в репозитории:

1. Напиши `main.tf` который создаёт локальные файлы:
   - `output/config.json` — JSON с конфигурацией окружения
   - `output/docker-compose.yml` — Compose файл сгенерированный из шаблона
2. Используй переменные: `environment`, `app_port`, `db_password`
3. Добавь output с путями к файлам

```bash
terraform init
terraform plan
terraform apply -var="environment=dev"
terraform apply -var="environment=prod" -var="app_port=9090"
```

**Что нужно сделать:** Убедись что при изменении переменной `environment` — пересоздаются файлы с новым содержимым. Изучи `terraform.tfstate` — что там хранится.

---

### Задание 16.2 — Remote State

Настрой хранение state в облаке:

**Вариант A (если есть Yandex Cloud):**
```bash
# Создать S3 бакет через CLI
yc storage bucket create --name my-terraform-state-$(date +%s)

# Настроить backend в main.tf
terraform {
  backend "s3" {
    endpoint = "storage.yandexcloud.net"
    bucket   = "my-terraform-state-..."
    key      = "dev/terraform.tfstate"
    ...
  }
}
terraform init -migrate-state
```

**Вариант B (локально через MinIO):**
```yaml
# docker-compose.yml дополнение
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
```

```hcl
terraform {
  backend "s3" {
    endpoint                    = "http://localhost:9000"
    bucket                      = "terraform-state"
    key                         = "dev/terraform.tfstate"
    region                      = "us-east-1"
    access_key                  = "minioadmin"
    secret_key                  = "minioadmin"
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}
```

---

### Задание 16.3 — Модули

Разбей конфигурацию на модули:

```
infra/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    ├── app-config/        # генерирует конфиги приложения
    └── monitoring-config/ # генерирует конфиги мониторинга
```

Каждый модуль должен:
- Принимать `environment` и `version` как входные переменные
- Генерировать соответствующие файлы конфигурации
- Возвращать пути к файлам через output

```hcl
module "app" {
  source      = "./modules/app-config"
  environment = var.environment
  version     = var.app_version
  db_host     = var.db_host
}

module "monitoring" {
  source      = "./modules/monitoring-config"
  environment = var.environment
  app_host    = module.app.internal_host
}
```

---

### Задание 16.4 — Terraform для Kubernetes

Terraform умеет управлять K8s ресурсами через провайдер `kubernetes`:

```hcl
# infra/k8s.tf

terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "minikube"
}

resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      "pod-security.kubernetes.io/enforce" = "baseline"
    }
  }
}

resource "kubernetes_config_map" "todo_config" {
  metadata {
    name      = "todo-config"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  data = {
    LOG_LEVEL = "info"
    PORT      = "8080"
  }
}

resource "kubernetes_secret" "todo_secrets" {
  metadata {
    name      = "todo-secrets"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  data = {
    database-url = base64encode(var.database_url)
  }
}

resource "kubernetes_deployment" "todo" {
  metadata {
    name      = "todo-service"
    namespace = kubernetes_namespace.production.metadata[0].name
  }
  spec {
    replicas = var.app_replicas
    selector {
      match_labels = { app = "todo-service" }
    }
    template {
      metadata {
        labels = { app = "todo-service" }
      }
      spec {
        container {
          name  = "todo-service"
          image = "ghcr.io/username/todo-service:${var.app_version}"
          env_from {
            config_map_ref { name = kubernetes_config_map.todo_config.metadata[0].name }
          }
        }
      }
    }
  }
}
```

**Что нужно сделать:** Создай namespace, ConfigMap и Deployment через Terraform. Убедись что `terraform apply` создаёт ресурсы в minikube. Измени `app_replicas` с 1 на 3 и выполни `terraform apply` — K8s должен масштабировать Deployment.

---

### Задание 16.5 — Финальный проект урока

Полная инфраструктура `todo-service` как код:

**Структура:**
```
infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf      # ссылается на модули
│   │   └── terraform.tfvars.example
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars.example
└── modules/
    ├── k8s-namespace/   # создать namespace с labels
    ├── k8s-app/         # Deployment + Service + ConfigMap + Secret
    └── k8s-monitoring/  # namespace для Prometheus + Grafana
```

**Требования:**
1. `dev` и `prod` — разные namespace, разные replicas, разные image tags
2. Remote state в MinIO (или облаке)
3. `terraform.tfvars` в `.gitignore`, есть `terraform.tfvars.example`
4. CI workflow: `terraform plan` в PR, `terraform apply` при merge в main
5. `Makefile` команды:
   ```makefile
   tf-plan-dev:
       cd infra/environments/dev && terraform plan

   tf-apply-dev:
       cd infra/environments/dev && terraform apply -auto-approve

   tf-destroy-dev:
       cd infra/environments/dev && terraform destroy
   ```

Закоммить с тегом `v1.0.0` — первая полноценная версия с полным IaC.

---

## Шпаргалка

| Команда | Что делает |
|---------|-----------|
| `terraform init` | Инициализировать, скачать провайдеры |
| `terraform validate` | Проверить синтаксис |
| `terraform plan` | Показать изменения |
| `terraform apply` | Применить изменения |
| `terraform apply -auto-approve` | Без подтверждения |
| `terraform destroy` | Удалить всё |
| `terraform output` | Показать outputs |
| `terraform state list` | Список ресурсов в state |
| `terraform state show RESOURCE` | Детали ресурса |
| `terraform import RESOURCE ID` | Импортировать существующий ресурс |
| `terraform fmt` | Отформатировать файлы |

---

## Ресурсы для изучения

- **Документация:** `https://developer.hashicorp.com/terraform/docs`
- **Registry провайдеров:** `https://registry.terraform.io`
- **Kubernetes провайдер:** `https://registry.terraform.io/providers/hashicorp/kubernetes`
- **Best practices:** `https://developer.hashicorp.com/terraform/language/style`
- **Книга:** "Terraform: Up and Running" — Yevgeniy Brikman

---

## Как понять что урок пройден

- [ ] Понимаю разницу между `plan` и `apply`
- [ ] Умею создавать ресурсы через `resource` блоки
- [ ] Переменные вынесены в `variables.tf`, значения в `tfvars`
- [ ] State хранится удалённо (не локальный файл)
- [ ] Конфигурация разбита на модули
- [ ] Terraform управляет K8s ресурсами через провайдер
- [ ] CI запускает `plan` при PR и `apply` при merge
- [ ] Секреты не хранятся в git

---

*Следующий урок: Безопасность — JWT, OWASP, Vault*
