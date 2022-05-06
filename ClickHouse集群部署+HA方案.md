





# ClickHouse集群部署+HA方案

[TOC]

## 什么是clickhouse数据库?

> 2021最火开源数据库，实时 OLAP 列式数据库 ClickHouse 是业界公认的一匹黑马，简单的说，ClickHouse作为分析型数据库，有三大特点：一是跑分快， ClickHouse有多少CPU，吃多少资源，所以飞快，比Vertica约快5倍，比Hive快279倍，比My SQL快801倍，二是功能多：ClickHouse支持数据统计分析各种场景，三是文艺范：目前ClickHouse的限制很多，生来就是为小资服务的。

------



## 部署clickhouse一分片两副本集群流程

#### 架构介绍：

> 搭建一分片两副本，这个架构比较简单可以认为是mysql的互为主从就好了，当用户在ch01节点上进行读/取操作的时候，由于拥有副本的功能，会将这份数据同步一份给ch02节点上，反之，在ch02节点操作也是一样，若其中一台节点宕掉了，由于有副本的数据同步还是一样可以进行操作，防止单点故障，之所以要用上zk是因为ch有个ReplicatedMergeTree引擎要依赖 zk，有数据写入或者修改时，就借助 zk 的分布式协同能力，实现多个副本之间的同步。因此需要安装 zk,所以至少需要三个节点，以达到高可用的效果。

#### 一、准备环境：

|         节点          |    系统    | 服务器配置 | 所需服务                                    |
| :-------------------: | :--------: | :--------: | ------------------------------------------- |
| zk01、ch01: 10.0.0.12 | CentOS 7.9 |    4C8G    | zookeeper（3.6.9）、clickhouse（21.11.3.6） |
| zk02、ch02: 10.0.0.7  | CentOS 7.9 |    4C8G    | zookeeper（3.6.9）、clickhouse（21.11.3.6） |
| zk03、ch02: 10.0.0.2  | CentOS 7.9 |    4C8G    | zookeeper（3.6.9）                          |



#### 二、下载所需要的软件包

```shell
#源码方式安装zookeeper
[root@zk01 ~]#wget https://dlcdn.apache.org/zookeeper/zookeeper-3.5.9/apache-zookeeper-3.5.9-bin.tar.gz  --no-check-certificate
#yum安装jdk,zookeeper依赖于jdk
[root@zk01 ~]#yum -y install java-1.8.0
```



```shell
#yum方式安装clickhouses
[root@zk01 ~]#rpm --import https://repo.clickhouse.tech/CLICKHOUSE-KEY.GPG
[root@zk01 ~]#yum-config-manager --add-repohttps://repo.clickhouse.tech/rpm/stable/x86_64
[root@zk01 ~]#yum -y install clickhouse-server clickhouse-client 
```

#### 三、配置启动zookeeper

##### 1）编辑 hosts 文件

```shell
#三台机器分别修改本地hosts文件
10.0.0.12   zk01 ch01
10.0.0.7    zk02 ch02
10.0.0.2    zk03 
```

##### 2）修改zookeeper配置文件

```shell
##注意！：需将自己定义一个配置文件为zoo.cfg，可以将已存在的zoo_sample.cfg进行改名。
##配置文件内容如下

#它用来控制心跳和超时，客户端与服务器或者服务器与服务器之间维持心跳，也就是每个tickTime时间就会发送一次心跳。通过心跳不仅能够用来监听机器的工作状态，还可以通过心跳来控制Flower跟Leader的通信时间，默认情况下最小的会话超时时间为两倍
tickTime=2000

#允许 follower （相对于 leader 而言的“客户端”）连接并同步到 leader 的初始化连接时间，它以 tickTime 的倍数来表示。当超过设置倍数的 tickTime 时间，则连接失败。
initLimit=10

# 也就是同步消息的最大数量，集群中Leader与Fo1lower之间的最大响应时间单位，假如响应超过syncLimit*tickTime，Leader认为Fo11wer死掉，从服务器列表中删除Fo1lwer。
syncLimit=5

#存储内存中数据库快照的位置，顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。   ##注意！：需要自己创建这个目录
dataDir=/opt/apache-zookeeper-3.5.9-bin/data

＃客户端连接的端口，默认是2181
clientPort=2181

#Cluster 集群模式的配置，3个节点，2个端口分别用于节点通信和集群选举
server.1=zk01:2888:3888
server.2=zk02:2888:3888
server.3=zk03:2888:3888
```

