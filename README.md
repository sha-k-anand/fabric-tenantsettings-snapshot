## Step 1. Create  a workspace
    Create a new workspace in fabric
## Step 2. Create  a Lakehouse
  Navigate to the newly created workspace and create a new lakehouse in it.
  
## Step 3. Create a new notebook


<details>

  <summary>View notebook code</summary>

```
from notebookutils.mssparkutils.credentials import getToken
from datetime import datetime
import requests

base_url = "https://api.fabric.microsoft.com/v1/admin"
token = getToken("https://analysis.windows.net/powerbi/api")
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
    SUM(CASE WHEN derived_NewFlag     = 'New'     THEN 1 ELSE 0 END)  NewCount
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

