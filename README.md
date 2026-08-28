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
