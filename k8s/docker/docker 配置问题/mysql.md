## Docker 启动 MySQL 闪退

要善于使用日志查看问题

docker logs <containerID> 

**问题如下：**

```bash
docker run --name=master -p 3306:3306 -d mysql
```

启动后发现闪退。查看日志：docker logs master

```bash
root@ubuntuhexo:# docker logs master
2022-11-11 08:03:05+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.31-1.el8 started.
2022-11-11 08:03:05+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2022-11-11 08:03:05+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.31-1.el8 started.
2022-11-11 08:03:05+00:00 [ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
    You need to specify one of the following as an environment variable:
    - MYSQL_ROOT_PASSWORD
    - MYSQL_ALLOW_EMPTY_PASSWORD
    - MYSQL_RANDOM_ROOT_PASSWORD
```

明显日志上说没有指定密码。加入密码后

```bash
docker run --name=master -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```

```bash
2022-11-11 08:04:29+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.31-1.el8 started.
2022-11-11 08:04:29+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2022-11-11 08:04:29+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.31-1.el8 started.
2022-11-11 08:04:29+00:00 [Note] [Entrypoint]: Initializing database files
2022-11-11T08:04:29.772339Z 0 [Warning] [MY-011068] [Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host_cache_size=0 instead.
2022-11-11T08:04:29.772424Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.31) initializing of server in progress as process 80
2022-11-11T08:04:29.777358Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-11-11T08:04:33.614888Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2022-11-11T08:04:37.621895Z 6 [Warning] [MY-010453] [Server] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option.
2022-11-11 08:04:41+00:00 [Note] [Entrypoint]: Database files initialized
2022-11-11 08:04:41+00:00 [Note] [Entrypoint]: Starting temporary server
2022-11-11T08:04:42.321690Z 0 [Warning] [MY-011068] [Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host_cache_size=0 instead.
2022-11-11T08:04:42.341394Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.31) starting as process 131
2022-11-11T08:04:42.401390Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-11-11T08:04:45.844897Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2022-11-11T08:04:46.575024Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2022-11-11T08:04:46.575058Z 0 [System] [MY-013602] [Server] Channel mysql_main configured to support TLS. Encrypted connections are now supported for this channel.
2022-11-11T08:04:46.575741Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory.
2022-11-11T08:04:46.584621Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: /var/run/mysqld/mysqlx.sock
2022-11-11T08:04:46.584640Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.31'  socket: '/var/run/mysqld/mysqld.sock'  port: 0  MySQL Community Server - GPL.
2022-11-11 08:04:46+00:00 [Note] [Entrypoint]: Temporary server started.
'/var/lib/mysql/mysql.sock' -> '/var/run/mysqld/mysqld.sock'
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leapseconds' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/tzdata.zi' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it.

2022-11-11 08:04:50+00:00 [Note] [Entrypoint]: Stopping temporary server
2022-11-11T08:04:50.284571Z 10 [System] [MY-013172] [Server] Received SHUTDOWN from user root. Shutting down mysqld (Version: 8.0.31).
2022-11-11T08:04:51.346488Z 0 [System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.31)  MySQL Community Server - GPL.
2022-11-11 08:04:52+00:00 [Note] [Entrypoint]: Temporary server stopped

2022-11-11 08:04:52+00:00 [Note] [Entrypoint]: MySQL init process done. Ready for start up.

2022-11-11T08:04:52.504332Z 0 [Warning] [MY-011068] [Server] The syntax '--skip-host-cache' is deprecated and will be removed in a future release. Please use SET GLOBAL host_cache_size=0 instead.
2022-11-11T08:04:52.505313Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.31) starting as process 1
2022-11-11T08:04:52.509649Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-11-11T08:04:52.625646Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2022-11-11T08:04:52.755430Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2022-11-11T08:04:52.755480Z 0 [System] [MY-013602] [Server] Channel mysql_main configured to support TLS. Encrypted connections are now supported for this channel.
2022-11-11T08:04:52.756754Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/mysqld' in the path is accessible to all OS users. Consider choosing a different directory.
2022-11-11T08:04:52.770087Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock
2022-11-11T08:04:52.770112Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.31'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
```

启动成功。登录MySQL：

```bash
docker exec -it master /bin/bash
```

```bash
bash-4.4# mysql -u root -p
Enter password:
```

```bash
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.31 MySQL Community Server - GPL

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

## MySQL 主从

主节点：

```bash
mysql> show master status;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000002 |      157 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

从节点：

```bash
mysql> CHANGE MASTER TO MASTER_HOST='192.168.100.26',MASTER_USER='root',MASTER_PASSWORD='123456',MASTER_LOG_FILE='binlog.000002',MASTER_LOG_POS=0;
Query OK, 0 rows affected, 7 warnings (0.01 sec)
```

```bash
mysql> start slave; # 启动主从模式
```

```bash
mysql> show slave status\G; # 查看状态
```

主从复制中出现问题：

```bash
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 192.168.100.26
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000002
          Read_Master_Log_Pos: 4
               Relay_Log_File: 3ed18b98fd78-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: binlog.000002
             Slave_IO_Running: No
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 4
              Relay_Log_Space: 157
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 13117
                Last_IO_Error: Fatal error: The slave I/O thread stops because master and slave have equal MySQL server ids; these ids must be different for replication to work (or the --replicate-same-server-id option must be used on slave but this does not always make sense; please check the manual before using it).
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 221111 08:39:57
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
1 row in set, 1 warning (0.00 sec)
```

```bash
Fatal error: The slave I/O thread stops because master and slave have equal MySQL server ids; these ids must be different for replication to work (or the --replicate-same-server-id option must be used on slave but this does not always make sense; please check the manual before using it).
```

这个问题就是主从节点的server_id相同了，需要不同。

**解决方法**：

先看一眼主节点和从节点的server_id : (都是这个)

```bash
mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
1 row in set (0.01 sec)
```

在从节点上停止主从复制模式：stop slave；

然后修改从节点的 server_id：set global server_id=2;

启动主从复制：start slave；

再看一眼从节点的 server_id： 

```bash
mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
1 row in set (0.01 sec)
```

然后看一下从节点的状态: mysql> show slave status\G;

```bash
 Slave_IO_Running: Yes
 Slave_SQL_Running: Yes
```

说明主从复制配置成功！

主从复制中出现问题：

> Authentication plugin 'caching_sha2_password' cannot be loaded: dlopen(/usr/local/mysql/lib/plugin/caching_sha2_password.so, 2): image not found

解决方法1：

ALTER USER 'username'@'ip_address' IDENTIFIED WITH mysql_native_password BY 'password';

解决方法2：

在创建容器的时候，在后面加上：

```bash
--default-authentication-plugin=mysql_native_password
```
