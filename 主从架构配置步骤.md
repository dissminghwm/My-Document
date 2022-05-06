#### **1：根据需求购买云主机配置数据库架构**

广州六区

Master1   192.168.1.72

Slave    192.168.1.8 

广州七区

Master2   192.168.2.10

上海六区

LVS     10.1.1.4

#### **2：配置Master1+Master2双主模式(互为主从)；**

分别添加master1和master2的配置文件参数;

master1：

server_id=1    

master2:   

server_id=2

Master1上操作；

查看master1的gtid

```shell
MariaDB [(none)]> show master status;
+---------------+----------+--------------+------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+---------------+----------+--------------+------------------+
| binlog.000003 |      744 |              |                  |
+---------------+----------+--------------+------------------+
1 row in set (0.000 sec)

MariaDB [(none)]> select binlog_gtid_pos("binlog.000003",610);
+--------------------------------------+
| binlog_gtid_pos("binlog.000003",610) |
+--------------------------------------+
| 0-1-5387                             |
+--------------------------------------+
1 row in set (0.000 sec)
```

添加同步用户；

```shell
MariaDB [(none)]> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%' IDENTIFIED BY "password";
Query OK, 0 rows affected (0.000 sec)
```

###### Master2操作；

Maste2设置GTID，指向master1，启动同步：

```shell
MariaDB [(none)]> set global gtid_slave_pos='0-1-5385';
Query OK, 0 rows affected (0.025 sec)

MariaDB [(none)]> change master to master_host='192.168.1.7', master_user='slave', master_password='password', master_port=3367, master_use_gtid=slave_pos;
Query OK, 0 rows affected (0.022 sec)

MariaDB [(none)]> start slave;
Query OK, 0 rows affected (0.011 sec)
```

查看slave状态是否异常

```shell
MariaDB [(none)]> show slave status \G
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 192.168.1.7
                   Master_User: slave
                   Master_Port: 3367
                 Connect_Retry: 60
               Master_Log_File: binlog.000003
           Read_Master_Log_Pos: 1182
                Relay_Log_File: relaylog.000002
                 Relay_Log_Pos: 1250
         Relay_Master_Log_File: binlog.000003
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB:
           Replicate_Ignore_DB: mysql,test,information_schema
            Replicate_Do_Table:
        Replicate_Ignore_Table:
       Replicate_Wild_Do_Table:
   Replicate_Wild_Ignore_Table:
                    Last_Errno: 0
                    Last_Error:
                  Skip_Counter: 0
           Exec_Master_Log_Pos: 1182
               Relay_Log_Space: 1552
               Until_Condition: None
```

将master1指向master2主机上；

```shell
MariaDB [(none)]> change master to master_host='192.168.2.10', master_user='slave', master_password='password', master_port=3367, master_use_gtid=current_pos;
 #此时gtid_slave_pos为空，为了能让master1能自动添加为Slave，需要用到current_pos选项
 
 start slave；
```

查看状态；

```shell
MariaDB [(none)]> show slave status \G
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 192.168.2.10
                   Master_User: slave
                   Master_Port: 3367
                 Connect_Retry: 60
               Master_Log_File: binlog.000003
           Read_Master_Log_Pos: 1337
                Relay_Log_File: relaylog.000002
                 Relay_Log_Pos: 678
         Relay_Master_Log_File: binlog.000003
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB:
           Replicate_Ignore_DB: mysql,test,information_schema
            Replicate_Do_Table:
        Replicate_Ignore_Table:
       Replicate_Wild_Do_Table:
   Replicate_Wild_Ignore_Table:
                    Last_Errno: 0
                    Last_Error:
                  Skip_Counter: 0
           Exec_Master_Log_Pos: 1337
               Relay_Log_Space: 980
               Until_Condition: None

```

此时双主模型已经构建完成，master2之所以不用创建复制账号是因为已将master1创建账号是的语句同步了过来

###### 测试数据是否同步

master1创建数据库；

```
MariaDB [(none)]> create   database 67_stat1;
Query OK, 1 row affected (0.000 sec)
```

