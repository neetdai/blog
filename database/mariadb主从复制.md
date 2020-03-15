## mariadb 主从复制

以前到现在都未曾试过有两台服务器去完成主从的配置。那么现在只能用容器去测试

1. 去创建docker-compose.yml

```docker
version: "2.0"
services: 
    mysql_1:
        image: mariadb:latest
        environment: 
            MYSQL_ROOT_PASSWORD: 123456
        networks:
            mysql_test:
                ipv4_address: 172.16.1.2
        volumes: 
            - "/home/neet/my/docker/mysql_test/mysql_1/config/:/etc/mysql/"
            - "/home/neet/my/docker/mysql_test/mysql_1/binlog/:/var/lib/mysql/"
    mysql_2:
        image: mariadb:latest
        environment: 
            MYSQL_ROOT_PASSWORD: 123456
        networks:
            mysql_test:
                ipv4_address: 172.16.1.3
        volumes: 
            - "/home/neet/my/docker/mysql_test/mysql_2/config/:/etc/mysql/"
            - "/home/neet/my/docker/mysql_test/mysql_2/binlog/:/var/lib/mysql/"
networks: 
    mysql_test:
        ipam: 
            config: 
                - subnet: 172.16.1.0/24
```

(这里出现了由于宿主机没有mariadb的文件，用宿主机直接映射的话，会覆盖容器里面的文件，所以我要docker cp 容器文件到宿主机)

2. 在主数据库的 /etc/mysql/my.cnf 中的 mysqld 项添加

```
    server-id = 1 # 这里的id要唯一的
    log-bin = master-bin
```

3. 在从数据库的 /etc/mysql/my.cnf 中的 mysqld 项添加

```
    server-id = 2 # 这里的id要唯一的
    log-bin = slave-bin
    log_slave_updates = 1
```

4. 然后重启两个数据库或者直接重启容器

5. 进入主数据库
```sql
    grant replication slave, replication client on *.* to repl@'172.16.1.%' identified by 'p4ssword';
```

6. 查看主数据库的状态
```sql
    show master status;
```

7. 进入从数据库, 启动复制
```sql
    change master to master_host = '172.16.1.2',
                    master_user = 'repl',
                    master_password = 'p4ssword',
                    master_log_file = 'master-bin.000001', 
                    master_log_pos = 0;
```

8. 查看从数据库状态， 如果没有报错， 应该就是可以的了
```sql
    show slave status;
```