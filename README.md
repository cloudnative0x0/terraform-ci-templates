# Структура Terraform-репозиториев

Репозитории разделены по ответственности: доступы, сети, вычисления, данные, наблюдаемость и продуктовые ресурсы. В платформенных репозиториях облака располагаются непосредственно в корне, без `live/`. В продуктовых репозиториях первым уровнем группировки является продукт.

## Шаблоны путей

| Репозиторий | Переиспользуемые модули | Terraform roots |
| --- | --- | --- |
| `tf-global-control-plane` | `modules/<cloud>/<module>/` | Варианты bootstrap и IAM приведены ниже |
| `tf-shared-network` | `modules/<cloud>/<module>/` | `<cloud>/<env>/<region>/<service>/` |
| `tf-infra` | `modules/<cloud>/<module>/` | `<cloud>/<workload>/<env>/<region>/<service>/` |
| `tf-data-plane` | `modules/<cloud>/<module>/` | `products/<product>/<cloud>/<env>/<region>/<datastore>/` |
| `tf-management-plane` | `modules/<cloud>/<module>/` | `<cloud>/<env>/<region>/<service>/` |
| `tf-products` | `modules/<cloud>/<module>/` | `products/<product>/<cloud>/<env>/<region>/<service>/` |
| `tf-vendor-integrations` | `modules/<vendor>/<module>/` | `<vendor>/organization/` или `<vendor>/environments/<env>/` |
| `tf-ci-templates` | Нет Terraform-модулей | Нет Terraform roots; код доставки: `.github/actions/<action>/` и `.github/workflows/<workflow>.yml` |

## Назначение репозиториев

### tf-global-control-plane

Облачный IAM baseline, deployment identities, федерации, политики и делегирование доступа. Также содержит обязательный bootstrap — основу для первоначального запуска Terraform CI.

```text
<cloud>/bootstrap/state/                 # Backend infrastructure, locking и защита state
<cloud>/bootstrap/ci/                    # Начальный trust и identity для доставки control plane
<cloud>/iam/platform/                    # Общий IAM baseline
<cloud>/iam/core/<env>/                  # IAM общего core по средам
<cloud>/iam/shared-network/<env>/        # IAM сетевой инфраструктуры по средам
<cloud>/iam/products/<product>/<env>/   # IAM конкретного продукта
```

Bootstrap имеет отдельные states и workflow. Первый запуск выполняется с исходным административным доступом; локальный state защищенно переносится в remote backend. Начальные identities не объявляются повторно в обычных IAM roots. Региональные уровни для IAM и bootstrap не обязательны.

### tf-shared-network

VPC/VNet, подсети, маршрутизация, NAT, межоблачная связность и общие DNS-зоны. Один network root владеет связанными сетевыми ресурсами своего deployment, включая подсети в других облачных каталогах.

Пример: `yc/prod/ru-central1/network/`. Общая нерегиональная DNS delegation при необходимости размещается в `<cloud>/dns/`.

### tf-infra

Kubernetes-кластеры, node/VM/GPU pools, общие registries и bootstrap кластеров. Приложения доставляются отдельным app/GitOps pipeline.

Пример: `yc/core/prod/ru-central1/kubernetes/`.

### tf-data-plane

Managed DB instances/clusters продуктов и общего core, HA, backup/PITR и DB-specific настройки защиты. Владелец — Platform/Data. Миграции бизнес-схем принадлежат delivery приложений.

Пример: `products/core/yc/prod/ru-central1/postgres-billing/`. Каждый независимый datastore получает отдельный root/state.

### tf-management-plane

Общие telemetry storage, retention, platform dashboards и мониторинг. Архив аудита имеет отдельные права и lifecycle.

Пример: `yc/prod/ru-central1/telemetry/`. Для аудита: `<cloud>/audit/<region>/archive/`. Настройки отправки логов на исходных ресурсах принадлежат владельцам этих ресурсов.

### tf-products

Buckets, queues, продуктовые CDN/DNS и разрешенные resource bindings. Владелец — продуктовая команда. Репозиторий потребляет инфраструктуру и DB endpoints; не управляет повторно кластерами из `tf-infra` или DB instances из `tf-data-plane`.

Пример: `products/product-a/yc/prod/ru-central1/storage-messaging/`. Для нерегионального edge: `products/<product>/<cloud>/<env>/edge/`.

### tf-vendor-integrations

Общие настройки внешних SaaS: organization baseline, SSO и shared integrations. Создается при появлении первой интеграции. Продуктовые SaaS-объекты при делегировании могут принадлежать `tf-products`, с единственным владельцем каждого объекта.

Примеры: `datadog/organization/`, `datadog/environments/prod/`.

### tf-ci-templates

Общие reusable workflows и actions для проверки, plan и apply Terraform. Не содержит облачных deployment roots или states. Приведенное назначение описывает целевой механизм доставки; недостающие workflows и поддержка реестра roots реализуются отдельно.

## Обозначения

- `<cloud>` — `yc`, `nebius`, `aws`, `azure`, `gcp`.
- `<env>` — `dev`, `stage`, `prod`.
- `<region>` — регион deployment, например `ru-central1`.
- `<workload>` — инфраструктурный deployment, например `core`.
- `<product>` — `core`, `product-a`, `product-b`; `core` обозначает общий бизнес-backend.
- `<service>` — независимо применяемый компонент: `network`, `kubernetes`, `registry`, `telemetry`.
- `<datastore>` — конкретный deployment хранилища: `postgres-primary`, `postgres-billing`.
- `<module>` — переиспользуемый строительный блок: `managed-postgresql`, `multi-folder-vpc`.
- `<vendor>` — внешний SaaS, например `datadog`, `pagerduty`.

## Общие правила

1. **Каждый конечный Terraform root — отдельный state и самостоятельный plan/apply.** Родительские каталоги только группируют roots. `modules/` содержит child modules без собственного backend и state.
2. Root обычно содержит `main.tf`, `backend.tf`, `providers.tf`, `versions.tf`, `.terraform.lock.hcl`; при необходимости — `variables.tf`, `outputs.tf`, `README.md`. Provider lockfile коммитится, state и credentials — нет.
3. Cloud/folder/project/account IDs задаются явно в конфигурации или `inventory/scopes.yaml`. Название пути не заменяет облачную изоляцию.
4. `stacks.yaml` перечисляет roots, стабильные IDs, target scopes и разрешенные auth/backend profiles. Это целевой контракт CI; текущий discovery нужно адаптировать. Перемещение каталога не должно автоматически менять backend key.
5. Один ресурс или IAM binding имеет одного владельца и один управляющий state. CODEOWNERS дополняется реальными ограничениями cloud/backend access.
6. Для нерегиональных ресурсов не добавляем искусственный регион. Общие ресурсы объявляются один раз. При нескольких однотипных deployments используются уникальные имена roots; для нескольких независимых tenants/accounts заранее согласуется дополнительный alias.
7. Создаем только необходимые deployments. Наличие `<cloud>` не означает обязательное развертывание каждого продукта во всех облаках.
