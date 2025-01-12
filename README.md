# Fabric Tenant settings


Tenant settings govern the functionality and features available within your organization’s Microsoft Fabric environment.
Any inadvertent changes to these settings can have a significant impact on your organization's security, compliance, and overall workflow.
Also, It is crucial to stay informed about newly released tenant settings and review their default configurations (enabled/disabled) and ensure these settings align with your organization policies and avoid unexpected disruptions.


More details on tenant settings here
    https://learn.microsoft.com/en-us/fabric/admin/about-tenant-settings

    https://learn.microsoft.com/en-us/fabric/admin/tenant-settings-index

    https://learn.microsoft.com/en-us/fabric/admin/about-tenant-settings#new-tenant-settings

    https://learn.microsoft.com/en-us/rest/api/fabric/admin/tenants/list-tenant-settings?tabs=HTTP
    
    https://purview.microsoft.com/audit/auditsearch

Here is an automated approach that you can use to snapshot the tenant settings on a daily basis and use it for tracking changes or new settings.
This process makes a call to the Admin APIs to get the list of tenant settings and saves it to a .JSON file into folder in lakehouse.
Notebook code reads all these daily JSON files and parses them and loads into a lakehouse tables.

It produces two tables
1. FabricTenantSettings_Snapshot  - Includes daily snapshot of all the tenant settings 
2. FabricTenantSettings_Summary - One row table, that indicates if a new settings/changes were made in the last day.  This table can be  used to send automated alerts.


## Prerequisites
* Access to fabric features (Capacity,Fabric is enabled at the tenant or capacity level)
* Basic knowledge of Fabric features such as lakehouse,notebooks and pipelines
* Authentication to Admin API options
    * Run the notebook/Pipeline as Fabric Admin (Fabric Admins needs to create the below notebook/pipelines)
    * or
    * Use Service Principal (Anyone can create the below notebooks,pipelines)

## Step 1. Create  a workspace
    Create a new workspace in fabric
## Step 2. Create  a Lakehouse
  Navigate to the newly created workspace and create a new lakehouse in it.
  
## Step 3. Create a new notebook

1. Create a new Notebook
2. Attach the lakehouse
3. Copy paste each of the  below codeblocks into new cell in the notebook


<details>

  <summary>View notebook code</summary>


For cell 1, use the below code block if API call needs to be made under the context of the current user
```
from notebookutils.mssparkutils.credentials import getToken
from datetime import datetime
import requests
token = getToken("https://analysis.windows.net/powerbi/api")
```

For cell 1, use the below code block if API call needs to be made under the context of service principal
```
from notebookutils.mssparkutils.credentials import getToken
from datetime import datetime
import requests

client_id = ""
client_secret = ""
tenant_id = ""

token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"
headers = {"Content-Type": "application/x-www-form-urlencoded"}
data = {
    "grant_type": "client_credentials",
    "client_id": client_id,
    "client_secret": client_secret,
    "scope": "https://analysis.windows.net/powerbi/api/.default",
}

response = requests.post(token_url, headers=headers, data=data)
token = response.json()["access_token"]
```

```
base_url = "https://api.fabric.microsoft.com/v1/admin"
headers = {"Authorization": f"Bearer {token}"}
folder_path = "Files/tenantsettings/year=" + datetime.now().strftime("%Y")  + "/month=" +  datetime.now().strftime("%Y%m")  
file_path =  "/lakehouse/default/"  + folder_path + "/" + datetime.now().strftime("%Y%m%d") + ".json"
mssparkutils.fs.mkdirs(folder_path) 

response = requests.get(f"{base_url}/tenantsettings", headers=headers)
with open(file_path, "w") as file:
    file.write(response.text)

```


