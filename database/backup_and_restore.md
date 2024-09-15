# 备份与还原

数据库操作很多时候都是人为操作失误导致，所以备份和还原是必不可少的操作。

首先在mysql中检查是否开启binlog

```sql
show variables like 'log_bin';
```

如果发现是`OFF`, 就要找到`server.cnf`文件, 一般在 `/etc/my.cnf.d/` 下, 如果找不到就用 `find / -name server.cnf ` 搜索。

在`[mysqld]`下添加
```
#[mysqld]
log-bin=mysql-bin # binlog日志目录，默认在/var/lib/mysql下
expire_logs_days=7 # binlog日志保留7天
```

再重启mysql服务, `systemctl restart mysql.service` 
然后连上去再使用 `show variables like 'log_bin';` 查看是否开启成功。

## 备份

### 全量备份

```bash
mysqldump -u root -p --master-data=2 --single-transaction --flush-logs --all-databases > all-databases.sql
```

`--master-data=2` 会在备份文件中添加binlog的文件名和位置，这样在还原的时候可以知道从哪里开始还原。

`--single-transaction` 是为了在备份的时候不锁表，保证数据的一致性。

`--flush-logs` 是为了在备份的时候刷新binlog日志，保证备份的数据是最新的。

`--all-databases` 是为了备份所有的数据库。

### 备份指定数据库指定表
```bash
mysqldump -u root -p --master-data=2 --single-transaction --flush-logs database table1 table2 > all-databases.sql
```

### 备份指定数据库排除某些表
```bash
mysqldump -u root -p --master-data=2 --single-transaction --flush-logs database --ignore-table=database.table1 --ignore-table=database.table2 > all-databases.sql
```

## 删除后还原

一般如果在生产上面执行`drop`命令，可以先把当前`binlog`文件备份，然后还原旧的数据。`binlog`文件一般存放在`/var/lib/mysql`下，文件名是`mysql-bin.000001`。

### 查看当前使用的binlog文件
```mysql
MariaDB [test]> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000004 |      328 |              |                  |
+------------------+----------+--------------+------------------+
```
会显示当前使用的binlog文件名和位置。


### 整体导入
在`mysql`中执行
```mysql
source all-databases.sql
```

然后再执行`mysqlbinlong`

## 修改后还原
基于有备份的前提下恢复数据
首先使用`show master status;`查看当前使用的binlog文件名和位置。
然后在`/var/lib/mysql/`底下找到该`binlog文件`, 使用`mysqlbinlog`命令查看该文件的明文内容

```bash
mysqlbinlog /var/lib/mysql/mysql-bin.000004 -vv > /tmp/mysql-bin.000004
```

然后根据`binlog`文件中的`update`语句，找到之前最后一个出现的`at`，这个`at`的位置将会是恢复数据的终点，而起点是备份数据的`master-log-pos`。


例如备份的`master-log-pos`是`328`，而`binlog`文件中最后一个`at`的位置是`556`，那么就可以使用`mysqlbinlog`命令进行恢复
```bash
mysqlbinlog /var/lib/mysql/mysql-bin.000004 --start-position=328 --stop-position=556 -vv > /tmp/mysql-restore.sql
```

最后在`mysql`中执行
```mysql
source /tmp/mysql-restore.sql
```

## 参考
- [MySQL备份与恢复](https://www.cnblogs.com/zhengyuxiang/p/6499486.html)