##### 3）创建myid文件

```shell
##三台机器分别在zookeeper工作目录下的data目录创建一个myid的文件，按顺序依次echo一个值进去，因为zk的选举机制会优先检查ZXID。ZXID比较大的服务器优先作为Leader，如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器
[root@zk01 apache-zookeeper-3.5.9-bin]# echo "1" > data/myid
[root@zk02 apache-zookeeper-3.5.9-bin]# echo "2" > data/myid
[root@zk03 apache-zookeeper-3.5.9-bin]# echo "3" > data/myid
```

##### 4) 启动zookeeper

```shell
##三台机器分别启动zookeeper，并查看他们的状态是否正常。
#启动zookeeper
[root@zk01 apache-zookeeper-3.5.9-bin]# ./bin/zkServer.sh  start
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.5.9-bin/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

#查看状态
[root@zk01 apache-zookeeper-3.5.9-bin]# ./bin/zkServer.sh  status
/usr/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.5.9-bin/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
```

#### 三、配置启动clickhouse

##### 1）创建需要的目录与文件

```shell
#分别在ch01和ch02上执行,
#创建ch的数据目录，
[root@ch01 ~]# mkdir -p /data/clickhouse
#创建集群配置文件，定义路径可以在config.xml中配置
[root@ch01 ~]# touch /etc/clickhouse-server/metrika.xml
#修改权限为clickhouse
[root@ch01 ~]# chown -R clickhouse:clickhouse /data/clikhouse
[root@ch01 ~]# chown  clickhouse:clickhouse /etc/clickhouse-server/metrika.xml
```

##### 2）修改config.xml文件