master2查看；

```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| 67_activity        |
| 67_ad              |
| 67_cms             |
| 67_core            |
| 67_cp              |
| 67_credit          |
| 67_form            |
| 67_plf             |
| 67_stat1           |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
12 rows in set (0.006 sec)
```

##### 添加新的一台slave指向master1

备份master1数据导入到slave上

```shell
[root@Master1 mysql]# ./bin/mysqldump --master-data=2 --single-transaction --routines --all-databases -uroot -p >> alldata.sql;
[root@VM-1-8-centos mysql]# ./bin/mysql -uroot -p < dump.sql
```

slave指向master1

```shell
#在备份文件dump.sql中会显示GTID值，
set global gtid_slave_pos='0-1-5392';  #取dump.sql中的GTID值

change master to master_host='192.168.1.7', master_user='slave', master_password='password', master_port=3367, master_use_gtid=slave_pos;

start slave；
```

##### 已完成双主一从架构，接下来进行测试；

模拟master1故障，在master2上插入数据看salve是否同步数据成功；

```shell
[root@Master1 mysql]# ./mysql.server  stop
#master2插入数据;
MariaDB [stat1]> use stat1
Database changed
MariaDB [stat1]> CREATE TABLE `games` (
    ->     `game_id` INT(11) NOT NULL AUTO_INCREMENT,
    ->     `game_name` VARCHAR(100) NOT NULL,
    ->     PRIMARY KEY (`game_id`)
    -> )
    -> COLLATE='utf8_general_ci'
    -> ENGINE=InnoDB;
Query OK, 0 rows affected (0.012 sec)
#slave重新指向并查看数据是否同步;
MariaDB [stat1]> stop slave;
Query OK, 0 rows affected (0.019 sec)

MariaDB [stat1]>  change master to master_host='192.168.2.10', master_user='slave', master_password='password', master_port=3367, master_use_gtid=slave_pos;
Query OK, 0 rows affected (0.013 sec)

MariaDB [stat1]> start slave;
Query OK, 0 rows affected (0.022 sec)

MariaDB [stat1]> use stat1;
Database changed
MariaDB [stat1]> show tables;
+-----------------+
| Tables_in_stat1 |
+-----------------+
| games           |
+-----------------+
1 row in set (0.000 sec)
```

启动master1，查看数据是否同步；

```shell
[root@Master1 mysql]# ./mysql.server  start
MariaDB [stat1]> use stat1;
Database changed
MariaDB [stat1]> show tables ;
+-----------------+
| Tables_in_stat1 |
+-----------------+
| games           |
+-----------------+
1 row in set (0.000 sec)
```

##### 不同地域之间进行内网通讯的配置

在私有网络中，创建对等连接，本端上海对端广州；

分别在不同区域路由表添加策略，目的端中填入对端 CIDR ，下一跳类型选择对等连接，下一跳选择已建立的对等连接。

###### 注意：

- 一定要在本端和对端都配置相关路由，才能通过对等连接通信。
- 两个 VPC 间，本端多个网段与对端多个网段通信，只需要增加对应的路由表项，不需要建立多个对等连接。

```shell
[root@VM-1-4-centos ~]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.1.1.4  netmask 255.255.255.0  broadcast 10.1.1.255
        inet6 fe80::5054:ff:feba:413c  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:ba:41:3c  txqueuelen 1000  (Ethernet)
        RX packets 457350  bytes 121296468 (115.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 400002  bytes 66993484 (63.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@VM-1-4-centos ~]# ip a s | grep 10.1.1.4
    inet 10.1.1.4/24 brd 10.1.1.255 scope global eth0
[root@VM-1-4-centos ~]# ping 192.168.2.10
PING 192.168.2.10 (192.168.2.10) 56(84) bytes of data.
64 bytes from 192.168.2.10: icmp_seq=1 ttl=62 time=35.5 ms
64 bytes from 192.168.2.10: icmp_seq=2 ttl=62 time=35.5 ms
64 bytes from 192.168.2.10: icmp_seq=3 ttl=62 time=35.5 ms
```



















