```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```





2592000



修改

```sql
SET GLOBAL binlog_expire_logs_seconds = 86400;

flush logs;
```





清掉某个时间前的



```sql
purge binary logs before '2022-11-10 12:00:00';
```

