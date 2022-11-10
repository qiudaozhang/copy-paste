假设有一个表的字段存储时间戳毫秒



当前

```sql
SELECT NOW();
```



然后加-10天

```sql
SELECT DATE_ADD(NOW(),INTERVAL -10 DAY)
```



变成秒



```sql
SELECT UNIX_TIMESTAMP(DATE_ADD(NOW(),INTERVAL -10 DAY))
```



我们需要的是毫秒



```sql
SELECT UNIX_TIMESTAMP(DATE_ADD(NOW(),INTERVAL -10 DAY)) * 1000;
```



现在我们依据时间删除就可以搞了



```sql
DELETE FROM future_kline WHERE i = '1m' AND t <= SELECT UNIX_TIMESTAMP(DATE_ADD(NOW(),INTERVAL -10 DAY)) * 1000
```

