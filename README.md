# Databricks-architect-certification
## AZURE 
## Account structure
<img width="1328" height="749" alt="image" src="https://github.com/user-attachments/assets/e4110356-7d71-4924-a646-7f588f1e791c" />


| Layer | Descr |
|---|---|
| Account | биллинг Microsoft (кто платит) |
| Tenant | Entra ID: users, groups, SSO |
| Subscription | счёт, Azure RBAC, квоты |
| Resource Group | папка: деплой / удаление пачкой |
| Resources | VNet, ADLS, Key Vault, Databricks workspace |
              
## Control plane and data plane
<img width="1054" height="595" alt="image" src="https://github.com/user-attachments/assets/f9fd603c-7270-4174-a1c6-7b2ede65201e" />

Сontrol plane:
- Web app — UI workspace
- Notebooks — исходный код
- Jobs — scheduler
- Metastore — metadata таблиц (имена, schema), не файлы
- Cluster manager — create/stop clusters
  
Data plane:
- Clusters — Spark-ноды
- Data sources — DB, lake, files, blobs
- Secure integrations — BI, keys, identity (RBAC)

## Reference architecture
<img width="1698" height="957" alt="image" src="https://github.com/user-attachments/assets/823a7ae8-5794-4aea-a11c-60ca622b778e" />

Поток: Data sources → Ingest → Store → Process → Serve → Apps / Power BI
| Stage | Purpose | Services |
|---|---|---|
| Data sources | Откуда данные | IoT, logs, media, files, business apps |
| Ingest | Как забрать | streaming: Event Hub, IoT Central, Kafka; batch: Azure Data Factory; Today: Auto Loader / Lakeflow Connect |
| Store | Куда класть сырьё | Azure Data Lake Storage (ADLS) |
| Process | Где считать | Azure Databricks: Spark, MLflow, Azure ML |
| Serve | Кому отдать | AKS, Cosmos DB, SQL DB, Synapse, Power BI, apps; Today: DBSQL / Model Serving |

**Data sources**
1. Sensors / IoT — unstructured
2. Logs — unstructured
3. Media — unstructured
4. Files — unstructured
5. Business / custom apps — structured
   
**Ingest**
1. Streaming / real-time → Event Hub, IoT Central, Kafka
2. Batch / orchestration → Azure Data Factory (ещё и запускает jobs в Databricks)
   
**Store**
1. Всё сходится в ADLS — landing / lake
2. Databricks читает и пишет обратно в ADLS

**Process (ядро слайда)**
1. Azure Databricks
2. Spark — distributed compute
3. MLflow + Azure ML — ML lifecycle
4. Ad-hoc analysis прямо из Databricks
   
**Serve**
1. Model serving → AKS
2. Operational DB → Cosmos DB, SQL DB → apps
3. Data warehouse → Azure Synapse → Power BI
4. Apps и Power BI — конечные потребители

<img width="1698" height="910" alt="image" src="https://github.com/user-attachments/assets/09c0fa8b-0c58-414a-b325-15309a39adbe" />

Flow:
1. Load raw data to ADLS (файлы as-is)
2. Azure Databricks: raw → Bronze Table (Delta)
3. Azure Databricks: Bronze → Silver → Gold (Delta)
4. Load Gold into serving layers
   
| Layer | Where lives | Data contract | Target |
|---|---|---|---|
| Raw | файлы в ADLS | нет таблицы; JSON/CSV/Parquet как пришли | только ingest |
| Bronze | Bronze Table, Delta | schema источника + техполя | replay, audit, reprocess |
| Silver | Silver Table, Delta | очищенные сущности, ключи, типы | engineering, DS, ML |
| Gold | Gold Table, Delta | витрины под бизнес-вопрос | BI, apps, serving |
| Serve | вне озера | Cosmos DB, Synapse (Polybase), Power BI, Apps | потребители |

