# ODBC Connectivity for NetSuite Data Ingestion in Microsoft Fabric

## Overview
Microsoft Fabric provides native support for ODBC-based connectivity through the **On-Premises Data Gateway**.  
When ingesting data from NetSuite using ODBC, a **middleware Virtual Machine (VM)** is required to host both the **ODBC driver** and the **gateway**.

This approach is ideal for **automated, scheduled, and production-grade pipelines**, where reliability and uptime are critical.  
Unlike personal laptops, a cloud VM runs continuously and ensures that data pipelines execute without human intervention.

Using an ODBC-based VM setup enables:
- Stable and secure NetSuite connectivity
- Seamless integration with Microsoft Fabric
- Enterprise-ready, always-on data ingestion pipelines

## Installing the NetSuite SuiteAnalytics ODBC Driver
To enable ODBC connectivity, the official **SuiteAnalytics Connect ODBC Driver** must be installed on the middleware VM.

### Steps
1. Go to the **NetSuite Help Center**
2. Search for **SuiteAnalytics Connect ODBC Driver**
3. Download the driver matching the VM operating system (**Windows 64-bit recommended**)
4. Install the driver using the standard installer: Next → Next → Accept License → Install

After installation, the NetSuite ODBC driver becomes available in the **Windows ODBC Data Source Manager**.

## Creating a System DSN (ODBC Configuration)
Microsoft Fabric accesses ODBC connections via the **On-Premises Data Gateway**, which requires a **System DSN**.

### Open ODBC Data Source Manager
1. On the Windows VM, search for **ODBC Data Sources (64-bit)**
2. Open the application
3. Navigate to the **System DSN** tab

### Add NetSuite ODBC Driver
1. Click **Add**
2. Select **NetSuite Driver 64-bit**
3. Click **Finish**

### Configure Connection Name
Provide a meaningful and consistent DSN name.
**Example** - ns_system_dsn_connection
> This DSN name will be referenced later inside Microsoft Fabric.  
> It must match **exactly**.

### Enter NetSuite Account ID
- The Account ID can be found in NetSuite at: Setup → Integration → Web Services Preferences → Account ID
- This value uniquely identifies the NetSuite account.

### Enter Role ID
1. Navigate to: Setup → Users/Roles → Manage Roles
2. Edit the required role
3. Identify the **Internal Role ID** (often visible in the URL or role details)

### Select Service Data Source
From the **Service Data Source** dropdown, select: **NetSuite2**

**Why NetSuite2?**
- Modern SuiteAnalytics schema
- Better query performance
- Improved SQL support

Click **OK** to save the configuration.

### Test the ODBC Connection
1. Click **Test Connect**
2. Enter:
   - NetSuite Username
   - NetSuite Password
   - Account ID
   - Role ID
3. Click **Test Connect**

A successful test confirms that the VM can connect to NetSuite.

> **Note**
> Ensure outbound port **1708** is open, as NetSuite uses this port for ODBC communication.

## Configuring Microsoft Fabric Connection
Once the VM-side setup is complete, the ODBC connection must be registered in Microsoft Fabric.  
All communication between Fabric and NetSuite flows through the **VM-hosted On-Premises Data Gateway**.

### Access Manage Connections
1. Log in to **Microsoft Fabric**
2. Navigate to: Settings → Manage Connections and Gateways

### Create a New Connection
1. Click **New**
2. A configuration panel will open from the right

### Connection Configuration

1. **Connection Location**
   - **On-Premises**
   - Although NetSuite is a cloud-based application, the ODBC driver is installed and runs on a VM.  
     Since the connection originates from this VM, Microsoft Fabric treats it as an **on-premises** data source.
2. **Gateway Cluster**
   - Select the **On-Premises Data Gateway** installed on the VM.
3. **Connection Name**
   - `ns_test`
4. **Connection Type**
   - `ODBC`
5. **Connection String**
   - `ns_system_dsn_connection`
6. **Authentication Method**
   - Select **Basic**
   - Enter the NetSuite **Username** and **Password**
7. **Privacy Level**
   - Set according to your organization’s data governance requirements.

Click **OK** to create the connection.

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
| `Target_Lakehouse`     | Target Lakehouse name where data will be stored                                                                                                                                                             | lh_raw                                               |
| `Target_Schema_Name`   | Target schema or Sink schema name                                                                                                                                                                           | raw, staging                                         |
| `Target_Table_Name`    | Target table or Sink table name                                                                                                                                                                             | account, partner, customer, service_equipment              |
| `Last_Modified_Column` | Column with Last Modified datetime/timestamp in the table                                                                                                                                                   | LastModifiedDate, UpdatedAt                          |
| `Write_Mode`           | Data write strategy in the target. `Append` = Add new records, `Overwrite` = Replace existing data.                                                                                                         | Append, Overwrite                                    |

The metadata file is read and loaded into the DB that you specify during pipeline execution, and the data extraction process is driven by the values defined in this file.

## Advantages of Using ODBC via Middleware VM
- Native support in Microsoft Fabric
- Centralized, always-on execution environment
- No dependency on personal laptops
- Secure credential handling via gateway
- Ideal for scheduled and automated pipelines
- Enterprise-approved architecture