```xml
<!-- ch01、ch02分别修改config.xml文件 -->
<!-- 配置文件内容如下 -->


<!-- 声明这是一个XML文件,版本1.0 -->
<?xml version="1.0"?>
<!-- 公司名 -->
<yandex>
    <!-- 日志记录设置 -->
    <logger>
        <!--  --日志记录级别。可接受的值： trace, debug, information, warning, error-->
        <level>trace</level> 
        <!-- 定义日志文件以及路径  -->
        <log>/data/clickhouse/log/server.log</log>
        <!-- 定义错误日志文件以及路径  -->
        <errorlog>/data/clickhouse/log/error.log</errorlog>
        <!-- 定义文件的大小,文件达到大小后，ClickHouse将对其进行存档并重命名，并在其位置创建一个新的日志文件 -->
        <size>1000M</size>
        <!-- 定义ClickHouse存储的已归档日志文件的数量 -->
        <count>10</count>
    </logger>
    <!-- 定义HTTP连接到服务器的端口 -->    
    <http_port>8123</http_port>
    <!-- 定义tcp连接到服务器的端口,默认是9000，这里做了修改--> 
    <tcp_port>6767</tcp_port>
    <!-- 定义通过MySQL与客户端通信的端口 默认9004，这里做了修改--> 
    <mysql_port>3167</mysql_port>
    <!-- 定义ClickHouse服务器之间交换数据的端口 -->
    <interserver_http_port>9009</interserver_http_port>
    <!-- 定义限制来源主机的请求， 指定“::”表示允许所有的ipv6 -->
    <listen_host>0.0.0.0</listen_host>
    <!-- 定义最大连接数 -->
    <max_connections>4096</max_connections>
    <!-- 定义ClickHouse在关闭连接之前等待传入请求的秒数。 默认为3秒 -->
    <keep_alive_timeout>3</keep_alive_timeout>
    <!-- 定义同时处理的最大请求数 -->
    <max_concurrent_queries>100</max_concurrent_queries>
    <!-- 表引擎从MergeTree使用的未压缩数据的缓存大小（以字节为单位，8G）。 -->
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>
    <!-- 标记缓存的大小，用于MergeTree系列的表中 -->
    <mark_cache_size>5368709120</mark_cache_size>
    <!-- 定义数据的目录路径 -->
    <path>/data/clickhouse/</path>
    <!-- 定义metrika.xml读取的路径 -->
    <include_from>/etc/clickhouse-server/metrika.xml</include_from>
    <!-- 定义用于处理大型查询的临时数据的路径 -->
    <tmp_path>/data/clickhouse/tmp/</tmp_path>
    <!-- 定义用户配置文件 -->
    <users_config>users.xml</users_config>
    <!-- 定义默认设置配置文件，在参数user_config中指定 -->
    <default_profile>default</default_profile>
    <!-- 定义默认数据库 -->
    <default_database>default</default_database>
    <!-- 定义服务器的时区 -->
    <timezone>Asia/Shanghai</timezone>
    <mlock_executable>false</mlock_executable>
    <!-- 定义远程服务器，分布式表引擎和集群表功能使用的集群的配置。 -->
    <remote_servers incl="clickhouse_remote_servers" />
    <!-- 定义配置的集群需要zookeeper的配置，默认在/etc/下读取metrika.xml -->
    <zookeeper incl="zookeeper-servers" optional="true" />
    <!-- 定义的创建复制时用到的宏定义常量，在metrika.xml -->
    <macros incl="macros" optional="true" />
 <!--重新加载内置词典的时间间隔（以秒为单位），默认3600。可以在不重新启动服务器的情况下“即时”修改词典-->        <builtin_dictionaries_reload_interval>3600</builtin_dictionaries_reload_interval>
    <!--定义最大的客户端连接session超时时间，默认3600-->
    <max_session_timeout>3600</max_session_timeout>
    <!--定义默认的客户端连接session超时时间，默认60-->
    <default_session_timeout>60</default_session_timeout>
    
    <!--查询记录在system.query_log表中-->
    <query_log>
        <!--库名-->
        <database>system</database>
        <!--表名-->
        <table>query_log</table>
        <!--自定义分区键-->
        <partition_by>toYYYYMM(event_date)</partition_by>
        <!--将数据从内存中的缓冲区刷新到表的时间间隔，单位毫秒，默认7500-->
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_log>
    
    <!-- trace_log系统表操作的设置 -->
    <trace_log>
        <database>system</database>
        <table>trace_log</table>
        <partition_by>toYYYYMM(event_date)</partition_by>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </trace_log>
    
    <!-- 使用log_query_threads = 1设置，记录接收到查询的线程。 -->
    <query_thread_log>
        <database>system</database>
        <table>query_thread_log</table>
        <partition_by>toYYYYMM(event_date)</partition_by>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_thread_log>
    <!--  外部词典的配置文件的路径，路径可以包含通配符*和？的绝对或则相对路径。-->
    <dictionaries_config>*_dictionary.xml</dictionaries_config>
    <!-- MergeTree引擎表的数据压缩设置，在metrika.xml- -->
    <compression incl="clickhouse_compression">
    </compression>
    <!-- 存储在zookeeper路径中的任务队列 -->
    <distributed_ddl>
        <path>/clickhouse/task_queue/ddl</path>
    </distributed_ddl>
    
     <!--数据汇总设置-->
    <graphite_rollup_example>
        <pattern>
            <regexp>click_cost</regexp>
            <function>any</function>
            <retention>
                <age>0</age>
                <precision>3600</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>60</precision>
            </retention>
        </pattern>
        <default>
            <function>max</function>
            <retention>
                <age>0</age>
                <precision>60</precision>
            </retention>
            <retention>
                <age>3600</age>
                <precision>300</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>3600</precision>
            </retention>
        </default>
    </graphite_rollup_example>
</yandex>
```

