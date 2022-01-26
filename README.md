Issue : https://stackoverflow.com/questions/70844029/azure-graph-query-to-get-data

# Workaround

## Azure get data from graph query

```sh
# get all backup policy with retention times, including vmnames
RecoveryServicesResources
    | where type == 'microsoft.recoveryservices/vaults/backuppolicies'
    | extend xvaultName = case(type == 'microsoft.recoveryservices/vaults/backuppolicies', split(split(id, 'microsoft.recoveryservices/vaults/')[1],'/')[0],type == 'microsoft.recoveryservices/vaults/backuppolicies', split(split(id, 'microsoft.recoveryservices/vaults/')[1],'/')[0],'--')
    | extend datasourceType = case(type == 'microsoft.recoveryservices/vaults/backuppolicies', properties.backupManagementType,type == 'microsoft.dataprotection/backupVaults/backupPolicies',properties.datasourceTypes[0],'--')
    | extend policyID = id
    | extend dailyDurationType = replace('"','',replace(':','',replace('durationType','',replace('{','',tostring(split(properties.retentionPolicy.dailySchedule.retentionDuration,',')[0])))))
    | extend daylyLTR = replace('"','',replace(':','',replace('count','',replace('}','',tostring(split(properties.retentionPolicy.dailySchedule.retentionDuration,',')[1])))))
    | extend DailyBackup = strcat("Daily, retention Duration ", daylyLTR, " ", dailyDurationType)
    | extend weeklyDurationType = replace('"','',replace(':','',replace('durationType','',replace('{','',tostring(split(properties.retentionPolicy.weeklySchedule.retentionDuration,',')[0])))))
    | extend weeklyLTR = replace('"','',replace(':','',replace('count','',replace('}','',tostring(split(properties.retentionPolicy.weeklySchedule.retentionDuration,',')[1])))))
    | extend weeklyStartDate = split(tostring(properties.retentionPolicy.weeklySchedule.daysOfTheWeek),'"')[1]
    | extend WeeklyBackup = strcat("Every ", weeklyStartDate, ", retention Duration ", weeklyLTR, " ", weeklyDurationType)
    | extend monthlyDurationType = replace('"','',replace(':','',replace('durationType','',replace('{','',tostring(split(properties.retentionPolicy.monthlySchedule.retentionDuration,',')[0])))))
    | extend monthlyLTR = replace('"','',replace(':','',replace('count','',replace('}','',tostring(split(properties.retentionPolicy.monthlySchedule.retentionDuration,',')[1])))))
    | extend monthlyStartDayWeek =  split(tostring(properties.retentionPolicy.monthlySchedule.retentionScheduleWeekly.daysOfTheWeek),'"')[1]
    | extend monthlyStartWeekMonth =  split(tostring(properties.retentionPolicy.monthlySchedule.retentionScheduleWeekly.weeksOfTheMonth),'"')[1]
    | extend MonthlyBackup = strcat("Every ", monthlyStartDayWeek, " ", monthlyStartWeekMonth, " Week, retention Duration ", monthlyLTR, " " , monthlyDurationType)
    | extend yearDurationType = replace('"','',replace(':','',replace('durationType','',replace('{','',tostring(split(properties.retentionPolicy.yearlySchedule.retentionDuration,',')[0])))))
    | extend yearLTR = replace('"','',replace(':','',replace('count','',replace('}','',tostring(split(properties.retentionPolicy.yearlySchedule.retentionDuration,',')[1])))))
    | extend yearlyStartDayWeek =  split(tostring(properties.retentionPolicy.yearlySchedule.retentionScheduleWeekly.daysOfTheWeek),'"')[1]
    | extend yearlyStartWeekMonth =  split(tostring(properties.retentionPolicy.yearlySchedule.retentionScheduleWeekly.weeksOfTheMonth),'"')[1]
    | extend yearlyStartMonth =  split(tostring(properties.retentionPolicy.yearlySchedule.monthsOfYear),'"')[1]
    | extend YearlyBackup = strcat("Every month ", yearlyStartWeekMonth, " ", yearlyStartDayWeek, ", retention Duration ", yearLTR, " ", yearDurationType)
    | project policyID,name,DailyBackup,WeeklyBackup,MonthlyBackup,YearlyBackup,daylyLTR,weeklyLTR,monthlyLTR,yearLTR
| join (RecoveryServicesResources 
    | where type in~ ('Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems')
    | extend vaultName = case(type =~ 'microsoft.dataprotection/backupVaults/backupInstances',split(split(id, '/Microsoft.DataProtection/backupVaults/')[1],'/')[0],type =~ 'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems',split(split(id, '/Microsoft.RecoveryServices/vaults/')[1],'/')[0],'--')
    | extend dataSourceType = case(type=~'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems',properties.backupManagementType,type =~ 'microsoft.dataprotection/backupVaults/backupInstances',properties.dataSourceSetInfo.datasourceType,'--')
    | extend friendlyName = properties.friendlyName
    | extend dsResourceGroup = split(split(properties.dataSourceInfo.resourceID, '/resourceGroups/')[1],'/')[0]
    | extend dsSubscription = split(split(properties.dataSourceInfo.resourceID, '/subscriptions/')[1],'/')[0]
    | extend primaryLocation = properties.dataSourceInfo.resourceLocation
    | extend policyName = case(type =~ 'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems',properties.policyName, type =~ 'microsoft.dataprotection/backupVaults/backupInstances', properties.policyInfo.name, '--')
    | extend protectionState = properties.currentProtectionState
    | extend vmProperties = properties
    | where protectionState in~ ('ConfiguringProtection','ProtectionConfigured','ConfiguringProtectionFailed','ProtectionStopped','SoftDeleted','ProtectionError')
    | project id, friendlyName, dataSourceType, dsResourceGroup, dsSubscription, vmProperties, vaultName, protectionState, policyName,primaryLocation)
    on $left.name == $right.policyName
```

