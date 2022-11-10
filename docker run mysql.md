mysql配置

/var/mysql/conf/my.cnf

```
[mysqld]
log-bin=/var/lib/mysql/mysql-bin  
server-id=1
binlog_format=MIXED
expire_logs_days=1
```



```bash
docker run -itd --name mysql8ubuntu -p 3306:3306 -e MYSQL_ROOT_PASSWORD=想要的密码 -e TZ=Asia/Shanghai  ubuntu/mysql:8.0-20.04_beta -v /var/mysql/conf:/etc/mysql/conf.d    --default-authentication-plugin=mysql_native_password
```