##### 3）修改metrika.xml文件

```xml
<!-- ch01、ch02分别修改config.xml文件 -->
<!-- 配置文件内容如下 -->

<yandex>
<!-- 集群配置 -->
<clickhouse_remote_servers>
    <!-- 定义集群名 -->
    <test_cluster>
        <shard>
            <!-- 分片的权重 -->
            <weight>2</weight>
            <!-- 开启表示副本数据自动同步 -->
            <internal_replication>true</internal_replication>
            <!-- 分片1的第一个副本 -->
            <replica>
                <host>ch02</host>
                <!-- 连接的ch端口 -->
                <port>6767</port>
                <!-- 连接的ch用户名 -->
                <user>stat67</user>
                <!-- 连接的ch密码 -->
                <password>wtf</password>
            </replica>
            <!-- 分片1的第二个副本 -->
            <replica>
                <host>ch03</host>
                <port>6767</port>
                <user>stat67</user>
                <password>wtf</password>
            </replica>
        </shard>
    </test_cluster>
</clickhouse_remote_servers>
<!-- 全局变量，使用时可以仅使用变量名，clickhouse服务器会替换成我们设定的变量值，目前用的最多的就是定义分片副本宏变量，然后再创建副本表时使用。 -->
<macros>
    <!-- 分片ID, 同一分片内的副本配置相同的分片id -->
    <shard>01</shard>
    <!-- 副本名，自己定义，但最好规范，不同节点的副本名不能一致 -->
    <replica>rep1_2</replica>
</macros>
<!-- 表示集群允许所有ip访问 -->
<networks>
   <ip>::/0</ip>
</networks>
<!-- 定义zookeeper的配置 -->
<zookeeper-servers>
    <node index="1">
        <host>zk01</host>
        <port>2181</port>
    </node>

    <node index="2">
        <host>zk02</host>
        <port>2181</port>
    </node>

    <node index="3">
        <host>zk03</host>
        <port>2181</port>
    </node>
</zookeeper-servers>
    
<!--压缩相关配置-->
<clickhouse_compression>
    <case>
        <min_part_size>10000000000</min_part_size>
        <min_part_size_ratio>0.01</min_part_size_ratio>
        <!--压缩算法lz4压缩比zstd快, 更占磁盘-->
        <method>lz4</method>
    </case>
</clickhouse_compression>
</yandex>

```

##### 4）修改users.xml文件

