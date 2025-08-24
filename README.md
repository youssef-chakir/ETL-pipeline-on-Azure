

# Azure ETL Demo (ADF + ADLS Gen2 + Azure SQL Serverless) 

A beginner-friendly **ETL pipeline on Azure**.  
We **extract** a CSV into **ADLS Gen2**, **transform** it with **Azure Data Factory (Mapping Data Flows)**, and **load** it into **Azure SQL Database (Serverless)**.  
Optimized for low cost (serverless + turn off debug when idle) and for recruiters/portfolio.

<img width="1637" height="189" alt="image" src="https://github.com/user-attachments/assets/d90ec874-9642-4994-b54a-0c2c29222284" />

---

## Table of Contents
- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Cost Notes](#cost-notes)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Quick Start (15 min)](#quick-start-15-min)
- [Step-by-Step Setup (Portal)](#step-by-step-setup-portal)
- [Optional: Setup via Azure CLI](#optional-setup-via-azure-cli)
- [Validate & Demo](#validate--demo)
- [Export ADF Artifacts to This Repo](#export-adf-artifacts-to-this-repo)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Roadmap / Next Steps](#roadmap--next-steps)
- [Talking Points (Interview Value)](#talking-points-interview-value)
- [License](#license)

---

## Features
- **ETL** with **ADF Mapping Data Flows** (no code, Spark under the hood)
- **Serverless** Azure SQL DB (auto-pause to reduce cost)
- **Managed Identity + RBAC** (no keys in code)
- Parameter-ready design and scheduling
- Clear monitoring screenshots (ADF Studio → Monitor)

---

## Architecture
- **Source:** CSV (`products.csv`)
- **Landing/Staging:** **ADLS Gen2** container `raw`
- **Transform:** **ADF Mapping Data Flow**
  - Trim names, default empty categories to `Unknown`, cast prices, filter invalid rows
- **Target:** **Azure SQL Database (Serverless)** table `dbo.Products`
- **Orchestration:** **ADF Pipeline** with a single Data Flow activity

---

## Tech Stack
- **Azure Data Factory v2** (pipelines, Mapping Data Flows)
- **Azure Data Lake Storage Gen2** (Hierarchical namespace ON)
- **Azure SQL Database – General Purpose Serverless**
- **Azure RBAC + Managed Identity** (ADF ↔ Storage)
- (Optional) **Azure Key Vault** for secrets later

---

## Cost Notes
- **Azure SQL Serverless** auto-pauses when idle → you pay only storage during pause.
- **ADF Debug** spins a Spark pool; **turn Debug OFF** when not designing.
- Use tiny CSVs to keep runs quick/cheap.

---

## Prerequisites
- Azure subscription (Student free tier is OK)
- Region used here: **France Central**
- Your **Resource Group**: `rg-azure-etl-demo`
- Local CSV (below) or any small CSV you prefer

**Sample `products.csv`**


ProductId,ProductName,Category,UnitPrice
1,  USB-C Cable 1m  ,Cables,10.99
2,HDMI Adapter,,14.50
3,"Wireless Mouse",Peripherals,19.90
4,Laptop Stand,Accessories,28.00
5,USB Hub 4-Port,Peripherals,0



---

## Quick Start (15 min)
<img width="1142" height="487" alt="image" src="https://github.com/user-attachments/assets/79160c74-1897-47a9-acc5-523e99026aa7" />

1) **Create resources**: Storage (ADLS Gen2), SQL Serverless DB, Data Factory.  
2) **Upload** `products.csv` to **ADLS** container `raw`.  
3) **RBAC**: give **ADF’s managed identity** the role **Storage Blob Data Contributor** on the storage account.  
4) In **ADF Studio**:
   - **Linked services**: `ls_adls` (Managed Identity), `ls_sqldb` (SQL auth)
   - **Datasets**: `ds_raw_products` (DelimitedText → ADLS), `ds_sql_products` (Azure SQL table)
   - **Data flow**: Source→Select→Derived→Filter→Sink  
     - Derived expressions:
       - `ProductName = trim(ProductName)`
       - `Category = iif(isNull(Category) || Category=='' , 'Unknown', Category)`
       - `UnitPrice = toDecimal(UnitPrice,10,2)`
       - Filter: `isInteger(ProductId) && toDecimal(UnitPrice,10,2) >= 0`
   - **Pipeline**: `pl_etl_products` runs the data flow
5) **Run** the pipeline → **Validate in SQL**:
```sql
SELECT TOP (20) * FROM dbo.Products ORDER BY LoadDate DESC;
````

---

## Step-by-Step Setup (Portal)

### 1) Storage (ADLS Gen2)

* **Storage accounts → Create**

  * Name: `eltstorage1` (lowercase)
  * Redundancy: LRS
  * **Advanced → Hierarchical namespace = On** (this makes it Gen2)
* After deploy: **Containers → + Container → `raw`**
* **Upload** `products.csv` into `raw`


### 2) Azure SQL Database (Serverless) + Networking

* **SQL databases → Create**

  * DB: `elt_demo`
  * Create a new server: `sqlsv-elt-demo` (set admin login + password)
  * Compute: **General Purpose – Serverless**, **Auto-pause** e.g., 60 min
* On the **server** (`sqlsv-elt-demo`) → **Networking**

  * **Allow Azure services and resources to access this server = Yes**
  * **+ Add client IPv4 address** (so you can use Query editor)
* Open **elt\_demo → Query editor** → run:

sql
CREATE TABLE dbo.Products (
  ProductId   INT           NOT NULL PRIMARY KEY,
  ProductName NVARCHAR(200) NOT NULL,
  Category    NVARCHAR(100) NULL,
  UnitPrice   DECIMAL(10,2) NULL,
  LoadDate    DATETIME2     NOT NULL DEFAULT SYSUTCDATETIME()
);

<img width="1145" height="880" alt="image" src="https://github.com/user-attachments/assets/b8823b0d-37f9-4df1-b72c-ba79e63af1f7" />


### 3) Data Factory & Managed Identity

* **Data factories → Create**: `adf-azure-etl-demo`
* Data Factory blade → **Identity → System assigned = On → Save**
* **Grant RBAC to ADF on Storage**:

  * Storage `eltstorage1` → **Access control (IAM) → Add role assignment**
  * Role: **Storage Blob Data Contributor**
  * **Assign access to: Managed identity**
  * Select: **Data Factory → adf-azure-etl-demo**


### 4) ADF Linked Services

**ADF Studio → Manage → Linked services → + New**

* `ls_adls`: **Azure Data Lake Storage Gen2**

  * Auth: **System-assigned managed identity**
  * URL: `https://eltstorage1.dfs.core.windows.net/`
  * **Test connection (To linked service)** = Success
  * (Optional) Test **To file path** with `file system = raw`

* `ls_sqldb`: **Azure SQL Database**

  * Server: `sqlsv-elt-demo.database.windows.net`
  * Database: `elt_demo`
  * Auth: **SQL authentication** (use the admin creds you set)
  * **Test connection** = Success


### 5) ADF Datasets (⚠️ Pick the correct types)

**Author → + → Dataset**

* **Azure** tab → **Azure Data Lake Storage Gen2** → **Continue**
  **DelimitedText** → **Continue**

  * Name: `ds_raw_products`
  * Linked service: `ls_adls`
  * File system: `raw`
  * File: `products.csv`
  * **First row as header = On**

* **Azure SQL Database** → **Continue**

  * Name: `ds_sql_products`
  * Linked service: `ls_sqldb`
  * Table: `dbo.Products`


### 6) Mapping Data Flow

**Author → Data flows → + Mapping data flow** → `df_products_to_sql`

* **Source**: `ds_raw_products` → **Projection → Import schema**
* **Select**: keep/rename → ProductId, ProductName, Category, UnitPrice
* **Derived Column**:

  * `ProductName = trim(ProductName)`
  * `Category = iif(isNull(Category) || Category=='' , 'Unknown', Category)`
  * `UnitPrice = toDecimal(UnitPrice,10,2)`
* **Filter**:

  ```
  isInteger(ProductId) && toDecimal(UnitPrice,10,2) >= 0
  ```
* **Sink**: `ds_sql_products`

  * Write: **Upsert** (key: `ProductId`) or **Insert** only
* **Publish**


### 7) Pipeline & Run

**Author → Pipelines → + Pipeline** → `pl_etl_products`

* Add **Data flow** activity → pick `df_products_to_sql`
* **Debug** → expect success
* **Publish → Add trigger → Trigger now**
* **Monitor** the run


---

## Optional: Setup via Azure CLI

```bash
# ---- Variables ----
RG=rg-azure-etl-demo
LOC=francecentral
ST=eltstorage1
SQLSV=sqlsv-elt-demo
DB=elt_demo
ADMIN=etladmin
PASS='<StrongP@ss!23>'

# Resource Group
az group create -n $RG -l $LOC

# ADLS Gen2
az storage account create -g $RG -n $ST -l $LOC \
  --sku Standard_LRS --kind StorageV2 --hierarchical-namespace true
az storage container create --account-name $ST -n raw

# SQL Server + DB (serverless)
az sql server create -g $RG -n $SQLSV -l $LOC --admin-user $ADMIN --admin-password $PASS
az sql db create -g $RG -s $SQLSV -n $DB \
  --service-objective GP_S_Gen5_1 --compute-model Serverless --auto-pause-delay 60

# (From portal) Set server networking:
# - Allow Azure services ON
# - Add your client IP

# Data Factory (create in portal to keep it simple)
```

---

## Validate & Demo

SQL check:

```sql
SELECT TOP (20) * FROM dbo.Products ORDER BY LoadDate DESC;
```

* Expect trimmed names, `Unknown` for empty categories, DECIMAL prices, non-negative.


---

## Export ADF Artifacts to This Repo

In **ADF Studio**:

* **Manage → ARM template → Export ARM template** (downloads a zip)
* Or right-click each **pipeline/data flow → Export JSON**
  Put files here:

```
adf/
  pipeline_pl_etl_products.json
  dataflow_df_products_to_sql.json
```

---

## Troubleshooting

**1) ADLS Gen2 `EndpointUnsupportedAccountFeatures` (Conflict)**
Message: *“This endpoint does not support BlobStorageEvents or SoftDelete…”*
Fix: Storage account → **Data protection** → Turn **Off** soft delete (blobs & containers). Remove Event Grid subscriptions tied to the blob service.
Alternative: switch `ls_adls` auth to **Account key/SAS** temporarily.

**2) ADF can’t list/read ADLS (`ls_adls` test fails)**

