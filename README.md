# 💰 Azure Cost Optimization: Archiving Old Billing Records from Cosmos DB

## 📘 Overview
We are optimizing a **read-heavy serverless Azure architecture** where billing records are stored in Cosmos DB. Records older than 3 months are rarely accessed, contributing significantly to storage and RU/s costs.

This solution archives old records to **Azure Blob Storage**, while maintaining access transparency for clients, **without changing the existing API contracts**.

---

## ✅ Goals
- **Reduce Cosmos DB storage and RU/s costs**
- **Seamless data access** for old records via fallback to Blob
- **No downtime or data loss** during transition
- **No changes to API contracts**

---

## 🔧 Architecture Diagram
```
+-------------------+
|   Client/API      |
+--------+----------+
         |
         v
+--------+----------+
|  Azure Function   | (API Wrapper - Unchanged)
+--------+----------+
         |
         v
+--------+--------+        +------------------------+
| Cosmos DB (Hot) |<------>| Azure Blob Storage     |
| (Recent ≤3mo)   |        | (Archived >3mo, cold)  |
+--------+--------+        +------------------------+
         ^                        |
         |                        v
+--------+------------------------+
| Timer-Triggered Azure Function |
| (Archival Job - Daily/Weekly)  |
+--------------------------------+
```

---

## 🧬 Core Logic

### 1. `archive_old_records.py` (Timer-triggered Azure Function)
```python
import datetime
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import gzip
import json

# Initialize clients
cosmos_client = CosmosClient(COSMOS_URL, COSMOS_KEY)
db = cosmos_client.get_database_client("billing_db")
container = db.get_container_client("records")

blob_client = BlobServiceClient.from_connection_string(BLOB_CONN_STRING)
blob_container = blob_client.get_container_client("billing-archive")

def archive_old_records():
    cutoff = datetime.datetime.utcnow() - datetime.timedelta(days=90)
    query = f"SELECT * FROM c WHERE c.timestamp < '{cutoff.isoformat()}'"
    
    old_records = list(container.query_items(query, enable_cross_partition_query=True))
    
    if old_records:
        file_name = f"archive-{cutoff.date()}.json.gz"
        blob = blob_container.get_blob_client(file_name)
        
        compressed = gzip.compress(json.dumps(old_records).encode("utf-8"))
        blob.upload_blob(compressed)
        
        for record in old_records:
            container.delete_item(record['id'], partition_key=record['partitionKey'])
```

---

### 2. `read_billing_record.py` (API Function Logic)
```python
def read_billing_record(record_id):
    try:
        return cosmos_container.read_item(item=record_id, partition_key=record_id)
    except:
        # Fallback to cold archive
        for blob in blob_container.list_blobs(name_starts_with="archive"):
            blob_data = blob_container.get_blob_client(blob).download_blob().readall()
            records = json.loads(gzip.decompress(blob_data))
            for record in records:
                if record['id'] == record_id:
                    return record
        raise Exception("Record not found")
```

---

## 📦 Terraform Example

### `main.tf`
```hcl
resource "azurerm_storage_account" "billing_storage" {
  name                     = "billingstoragexyz"
  resource_group_name      = var.resource_group
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "archive" {
  name                  = "billing-archive"
  storage_account_name  = azurerm_storage_account.billing_storage.name
  container_access_type = "private"
}

resource "azurerm_function_app" "archiver" {
  name                       = "billing-archiver"
  location                   = var.location
  resource_group_name        = var.resource_group
  app_service_plan_id        = azurerm_app_service_plan.this.id
  storage_account_name       = azurerm_storage_account.billing_storage.name
  storage_account_access_key = azurerm_storage_account.billing_storage.primary_access_key
  version                    = "~4"
  os_type                    = "linux"
  app_settings = {
    "COSMOS_URL" = var.cosmos_url
    "COSMOS_KEY" = var.cosmos_key
    "BLOB_CONN_STRING" = azurerm_storage_account.billing_storage.primary_connection_string
  }
}
```

---

## 🔧 Azure CLI Snippets
```bash
# Create storage
az storage account create \
  --name billingstoragexyz \
  --resource-group my-rg \
  --location eastus \
  --sku Standard_LRS

az storage container create \
  --account-name billingstoragexyz \
  --name billing-archive

# Deploy Function App
az functionapp create \
  --resource-group my-rg \
  --consumption-plan-location eastus \
  --runtime python \
  --functions-version 4 \
  --name billing-archiver \
  --storage-account billingstoragexyz
```

---

## 🚀 Next Steps
- ✅ Setup Blob container and Cosmos access
- ✅ Deploy archival function on a timer (daily or weekly)
- ✅ Modify API backend with cold-storage fallback logic

---

## 📊 Optional Enhancements
- Add **Azure Cache for Redis** to cache archived records
- Use **Azure Data Factory** for batch migration
- Store blobs in **Cool or Archive tier** for more savings

--