```xml
<!-- 若不进行添加用户可以忽略此步骤，不修改users.xml登录数据库默认用户为default，密码为空-->
<!-- ch01、ch02分别修改users.xml文件 -->
<!-- 配置文件内容如下 -->

<?xml version="1.0"?>
<yandex>
        <!--     类似于用户角色，可以实现最大内存、负载方式等配置的服用,profile是一组设置的集合，类似于角色的概念，每个用户都有一个profile。 -->
        <profiles>
                <!-- 一个default,必须始终存在并且在启动服务器时应用 -->
                <default>
                        <!--  用于在单个服务器上运行查询的最大RAM量，在默认配置文件中，最大为10 GB -->
                        <max_memory_usage>30000000000</max_memory_usage>
                        <!-- 是否使用未压缩块的缓存。接受0或1。默认情况下，0（禁用）  -->
                        <use_uncompressed_cache>0</use_uncompressed_cache>
                        <!-- 指定用于分布式查询处理的副本选择算法。默认：Random  -->
                        <load_balancing>random</load_balancing>

                </default>
                <!-- 自己定义的profile  -->
                <stat67ro>
                        <max_memory_usage>30000000000</max_memory_usage>
                        <use_uncompressed_cache>0</use_uncompressed_cache>
                        <load_balancing>random</load_balancing>
                        <!-- 0-允许所有查询,1-仅允许读取数据查询，2-允许读取数据和更改设置查询 -->
                        <readonly>2</readonly>
                </stat67ro>

                <readonly>
                        <readonly>1</readonly>
                </readonly>

        </profiles>
        <!-- user标签内设置包括用户名、密码、权限 -->
        <users>
               <!-- 自己定义的用户  -->
                <stat67>
                        <!-- 加密的密码，若希望明文可以添加<password></password>             使用 echo -n {你的密码} | openssl dgst -sha256 可生成加密的 -->
                        <password_sha256_hex>e57ee9f15034c399f4e0db906e79f195acbdcc9f30a8846732ea2b271de91e13</password_sha256_hex>
                        <!-- 用于给用户分配quota，这里quota是在quotas标签下配置的。quota用于在一定时间内跟踪或限制资源的使用。 -->
                        <quota>default</quota>
                        <!-- 使用default的profile -->
                        <profile>default</profile>
                        <!--配置网络列表，只有网络列表范围内的用户可以连接到ClickHouse -->
                        <networks incl="networks" replace="replace">
                                <!-- 用户可以从任意网络访问 -->
                                <ip>::/0</ip>
                        </networks>

                        <!-- 配置用户访问数据库权限  -->
                        <allow_databases>
                                <database>stat67</database>
                        </allow_databases>

                </stat67>
                <!-- 系统默认的用户  -->
                <default>   
 <!-- 这里是默认是 <password></password> 标签无密码的，为了安全，我这里改了-->                   <password_sha256_hex>1837bc2c546d46c705204cf9f857b90b1dbffd2a7988451670119945ba39a10b</password_sha256_hex>
                        <networks incl="networks" replace="replace">
                                <ip>::/0</ip>
                        </networks>
                        <profile>default</profile>
                        <quota>default</quota>
                </default>
        </users>

        <!-- 配额，限制使用资源，限制有二种类型：一是在固定周期里的执行次数(quotas)，二是限制用户或则查询的使用资源（profiles）  -->
        <quotas>
                <!-- 自定义名称  -->
                <default>
                        <!-- 配置时间间隔，每个时间内的资源消耗限制  -->
                        <interval>
                                <!-- 时间周期，单位秒  -->
                                <duration>3600</duration>
                                <!-- 时间周期内允许的请求总数，0表示不限制  -->
                                <queries>0</queries>
                                <!-- 时间周期内允许的异常总数，0表示不限制  -->
                                <errors>0</errors>
                                <!-- 时间周期内允许返回的行数，0表示不限制  -->
                                <result_rows>0</result_rows>
                                <!-- 时间周期内允许在分布式查询中，远端节点读取的数据行数，0表示不限制 -->
                                <read_rows>0</read_rows>
                                <!-- 时间周期内允许执行的查询时间，单位是秒，0表示不限制  -->
                                <execution_time>0</execution_time>
                        </interval>
                </default>
        </quotas>
</yandex>
```

##### 5）验证数据同步功能