```
%%sql
DROP TABLE IF EXISTS tenantsettings_snapshot_step01;
CREATE TABLE         tenantsettings_snapshot_step01
(
tenantSettings array<struct<canSpecifySecurityGroups:boolean,enabled:boolean,enabledSecurityGroups:array<struct<graphId:string,name:string>>,
properties:array<struct<name:string,type:string,value:string>>,
settingName:string,
tenantSettingGroup:string,
title:string>>
) using  json
OPTIONS (
multiLine true,
path "Files/tenantsettings/*/*/*.json"
);
```

```
%%sql
DROP VIEW IF EXISTS tenantsettings_snapshot_step02;
CREATE VIEW  tenantsettings_snapshot_step02 as
SELECT 
    REPLACE(RIGHT(input_file_name(),13),'.json','') as SnapshotDateYYYYMMDD,
    TO_DATE(REPLACE(RIGHT(input_file_name(), 13), '.json', ''), 'yyyyMMdd') as SnapshotDate,
    c1 as rowid,c2.tenantSettingGroup,c2.settingName,c2.title,c2.enabled,c2.canSpecifySecurityGroups,c2.properties as ANY_properties ,CAST(c2.properties AS STRING) as properties
FROM  
    tenantsettings_snapshot_step01 j1
LATERAL VIEW  
    posexplode(tenantSettings) c01 as c1,c2
```

```
%%sql
DROP VIEW IF EXISTS tenantsettings_snapshot_step03;
CREATE VIEW  tenantsettings_snapshot_step03 as
SELECT 
   j1.SnapshotDateYYYYMMDD,j1.SnapshotDate,
    j1.tenantSettingGroup,j1.settingName,
    c3 as propertyid, c4.name,c4.value,c4.type
FROM  
    tenantsettings_snapshot_step02 j1
LATERAL VIEW  posexplode(j1.ANY_properties)  c02  as c3,c4
```


```
%%sql
DROP VIEW IF EXISTS tenantsettings_snapshot_step04;
CREATE VIEW tenantsettings_snapshot_step04 AS
SELECT 
    SnapshotDateYYYYMMDD,SnapshotDate,rowid,tenantSettingGroup,settingName,title,canSpecifySecurityGroups,
    enabled,
    LAG(ts1.enabled)              OVER (PARTITION BY ts1.tenantSettingGroup,ts1.settingName ORDER BY ts1.SnapshotDate) AS derived_previousvalue_enabled,
    properties,
    LAG(ts1.properties)           OVER (PARTITION BY ts1.tenantSettingGroup,ts1.settingName ORDER BY ts1.SnapshotDate) AS derived_previousvalue_properties,
    FIRST_VALUE(ts1.SnapshotDate) OVER (PARTITION BY ts1.tenantSettingGroup,ts1.settingName ORDER BY ts1.SnapshotDate) AS derived_setting_first_available_date,
    CASE WHEN (enabled <> derived_previousvalue_enabled) OR  (concat(properties,'') <> concat(derived_previousvalue_properties,''))       THEN 'Changed' ELSE '' END as derived_ChangedFlag,
    CASE WHEN derived_previousvalue_enabled IS NULL  THEN 'New'     ELSE '' END as derived_NewFlag,
    CASE WHEN SnapshotDateYYYYMMDD = (SELECT MAX(SnapshotDateYYYYMMDD) FROM tenantsettings_snapshot_step02) THEN 'Current' ELSE '' END as derived_RowStatus
FROM
    tenantsettings_snapshot_step02 ts1
```

```
%%sql
DROP VIEW IF EXISTS tenantsettings_snapshot_step05;
CREATE VIEW tenantsettings_snapshot_step05 AS
SELECT 
    Current_Date() as LastRunDateUTC,
    SUM(CASE WHEN derived_ChangedFlag = 'Changed' THEN 1 ELSE 0 END)  ChangedCount,
    SUM(CASE WHEN derived_NewFlag     = 'New'     THEN 1 ELSE 0 END)  NewCount,
    SUM(CASE WHEN derived_ChangedFlag = 'Changed' THEN 1 ELSE 0 END)  + 
    SUM(CASE WHEN derived_NewFlag     = 'New'     THEN 1 ELSE 0 END)  NotificationCount
FROM 
    tenantsettings_snapshot_step04 
WHERE
    SnapshotDateYYYYMMDD = (SELECT MAX(SnapshotDateYYYYMMDD) FROM tenantsettings_snapshot_step04)
```

