# Databricks-architect-certification
## AZURE 
### Account structure
<img width="1328" height="749" alt="image" src="https://github.com/user-attachments/assets/e4110356-7d71-4924-a646-7f588f1e791c" />


| Layer | Descr |
|---|---|
| Account | биллинг Microsoft (кто платит) |
| Tenant | Entra ID: users, groups, SSO |
| Subscription | счёт, Azure RBAC, квоты |
| Resource Group | папка: деплой / удаление пачкой |
| Resources | VNet, ADLS, Key Vault, Databricks workspace |
              
### Control plane and data plane
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

### Reference architecture
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

### Cost Management
1. https://www.databricks.com/product/azure-pricing
2. https://azure.microsoft.com/en-us/pricing/calculator/

<img width="1696" height="554" alt="image" src="https://github.com/user-attachments/assets/195f978f-d8f3-49c6-9b04-a73f10466bee" />

### Planning Azure Workspace
<img width="1679" height="790" alt="image" src="https://github.com/user-attachments/assets/6f27d08a-1ec5-4ec5-a8e5-9ee3aadf34c3" />
<img width="1686" height="652" alt="image" src="https://github.com/user-attachments/assets/9276c8c2-f757-44c0-ae5c-60bb75ad5571" />

### VNet
<img width="1694" height="850" alt="image" src="https://github.com/user-attachments/assets/46d5ebea-f47e-4d44-b4e2-3f36a027a9d3" />

VNet задаёт сетевой периметр, в котором выполняется обработка данных. Цель — чтобы compute и трафик к storage оставались в контуре организации: контролируемый доступ к корпоративным системам, ограничение исходящего трафика и выполнение требований security и compliance.

Без VNet injection кластеры размещаются в сети Databricks. Организация слабо управляет маршрутизацией, политиками доступа и подключением к on-premises. С VNet injection узлы classic compute работают в виртуальной сети подписки заказчика. Это позволяет применять собственные NSG, firewall, Private Endpoint и соединение с корпоративной сетью.

<img width="1701" height="834" alt="image" src="https://github.com/user-attachments/assets/f8e6a945-0f29-4675-9988-87a48faf0508" />
<img width="1690" height="898" alt="image" src="https://github.com/user-attachments/assets/de0fc6fa-a5d8-480a-8fa5-858862855075" />
<img width="1689" height="844" alt="image" src="https://github.com/user-attachments/assets/548abe37-af22-4db4-9434-de11c0bc33f8" />

### Azure Active Directory
<img width="1615" height="866" alt="image" src="https://github.com/user-attachments/assets/81026b6c-f69d-475d-812d-82a9b29dbf60" />
<img width="1692" height="826" alt="image" src="https://github.com/user-attachments/assets/2e9fe40b-a145-4812-92e4-0decca54a580" />

### Unity Catalog and Accoun Administration
<img width="1675" height="861" alt="image" src="https://github.com/user-attachments/assets/4fc87c69-5d6e-4069-8ecf-b32f0bfce74e" />