```mysql
##两节点同时启动clickhouse
[root@ch01 ~]# systemctl restart clickhouse-server.service
##登录集群查看状态
#因为添加了用户名改了端口进入数据库要指定，不指定默认是以default用户、9000端口进入clickhouse
[root@ch01 ~]# clickhouse-client -h ch01 -u stat67  --password wtf --port 6767
#查看集群是否存在
ch01 :)  select * from system.clusters

SELECT *
FROM system.clusters

Query id: 4585560f-ab2c-464d-86c3-009cceac42d4

┌─cluster──────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name─┬─host_address─┬─port─┬─is_local─┬─user───┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┐
│ test_cluster │         1 │            1 │           1 │ ch02      │ 10.0.0.7     │ 6767 │        1 │ stat67 │                  │            0 │               0 │                       0 │
│ test_cluster │         1 │            1 │           2 │ ch03      │ 10.0.0.2     │ 6767 │        0 │ stat67 │                  │            0 │               0 │                       0 │
└──────────────┴───────────┴──────────────┴─────────────┴───────────┴──────────────┴──────┴──────────┴────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┘

2 rows in set. Elapsed: 0.001 sec.
##验证数据同步功能
#在ch01创建数据库testdb
##!!这里创建库需要用户default用户，因为stat67用户没有权限，或者将stat67用户的allow_database标签去掉，
#建库建表带 ON CLUSTER语句，这样就可以在整个集群发挥作用。
ch01 :) create database  testdb on cluster test_cluster;

CREATE DATABASE testdb ON CLUSTER test_cluster

Query id: b35abc22-cdf0-4796-919e-d586fc953661

┌─host─┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ ch03 │ 6767 │      0 │       │                   1 │                0 │
│ ch02 │ 6767 │      0 │       │                   0 │                0 │
└──────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘
2 rows in set. Elapsed: 0.110 sec.

##创建表test01
##ENGINE后面跟着的参数，是提供给ZooKeeper作区分的该表的路径，对于不同分片上的表，都必须要有不同的路径，所以路径都必须唯一。第一个参数的/clickhouse/tables是官方要求的固定前缀，后面test01是自己表名，表名前面其实还可以加库名，总的来说可以写成：/clickhouse/tables/分片名/库/表名 。 第二个参数{replica}指的是副本名称，{副本名}是直接引用配置文件中的你自己配置的副本名称信息，如果是按照我的方式部署的集群，就在metrika.xml 文件里面，找到这一行，记住建表之前一定要确认数据库在每个节点都存在并执行 Use XXX 指定你要操作的数据库，不然会出错，我这里是没有执行use 是因为我建表指定了库。
ch01 :) create table test01  on cluster test_cluster (id int,name String)ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/testdb/test01','{replica}')ORDER BY id

CREATE TABLE test01 ON CLUSTER test_cluster
(
    `id` int,
    `name` String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/test01', '{replica}')
ORDER BY id

Query id: bff7a4ac-3091-47e1-a481-3da62f71fc56

┌─host─┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ ch03 │ 6767 │      0 │       │                   1 │                0 │
│ ch02 │ 6767 │      0 │       │                   0 │                0 │
└──────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘

2 rows in set. Elapsed: 0.112 sec.


##登录ch02节点上查看是否有同步过去，-d 指定数据库
[root@ch02 ~]# clickhouse-client  --port 6767 -h ch03 -u stat67 --password wtf  -d testdb
ClickHouse client version 21.11.3.6 (official build).
Connecting to database testdb at ch03:6767 as user stat67.
Connected to ClickHouse server version 21.11.3 revision 54450.

VM-0-2-centos :) show tables;

SHOW TABLES

Query id: 028b8ae1-0aee-4a3a-b2ec-c63411be5f53

┌─name───┐
│ test01 │
└────────┘

1 rows in set. Elapsed: 0.001 sec.

##在ch02节点上给test01表加上数据，然后去ch01上查看数据是否有同步
ch02 :) insert into test01 values('1','tony');

INSERT INTO test01 FORMAT Values

Query id: 5f97e229-d4b1-41be-a9ed-1ffbab2a52fe

Connecting to database testdb at ch03:6767 as user stat67.
Connected to ClickHouse server version 21.11.3 revision 54450.

Ok.

1 rows in set. Elapsed: 0.011 sec.
##登录ch01节点查看，如果数据查询到了 ，并且一致，则成功，否则需重新检查配置
#-q 可以非交互式查询
[root@ch01 ~]# clickhouse-client  --port 6767  -u stat67 --password wtf  -q 'select * from test01;'
1       tony

#关闭ch02上的clickhouse，然后在ch01上面插入多条数据，最后启动ch02的clikckhouse登陆查看数据是否有同步
#ch01操作：
INSERT INTO test01  VALUES (2,'james');
INSERT INTO test01  VALUES (3,'Bob');
INSERT INTO test01  VALUES (4,'jack');
INSERT INTO test01  VALUES (5,'Abby');
#ch02操作：
#若查不到，说明配置有误
[root@ch02 ~]# systemctl restart clickhouse-server.service
[root@ch02 ~]# clickhouse-client  --port 6767    -q 'select * from test01;'
1       tony
2       james
3       Bob
4       jack
5       Abby
```