```
%%pyspark
resultsDF1=spark.sql("SELECT * FROM tenantsettings_snapshot_step04")
resultsDF1.write.mode("overwrite").option("overwriteSchema", "true").format("delta").save("Tables/FabricTenantSettings_Snapshot")

resultsDF2=spark.sql("SELECT * FROM tenantsettings_snapshot_step05")
resultsDF2.write.mode("overwrite").option("overwriteSchema", "true").format("delta").save("Tables/FabricTenantSettings_Summary")
```

```
%%sql
DROP TABLE IF EXISTS tenantsettings_snapshot_step01;
DROP VIEW  IF EXISTS tenantsettings_snapshot_step02;
DROP VIEW  IF EXISTS tenantsettings_snapshot_step03;
DROP VIEW  IF EXISTS tenantsettings_snapshot_step04;
```


</details>


## Step 4. Create a pipeline to run the notebook and schedule it to run daily

1. Add a Notebook activity
2. Add Lookup activity to read contents from FabricTenantSettings_Summary 
![Link](/screenshots/pipeline%20lookup%20activity.png)
3. Add If condition that checks the lookup values and send Teams channel alerts
![Link](/screenshots/teams%20notification.png)
```
@not(equals(activity('CheckSummaryTable').output.firstRow.NotificationCount, 0))
```




### FabricTenantSettings_Snapshot


|Table Name|Column Name| Column Value| Remarks|
|--|--|--|--|
|FabricTenantSettings_Snapshot|derived_previousvalue_properties|Previous values|
|FabricTenantSettings_Snapshot|derived_setting_first_available_date|New setting was added on|
|FabricTenantSettings_Snapshot|derived_ChangedFlag| = 'Changed '|Indicates if the setting was changed - compared to the previous snapshot|
|FabricTenantSettings_Snapshot|derived_NewFlag| = 'New' | Indicates if the setting is new|
|FabricTenantSettings_Snapshot|derived_RowStatus| = 'Current' All the rows from the latest snapshot will be marked as 'Current'|

### FabricTenantSettings_Summary


Summar of the last snapshot run - This table will always have only one row and it can be used for monitoring and sending automated alerts

|Table Name|Column Name| Column Value| Remarks|
|--|--|--|--|
|FabricTenantSettings_Summary|LastRunDateUTC|Date of the last snapshot run|
|FabricTenantSettings_Summary|ChangedCount|Count of settings that were changed|
|FabricTenantSettings_Summary|NewCount|Count of settings that were newly added|
|FabricTenantSettings_Summary|NotificationCount| Sum of changed  + new|


## Step 4. Sample SQL queries to run on the SQL endpoint of the lakehouse



Query to get  summary of the new/changed count
```
%%sql
SELECT
	SnapshotDate,
	SUM(CASE WHEN derived_ChangedFlag = 'Changed' THEN 1 ELSE NULL END) as ChangedCount,
	SUM(CASE WHEN derived_NewFlag = 'New' THEN 1 ELSE NULL END) as NewCount,
	COUNT(*) as SettingsCount
FROM
	[dbo].[FabricTenantSettings_Snapshot]
GROUP BY
	SnapshotDate
ORDER BY 1
```

Query to get settings from the last snapshot
```
SELECT * FROM  [dbo].[FabricTenantSettings_Snapshot] WHERE derived_RowStatus = 'Current'
```

Query to get all the setting changes from all snapshots
```
SELECT * FROM  [dbo].[FabricTenantSettings_Snapshot] WHERE derived_RowStatus = 'Changed'
```
