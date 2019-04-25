# Docker 搭建高可用Redis 主从、哨兵集群



# 1. 拓扑结构

![img](https://upload-images.jianshu.io/upload_images/6287954-78c11a101a34e462.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



# 2. 搭建主从复制结构

## 2.1 创建Redis主从资源编排文件

创建redis集群的docker-compose

设置一个主容器和两个从容器，并在启动时设置密码。
由于是docker容器内环境，主从需要相互通信，所以给它们配置网络，同时也是为了下面哨兵能够ping通集群做准备。

创建一个docker网络

```shell
docker network create sentinel-master
```

创建redis.yml编排文件

```shell
version: '3'
services:
  redis-master:
    image: redis:5.0       ## 镜像
    container_name: redis-master
    command: redis-server --masterauth 123456 --requirepass 123456
    ports:
    - "6379:6379"
    networks:
    - sentinel-master
  redis-slave1:
    image: redis:5.0                 ## 镜像
    container_name: redis-slave-1
    ports:
    - "6380:6379"           ## 暴露端口
    command: redis-server --slaveof redis-master 6379  --masterauth 123456 --requirepass 123456 
    depends_on:
    - redis-master
    networks:
    - sentinel-master
  redis-slave2:
    image: redis:5.0                 ## 镜像
    container_name: redis-slave-2
    ports:
    - "6381:6379"           ## 暴露端口
    command: redis-server --slaveof redis-master 6379  --masterauth 123456 --requirepass 123456
    depends_on:
    - redis-master
    networks:
    - sentinel-master
networks:
  sentinel-master:
```

## 2.2 启动Redis集群

  docker-compose -f redis.yml up -d

```
​```

​```
```

## 2.3 查看是否部署成功

```shell
[root@centos101 cbs_cloud]# docker exec -it 37e581aa072f /bin/bash     #进入主节点
root@37e581aa072f:/data# redis-cli -a 123456
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=172.23.0.4,port=6379,state=online,offset=4181,lag=1
slave1:ip=172.23.0.3,port=6379,state=online,offset=4181,lag=0
master_replid:81121ca088065ddc12bd65c3ecdaa5f8a0ab1c83
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:4181
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:4181
```

可以看到两个从节点的ip、端口、状态信息

# 3. 配置redis哨兵

### 3.1  redis哨兵配置文件

哨兵配置文件sentinel.conf如下：

```shell
port 26379
dir /tmp
sentinel monitor mymaster 172.23.0.2 6379 2 
sentinel auth-pass mymaster 123456 
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000  
sentinel deny-scripts-reconfig yes
```

第三行表示Redis监控一个叫做mymaster的运行在172.28.0.3:6379的master，投票达到2则表示master以及挂掉了。  
第四行设置主节点的密码  
第五行表示在一段时间范围内sentinel向master发送的心跳PING没有回复则认为master不可用了。 
第六行的parallel-syncs表示设置在故障转移之后，同时可以重新配置使用新master的slave的数量。数字越低，更多的时间将会用故障转移完成，但是如果slaves配置为服务旧数据，你可能不希望所有的slave同时重新同步master。因为主从复制对于slave是非阻塞的，当停止从master加载批量数据时有一个片刻延迟。通过设置选项为1，确信每次只有一个slave是不可到达的。
第七行表示10秒内mymaster还没活过来，则认为master宕机了。

将sentiner.conf复制三份sentinel.conf、sentine2.conf、sentine3.conf

### 3.2  创建哨兵集群资源编排文件sentinel.yml

```shell
version: '3'
services:
  sentinel1:
    image: redis:5.0       ## 镜像
    container_name: redis-sentinel-1
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
    - "./sentinel.conf:/usr/local/etc/redis/sentinel.conf"
  sentinel2:
    image: redis:5.0                ## 镜像
    container_name: redis-sentinel-2
    ports:
    - "26380:26379"           
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
    - "./sentinel2.conf:/usr/local/etc/redis/sentinel.conf"
  sentinel3:
    image: redis:5.0                ## 镜像
    container_name: redis-sentinel-3
    ports:
    - "26381:26379"           
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
    volumes:
    - ./sentinel3.conf:/usr/local/etc/redis/sentinel.conf
networks:
  default:
    external:
      name: sentinel-master
```

 ps：网络名称要与主从节点的网络一致

### 3.3  启动哨兵集群

```shell
docker-compose -f sentinel.yml up -d
```

### 3.4 进入哨兵节点，查看是否有主节点

```shell
[root@centos101 cbs_cloud]# docker exec -it 393c2914451c /bin/bash
root@393c2914451c:/data# redis-cli -p 26379
127.0.0.1:26379> sentinel master  mymaster
 1) "name"
 2) "mymaster"
 3) "ip"
 4) "172.23.0.2"             #主节点ip
 5) "port"
 6) "6379"
 7) "runid"
 8) "b0efb050ef25c6bb1db6d78b8ad5299980418b0f"
 9) "flags"
10) "master"
11) "link-pending-commands"
12) "0"
13) "link-refcount"
14) "1"
15) "last-ping-sent"
16) "0"
17) "last-ok-ping-reply"
18) "448"
19) "last-ping-reply"
20) "448"
21) "down-after-milliseconds"
22) "30000"
23) "info-refresh"
24) "1570"
25) "role-reported"
26) "master"
27) "role-reported-time"
28) "81865"
29) "config-epoch"
30) "0"
31) "num-slaves"               #两个从节点
32) "2"
33) "num-other-sentinels"
34) "2"
35) "quorum"
36) "2"
37) "failover-timeout"
38) "10000"
39) "parallel-syncs"
40) "1"
```

# 4.  测试故障节点

   当你把主节点关掉，间隔10s(自己配置的failover-timeout)，查看哨兵的是否会检测到master节点宕机，并做主从切换:
再次查看哨兵信息，可发现master节点的ip已经更换位从节点了，也就是从节点升级成了主节点，而当宕掉的主节点从新恢复时，它便会成为从节点了。