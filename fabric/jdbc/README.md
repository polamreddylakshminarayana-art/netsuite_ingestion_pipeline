# JDBC Connectivity for NetSuite Data Ingestion in Microsoft Fabric

## Overview
Microsoft Fabric does not provide a native JDBC connector for NetSuite (unlike the available ODBC connector).  
To ingest NetSuite data into Fabric using JDBC, a custom NetSuite JDBC driver (JAR file) must be downloaded from NetSuite and uploaded into a custom Fabric Environment.

This approach enables secure, scalable, and high-performance data ingestion from NetSuite into Fabric notebooks using Spark JDBC.

## Downloading the NetSuite JDBC Driver
To enable JDBC connectivity, the official NetSuite JDBC driver must be downloaded from the NetSuite UI.

### Steps in NetSuite
1. Go to **Home Page**
2. Navigate to **Settings**
3. Click **Set Up SuiteAnalytics Connect**
4. Select **JDBC Driver**
5. Download the **JAR file**

This JAR file will later be uploaded into Microsoft Fabric.

## Uploading the JDBC Driver to Microsoft Fabric
Since Fabric notebooks do not support system-level driver installation, the JDBC JAR must be uploaded into a custom Environment.

### Steps in Fabric
1. Create a **New Environment** in Microsoft Fabric
2. Open the environment settings
3. Under **Built-in Functions**, upload the downloaded **NetSuite JDBC JAR file**
4. **Publish** the environment
5. Switch your notebook from the default environment to this custom environment

After publishing, the JDBC driver becomes available for all notebooks using this environment.

## Establishing JDBC Connection to NetSuite
Once the environment is ready, Spark can connect to NetSuite using JDBC.

### JDBC Driver Class
```python
driver = "com.netsuite.jdbc.openaccess.OpenAccessDriver"
```

This is the fully qualified class name provided by the NetSuite JDBC driver and must match the uploaded JAR file.

### JDBC Connection URL

The JDBC connection URL is used to establish a secure connection between Microsoft Fabric and NetSuite using the uploaded JDBC driver.

```python
jdbc_url = (
    "jdbc:ns://<AccountId>.connect.api.netsuite.com:1708;"
    "ServerDataSource=NetSuite2.com;"
    "Encrypted=1;"
    "NegotiateSSLClose=false;"
    "CustomProperties=("
    "AccountID=<AccountId>;"
    "RoleID=<RoleId>;"
    "Email=<Email>;"
    "Password=<Password>)"
)
```

#### Connection Parameters

- **AccountId** – NetSuite account identifier
- **RoleId** – Role with SuiteAnalytics Connect permission
- **Email** – NetSuite user email
- **Password** – NetSuite user password
- **Port** – NetSuite JDBC uses port `1708`

All placeholder values must be replaced with valid NetSuite credentials.

### Reading Data from NetSuite

Once the JDBC URL and driver are configured, NetSuite data can be read into a Spark DataFrame.

``` python
query = f"(SELECT * FROM {table_name}) AS t"

df = (
    spark.read.format("jdbc")
        .option("url", jdbc_url)
        .option("driver", driver)
        .option("dbtable", query)
        .option("fetchSize", 1000)
        .option("user", "<Email>")
        .option("password", "<Password>")
        .load()
)
```
The fetchSize option helps optimize performance when querying large NetSuite tables.

## **Source Configuration Metadata File Structure**:                                                                                                                                                                                                                         

The metadata file (CSV or JSON) should contain the following fields:

| Field                  | Description                                                                                                                                                                                                 | Example                                              |
|------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| `Id`                   | Unique identifier for each record                                                                                                                                                                           | 1, 2, 3, 4…                                          |
| `Source_Name`          | Name of the Data Source                                                                                                                                                                                     | NetSuite                      |
| `Object_Name`          | Name of the Object                                                                                                                                                                                          | Account, Partner, Customer, CUSTITEM1210              |
| `Source_Primary_Key`   | Primary key column in the source table                                                                                                                                                                      | id, partner_id, tranid, uniquey                                                   |
| `Required_Columns`     | List of columns to extract from the source. (If all columns needed, leave this column blank. Script will automatically consider all columns)                                          | id, name, address1, city, create_date, modifydate                     |    
| `Data_Load_Type`       | Type of data load strategy. Incremental or Full Load | Incremental, Full Load |
| `Is_Active`            | Flag indicating whether the object is active for ingestion (TRUE = Active, FALSE = Inactive)                                                                                                                       | TRUE, FALSE                                                    |
| `Table_Size`           | Size classification of the table (`small` / `big`)                                                                                                                                                                                           | small or big                                         |
| `Target_Lakehouse`     | Target Lakehouse name where data will be stored                                                                                                                                                             | lh_raw                                               |
| `Target_Schema_Name`   | Target schema or Sink schema name                                                                                                                                                                           | raw, staging                                         |
| `Target_Table_Name`    | Target table or Sink table name                                                                                                                                                                             | account, partner, customer, service_equipment              |
| `Last_Modified_Column` | Column with Last Modified datetime/timestamp in the table                                                                                                                                                   | LastModifiedDate, UpdatedAt                          |
| `Write_Mode`           | Data write strategy in the target. `Append` = Add new records, `Overwrite` = Replace existing data.                                                                                                         | Append, Overwrite                                    |

The metadata file is read during pipeline execution and drives the entire data extraction process. It can either be loaded into a database specified in the pipeline or read directly from Fabric after uploading the file. The values defined in this file control how the ingestion runs.

> **Note on `Table_Size = big`:**  
 If `Table_Size` is set to `big`, the ingestion framework will partition the data into four equal time-based ranges using the `Last_Modified_Column`. Each partition is processed separately to improve performance, scalability, and reliability during ingestion.

## Process Flow
### Step 1: Create the Metadata File
Create a `metadata.csv` file as per the metadata file structure mentioned above.  
It defines which NetSuite tables to load, load type (Full/Incremental), and target Lakehouse details.  
This file controls the entire ingestion process.

### Step 2: Import the Notebook
Import `nb_netsuite_ingestion.ipynb` into your Fabric workspace.  
The notebook connects to NetSuite using JDBC and reads the metadata file.  
It extracts the data and writes it to the Lakehouse.

### Step 3: Create the Pipeline
Create a Data Pipeline and attach the notebook activity.  
Use `netsuite_jdbc_ingestion_pipeline.json` if deploying via JSON.  
Configure parameters such as metadata location and target Lakehouse.

### Step 4: Run the Pipeline
Execute the pipeline to start the ingestion process.  
It reads metadata, loads active tables, and applies full or incremental logic.  
The data is written to the configured Lakehouse.

## Advantages of Using JDBC

- Faster performance due to optimized Java-based drivers
- Easy deployment using JDBC JAR files in Fabric environments
- Supports encrypted connections and secure authentication
- Seamless integration with Spark and Fabric notebooks