#### second query
```sh
# get all vm details with backup policy
Resources
| where type in~ ('microsoft.compute/virtualmachines','microsoft.classiccompute/virtualmachines') 
| extend resourceId=tolower(id) 
| extend sku = properties.storageProfile.imageReference.sku
| extend publisher = properties.storageProfile.imageReference.publisher
| extend offer = properties.storageProfile.imageReference.offer
| extend ostype = properties.storageProfile.osDisk.osType
| extend hardwareType = properties.hardwareProfile
| join kind = leftouter ( 
    RecoveryServicesResources
    | where type == "microsoft.recoveryservices/vaults/backupfabrics/protectioncontainers/protecteditems"
    | where properties.backupManagementType == "AzureIaasVM"
    | project resourceId = tolower(tostring(properties.sourceResourceId)), backupItemid = id, isBackedUp = isnotempty(id), policyNamex = properties.policyInfo.name, vaultName = case(type =~ 'microsoft.dataprotection/backupVaults/backupInstances',split(split(id, '/Microsoft.DataProtection/backupVaults/')[1],'/')[0],type =~ 'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems',split(split(id, '/Microsoft.RecoveryServices/vaults/')[1],'/')[0],'--'), dataSourceType = case(type=~'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems',properties.backupManagementType,type =~ 'microsoft.dataprotection/backupVaults/backupInstances',properties.dataSourceSetInfo.datasourceType,'--'), policyName = case(type =~ 'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems',properties.policyName, type =~ 'microsoft.dataprotection/backupVaults/backupInstances', properties.policyInfo.name, '--') ) on resourceId
    | extend isProtected = isnotempty(backupItemid) 
```


# mysql
## use mysql docker container
```sh
docker run -d --name ms -p 3306:3306  -v mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=password mysql

# in wsl2 install mysql-client and connect to the db
mysql --local-infile=1 -uroot -p -h 127.0.0.1 backup_data
```

#### inner join query to combine this two files
```sql
select * from vms left join retention on vms.vmname = retention.FRIENDLYNAME union all select * from vms right join retention on vms.vmname = retention.FRIENDLYNAME where vms.vmname is null;

# create table to store data
create table all_data (select * from vms left join retention on vms.vmname = retention.FRIENDLYNAME union all select * from vms right join retention on vms.vmname = retention.FRIENDLYNAME where vms.vmname is null);
```

## Issue
### ERROR 2068 (HY000): LOAD DATA LOCAL INFILE file request rejected due to restrictions on access.
### ERROR 3948 (42000): Loading local data is disabled; this must be enabled on both the client and server sides
#### Solution : 
```sh
# step 01
# on mysql root user
SET GLOBAL local_infile=1;
# step 02
mysql --local-infile=1 -u root -p
# mysql> LOAD DATA LOCAL INFILE  'vm_details_with_backup_policy.csv' INTO TABLE vms FIELDS TERMINATED BY ','  ENCLOSED BY '"' LINES TERMINATED BY '\n' IGNORE 1 ROWS;
#Query OK, 435 rows affected, 438 warnings (0.22 sec)
#Records: 435  Deleted: 0  Skipped: 0  Warnings: 438
```

#### Referance : https://www.prisma.io/dataguide/mysql/reading-and-querying-data/joining-tables