Что делает каждый слой:
1. **Raw** — dump в ADLS. Ещё не Delta. Streaming: Event Hub / IoT Hub / Kafka. Batch: Azure Data Factory.
2. **Bronze** — Spark склеивает batch + stream, пишет Delta «почти as-is». Не чистим бизнес-логикой. Добавляем `_ingest_time`, source, filename. Append-mostly.
3. **Silver** — join, enrich, clean, transform. Дедуп, ключи, CDC merge, качество. Grain = сущность (заказ, клиент, событие). На слайде отсюда Azure ML + MLflow.
4. **Gold** — агрегаты и витрины «готово к отчёту».
5. **Serve** — копия/выдача наружу: Cosmos (низкая latency), Synapse + Power BI, Apps.
Compute на всех стрелках внутри озера — Azure Databricks. Storage слоёв — ADLS.

## Cost Management
1. https://www.databricks.com/product/azure-pricing
2. https://azure.microsoft.com/en-us/pricing/calculator/

<img width="1696" height="554" alt="image" src="https://github.com/user-attachments/assets/195f978f-d8f3-49c6-9b04-a73f10466bee" />

## Planning Azure Workspace
<img width="1679" height="790" alt="image" src="https://github.com/user-attachments/assets/6f27d08a-1ec5-4ec5-a8e5-9ee3aadf34c3" />
<img width="1686" height="652" alt="image" src="https://github.com/user-attachments/assets/9276c8c2-f757-44c0-ae5c-60bb75ad5571" />

## VNet
<img width="1694" height="850" alt="image" src="https://github.com/user-attachments/assets/46d5ebea-f47e-4d44-b4e2-3f36a027a9d3" />

VNet задаёт сетевой периметр, в котором выполняется обработка данных. Цель — чтобы compute и трафик к storage оставались в контуре организации: контролируемый доступ к корпоративным системам, ограничение исходящего трафика и выполнение требований security и compliance.

Без VNet injection кластеры размещаются в сети Databricks. Организация слабо управляет маршрутизацией, политиками доступа и подключением к on-premises. С VNet injection узлы classic compute работают в виртуальной сети подписки заказчика. Это позволяет применять собственные NSG, firewall, Private Endpoint и соединение с корпоративной сетью.

<img width="1701" height="834" alt="image" src="https://github.com/user-attachments/assets/f8e6a945-0f29-4675-9988-87a48faf0508" />
<img width="1690" height="898" alt="image" src="https://github.com/user-attachments/assets/de0fc6fa-a5d8-480a-8fa5-858862855075" />
<img width="1689" height="844" alt="image" src="https://github.com/user-attachments/assets/548abe37-af22-4db4-9434-de11c0bc33f8" />

## Azure Active Directory
<img width="1615" height="866" alt="image" src="https://github.com/user-attachments/assets/81026b6c-f69d-475d-812d-82a9b29dbf60" />
<img width="1692" height="826" alt="image" src="https://github.com/user-attachments/assets/2e9fe40b-a145-4812-92e4-0decca54a580" />

## Unity Catalog and Accoun Administration
<img width="1688" height="737" alt="image" src="https://github.com/user-attachments/assets/5cc16b29-e9b3-4d7b-bb52-73a0a4afba5f" />

## Databricks Network Administration

### 1. Azure Software Defined Networks
<img width="1663" height="592" alt="image" src="https://github.com/user-attachments/assets/1d590113-1f27-45df-8f78-774bfa99c52e" />
<img width="1682" height="891" alt="image" src="https://github.com/user-attachments/assets/a6255e6c-de3c-45a2-ac74-2a52177e5163" />

### 2.Subnets
Адрес нужен, чтобы у каждой машины обработки была уникальная идентичность в периметре: control plane мог отдать ей job, а она могла открыть соединение к ADLS. Без уникального IP узлы нельзя различить — как процессы без имени в оркестрации.

Кто что делает:

Вы задаёте диапазон VNet и каждой subnet (например 10.10.0.0/16, внутри два куска). Это вместимость: сколько узлов одновременно влезет.
Azure при старте кластера сам выдаёт свободный частный IP из subnet каждой сетевой карте VM. Вручную IP нодам обычно не раздают.
Когда кластер гаснет, адреса возвращаются в пул.
Мало адресов — в пик ETL не стартует ещё один job. 

### 3.CIDR Ranges
CIDR (Classless Inter-Domain Routing) — нотация размера адресного пространства: 10.10.0.0/24. Слева начало диапазона, после / — сколько адресов в пуле. Меньше число после / — больше адресов.

| Часть | Значение |
| --- | --- |
| `10.10.0.0` | начало диапазона |
| `/24` | длина маски: сколько адресов в пуле |
<img width="1688" height="611" alt="image" src="https://github.com/user-attachments/assets/cb91aea9-5f18-4960-a3ff-7756c55d0d02" />

### 4. VNet Peering
<img width="1679" height="832" alt="image" src="https://github.com/user-attachments/assets/9c44a901-af44-493a-b8ba-26454381f9ab" />
Peering — договорённость, что две сети доверяют друг другу прямой обмен трафиком по согласованным правилам, без посредника вроде public internet.

В Azure это VNet peering (Virtual Network peering): две VNet (Virtual Network) объявляют, что машины в одной видят частные IP другой. Кластер в сети Databricks может обратиться к ADLS (Azure Data Lake Storage) или к другой VNet так, как если бы они были соседними сегментами одной инфраструктуры.

Без peering каждая VNet — изолированный контур: свои адреса, своего выхода наружу. С peering появляется частный мост между контурами (в одном region или, как global peering, между регионами).

## UDRs (User Defined Routing) with Azure Databricks
# UDR (User-Defined Routes)

**UDR (User-Defined Routes)** — таблица маршрутов на subnet (подсеть) в VNet (Virtual Network): трафик к заданному префиксу направляется на указанный next hop (firewall, VPN-шлюз, NVA (Network Virtual Appliance)).

Системные маршруты Azure уже связывают ресурсы внутри сети и с интернетом. UDR нужен, когда исходящий путь compute (кластеры Azure Databricks к ADLS (Azure Data Lake Storage), интернету, on-premises) должен идти **через выбранную точку контроля**, а не напрямую.

| Цель | Роль UDR |
| --- | --- |
| Контроль исходящих данных | `0.0.0.0/0` → firewall |
| Доступ к корпоративным системам | CIDR (Classless Inter-Domain Routing) on-premises → VPN / ExpressRoute |

Если весь трафик отправить на firewall и не разрешить control plane и SCC (Secure Cluster Connectivity), кластер не стартует. NSG (Network Security Group) решает *можно ли*, UDR — *каким путём*.

## Data Loss Prevention and Exfiltration 

**Data exfiltration** — несанкционированный вынос данных из среды Azure Databricks: открытый исходящий путь, слишком широкий доступ к storage, скомпрометированные учётные данные, выгрузка query results или запись во внешний контур, который политика не разрешает.
**Решение.** Регулируемые данные: сеть (VNet injection, SCC (Secure Cluster Connectivity), egress через firewall, Private Link к storage) + Unity Catalog. IP access lists — дополнительно к тому, *откуда* открывают UI.
<img width="1685" height="537" alt="image" src="https://github.com/user-attachments/assets/8c6ac81f-8e44-44c2-afd0-c2f0aa7aba32" />
<img width="1679" height="882" alt="image" src="https://github.com/user-attachments/assets/ae793d07-3ae7-4937-acbf-93734c6ef541" />
<img width="1692" height="492" alt="image" src="https://github.com/user-attachments/assets/74faae11-a9f1-4b63-a131-e61acf975f91" />

## Azure Databrikcs Endpoints 
<img width="1699" height="834" alt="image" src="https://github.com/user-attachments/assets/ae4e29a0-7450-4b3f-9748-6a03a88d944e" />
<img width="1688" height="904" alt="image" src="https://github.com/user-attachments/assets/aa3bc8c9-01e2-472c-8e76-26487cf15ae4" />



