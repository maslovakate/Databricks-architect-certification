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
