# Docker+redis哨兵HA集群部署

[TOC]

## 集群介绍：

![redis](https://cdn.jsdelivr.net/gh/dissminghwm/Tuchuang/img/redis.PNG)



> 普通redis集群有主从复制、读写分离的功能，能有效的进行备份，但是缺点是不具备自动容错和恢复功能，主机从机的宕机都会导致前端部分读写请求失败，需要等待机器重启或者手动切换前端的 IP 才能恢复，主机宕机，宕机前有部分数据未能及时同步到从机，切换 IP 后还会引入数据不一致的问题，降低了系统的可用性，
> 加入哨兵模式，这种模式下，master 宕机，哨兵会自动选举 master 并将其他的 slave 指向新的 master，在主从模式下，redis 同时提供了哨兵命令`redis-sentinel`，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是哨兵进程向所有的 redis 机器发送命令，等待 Redis 服务器响应，从而监控运行的多个 Redis 实例，哨兵可以有多个，一般为了便于决策选举，使用奇数个哨兵。哨兵可以和 redis 机器部署在一起，也可以部署在其他的机器上。多个哨兵构成一个哨兵集群，哨兵直接也会相互通信，检查哨兵是否正常运行，同时发现 master 宕机哨兵之间会进行决策选举新的 master。
>
> 哨兵模式的作用:
>
> - 通过发送命令，让 Redis 服务器返回监控其运行状态，包括主服务器和从服务器;
> - 当哨兵监测到 master 宕机，会自动将 slave 切换到 master，通过其他的从服务器，修改配置文件，让它们切换主机;
> - 然而一个哨兵进程对 Redis 服务器进行监控，也可能会出现问题，为此，我们可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式。
>
> 哨兵很像zookeeper 的功能。

### 所需节点：

![redis1](https://cdn.jsdelivr.net/gh/dissminghwm/Tuchuang/img/redis1.PNG)



## 部署redis主从

### 一、构建redis基本镜像

#### 1）下载centos基础镜像

```shell
#这里使用centos7的镜像
docker pull centos:7
#创建容器，进入容器里面安装我们需要的最新redis版本,创建我们需要的目录、文件
docker run -itd --name myredis centos:7 /bin/bash 
#给容器配置yum
/]# rm -rf /etc/yum.repo.d/*
/]# curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
/]# yum clean all && yum repo list
#安装一些依赖包（这里可以根据需求多装点）
/]# yum -y install wget  gcc net-tools tar make  kernel-devel
#下载最新的redis，
/]# wget http://download.redis.io/releases/redis-6.2.6.tar.gz
/]# tar xf redis-6.2.6.tar.gz  -C /opt/
#编译安装（指定安装路径纯属个人爱好）
/]# cd /opt/redis-6.2.6 && make && make PREFIX=/usr/local/redis install
#放配置文件
/]# mkdir /usr/local/redis/etc/
#配置环境变量（方法多种，采用最简单的）
/]# cd /usr/local/redis/bin/ && cp redis-cli redis-sentinel redis-server redis-benchmark  /usr/bin/
#创建存放数据的目录
/]# mkdir /data
/]# mkdir /var/log/redis/
```

#### 2）保存容器创建镜像

```shell
#将容器转化为镜像
docker commit myredis myredis:6.2.6
#查看
docker images 
```

### 二、准备配置文件

#### 1）创建编辑redis.conf

```shell
##slave1、2配置文件只需要加上 “ replicaof <ip> <端口> ” 一行配置指定master即可，其他配置跟master一致
#指定Redis 只接收来自于该IP 地址的请求，如果不进行设置，那么将处理所有请求，这里是接收所有ip
bind 0.0.0.0
#关闭protected-mode模式，此时外部网络可以直接访问,开启protected-mode保护模式，需配置bind ip或者设置访问密码
protected-mode no
#redis工作端口
port 6379
#此参数确定了TCP连接中已完成队列(完成三次握手之后)的长度，默认值511，而Linux的默认参数值是128。当系统并发量大并且客户端速度缓慢的时候，可以将这二个参数一起参考设定。
tcp-backlog 511
#单位秒，默认0，如果在一个 timeout 时间内，没有数据的交互，是否断开连接。0代表永不断开。
timeout 0
#默认是300；客户端与服务器端如果没有任何数据交互，多少秒会进行一次ping,pong 交互。用于校验是否有机器已经挂了
tcp-keepalive 300
#redis是否后台运行，默认yes  （因是基于docker部署，所以需要设置为no）
daemonize no
#redis进程文件
pidfile "/var/run/redis_6379.pid"
#日志等级，生产模式一般选用notice，开发测试阶段可以用debug
loglevel notice
#日志路径
logfile "/var/log/master.access.log"
#设置提供库的数量，默认16个
databases 16
#redis启动时是否显示Logo （前台运行的话你就可以好好享受了。。）
always-show-logo no
#设置进程标题，默认yes
set-proc-title yes
proc-title-template "{title} {listen-addr} {server-mode}"
#如果启用如上的快照（RDB），在一个存盘点之后，可能磁盘会坏掉或者权限问题，redis将依然能正常工作
stop-writes-on-bgsave-error yes
#是否将字符串用LZF压缩到.rdb 数据库中，如果想节省CPU资源可以将其设置成no，但是字符串存储在磁盘上占用空间会很大，默认是yes
rdbcompression yes
#rdb文件的校验，如果校验将避免文件格式坏掉，如果不校验将在每次操作文件时要付出校验过程的资源新能，将此参数设置为no，将跳过校验
rdbchecksum yes
#转储数据的文件名
dbfilename "dump.rdb"
#在没有持久化的情况下删除复制中使用的rdb文件，通常情况下保持默认即可
rdb-del-sync-files no
#数据存放的目录
dir "/data"
## 当一个slave失去和master的连接，或者同步正在进行中，slave的行为有两种可能：
# 1) 如果 replica-serve-stale-data 设置为 "yes" (默认值)，slave会继续响应客户端请求，可能是正常数据，也可能是还没获得值的空数据。
# 2) 如果 replica-serve-stale-data 设置为 "no"，slave会回复"正在从master同步（SYNC with master in progress）"来处理各种请求，除了 info 和 replicaof 命令。
replica-serve-stale-data yes
# 你可以配置salve实例是否接受写操作。可写的slave实例可能对存储临时数据比较有用(因为写入salve# 的数据在同master同步之后将很容被删除)，但是如果客户端由于配置错误在写入时也可能产生一些问题。
# 从Redis2.6默认所有的slave为只读
# 注意:只读的slave不是为了暴露给互联网上不可信的客户端而设计的。它只是一个防止实例误用的保护层。
# 一个只读的slave支持所有的管理命令比如config,debug等。为了限制你可以用'rename-command'来隐藏所有的管理和危险命令来增强只读slave的安全性。
replica-read-only yes
# 同步策略: 磁盘或socket，默认磁盘方式
repl-diskless-sync no
# 如果非磁盘同步方式开启，可以配置同步延迟时间，以等待master产生子进程通过socket传输RDB数据给slave。
# 默认值为5秒，设置为0秒则每次传输无延迟。
repl-diskless-sync-delay 5
#是否使用无磁盘加载，有三项：
#disabled：不要使用无磁盘加载，先将rdb文件存储到磁盘
#on-empty-db：只有在完全安全的情况下才使用无磁盘加载
#swapdb：解析时在RAM中保留当前db内容的副本，直接从套接字获取数据。
repl-diskless-load disabled
#在slave和master同步后（发送psync/sync），后续的同步是否设置成TCP_NODELAY . 假如设置成yes，则redis会合并小的TCP包从而节省带宽，但会增加同步延迟（40ms），造成master与slave数据不一致 假如设置成no，则redis master会立即发送同步数据，没有延迟。
repl-disable-tcp-nodelay no
#当 master 不能正常工作的时候，Redis Sentinel 会从 slaves 中选出一个新的 master，这个值越小，就越会被优先选中，但是如果是 0 ， 那是意味着这个 slave 不可能被选中。 默认优先级为 100
replica-priority 100
#ACL日志存储在内存中并消耗内存，设置此项可以设置最大值来回收内存。
acllog-max-len 128
#针对redis内存使用达到maxmeory，并设置有淘汰策略时，在被动淘汰键时，是否采用lazy free机制。因为此场景开启lazy free, 可能使用淘汰键的内存释放不及时，导致redis内存超用，超过maxmemory的限制。
lazyfree-lazy-eviction no
#针对设置有TTL的键，达到过期后，被redis清理删除时是否采用lazy free机制。此场景建议开启，因TTL本身是自适应调整的速度。
lazyfree-lazy-expire no
#针对有些指令在处理已存在的键时，会带有一个隐式的DEL键的操作。如rename命令，当目标键已存在,redis会先删除目标键，如果这些目标键是一个big key,那就会引入阻塞删除的性能问题。 此参数设置就是解决这类问题，建议可开启。
lazyfree-lazy-server-del no
#针对slave进行全量数据同步，slave在加载master的RDB文件前，会运行flushall来清理自己的数据场景，参数设置决定是否采用异常flush机制。如果内存变动不大，建议可开启。可减少全量同步耗时，从而减少主库因输出缓冲区爆涨引起的内存使用增长。
replica-lazy-flush no
#对于替换用户代码DEL调用的情况，也可以这样做使用UNLINK调用是不容易的，要修改DEL的默认行为命令的行为完全像UNLINK。
lazyfree-lazy-user-del no
lazyfree-lazy-user-flush no
#内存的一些设置，默认即可
oom-score-adj no
oom-score-adj-values 0 200 800
#是否关闭thp
disable-thp yes
#开启aof模式
appendonly yes
#AOF文件名
appendfilename "appendonly.aof"
#配置为everysec，是建议的同步策略，也是默认配置，做到兼顾性能和数据安全性。理论上只有在系统突然宕机的情况下丢失1秒的数据。
appendfsync everysec
#当"no-appendfsync-on-rewrit"设置为no,那么有一个进程在执行SAVE操作，AOF持久化模式相当于被设置成了"no"，也就是说根据OS的设置，糟糕的情况下可能丢失30秒以上的数据
no-appendfsync-on-rewrite no
#自动触发AOF重写，触发重写百分比 （指定百分比为0，将禁用aof自动重写功能）
auto-aof-rewrite-percentage 100
#触发自动重写的最低文件体积（小于64mb不自动重写）
auto-aof-rewrite-min-size 64mb
#指定当发生AOF文件末尾截断时，加载文件还是报错退出
aof-load-truncated yes
#开启混合持久化，更快的AOF重写和启动时数据恢复
aof-use-rdb-preamble yes
#设置lua脚本的最大运行时间，单位为毫秒
lua-time-limit 5000
#redis的slow log是一个系统OS进行的记录查询，它是超过了指定的执行时间的。执行时间不包括类似与client进行交互或发送回复等I/O操作，它只是实际执行指令的时间。
#有2个参数可以配置，一个用来告诉redis执行时间，这个时间是微秒级的（1秒=1000000微秒），这是为了不遗漏命令。另一个参数是设置slowlog的长度，当一个新的命令被记录时，最旧的命令将会从命令记录队列中移除。
slowlog-log-slower-than 10000
slowlog-max-len 128
#延迟监控，用于记录等于或超过了指定时间的操作，默认是关闭状态，即值为0。
latency-monitor-threshold 0
#件通知，默认不启用，具体参数查看配置文件
notify-keyspace-events ""
#当条目数量较少且最大不会超过给定阀值时，哈希编码将使用一个很高效的内存数据结构，阀值由以下参数来进行配置。
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
#与哈希类似，少量的lists也会通过一个指定的方式去编码从而节省更多的空间，它的阀值通过以下参数来进行配置。
list-max-ziplist-size -2
list-compress-depth 0
#当集合(set)存放的值都是64位的无符号10进制整数时，且条目数小于512时会采用intset作为集合的底层数据结构。
set-max-intset-entries 512
zset-max-ziplist-entries
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
#设置HyeperLogLog的字节数限制，这个值通常在0~15000之间，默认为3000，基本不超过16000
hll-sparse-max-bytes 3000
#设置Stream的单个节点最大字节数和最多能有多少个条目，如果任何一个条件满足就会新增加一个节点用以保存新的数据
# 如果将任何一个配置项设置为0，表示不限制
stream-node-max-bytes 4kb
stream-node-max-entries 100
# 如果没有非常严格的要求，建议将"activerehashing"设置为yes，这样可以让内存尽可能快的释放
activerehashing yes
#客户端输出缓冲区的配置 
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
#Redis有后台任务，通过设置"hz"可提高或者降低检查这些任务是否应该执行的频率，值越大消耗的CPU越多，反之越少
# 值可以设置在1到500之间，通常不建议将该值设置得比100大，一般都使用10这个默认值，除非我们的系统有非常严格的延时要求，才会将"hz"设置得等于或者超过100
hz 10
#当设置为yes时，可以基于"ht"配置值动态的调整使用的"ht"值，比如连接的客户端很多事，动态将ht调高，可以减少延迟。而当连接客户端比较少，又可以动态降低"ht"，这样消耗的CPU会很少
dynamic-hz yes
#当子进程在重写AOF文件时，如果将"aof-rewrite-incremental-fsync"设置为yes，那么一旦生成32M数据才会调用一次OS的fsync函数，这样可以降低出现访问峰值时系统的延迟。因为可以减少fsync调用次数和IO请求
aof-rewrite-incremental-fsync yes
#如果持久化采用的混合方式，即AOF文件是由"RDB部分+AOF部分"组成的话，我想"aof-rewrite-incremental-fsync"和"rdb-save-incremental-fsync"都会使用到
rdb-save-incremental-fsync yes
#线程的参数开启，默认不用管
jemalloc-bg-thread yes
#save 900 1表示如果900秒内至少1个key发生变化（新增、修改和删除），则重写rdb文件；
#save 300 10表示如果每300秒内至少10个key发生变化（新增、修改和删除），则重写rdb文件；
#save 60 3600表示如果每60秒内至少10000个key发生变化（新增、修改和删除），则重写rdb文件。
save 3600 1
save 300 100
save 60 10000
#其中user为关键词，default为用户名，后面的内容为ACL规则描述，on表示活跃的，nopass表示无密码， ~* 表示所有key，+@all表示所有命令。所以上面的命令表示活跃用户default无密码且可以访问所有命令以及所有数据。
user default on nopass ~* &* +@all
#replicaof用于追随某个节点的redis，被追随的节点为主节点，追随的为从节点。 !!!从节点需要配置这个
#replicaof redis-master 3367 
```

#### 2）创建配置文件需要的目录

```shell
#在redis配置文件和sentinel配置文件定义的目录是容器里面的，无需关心，我们只需创建需要映射的目录就好
-redis ~]# mkdir -p /data/redis/{master,slave1,slave2}
-redis ~]# mkdir  /var/log/redis/
-redis ~]# touch /var/log/redis/{master,slave1,slave2}.access.log
#配置文件最好放在一个目录里
-redis ~]# mkdir  /etc/redis_conf/
```

#### 3）创建编辑Sentinel.conf

```yaml

---
#docke-compose文件格式版本
version: '3.7'
#调用已经创建的网络并使用
networks:
    redis_net:
       external: true
# 定义所有的 service 信息
services:
  redis-master:
    #调用的镜像
    image: myredis:6.2.6
    #指定容器的名称 (等同于 docker run --name 的作用
    container_name: redis-master
    #重启策略
    restart: always
    #设置容器的权限为root
    privileged: true
    #设置时区
    environment:
       - TZ=Asia/Shanghai
    #加入指定网络 (等同于 docker network connect 的作用), networks 可以位于 compose 文件顶级键和 services 键的二级键
    networks:
      redis_net:
          #配置ipv4
          ipv4_address: 192.168.0.10
    #映射的端口
    ports:
      - "6379:6379"
    #映射的目录文件
    volumes:
      - /etc/redis_conf/master.conf:/usr/local/etc/redis/redis.conf
      - /data/redis/master/:/data
      - /var/log/redis/master.access.log:/var/log/master.access.log
    #启动命令
    command: ["redis-server","/usr/local/etc/redis/redis.conf"]

  redis-slave1:
    image: myredis:6.2.6
    container_name: redis-slave1
    restart: always
    environment:
       - TZ=Asia/Shanghai
    depends_on:
      - redis-master
    networks:
       redis_net:
           ipv4_address: 192.168.0.11
    ports:
      - "6380:6379"
    volumes:
      - /etc/redis_conf/slave1.conf:/usr/local/etc/redis/redis.conf
      - /data/redis/slave1:/data
      - /var/log/redis/slave1.access.log:/var/log/slave1.access.log
    command: ["redis-server","/usr/local/etc/redis/redis.conf"]
  redis-slave2:
    image: myredis:6.2.6
    container_name: redis-slave2
    restart: always
    environment:
       - TZ=Asia/Shanghai
    depends_on:
      - redis-master
    networks:
        redis_net:
            ipv4_address: 192.168.0.12
    ports:
      - "6381:6379"
    volumes:
      - /etc/redis_conf/slave2.conf:/usr/local/etc/redis/redis.conf
      - /data/redis/slave2:/data
      - /var/log/redis/slave2.access.log:/var/log/slave2.access.log
    command: ["redis-server","/usr/local/etc/redis/redis.conf"]
```

#### 4）启动测试复制

```shell
#启动需要docker-compose
#curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose && chmod 777 /usr/local/bin/docker-compose 
#需要在docker-compose当前目录执行
-redis ~] docker-compose up -d 
Creating redis-master ... done
Creating redis-slave2 ... done
Creating redis-slave1 ... done
##查看master的主从信息
#获得redis客户端
-redis ~] docker cp redis-master:/usr/bin/redis-cli /usr/bin
#slave连接master成功
[root@redis redis_conf]# redis-cli  -h 192.168.0.10 info | grep slave
mem_clients_slaves:41024
slave_expires_tracked_keys:0
connected_slaves:2
slave0:ip=192.168.0.12,port=6379,state=online,offset=56,lag=0
slave1:ip=192.168.0.11,port=6379,state=online,offset=56,lag=0
#测试主从数据是否同步
[root@redis redis_conf]# redis-cli  -h 192.168.0.10 set hello world
OK
#slave节点查看
[root@redis redis_conf]# redis-cli  -h 192.168.0.11 get hello
"world"
[root@redis redis_conf]# redis-cli  -h 192.168.0.12 get hello
"world"

```

## 部署redis哨兵

### 一、准备配置文件

#### 1)创建编辑Sentinel.conf

```bash
#sentinel端口
port 26379
#进程文件
pidfile "/var/run/redis_6379.pid"
#工作目录路径
dir "/data"
#是否后台运行，默认yes ,这里因为是基于docker，所以设置no
daemonize no
#关闭保护模式
protected-mode no
#日志文件路径
logfile "/var/log/redis-sentinel.log"
#这个后面的数字2,是指当有两个及以上的sentinel服务检测到master宕机，才会去执行主从切换的功能。
sentinel monitor redisMaster 192.168.0.10 6379 2
#若sentinel在该配置值内未能完成failover操作（即故障时master/slave自动切换），则认为本次failover失败。
sentinel down-after-milliseconds redisMaster 10000
#sentinel在该配置值内未能完成failover操作（即故障时master/slave自动切换），则认为本次failover失败。
sentinel failover-timeout redisMaster 60000
```

#### 2）创建配置文件需要的目录

```shell
-redis ~]# touch /var/log/redis/sentinel{1..3}.log
-redis ~]# mkdir -p /data/redis/sentinel{1..3}
```

#### 3）创建配置文件需要的目录

```yaml
#
---
version: '3.7'
networks:
    redis_net:
       external: true
services:
  redis-sentinel1:
    image: myredis:6.2.6
    container_name: redis-sentinel1
    restart: always
    environment:
       - TZ=Asia/Shanghai
    networks:
       redis_net:
           ipv4_address: 192.168.0.13
    networks:
     - "redis_net"
    ports:
      - "26379:26379"
    volumes:
      - /etc/redis_conf/sentinel.conf:/opt/sentinel.conf
      - /data/redis/sentinel1:/data
      - /var/log/redis/redis-sentinel.log:/var/log/redis-sentinel.log
    command: ["redis-sentinel","/usr/local/etc/sentinel.conf"]

  redis-sentinel2:
    image: myredis:6.2.6
    container_name: redis-sentinel2
    restart: always
    environment:
       - TZ=Asia/Shanghai
    networks:
       redis_net:
          ipv4_address: 192.168.0.14
    ports:
      - "26380:26379"
    volumes:
      - /etc/redis_conf/sentinel.conf:/opt/sentinel.conf
      - /data/redis/sentinel2:/data
    command: ["redis-sentinel","/usr/local/etc/sentinel.conf"]

  redis-sentinel3:
    image: myredis:6.2.6
    container_name: redis-sentinel3
    restart: always
    environment:
       - TZ=Asia/Shanghai
    networks:
       redis_net:
          ipv4_address: 192.168.0.15
    ports:
      - "26381:26379"
    volumes:
      - /etc/redis_conf/sentinel.conf:/opt/sentinel.conf
      - /data/redis/sentinel3:/data
    command: ["redis-sentinel","/usr/local/etc/sentinel.conf"]
```

#### 4）启动测试高可用

```bash
#启动容器，查看sentinel状态
redis sentinel]# docker-compose up -d 
redis sentiment]# tail /var/log/redis/sentinel1.log
1:X 02 Dec 2021 09:48:50.125 * +fix-slave-config slave 192.168.0.12:6379 192.168.0.12 6379 @ redisMaster 192.168.0.10 6379
1:X 02 Dec 2021 09:48:50.126 * +fix-slave-config slave 192.168.0.11:6379 192.168.0.11 6379 @ redisMaster 192.168.0.10 6379
#sentinel正常，正在监控
#模拟故障，关掉master,
redis sentinel]# docker stop redis-master
#查看slave2或者slave3的信息
redis sentinel]# redis-cli -h 192.168.0.11 info  Replication
# Replication
role:master
connected_slaves:1
slave0:ip=192.168.0.12,port=6379,state=online,offset=76690,lag=1
master_failover_state:no-failover
master_replid:6b2b2bc42d535ecc0c04c4c2add7162aec5e1c53
master_replid2:3880cf9d6e36d416175476a2476dbc6c217f90be
master_repl_offset:76832
second_repl_offset:36514
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:76832
#可以看到已经转换成功，
#重启reids-master,查看信息
sentinel]# redis-cli -h 192.168.0.10 info  Replication
# Replication
role:slave
master_host:192.168.0.11
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:91735
slave_repl_offset:91735
#这里看到之前的master已经是slave状态并现在指向了slave1作为它的master
```