## 部署clickhouse集群HA高可用

#### 一、HA架构（clickhouse+云LB）：



| 用户一访问 ----------》 云LB的IP:端口 | 云LB 依次轮询转发 ------》 | ch01接收请求 |
| :-----------------------------------: | :------------------------: | :----------: |
| 用户二访问 ----------》 云LB的IP:端口 | 云LB 依次轮询转发 ------》 | ch02接收请求 |

> clickhouse集群的高可用采用通过云LB转发后端的clickhouse集群,让用户访问云LB的IP，然后LB再负责转发到后端各个节点，优化了访问请求在服务器组之间的分配，消除了服务器之间的负载不平衡，提高系统的反应速度与总体性能，
>
> 负载均衡通过健康检查来判断后端服务的可用性，避免后端服务异常影响前端业务，开启云LB的健康检查，无论后端的节点权重是多少（包括权重为0），都会进行健康检查，当后端其中一台节点被判定为异常后，会自动将新的请求转发给其他正常的 节点，而不会转发到异常的 节点上，从而不会影响请求的丢失



#### 二、容灾演练-故障切换

##### 1）编写脚本模拟持续数据插入

```shell
##创建新表，往新表插入数据

#! /bin/bash
#已经准备好的1000条数据
name=`cat /opt/name.txt` 
id=1
for i in $name;
do
    echo "插入的第${id}条数据"
    clickhouse-client --port 6767 -h ch_lb <<EOF
    insert into test01 values($id,"$i");
EOF
    let id++
done
##脚本执行完成大概需要30几秒
```

##### 2)模拟故障发生

```shell
##执行脚本，模拟数据插入
##在脚本执行过程中,模拟节点宕机，直接reboot(相当于停掉zookeeper和clickhouse)注意！要把zookeeper和clickhouse设置成开机自启动！！！
##reboot掉ch01, 等ch01重启好后，等待几秒，再reboot掉ch02,最后查看双方节点的数据是否一致，数据是否有丢失

##在reboot ch01节点的时候，去执行脚本的终端可以看到，有些是插入失败的，但是基于云LB的健康检查故障切换，请求会转发到另一台节点上，从而不会丢任何一条请求。
插入的第169条数据
插入的第170条数据
Code: 210. DB::NetException: Connection refused (ch_lb:6767). (NETWORK_ERROR)

插入的第171条数据
插入的第172条数据
Code: 210. DB::NetException: Connection refused (ch_lb:6767). (NETWORK_ERROR)

插入的第173条数据
插入的第174条数据
Code: 210. DB::NetException: Connection refused (ch_lb:6767). (NETWORK_ERROR)

插入的第175条数据
插入的第176条数据
Code: 210. DB::NetException: Connection refused (ch_lb:6767). (NETWORK_ERROR)

插入的第177条数据
插入的第178条数据
Code: 210. DB::NetException: Connection refused (ch_lb:6767). (NETWORK_ERROR)

插入的第179条数据
插入的第180条数据
Code: 210. DB::NetException: Connection refused (ch_lb:6767). (NETWORK_ERROR)

插入的第181条数据

##查看双方节点的数据
[root@ch01 ~]# clickhouse-client --port 6767 -q "select * from test01"
995     Haley02
996     Hamiltion02
997     Hardy02
998     Harlan02
999     Harley02
1000    Harold02

[root@ch02 ~]# clickhouse-client --port 6767 -q "select * from test01"
995     Haley02
996     Hamiltion02
997     Hardy02
998     Harlan02
999     Harley02
1000    Harold02
```