* Ensure Data Factory **System-assigned managed identity = On**
* Storage → **IAM → Add role assignment**: **Storage Blob Data Contributor** to **Managed identity → Data Factory → adf-azure-etl-demo**
* Refresh ADF Studio and retest.

**3) Dataset dialog doesn’t show DelimitedText**

* In **Dataset** wizard, pick **Azure → Azure Data Lake Storage Gen2** **first**, **then** choose **DelimitedText**.

**4) SQL connection errors**

* SQL server → **Networking**: **Allow Azure services = Yes**, **Add client IP**
* Use full server FQDN: `sqlsv-elt-demo.database.windows.net`
* Re-enter SQL admin user/password; confirm DB name `elt_demo`.

**5) Data Flow Debug slow/expensive**

* Only enable **Debug** while designing; **turn it Off** when done.

---

## Security Notes

* Prefer **Managed Identity + RBAC** (no keys in code).
* When you later add external sources (APIs), store secrets in **Azure Key Vault**, and connect **ADF linked services** to Key Vault references.
* If you must use keys/SAS for a demo, rotate and keep them out of screenshots.

---

## Roadmap / Next Steps

* Add ingestion **HTTP → ADLS** with ADF **Copy Data tool**
* Parameterize date-partitioned paths (`raw/products/yyyy/MM/dd/…`)
* Add **Power BI** report on top of Azure SQL
* Switch **T** to **Synapse Serverless** or **Fabric** (lakehouse style)
* CI/CD: ADF Git integration + ARM/Bicep deployment
* Data quality checks & a **quarantine** folder/table for bad rows

---

## Talking Points (Interview Value)

* Designed a cloud ETL with **ADF Mapping Data Flows** (visual Spark) and **serverless SQL** for low cost.
* Implemented **Managed Identity + RBAC** to access ADLS securely.
* Solved real Azure issues (RBAC, firewall, ADLS feature conflicts).
* Documented with architecture, screenshots, and exported artifacts.




Need me to also generate a **clean architecture PNG** (ADF → ADLS → SQL DB) to drop into `docs/architecture.png` and a **products table SQL** file ready for commit?
::contentReference[oaicite:0]{index=0}
```
