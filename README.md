# SQL Server Data Warehouse

A complete SQL Server data warehouse project that turns raw CRM and ERP CSV files into cleaned, integrated, and analytics-ready datasets. The project demonstrates a practical ETL pipeline using a three-layer medallion architecture:

```text
CSV source files -> Bronze (raw) -> Silver (cleansed) -> Gold (business-ready)
```

## Project goals

- Build a reusable data warehouse in Microsoft SQL Server.
- Load CRM and ERP source data into a raw staging layer.
- Clean, standardize, validate, and enrich data with SQL transformations.
- Create a Gold-layer star schema for reporting and analytics.
- Provide SQL-based data-quality checks for the Silver and Gold layers.

## Architecture

### Bronze layer

The Bronze layer stores source data with minimal transformation. Tables are created by `scripts /bronze/ddl_bronze.sql` and loaded from CSV files by `bronze.load_bronze`.

### Silver layer

The Silver layer contains cleansed and standardized data. Transformations include trimming text, normalizing categorical values, removing duplicate customer records, converting integer date values to SQL dates, deriving product categories, and correcting invalid sales values. The layer is created by `scripts /silver/ddl_silver.sql` and populated by `silver.load_silver`.

### Gold layer

The Gold layer exposes business-ready views that follow a star-schema design:

- `gold.dim_customers` - customer attributes combined from CRM and ERP data.
- `gold.dim_products` - current product and category attributes.
- `gold.fact_sales` - sales transactions joined to customer and product surrogate keys.

These views are created by `scripts /gold/ddl_gold._views.sql`.

## Repository structure

```text
.
├── datasets/
│   ├── CUST_AZ12.csv
│   ├── LOC_A101.csv
│   ├── PX_CAT_G1V2.csv
│   ├── cust_info.csv
│   ├── prd_info.csv
│   └── sales_details.csv
├── docs/
├── scripts /
│   ├── create_database.sql
│   ├── bronze/
│   │   ├── ddl_bronze.sql
│   │   └── storedprocedure_load_bronze.sql
│   ├── silver/
│   │   ├── ddl_silver.sql
│   │   └── storedprocedure_load_silver.sql
│   └── gold/
│       └── ddl_gold._views.sql
├── tests/
│   ├── silver_DataQualityCheck.sql
│   └── gold_DataQualityCheck.sql
└── README.md
```

> **Note:** The repository directory is currently named `scripts ` with a trailing space. Preserve the name when using file paths, or rename it locally if your tooling does not handle trailing spaces reliably.

## Prerequisites

- Microsoft SQL Server 2017 or later.
- SQL Server Management Studio (SSMS) or another SQL Server client.
- Permission to create/drop the `DataWarehouse` database and execute `BULK INSERT`.
- Access to the CSV files from the SQL Server machine. `BULK INSERT` reads files from the server, not necessarily from the client running SSMS.

## Getting started

### 1. Create the database and schemas

Run the following script in SSMS:

```text
scripts /create_database.sql
```

This creates the `DataWarehouse` database and the `bronze`, `silver`, and `gold` schemas.

> **Warning:** The script drops and recreates `DataWarehouse` if it already exists. Back up any data you need before running it.

### 2. Create Bronze tables

Run:

```text
scripts /bronze/ddl_bronze.sql
```

### 3. Configure the CSV paths

The Bronze load procedure currently contains Windows paths similar to:

```text
C:\sql\dwh_project\01_datasets\source_crm\cust_info.csv
```

Before running it, either:

1. Place the source files in the expected `source_crm` and `source_erp` folders; or
2. Edit the `FROM` paths in `scripts /bronze/storedprocedure_load_bronze.sql` to match the location of the checked-out `datasets` directory.

The source-file mapping is:

| Source | Repository file | Bronze table |
| --- | --- | --- |
| CRM | `datasets/cust_info.csv` | `bronze.crm_cust_info` |
| CRM | `datasets/prd_info.csv` | `bronze.crm_prd_info` |
| CRM | `datasets/sales_details.csv` | `bronze.crm_sales_details` |
| ERP | `datasets/LOC_A101.csv` | `bronze.erp_loc_a101` |
| ERP | `datasets/CUST_AZ12.csv` | `bronze.erp_cust_az12` |
| ERP | `datasets/PX_CAT_G1V2.csv` | `bronze.erp_px_cat_g1v2` |

### 4. Load the Bronze layer

Create the procedure, then execute it:

```sql
:r "scripts /bronze/storedprocedure_load_bronze.sql"
GO

EXEC bronze.load_bronze;
```

If SQLCMD mode is unavailable, open and execute the procedure file directly in SSMS, then run the `EXEC` statement.

### 5. Create and load the Silver layer

Run the table DDL and load procedure:

```text
scripts /silver/ddl_silver.sql
scripts /silver/storedprocedure_load_silver.sql
```

Then execute:

```sql
EXEC silver.load_silver;
```

### 6. Create the Gold views

Run:

```text
scripts /gold/ddl_gold._views.sql
```

The Gold views are now available for reporting and analysis.

## Example queries

```sql
USE DataWarehouse;
GO

-- Inspect the customer dimension
SELECT TOP (20) *
FROM gold.dim_customers
ORDER BY customer_key;

-- Inspect the product dimension
SELECT TOP (20) *
FROM gold.dim_products
ORDER BY product_key;

-- Summarize sales by product line
SELECT
    p.product_line,
    SUM(f.sales_amount) AS total_sales,
    SUM(f.quantity) AS total_quantity
FROM gold.fact_sales AS f
LEFT JOIN gold.dim_products AS p
    ON f.product_key = p.product_key
GROUP BY p.product_line
ORDER BY total_sales DESC;
```

## Data quality checks

Run the checks after the corresponding ETL layer has been loaded:

```text
tests/silver_DataQualityCheck.sql
tests/gold_DataQualityCheck.sql
```

The checks cover duplicate or null keys, unwanted spaces, standardized categorical values, invalid date ranges, invalid sales calculations, out-of-range birth dates, and referential consistency in the Gold layer. Most checks are written with an expectation of no returned rows; investigate any returned records before using the data for reporting.

## ETL workflow

```text
1. Create/recreate DataWarehouse and schemas
2. Create Bronze tables
3. Configure server-accessible CSV paths
4. Execute bronze.load_bronze
5. Create Silver tables
6. Execute silver.load_silver
7. Run Silver data-quality checks
8. Create Gold views
9. Run Gold data-quality checks
10. Query Gold views for analytics
```

Both load procedures truncate their target tables before loading. They are therefore designed for repeatable full-refresh loads rather than incremental processing.

## Technologies

- Microsoft SQL Server
- T-SQL
- `BULK INSERT`
- SQL Server views and stored procedures
- Medallion architecture
- Star-schema dimensional modeling

## Safety considerations

- Do not run `scripts /create_database.sql` against a database containing data you need to preserve.
- Use least-privilege SQL Server accounts where possible.
- Ensure the SQL Server service account can read the CSV directory.
- Validate the Silver and Gold quality checks after every full refresh.

## License

No license has been specified for this repository.