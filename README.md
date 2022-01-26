# mysql
mysql cheets

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


#### inner join query
```sh
select * from vms left join retention on vms.vmname = retention.FRIENDLYNAME union all select * from vms right join retention on vms.vmname = retention.FRIENDLYNAME where vms.vmname is null;

```
