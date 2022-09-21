# MySQL Master-Slave(Readonly)

> MySQL 设置了主从复制，为了保证数据一致性，防止从库乱写入，需要设置从库为只读状态

1）查看默认读写状态

```mysql
mysql> show global variables like "%read_only%";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_read_only      | OFF   |
| read_only             | OFF   |
| super_read_only       | OFF   |
| transaction_read_only | OFF   |
| tx_read_only          | OFF   |
+-----------------------+-------+
5 rows in set (0.01 sec)
```

2）设置只读

```sql
mysql> set global read_only=1;         # 普通用户设置只读
mysql> set global super_read_only=1;   # 超级用户设置只读
```

```sql
# 再查看状态，已设置为只读
mysql> show global variables like "%read_only%";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_read_only      | OFF   |
| read_only             | ON    |
| super_read_only       | ON    |
| transaction_read_only | OFF   |
| tx_read_only          | OFF   |
+-----------------------+-------+
5 rows in set (0.00 sec)
```

3）创建一个库查看是否设置成功

```sql
mysql> create database test;
ERROR 1290 (HY000): The MySQL server is running with the --super-read-only option so it cannot execute this statement
```

**注意：**主从复制不受影响，可正常写入，如在做数据恢复的时候希望从库也无法进行任何写入，需要进行锁表

```sql
mysql> flush tables with read unlock;    # 锁表
mysql> unlock tables;                    # 解锁
```

**设置在 my.cnf 使用 MySQL 重启也能生效**

```bash
[mysqld]
read_only=1
super_read_only=1
```
