# Azure Databricks Platform Architect

Справочник по платформе Azure Databricks: определение, назначение и ограничения
каждого ресурса и правила, со ссылками на официальную документацию. Разделы
повторяют модули курса Azure Databricks Platform Architect.

## Оглавление

**[1. Workspace Administration Fundamentals](#1-workspace-administration-fundamentals)**

[First-Party Service on Azure](#first-party-service-on-azure) ·
[Platform Administrator and Platform Architect](#platform-administrator-and-platform-architect) ·
[Azure Hierarchical Structure](#azure-hierarchical-structure) ·
[Control Plane and Data Plane](#control-plane-and-data-plane) ·
[Lakehouse Reference Architecture](#lakehouse-reference-architecture) ·
[Medallion Architecture](#medallion-architecture) ·
[Cost Saving Features](#cost-saving-features) ·
[Workspace Planning Questions](#workspace-planning-questions) ·
[Requirements to Get Started](#requirements-to-get-started) ·
[Backing Resources for a Workspace](#backing-resources-for-a-workspace) ·
[Workspace Deployment Options](#workspace-deployment-options) ·
[Terraform for Azure Databricks](#terraform-for-azure-databricks)

**[2. Networking and Security Fundamentals](#2-networking-and-security-fundamentals)**

[Azure Active Directory](#azure-active-directory) ·
[System for Cross-Domain Identity Management (SCIM)](#system-for-cross-domain-identity-management-scim) ·
[External User Types in Azure Active Directory](#external-user-types-in-azure-active-directory) ·
[Unity Catalog](#unity-catalog) ·
[Software Defined Network (SDN)](#software-defined-network-sdn) ·
[Additional Networking Topics and Concepts](#additional-networking-topics-and-concepts) ·
[VNet Injection](#vnet-injection) ·
[Subnets](#subnets) ·
[Network Security Groups (NSG)](#network-security-groups-nsg) ·
[CIDR Ranges](#cidr-ranges) ·
[VNet Peering](#vnet-peering) ·
[Secure Cluster Connectivity (SCC)](#secure-cluster-connectivity-scc) ·
[IP Access Lists](#ip-access-lists) ·
[User-Defined Routes (UDR)](#user-defined-routes-udr) ·
[Data Exfiltration](#data-exfiltration) ·
[Azure Features for DLP](#azure-features-for-dlp) ·
[Azure Databricks Features for DLP](#azure-databricks-features-for-dlp) ·
[Additional DLP Topics](#additional-dlp-topics) ·
[Private Endpoints](#private-endpoints) ·
[Service Endpoints](#service-endpoints) ·
[Azure Private Link](#azure-private-link) ·
[Private Endpoints for an Azure Databricks Workspace](#private-endpoints-for-an-azure-databricks-workspace) ·
[Azure Databricks Private Endpoint Types](#azure-databricks-private-endpoint-types) ·
[Azure Private DNS](#azure-private-dns) ·
[Access Errors](#access-errors)

**[3. Cloud Integrations](#3-cloud-integrations)**

[Cloud Integrations Overview](#cloud-integrations-overview) ·
[Azure Storage Account Integration](#azure-storage-account-integration) ·
[Azure Data Factory Integration](#azure-data-factory-integration) ·
[Power BI Integration](#power-bi-integration)

**[Проверка знаний](#проверка-знаний)**

## 1. Workspace Administration Fundamentals

### First-Party Service on Azure

**Определение.** First-Party Service on Azure — модель поставки, в которой Azure
Databricks предоставляется как собственный сервис Azure, а не как сторонняя
подписка из marketplace.

**Назначение.** Workspace существует как ресурс Azure: оплата идёт через
подписку Azure, identity через Azure Active Directory, поддержка через Microsoft.
Отдельный договор с Databricks и отдельный счёт не нужны.

**Как работает.**

- Тип ресурса `Microsoft.Databricks/workspaces` в Azure Resource Manager.
- Стоимость DBU и инфраструктуры попадает в счёт подписки Azure.
- Аутентификация через Azure Active Directory, управление через Azure RBAC.
- Развёртывание из Azure Portal, Azure CLI, ARM template и Terraform.

**Документация.**
[What is Azure Databricks?](https://learn.microsoft.com/en-us/azure/databricks/introduction/) ·
[Azure Databricks architecture overview](https://learn.microsoft.com/en-us/azure/databricks/getting-started/overview)

### Platform Administrator and Platform Architect

**Назначение.** Роль отвечает за то, что нельзя изменить из ноутбука: сетевой
периметр, identity, governance и стоимость. Решения этого уровня принимаются до
развёртывания, поскольку часть из них исправляется только пересозданием
workspace.

**Как работает.**

- Проектирует сеть: VNet injection, SCC, private endpoint, DNS, egress.
- Настраивает identity: SCIM из Azure Active Directory, account admin, service principal.
- Отвечает за governance: metastore Unity Catalog, external location, права.
- Управляет стоимостью: tier, pre-purchase, auto-termination, autoscaling.
- Автоматизирует развёртывание через ARM template или Terraform.

**Документация.**
[Administration overview](https://learn.microsoft.com/en-us/azure/databricks/admin/) ·
[Azure Databricks account settings](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/)

### Azure Hierarchical Structure

<img width="1328" height="749" alt="image" src="https://github.com/user-attachments/assets/e4110356-7d71-4924-a646-7f588f1e791c" />

**Назначение.** Определяет, кто платит, где живут identity и на каком уровне
применяются права и квоты. Workspace Azure Databricks создаётся внутри
конкретной пары Subscription и Resource Group.

**Как работает.**

| Level | Роль |
| --- | --- |
| `Account` | биллинг Microsoft: кто платит |
| `Tenant` | Azure Active Directory (ныне Microsoft Entra ID): users, groups, SSO (Single Sign-On) |
| `Subscription` | счёт, Azure RBAC (Role-Based Access Control), квоты |
| `Resource Group` | контейнер жизненного цикла: деплой и удаление пачкой |
| `Resources` | VNet (Virtual Network), ADLS (Azure Data Lake Storage), Key Vault, Databricks workspace |

**Ограничения.** Квоты subscription на vCPU и публичные IP-адреса ограничивают
предельный размер кластеров.

**Документация.**
[Organize your Azure resources and hierarchy](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-setup-guide/organize-resources) ·
[What is Azure Resource Manager?](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview)

### Control Plane and Data Plane

<img width="1054" height="595" alt="image" src="https://github.com/user-attachments/assets/f9fd603c-7270-4174-a1c6-7b2ede65201e" />

**Определение.** Control Plane and Data Plane — разделение платформы на
control plane в подписке Azure Databricks и data plane в подписке заказчика.

**Назначение.** Управление отделено от обработки: Databricks отвечает за
управляющие сервисы, данные и вычисления остаются в облаке заказчика.

**Как работает.**

| Plane | Где | Состав |
| --- | --- | --- |
| Control plane | Azure Databricks Cloud | `Web App`, `Notebooks`, `Jobs`, `Metastore`, `Cluster Manager` |
| Data plane | Azure Customer Cloud | `Clusters`, `Data Sources`, `Secure Integrations` |

- `Metastore` хранит метаданные таблиц: имена и schema, а не сами файлы.
- Control plane и data plane обмениваются данными по TLS.
- Доступ к `Secure Integrations` идёт по TLS с проверкой identity.

**Документация.**
[Azure Databricks architecture overview](https://learn.microsoft.com/en-us/azure/databricks/getting-started/overview) ·
[Networking in Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/security/network/)

### Lakehouse Reference Architecture

<img width="1698" height="957" alt="image" src="https://github.com/user-attachments/assets/823a7ae8-5794-4aea-a11c-60ca622b778e" />

**Назначение.** Задаёт, какой сервис Azure отвечает за каждую стадию, чтобы
compute, storage и подача данных не смешивались в одном компоненте.

**Как работает.**

| Stage | Purpose | Services |
| --- | --- | --- |
| Data sources | откуда данные | sensors / IoT, logs, media, files, business apps |
| Ingest | как забрать | streaming: Event Hubs, IoT Central, Kafka; batch: Azure Data Factory |
| Store | куда класть сырьё | Azure Data Lake Storage (ADLS) |
| Process | где считать | Azure Databricks: Spark, MLflow, Azure ML |
| Serve | кому отдать | AKS, Cosmos DB, SQL Database, Synapse, Power BI, apps |

**Актуально.** Для ingest сегодня используются Auto Loader и Lakeflow Connect,
для serve — Databricks SQL и Mosaic AI Model Serving.

**Документация.**
[Lakehouse reference architectures](https://learn.microsoft.com/en-us/azure/databricks/lakehouse-architecture/reference) ·
[What is Auto Loader?](https://learn.microsoft.com/en-us/azure/databricks/ingestion/cloud-object-storage/auto-loader/)

### Medallion Architecture

<img width="1698" height="910" alt="image" src="https://github.com/user-attachments/assets/09c0fa8b-0c58-414a-b325-15309a39adbe" />

**Определение.** Medallion Architecture — многослойная модель данных, в которой
качество данных растёт при переходе Raw → Bronze → Silver → Gold.

**Назначение.** Каждый слой имеет свой контракт данных, поэтому ошибка
обработки исправляется переигрыванием слоя, а не повторным сбором источников.

**Как работает.**

| Layer | Где живёт | Контракт данных | Потребитель |
| --- | --- | --- | --- |
| Raw | файлы в ADLS | JSON, CSV, Parquet как пришли | только ingest |
| Bronze | Delta table | schema источника плюс технические поля | replay, audit, reprocess |
| Silver | Delta table | очищенные сущности, ключи, типы | engineering, DS, ML |
| Gold | Delta table | витрины под бизнес-вопрос | BI, apps, serving |
| Serve | вне озера | Cosmos DB, Synapse, Power BI, apps | конечные потребители |

- Bronze пишется «почти as-is» с полями `_ingest_time`, source, filename.
- Silver выполняет join, dedup, CDC merge и проверки качества; grain — сущность.
- Compute на всех переходах внутри озера — Azure Databricks.

**Документация.**
[What is the medallion lakehouse architecture?](https://learn.microsoft.com/en-us/azure/databricks/lakehouse/medallion) ·
[Delta Lake on Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/delta/)

### Cost Saving Features

<img width="1696" height="554" alt="image" src="https://github.com/user-attachments/assets/195f978f-d8f3-49c6-9b04-a73f10466bee" />

**Назначение.** Счёт складывается из DBU (Databricks Unit) и инфраструктуры
Azure. Механизмы уменьшают либо время работы compute, либо ставку за DBU.

**Как работает.**

- `Cluster Auto-shutdown` — остановка кластера после заданного простоя.
- `Elastic cluster sizes` — autoscaling числа worker в границах min и max.
- `Pre-purchase plan` — предоплата DBCU (Databricks Commit Units) на 1 или 3 года.
- `Standard vs. Premium tier` — Premium дороже за DBU и добавляет функции контроля доступа.

**Ограничения.** Pre-purchase покрывает только DBU: VM, storage и трафик
тарифицируются Azure отдельно.

**Документация.**
[Azure Databricks pricing](https://www.databricks.com/product/azure-pricing) ·
[Azure pricing calculator](https://azure.microsoft.com/en-us/pricing/calculator/) ·
[Save on Azure Databricks DBUs with pre-purchase](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/prepay-databricks-reserved-capacity) ·
[Compute configuration reference](https://learn.microsoft.com/en-us/azure/databricks/compute/configure)

### Workspace Planning Questions

<img width="1679" height="790" alt="image" src="https://github.com/user-attachments/assets/6f27d08a-1ec5-4ec5-a8e5-9ee3aadf34c3" />

**Назначение.** Сетевой режим, публичные IP и подключение к on-premises
задаются при развёртывании, поэтому ошибка на этом шаге исправляется
пересозданием workspace, а не изменением настройки.

**Как работает.**

- Назначение workspace и срок его существования.
- Нужен ли публичный IP; если нет — как организована связь.
- Требования безопасности и доступа.
- Наличие on-premises систем, с которыми нужна связь.
- Как доступ будет отслеживаться и ограничиваться; используется ли Unity Catalog.

**Документация.**
[Networking in Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/security/network/) ·
[Azure Databricks account settings](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/)

### Requirements to Get Started

<img width="1686" height="652" alt="image" src="https://github.com/user-attachments/assets/9276c8c2-f757-44c0-ae5c-60bb75ad5571" />

**Назначение.** Развёртывание запрашивает всё перечисленное сразу: отсутствие
VNet или storage account останавливает создание workspace.

**Как работает.**

- При VNet injection — существующая или новая VNet в Azure.
- Storage account (или несколько) для данных.
- Credentials и конфигурации дополнительных сервисов для интеграции.
- Имя workspace.
- Subscription, resource group и region для workspace.

**Документация.**
[Deploy Azure Databricks in your VNet (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)

### Backing Resources for a Workspace

**Назначение.** Часть ресурсов Azure создаёт сама платформа, часть готовит
администратор. Различать их обязательно: managed resource group изменять вручную
нельзя, а storage account и Key Vault полностью на стороне заказчика.

**Как работает.**

| Ресурс | Кто создаёт | Роль |
| --- | --- | --- |
| `Managed resource group` | Azure Databricks | VM, диски, NSG и служебный storage account workspace |
| `DBFS root storage account` | Azure Databricks | системное хранилище workspace |
| `ADLS Gen2 storage account` | заказчик | данные lakehouse, external location Unity Catalog |
| `Azure Databricks access connector` | заказчик | managed identity для доступа Unity Catalog к storage |
| `Key Vault` | заказчик | секреты для secret scope |

**Ограничения.** DBFS root не предназначен для продакшн-данных заказчика: их
держат в отдельном storage account.

**Документация.**
[DBFS root](https://learn.microsoft.com/en-us/azure/databricks/dbfs/root-locations) ·
[Managed identities for Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/azure-managed-identities) ·
[Secret management](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/)

### Workspace Deployment Options

**Назначение.** Сетевой режим выбирается на этом шаге и определяет, в какой сети
работают узлы кластера и какие средства контроля доступны организации.

**Варианты и выбор.**

| Вариант | Где работает compute | Когда выбирают |
| --- | --- | --- |
| `Default deployment` | managed VNet в подписке Databricks | быстрый старт, нет требований к сети |
| `VNet injection` | VNet заказчика | нужны NSG, UDR, private endpoint, связь с on-premises |

**Как работает.**

- Инструменты развёртывания: Azure Portal, Azure CLI, ARM template, Terraform.
- Тип ресурса в обоих вариантах один: `Microsoft.Databricks/workspaces`.
- Tier (`Standard`, `Premium`) выбирается там же и определяет доступные функции безопасности.

**Ограничения.** Смена сетевого режима существующего workspace не
поддерживается: создаётся новый workspace.

**Документация.**
[Deploy Azure Databricks in your VNet (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject) ·
[Manage workspaces](https://learn.microsoft.com/en-us/azure/databricks/admin/workspace/)

### Terraform for Azure Databricks

**Определение.** Terraform for Azure Databricks — управление workspace и его
объектами как infrastructure as code через Databricks Terraform provider.

**Назначение.** Развёртывание становится повторяемым: одна конфигурация создаёт
сеть, workspace и права, изменения проходят review и хранятся в системе версий.

**Как работает.**

- Provider `azurerm` создаёт VNet, subnet, NSG и сам ресурс workspace.
- Provider `databricks` настраивает объекты внутри: users, groups, clusters, jobs, permissions.
- State-файл позволяет увидеть diff до применения изменений.
- Один код разворачивает dev, test и prod, отличаясь только переменными.

**Документация.**
[Databricks Terraform provider](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/terraform/)

## 2. Networking and Security Fundamentals

### Azure Active Directory

**Определение.** Azure Active Directory (ныне Microsoft Entra ID) — служба
identity в Azure: users, groups, service principal и SSO для всех ресурсов
подписки.

**Назначение.** Единый источник identity: доступ к workspace выдаётся той же
учётной записи, что и к остальным ресурсам Azure, а отзыв в источнике закрывает
все пути сразу.

**Как работает.**

- Вход в workspace выполняется через Azure Active Directory.
- Azure RBAC на ресурсе workspace определяет, кто управляет им в Azure.
- SCIM синхронизирует users и groups на уровень account и workspace.
- Service principal используется для автоматизации и jobs вместо личной учётной записи.
- Токены Azure Active Directory применяются при обращении к REST API.

**Документация.**
[What is Microsoft Entra ID?](https://learn.microsoft.com/en-us/entra/fundamentals/whatis) ·
[Authentication and access control](https://learn.microsoft.com/en-us/azure/databricks/security/auth/) ·
[Service principals](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals)

### System for Cross-Domain Identity Management (SCIM)

<img width="1615" height="866" alt="image" src="https://github.com/user-attachments/assets/81026b6c-f69d-475d-812d-82a9b29dbf60" />

**Определение.** System for Cross-Domain Identity Management (SCIM) —
стандартный протокол синхронизации users, groups и service principal из Azure
Active Directory в Azure Databricks.

**Назначение.** Одна identity в Azure Active Directory становится account user
и далее workspace user в нескольких workspace. Без синхронизации учётные записи
создаются в каждом workspace вручную.

**Как работает.**

- Identity создаётся в Azure Active Directory.
- Provisioning передаёт её на уровень account: account user или service principal.
- С уровня account identity назначается в конкретные workspace.

**Ограничения.** Отзыв доступа выполняется в источнике: identity, оставшаяся в
workspace без синхронизации, сохраняет доступ.

**Документация.**
[Sync users and groups from Microsoft Entra ID using SCIM](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/scim/aad)

### External User Types in Azure Active Directory

<img width="1692" height="826" alt="image" src="https://github.com/user-attachments/assets/2e9fe40b-a145-4812-92e4-0decca54a580" />

**Определение.** External User Types in Azure Active Directory — два типа
внешних пользователей: B2B (Business-to-Business) и B2C (Business-to-Customer).

**Назначение.** Определяет, кому вообще можно выдать доступ к workspace:
подрядчику или партнёру — можно, потребительской identity — нет.

**Как работает.**

| Тип | Кто это | Доступ к Azure Databricks |
| --- | --- | --- |
| B2B | vendors, contractors, partners; добавляются в Azure Active Directory напрямую, роль Guest, вне домена | доступ возможен |
| B2C | аутентификация через сторонние системы (Facebook, Google, Apple), в Azure Active Directory не добавляются, ограниченный доступ | workspace использовать нельзя |

**Документация.**
[B2B collaboration overview](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b)

### Unity Catalog

<img width="1688" height="737" alt="image" src="https://github.com/user-attachments/assets/5cc16b29-e9b3-4d7b-bb52-73a0a4afba5f" />

**Определение.** Unity Catalog — централизованный уровень governance для данных
и AI-объектов: единый каталог, права доступа и аудит для всех workspace в account.

**Назначение.** Права определяются один раз на уровне account и действуют во
всех workspace, вместо повторной настройки доступа в каждом workspace.

**Как работает.**

- `Define once, secure everywhere` — единая модель прав для всех workspace.
- `Standards-compliant security model` — ANSI SQL GRANT на объекты каталога.
- `Built-in auditing` — журнал обращений к объектам.
- Metastore и права управляются из account console, а не из отдельного workspace.

**Ограничения.** Metastore создаётся на region: workspace подключается к
metastore своего региона.

**Документация.**
[What is Unity Catalog?](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/) ·
[Azure Databricks account settings](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/)

### Software Defined Network (SDN)

<img width="1663" height="592" alt="image" src="https://github.com/user-attachments/assets/1d590113-1f27-45df-8f78-774bfa99c52e" />

**Определение.** Software Defined Network (SDN) — сетевая модель, в которой
топология и политики задаются программно, а не настройкой оборудования.

**Назначение.** Сеть Azure создаётся и меняется как ресурс через API и
шаблоны, поэтому сетевой периметр workspace воспроизводим и версионируем.

**Как работает.** Три основных компонента:

- `Network Controller` — централизованное управление конфигурацией сети.
- `Software Load Balancer` — распределение трафика без физического устройства.
- `Gateway` — выход в другие сети: on-premises, другие VNet, интернет.

**Документация.**
[What is Azure Virtual Network?](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)

### Additional Networking Topics and Concepts

<img width="1682" height="891" alt="image" src="https://github.com/user-attachments/assets/a6255e6c-de3c-45a2-ac74-2a52177e5163" />

**Назначение.** Задаёт минимальный словарь сети: адресное пространство, размер
сети, связность, маршрутизация и два способа приватного доступа к сервисам.

**Как работает.**

| Термин | Определение | Подробно |
| --- | --- | --- |
| `RFC 1918` | непубличное, не маршрутизируемое в интернете адресное пространство: блоки `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | — |
| `CIDR` | адрес плюс число значащих бит, образующих маршрутную часть; альтернатива традиционным subnet mask | [CIDR Ranges](#cidr-ranges) |
| `Peering` | соединение двух и более VNet, после которого для связности они выглядят как одна сеть | [VNet Peering](#vnet-peering) |
| `Routes` | custom или user-defined (static) маршруты, переопределяющие системные маршруты Azure или дополняющие route table подсети | [User-Defined Routes (UDR)](#user-defined-routes-udr) |
| `Service Endpoints` | безопасное прямое подключение к сервисам Azure по оптимизированному маршруту через Azure backbone | [Service Endpoints](#service-endpoints) |
| `Private Endpoints` | сетевой интерфейс с частным адресом из VNet, подключающий сервис через Azure Private Link | [Private Endpoints](#private-endpoints) |

**Ограничения.** Адресные пространства связываемых сетей не должны
перекрываться, иначе адрес не разрешается однозначно.

**Документация.**
[What is Azure Virtual Network?](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) ·
[Networking in Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/security/network/)

### VNet Injection

<img width="1694" height="850" alt="image" src="https://github.com/user-attachments/assets/46d5ebea-f47e-4d44-b4e2-3f36a027a9d3" />

**Определение.** VNet Injection — развёртывание classic compute plane в VNet
подписки заказчика вместо управляемой сети Databricks.

**Назначение.** Даёт организации контроль над маршрутизацией, NSG (Network
Security Group), Private Endpoint и связью с on-premises. Без injection
периметр compute задаёт Databricks.

**Как работает.**

- VNet содержит две delegated subnet: private и public.
- Каждый узел получает NIC (Network Interface Card) с частным IP из subnet.
- NSG применяется на subnet, container rules — на контейнер узла.
- Связь с control plane идёт через relay и load balancer (NPIP v2).
- Peering связывает VNet с остальными ресурсами Azure по частным адресам.

**Ограничения.** Диапазоны VNet и subnet задаются при создании workspace и
определяют предел одновременно работающих узлов.

**Документация.**
[Deploy Azure Databricks in your VNet (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject) ·
[What is Azure Virtual Network?](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)

### Subnets

**Определение.** Subnets — диапазоны адресов внутри VNet, из которых узлы
получают частные IP.

**Назначение.** Даёт каждому узлу уникальную идентичность в периметре: control
plane адресует ему job, а узел открывает соединение к ADLS. Без уникального IP
узлы неразличимы.

**Как работает.**

- Диапазоны VNet и subnet задаёт администратор при создании workspace.
- Azure выдаёт свободный частный IP каждой NIC при старте кластера.
- При остановке кластера адреса возвращаются в пул.
- Workspace с VNet injection использует две delegated subnet.

**Ограничения.** Исчерпание адресов не даёт стартовать новым кластерам в пик
нагрузки; размер subnet после создания workspace не меняется.

**Документация.**
[Deploy Azure Databricks in your VNet (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)

### Network Security Groups (NSG)

**Определение.** Network Security Groups (NSG) — наборы правил allow и deny на
subnet или сетевом интерфейсе, фильтрующие трафик по источнику, назначению,
порту и протоколу.

**Назначение.** Определяет, разрешено ли соединение вообще. NSG решает,
*можно ли*, а UDR — *каким путём*: это разные средства и подменять одно другим
нельзя.

**Как работает.**

- Правила применяются к host и container subnet workspace VNet.
- Обязательные правила создаёт и поддерживает платформа: их удаление ломает кластеры.
- Собственные правила добавляются рядом, для доступа к корпоративным ресурсам.
- На subnet с private endpoint правила задаются отдельно от subnet кластеров.

**Ограничения.** При NSG policy на subnet private endpoint нужны inbound порты
443, 6666, 3306 и 8443-8451.

**Документация.**
[Network security groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview) ·
[Deploy Azure Databricks in your VNet (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)

### CIDR Ranges

**Определение.** CIDR Ranges (Classless Inter-Domain Routing) — диапазоны
адресного пространства в нотации `10.10.0.0/24`: адрес плюс число значащих бит,
образующих маршрутную часть. Альтернатива традиционным subnet mask.

**Назначение.** Задаёт вместимость VNet и subnet, то есть предельное число
одновременно работающих узлов.

**Как работает.** Rule of 4:

- Определить ожидаемое число одновременных узлов кластера.
- Умножить на 2 — public и private subnet.
- Умножить на 2 — запуск и остановка (flash-over).
- Итог примерно вчетверо больше числа одновременных узлов.
- Взять CIDR, дающий следующее по величине адресное пространство VNet.

**Ограничения.** Чем меньше число после `/`, тем больше адресов в пуле.

<img width="1688" height="611" alt="image" src="https://github.com/user-attachments/assets/cb91aea9-5f18-4960-a3ff-7756c55d0d02" />

**Документация.**
[Deploy Azure Databricks in your VNet (VNet injection)](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/vnet-inject)

### VNet Peering

<img width="1679" height="832" alt="image" src="https://github.com/user-attachments/assets/9c44a901-af44-493a-b8ba-26454381f9ab" />

**Определение.** VNet Peering — соединение двух и более VNet в Azure, после
которого для целей связности они выглядят как одна сеть.

**Назначение.** Даёт частный маршрут между контурами: кластер обращается к ADLS
или к другой VNet по частным IP, минуя публичный интернет.

**Варианты и выбор.**

| Вариант | Область |
| --- | --- |
| `Virtual Network Peering` | VNet в одном region |
| `Global Network Peering` | VNet в разных region |

**Как работает.**

- VNet не обязаны находиться в одной subscription.
- Peering подсетей происходит неявно, отдельной настройки не требует.
- Трафик остаётся в локальной сети и идёт по Azure backplane.

**Документация.**
[Virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)

### Secure Cluster Connectivity (SCC)

<img width="1701" height="834" alt="image" src="https://github.com/user-attachments/assets/f8e6a945-0f29-4675-9988-87a48faf0508" />

**Определение.** Secure Cluster Connectivity (SCC), также No Public IP (NPIP), —
режим, в котором узлы classic compute не имеют публичных IP-адресов и не
принимают входящие соединения.

**Назначение.** До SCC control plane обращался к VM data plane через публичный
интернет. SCC разворачивает поток: узел сам инициирует egress-соединение.

**Как работает.**

- Узлы получают только частные IP из subnet заказчика.
- Трафик к control plane идёт только на исходящее соединение узла.
- Снимается зависимость от квоты публичных IP-адресов.

**Варианты и выбор.** SCC вместе с VNet injection считается best practice:
такая комбинация даёт максимум безопасности и гибкости для бизнеса.

**Документация.**
[Secure cluster connectivity](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/secure-cluster-connectivity)

### IP Access Lists

**Определение.** IP Access Lists — списки разрешённых и запрещённых публичных
IP-диапазонов, из которых открывают UI и REST API workspace.

**Назначение.** Ограничивает, *откуда* приходит пользователь: вход возможен
только из корпоративной сети или VPN, даже если учётные данные скомпрометированы.

**Как работает.**

- Allow list задаёт разрешённые CIDR, block list — исключения внутри них.
- Правила действуют на front-end доступ: web application и REST API.
- Настраиваются на уровне workspace и требуют Premium plan.

**Варианты и выбор.** С front-end Private Link публичный вход отключают целиком,
и списки не нужны. IP access lists применяют, когда публичный вход остаётся
открытым и его требуется сузить.

**Ограничения.** Контролируют только вход в workspace, но не то, куда кластер
отправляет данные: исходящий путь закрывается сетевыми средствами.

**Документация.**
[Configure IP access lists for workspaces](https://learn.microsoft.com/en-us/azure/databricks/security/network/front-end/ip-access-list)

### User-Defined Routes (UDR)

**Определение.** User-Defined Routes (UDR) — пользовательская таблица маршрутов
на subnet, переопределяющая системные маршруты Azure: трафик к заданному
префиксу направляется на указанный next hop.

**Назначение.** Нужен, когда исходящий путь compute должен идти через выбранную
точку контроля — firewall, VPN gateway, NVA (Network Virtual Appliance), — а не
напрямую.

**Как работает.**

| Цель | Маршрут |
| --- | --- |
| Контроль исходящих данных | `0.0.0.0/0` → Azure Firewall |
| Доступ к корпоративным системам | CIDR on-premises → VPN или ExpressRoute |

- UDR применяется вместе с VNet injection, SCC и private endpoint к control plane.

**Ограничения.** Если весь трафик уходит на firewall без разрешённых маршрутов
к control plane и SCC, кластер не стартует.

**Документация.**
[User-defined route settings for Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/udr) ·
[Virtual network traffic routing](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)

### Data Exfiltration

**Определение.** Data Exfiltration — несанкционированный вынос данных за
пределы среды Azure Databricks.

**Назначение.** Тема определяет набор обязательных контролей для регулируемых
данных: без них открытый исходящий путь или широкий доступ к storage делают
вынос данных технически возможным.

**Как работает.** Основные векторы:

- открытый egress из compute;
- слишком широкий доступ к storage account;
- скомпрометированные credentials;
- выгрузка query results или запись во внешний контур.

**Варианты и выбор.** Сетевые меры (VNet injection, SCC, egress через firewall,
Private Link к storage) закрывают путь наружу; Unity Catalog ограничивает доступ
к объектам. IP access lists дополняют их контролем того, откуда открывают UI.

**Документация.**
[Networking in Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/security/network/)

### Azure Features for DLP

<img width="1685" height="537" alt="image" src="https://github.com/user-attachments/assets/8c6ac81f-8e44-44c2-afd0-c2f0aa7aba32" />

**Назначение.** Средства Azure для DLP (Data Loss Prevention) ограничивают
сетевые пути, по которым данные могут покинуть периметр, и дают наблюдаемость
исходящего трафика.

**Как работает.**

- `User Defined Routes (UDRs)` — направляют egress на точку контроля.
- `Azure Firewalls` — фильтруют исходящие соединения по правилам.
- `Network Behavior Analysis programs` — выявляют аномальный трафик.
- `Private Link (Private Endpoints)` — приватный доступ к сервисам.
- `Service Endpoints` — оптимизированный маршрут к сервисам Azure.

**Документация.**
[What is Azure Private Endpoint?](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) ·
[Virtual network traffic routing](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)

### Azure Databricks Features for DLP

<img width="1679" height="882" alt="image" src="https://github.com/user-attachments/assets/ae793d07-3ae7-4937-acbf-93734c6ef541" />

**Назначение.** Сетевые меры Azure не ограничивают, кто открывает workspace и
какие объекты он видит. `IP Access Lists` и `Unity Catalog` закрывают именно это.

**Как работает.**

- `IP Access Lists` ограничивают, с каких адресов открывают UI и API workspace.
- `Unity Catalog` управляет доступом к объектам данных из account console.
- Диапазоны Azure Active Directory берутся из service tag `AzureActiveDirectory`.

**Ограничения.** IP access lists контролируют вход в workspace, но не то, куда
кластер отправляет данные: исходящий путь закрывается сетевыми средствами.

**Документация.**
[Configure IP access lists for workspaces](https://learn.microsoft.com/en-us/azure/databricks/security/network/front-end/ip-access-list) ·
[What is Unity Catalog?](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/)

### Additional DLP Topics

<img width="1692" height="492" alt="image" src="https://github.com/user-attachments/assets/74faae11-a9f1-4b63-a131-e61acf975f91" />

**Назначение.** Принципы задают, как выдаются права, независимо от конкретного
сервиса: сначала подтверждение identity, затем минимально достаточные права.

**Как работает.**

- Authentication подтверждает identity, authorization определяет разрешённые действия.
- Least privilege выдаёт только права, необходимые для задачи.
- Zero trust не считает сеть доверенной: каждое обращение проверяется.

**Документация.**
[Azure Databricks account settings](https://learn.microsoft.com/en-us/azure/databricks/admin/account-settings/)

### Private Endpoints

<img width="1699" height="834" alt="image" src="https://github.com/user-attachments/assets/ae4e29a0-7450-4b3f-9748-6a03a88d944e" />

**Определение.** Private Endpoints — сетевые интерфейсы с частными адресами
RFC 1918 из VNet заказчика, создающие приватное подключение к сервису через
Azure Private Link.

**Назначение.** Даёт публичному ресурсу локальную идентичность внутри сети:
обращение к нему идёт по частному IP, доступному в локальной сети.

**Как работает.**

- Занимает частный IP из subnet заказчика и имеет собственный NIC.
- Создаёт приватное подключение к сервису, доступное внутри локальной сети.
- Работает и для доступа из on-premises через ExpressRoute или VPN.

**Ограничения.** Не убирает публичный доступ к самому ресурсу: публичный
доступ отключается отдельной настройкой на ресурсе.

**Документация.**
[What is Azure Private Endpoint?](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)

### Service Endpoints

<img width="1688" height="904" alt="image" src="https://github.com/user-attachments/assets/aa3bc8c9-01e2-472c-8e76-26487cf15ae4" />

**Определение.** Service Endpoints — настройка subnet, направляющая трафик к
сервисам Azure по оптимизированному маршруту через Azure backbone.

**Назначение.** Ограничивает доступ к storage account адресами конкретной
subnet, не выделяя сервису частный IP в сети заказчика.

**Как работает.**

- Включается на subnet VNet, там же где `Delegate subnet to a service`.
- Обходит default route, SNAT и firewall.
- Бесплатная функция Azure: не требует DNS и не создаёт NIC.
- Управляет только исходящим трафиком.

**Варианты и выбор.**

| Средство | Что даёт | Когда выбирают |
| --- | --- | --- |
| `Service Endpoint` | оптимизированный маршрут от subnet к сервису | достаточно ограничить доступ адресами subnet |
| `Private Endpoint` | частный IP сервиса в VNet | нужен приватный адрес и доступ из on-premises |

**Ограничения.** Нельзя включить в on-premises сетях, только в VNet.

**Документация.**
[Virtual Network service endpoints](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)

### Azure Private Link

<img width="1690" height="898" alt="image" src="https://github.com/user-attachments/assets/de0fc6fa-a5d8-480a-8fa5-858862855075" />
<img width="1689" height="844" alt="image" src="https://github.com/user-attachments/assets/548abe37-af22-4db4-9434-de11c0bc33f8" />

**Определение.** Azure Private Link — приватное подключение к Azure Databricks
через private endpoint с адресом из VNet заказчика.

**Назначение.** Убирает публичные URL из пути доступа: трафик идёт по
корпоративной сети и Azure backplane, не выходя в интернет.

**Варианты и выбор.**

| Option | Путь | Когда применяют |
| --- | --- | --- |
| 1. End User to Control Plane | пользователь → ExpressRoute или VPN → transit VNet → private endpoint → control plane | workspace нельзя открывать по публичным URL |
| 2. Data Plane to Control Plane | clusters в customer VNet → private endpoint → control plane, metastore, artifact store | control plane доступен только по частным IP |

- В option 2 правила firewall для доступа к control plane не нужны.
- В обоих вариантах трафик маршрутизируется по Azure backplane.

**Актуально.** К этим двум вариантам добавился третий — outbound Private Link от
serverless compute к ресурсам Azure заказчика.

**Документация.**
[Enable Azure Private Link](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/private-link)

### Private Endpoints for an Azure Databricks Workspace

<img width="3324" height="1778" alt="image" src="https://github.com/user-attachments/assets/d6576a16-30f3-4342-be7e-7acd8292f826" />

**Определение.** Private Endpoints for an Azure Databricks Workspace — private
endpoint в подписке заказчика, через которые workspace и его зависимости
доступны по частным IP.

**Назначение.** Control plane остаётся доступен только по частным адресам,
поэтому правила firewall для доступа к control plane не нужны.

**Как работает.**

- `NPIP Relay + Webapp PE` — общий endpoint к webapp и SCC relay control plane.
- `DBFS Storage PE` — приватный доступ к DBFS storage account.
- `External Metastore PE` — приватный доступ к внешнему metastore на SQL.
- Endpoint размещают в hub или bastion VNet либо в отдельной subnet workspace VNet.
- Кластеры работают в workspace VNet в host и container subnet, трафик по TLS 1.2/1.3.

**Ограничения.** Front-end и back-end требуют Premium plan, VNet injection и SCC.

**Документация.**
[Private Link concepts](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/private-link) ·
[Enable Azure Private Link](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/private-link-standard)

### Azure Databricks Private Endpoint Types

**Определение.** Azure Databricks Private Endpoint Types — четыре типа private
endpoint, различающиеся значением sub-resource, которое задаёт закрываемый
трафик.

**Назначение.** Sub-resource определяет, какой путь пойдёт через endpoint:
неверное значение оставляет часть трафика на публичном маршруте.

**Как работает.**

- `databricks_ui_api` — REST API и web application; front-end и back-end.
- `browser_authentication` — SSO-логины в браузере: callbacks от Microsoft Entra ID.
- `general_access` — workspace и ресурсы account, custom URL, разные region.
- `service_direct` — доступ к performance-intensive services.
- Front-end endpoint размещаются в transit VNet, classic — в отдельной subnet workspace VNet.

**Ограничения.**

- Один `browser_authentication` endpoint на region и private DNS zone: удаление workspace-хоста ломает web login остальным workspace региона.
- При NSG policy на subnet private endpoint нужны inbound порты 443, 6666, 3306 и 8443-8451.

**Документация.**
[Private Link concepts](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/private-link) ·
[Azure Private Endpoint private DNS zone values](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)

### Azure Private DNS

<img width="3376" height="1290" alt="image" src="https://github.com/user-attachments/assets/ac7332e5-456f-43c4-a5d5-8776e0325f8a" />

**Определение.** Azure Private DNS — служба DNS для VNet, в которой private DNS
zone разрешает FQDN (Fully Qualified Domain Name) сервиса в частный адрес private
endpoint.

**Назначение.** FQDN по умолчанию разрешается в публичный IP: без записи в зоне
клиент уходит на публичный адрес, и private endpoint не используется.

**Как работает.**

- Зона `privatelink.azuredatabricks.net` создаётся для поддержки private link.
- Запись для адреса workspace `adb-<id>.<n>.azuredatabricks.net` указывает на частный IP endpoint.
- OAuth CNAME для `<region>.pl-auth.azuredatabricks.net` включает авторизацию пользователя.
- Private link нужен и для data plane с control plane, и для пользователя к control plane.
- Зона должна быть назначена VNet через virtual network link, иначе не работает.

**Ограничения.** Conditional forwarding настраивается на публичную зону
`azuredatabricks.net`, а не на `privatelink.azuredatabricks.net`.

**Документация.**
[What is Azure Private DNS?](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview) ·
[Azure Private Endpoint DNS integration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration) ·
[Azure Private Endpoint private DNS zone values](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns)

### Access Errors

<img width="3276" height="1782" alt="image" src="https://github.com/user-attachments/assets/d9c94bb5-3d77-4670-b652-feb194cebe87" />

**Назначение.** Оба отказа выглядят как недоступность сайта, но причина сетевая:
имя не разрешается, потому что запись в private DNS zone отсутствует или зона не
связана с VNet.

**Как работает.**

| Отказ | URL в браузере | Что не разрешилось |
| --- | --- | --- |
| `Workspace Denial` | `adb-<id>.<n>.azuredatabricks.net` | адрес workspace |
| `OAuth Denial` | `<region>.pl-auth.azuredatabricks.net/aad/redirect` | адрес OAuth-редиректа |

- Браузер показывает `This site can't be reached` и код `DNS_PROBE_FINISHED_NXDOMAIN`.
- `OAuth Denial` возникает при рабочем адресе workspace: логин доходит до редиректа и обрывается.

**Документация.**
[Azure Private Endpoint DNS integration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns-integration) ·
[Private Link concepts](https://learn.microsoft.com/en-us/azure/databricks/security/network/classic/private-link)

## 3. Cloud Integrations

### Cloud Integrations Overview

**Назначение.** Workspace не хранит данные и не оркеструет конвейеры целиком:
storage, orchestration и BI подключаются как отдельные сервисы Azure. Способ
подключения определяет, где живут credentials и кто отвечает за права.

**Как работает.** Общие способы подключения:

- `Storage credential` и `external location` Unity Catalog — доступ к storage по managed identity.
- `Service principal` или managed identity — аутентификация сервиса без личной учётной записи.
- `Secret scope` на Key Vault — хранение ключей и строк подключения.
- `SQL warehouse` с JDBC или ODBC — точка подключения BI-инструментов.
- `REST API` и `jobs` — запуск задач из внешних оркестраторов.

**Документация.**
[Integrations](https://learn.microsoft.com/en-us/azure/databricks/integrations/) ·
[Secret management](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/)

### Azure Storage Account Integration

**Определение.** Azure Storage Account Integration — подключение ADLS Gen2 к
workspace как основного хранилища данных lakehouse.

**Назначение.** DBFS root не предназначен для продакшн-данных, поэтому данные
держат в отдельном storage account: жизненный цикл, права и стоимость
управляются вне workspace.

**Как работает.**

- Создаётся `Azure Databricks access connector` с managed identity.
- Identity получает роль `Storage Blob Data Contributor` на storage account.
- В Unity Catalog создаётся storage credential, затем external location.
- Данные адресуются по `abfss://<container>@<account>.dfs.core.windows.net/`.
- Приватный доступ обеспечивают private endpoint или service endpoint.

**Ограничения.** Требуется storage account ADLS Gen2 с включённым hierarchical
namespace.

**Документация.**
[Connect to Azure Data Lake Storage](https://learn.microsoft.com/en-us/azure/databricks/connect/storage/azure-storage) ·
[Managed identities for Unity Catalog](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/cloud-storage/azure-managed-identities) ·
[External locations](https://learn.microsoft.com/en-us/azure/databricks/connect/unity-catalog/external-locations)

### Azure Data Factory Integration

**Определение.** Azure Data Factory Integration — запуск notebook, JAR или
Python-задачи Azure Databricks как activity в pipeline Azure Data Factory.

**Назначение.** Data Factory отвечает за расписание, зависимости и перезапуски
по всему ландшафту Azure, Databricks — за вычисления. Без внешнего оркестратора
связь с шагами вне Databricks собирается вручную.

**Как работает.**

- В Data Factory создаётся linked service к workspace.
- Аутентификация: access token, managed identity или service principal.
- Activity `Notebook`, `Jar` или `Python` выполняется на job cluster или существующем кластере.
- Параметры передаются в notebook, результат возвращается в pipeline.

**Документация.**
[Transform data by running a Databricks notebook](https://learn.microsoft.com/en-us/azure/data-factory/transform-data-using-databricks-notebook) ·
[Jobs](https://learn.microsoft.com/en-us/azure/databricks/jobs/)

### Power BI Integration

**Определение.** Power BI Integration — подключение Power BI к Azure Databricks
через встроенный connector к SQL warehouse или кластеру.

**Назначение.** Отчёты читают Gold-слой прямо из lakehouse: отдельная копия
данных в BI-базе не нужна, а права остаются за Unity Catalog.

**Как работает.**

- Источник — SQL warehouse или кластер; берутся `Server hostname` и `HTTP path`.
- Аутентификация через Azure Active Directory или personal access token.
- `DirectQuery` даёт актуальность, `Import` — скорость отчёта.
- Публикация из Unity Catalog создаёт semantic model по выбранным таблицам.

**Ограничения.** Warehouse должен быть запущен: на остановленном отчёт
`DirectQuery` не обновится.

**Документация.**
[Connect Power BI to Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/partners/bi/power-bi) ·
[SQL warehouses](https://learn.microsoft.com/en-us/azure/databricks/compute/sql-warehouse/)

## Проверка знаний

Контрольные вопросы курса с краткими ответами. Подробности — в темах выше.

### 1. Workspace Administration Fundamentals

**Explain the first-party service relationship Databricks has with Microsoft.**
Azure Databricks — first-party сервис Azure: тип ресурса
`Microsoft.Databricks/workspaces`, биллинг через подписку Azure, identity через
Azure Active Directory, поддержка через Microsoft. Отдельного договора с
Databricks не требуется.

**Identify the responsibilities of the Platform Administrator/Platform Architect.**
Сетевой периметр (VNet injection, SCC, private endpoint, DNS), identity (SCIM,
account admin, service principal), governance (Unity Catalog, external location)
и стоимость (tier, pre-purchase, auto-termination).

**Describe foundational concepts of the Azure cloud ecosystem.**
Иерархия Account → Tenant → Subscription → Resource Group → Resources: кто
платит, где identity, где квоты и Azure RBAC, в каком контейнере живут ресурсы.

**Describe additional resources that may be included with Azure Databricks.**
ADLS Gen2 для данных, VNet и NSG для сети, Key Vault для секретов, access
connector для managed identity, Event Hubs или Data Factory для ingest,
Power BI и Cosmos DB для serve.

**Recognize the impact of Azure Databricks on cost management and planning.**
Счёт складывается из DBU и инфраструктуры Azure. Снижают стоимость
auto-shutdown, autoscaling, pre-purchase DBCU и выбор tier; VM, storage и трафик
тарифицируются отдельно.

**Review the decisions necessary to implement Azure Databricks.**
Назначение и срок жизни workspace, нужен ли публичный IP, требования
безопасности и доступа, связь с on-premises, мониторинг доступа, использование
Unity Catalog.

**Identify the resources needed to implement a workspace.**
VNet при VNet injection, storage account, credentials интеграций, имя workspace,
subscription, resource group и region.

**Create necessary backing resources for Azure Databricks.**
Платформа создаёт managed resource group и DBFS root storage account. Заказчик
готовит ADLS Gen2, access connector с managed identity, Key Vault и VNet.

**Differentiate between the available options for deploying a workspace.**
`Default deployment` размещает compute в managed VNet Databricks;
`VNet injection` — в VNet заказчика, что открывает NSG, UDR, private endpoint и
связь с on-premises. Сменить режим у существующего workspace нельзя.

**Determine the impact of networking for Azure on your workspace.**
Размеры VNet и subnet задают предел одновременных узлов, NSG определяет
разрешённые соединения, UDR — путь egress, private endpoint и DNS — доступность
control plane.

**Deploy an Azure Databricks workspace using the default method.**
Создание ресурса `Azure Databricks` в портале: subscription, resource group,
имя, region, tier. Сеть создаёт платформа, дополнительная настройка не нужна.

**Deploy an Azure Databricks workspace with VNet Injection.**
Заранее готовятся VNet и две delegated subnet с NSG, затем при создании
workspace выбирается собственная VNet и указываются host и container subnet.

**Describe how Terraform can automate the deployment process.**
Provider `azurerm` создаёт сеть и сам workspace, provider `databricks` —
объекты внутри (users, groups, clusters, jobs, permissions). Конфигурация
версионируется, а diff виден до применения.

### 2. Networking and Security Fundamentals

**Describe the impact of Azure Active Directory on identity and access control.**
Azure Active Directory — единый источник identity: users, groups, service
principal и SSO. Отзыв доступа в источнике закрывает все пути.

**Explain how AAD is used by Azure Databricks.**
Вход в workspace идёт через Azure Active Directory, Azure RBAC определяет
управление ресурсом, SCIM синхронизирует users и groups, токены применяются к
REST API, service principal — для автоматизации.

**Describe how Unity Catalog is used for account administration and control.**
Metastore и права живут на уровне account: политика задаётся один раз и
действует во всех workspace, доступ выдаётся через GRANT, обращения пишутся в
журнал аудита.

**Describe Azure Software Defined Networks.**
Сеть задаётся программно, а не настройкой оборудования. Три компонента:
`Network Controller`, `Software Load Balancer`, `Gateway`.

**Explain the value Azure SDNs bring to cloud computing with Azure Databricks.**
Периметр workspace создаётся и меняется через API и шаблоны, поэтому он
воспроизводим, версионируем и одинаков для dev, test и prod.

**Describe the purpose of subnets in a VNet used by Azure Databricks.**
Из subnet каждый узел получает уникальный частный IP: control plane адресует ему
job, узел открывает соединение к storage. Workspace с VNet injection использует
host и container subnet.

**Explain how Network Security Groups are used to control access and traffic.**
NSG — правила allow и deny на subnet: определяют, разрешено ли соединение.
Обязательные правила создаёт платформа, их удаление ломает кластеры.

**Set appropriate CIDR ranges for your Azure Databricks VNet.**
Rule of 4: ожидаемое число одновременных узлов умножить на 2 (public и private
subnet) и ещё на 2 (запуск и остановка), затем взять следующий по величине CIDR.

**Explain how these ranges impact your subnet sizes.**
Меньшее число после `/` даёт больше адресов. Нехватка адресов не даёт стартовать
новым кластерам в пик, а размер subnet после создания workspace не меняется.

**Define peering in the context of Azure SDNs.**
Соединение двух и более VNet, после которого для связности они выглядят как одна
сеть: `Virtual Network Peering` в одном регионе, `Global Network Peering` — между
регионами.

**Describe how peering is used in a VNet for Azure Databricks.**
Даёт кластерам частный маршрут к ADLS, другим VNet и корпоративным системам по
частным IP через Azure backplane, без выхода в интернет.

**Describe what an IP Access list is and how it is used.**
Списки разрешённых и запрещённых публичных IP-диапазонов для UI и REST API
workspace: вход только из корпоративной сети или VPN. Требуется Premium plan.

**Explain how IP Access lists work with Private Link.**
С front-end Private Link публичный вход отключают целиком, и списки не нужны.
Списки применяют, когда публичный вход остаётся открытым и его надо сузить.

**Describe User Defined Route (UDR) tables.**
Таблица маршрутов на subnet, переопределяющая системные маршруты Azure: трафик к
заданному префиксу идёт на указанный next hop — firewall, VPN gateway, NVA.

**Explain use cases where UDRs are applicable with Azure Databricks.**
Контроль исходящих данных через `0.0.0.0/0` на Azure Firewall и доступ к
on-premises через VPN или ExpressRoute по CIDR корпоративной сети.

**List the features available that support UDRs with Azure Databricks.**
VNet injection, SCC, Azure Firewall, private endpoint к control plane, service
endpoint к storage.

**Define Data Loss Prevention (DLP) and Data Exfiltration.**
DLP — набор мер против утечки данных. Data exfiltration — несанкционированный
вынос данных за пределы среды: открытый egress, широкий доступ к storage,
скомпрометированные credentials, выгрузка результатов запросов.

**Describe the tools and features available in Azure and Azure Databricks for DLP.**
Azure: UDR, Azure Firewall, network behavior analysis, private endpoint, service
endpoint. Azure Databricks: IP access lists и Unity Catalog.

**Differentiate between private and service endpoints.**
Private endpoint — частный IP сервиса в VNet, есть NIC, нужен DNS, работает из
on-premises. Service endpoint — маршрут от subnet к сервису, без NIC и DNS,
бесплатен, только внутри VNet и только для egress.

**Describe how Azure Databricks can use both types.**
Private endpoint закрывает доступ к control plane, DBFS storage и внешнему
metastore. Service endpoint ограничивает доступ к storage account адресами
subnet кластеров.

**Establish a private endpoint between control plane and data plane.**
В подписке заказчика создаётся private endpoint с sub-resource
`databricks_ui_api` в отдельной subnet, control plane становится доступен только
по частным IP, правила firewall для него не нужны. Требуются Premium plan, VNet
injection и SCC.

**Identify challenges of using private endpoints and how to accommodate them.**
Публичный доступ к ресурсу не отключается автоматически; `browser_authentication`
endpoint один на регион и private DNS zone, и удаление workspace-хоста ломает
вход остальным; при NSG policy нужны порты 443, 6666, 3306 и 8443-8451; без
записей DNS появляются ошибки NXDOMAIN.

**Explain why an Azure DNS is important for Private Link enabled workspaces.**
FQDN по умолчанию разрешается в публичный IP. Без private DNS zone, назначенной
VNet, клиент уходит на публичный адрес, и private endpoint не используется.

**Describe the key records to include in an Azure DNS for a workspace.**
Зона `privatelink.azuredatabricks.net`: запись для адреса workspace
`adb-<id>.<n>.azuredatabricks.net` на частный IP endpoint и OAuth CNAME для
`<region>.pl-auth.azuredatabricks.net`.

### 3. Cloud Integrations

**Describe the basics of establishing cloud integrations with a workspace.**
Внешний сервис получает identity (managed identity или service principal),
секреты хранятся в Key Vault через secret scope, доступ к данным описывается
external location Unity Catalog, сетевой путь закрывается endpoint.

**Identify common integration methods used in a workspace.**
Storage credential и external location, managed identity или service principal,
secret scope на Key Vault, SQL warehouse по JDBC или ODBC, REST API и jobs.

**Explain the importance of integrating a storage account with Azure Databricks.**
DBFS root не предназначен для продакшн-данных. Отдельный ADLS Gen2 отделяет
жизненный цикл, права и стоимость данных от жизненного цикла workspace.

**Connect an Azure storage account to a workspace.**
Создать access connector с managed identity, выдать ей `Storage Blob Data
Contributor` на storage account, создать storage credential и external location
в Unity Catalog, обращаться по `abfss://`.

**Explain why you would use Azure Data Factory with Azure Databricks.**
Data Factory даёт расписание, зависимости и перезапуски по всему ландшафту
Azure, включая шаги вне Databricks; Databricks выполняет вычисления.

**Connect Azure Data Factory to a workspace.**
Создать linked service к workspace с аутентификацией по access token, managed
identity или service principal, затем добавить activity `Notebook`, `Jar` или
`Python` с параметрами.

**Explain why you would use Power BI with Azure Databricks.**
Отчёты читают Gold-слой прямо из lakehouse: копия данных в BI-базе не нужна,
права остаются за Unity Catalog.

**Connect Power BI to a workspace.**
Взять `Server hostname` и `HTTP path` у SQL warehouse или кластера,
аутентифицироваться через Azure Active Directory или personal access token,
выбрать `DirectQuery` или `Import`.
