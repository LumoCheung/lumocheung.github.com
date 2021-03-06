---
layout: post_layout
title: redis-cluster研究和使用
date: 2016-12-16
location: 北京
pulished: true
excerpt_separator: "#"
---

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。 Redis 内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的 磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

## 一 关于redis cluster

### 1 redis cluster的现状

reids-cluster计划在redis3.0中推出，可以看作者antirez的声明:   
http://antirez.com/news/49   
(ps:跳票了好久，今年貌似加快速度了),目前的最新版本见:
https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES  


目前redis支持的cluster特性(已测试):  
1) 节点自动发现   
2) slave->master 选举,集群容错   
3) Hot resharding:在线分片   
4) 集群管理:cluster xxx   
5) 基于配置(nodes-port.conf)的集群管理   
6) ASK 转向/MOVED 转向机制.   

### 2 redis cluster 架构

#### 1) redis-cluster架构图

![redis-cluster架构图](/img/ewcx2640qfhdp35lmrjsnyu9aoiv8kbgtz71.jpg "")
 
架构细节:  
a) 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽.   
b) 节点的fail是通过集群中超过半数的节点检测失效时才生效.   
c) 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可   
d) redis-cluster把所有的物理节点映射到[0-16383]slot上,cluster 负责维护node<->slot<->key   

#### 2) redis-cluster选举:容错
 
![redis-cluster选举图](/img/863u92amvl1n4w7r5pxbkoczdsjtyq0egihf.jpg)
 
a) 领着选举过程是集群中所有master参与,如果半数以上master节点与master节点通信超过(cluster-node-timeout),认为当前master节点挂掉.   
b) 什么时候整个集群不可用(cluster_state:fail)?   
    i. 如果集群任意master挂掉,且当前master没有slave.集群进入fail状态,也可以理解成集群的slot映射[0-16383]不完成时进入fail状态.   
    ps : redis-3.0.0.rc1加入cluster-require-full-coverage参数,默认关闭,打开集群兼容部分失败.   
    ii. 如果集群超过半数以上master挂掉，无论是否有slave集群进入fail状态.   
    ps:当集群不可用时,所有对集群的操作做都不可用，收到((error) CLUSTERDOWN The cluster is down)错误   
  
## 二 redis cluster的使用

### 1 安装redis cluster

*安装redis-cluster依赖:redis-cluster的依赖库在使用时有兼容问题,在reshard时会遇到各种错误,请按指定版本安装.*

*参考文档中有关于ruby比较新版本的安装方法*

1) 确保系统安装zlib,否则gem install会报(no such file to load -- zlib)

*centos自带，如需要安装，最好官网最新版*

```bash
#download:zlib-1.2.6.tar  
./configure  
make  
make install  
```

2) 安装ruby:version(1.9.2)

```bash
# ruby1.9.2   
cd /path/ruby  
./configure -prefix=/usr/local/ruby  
make  
make install  
sudo cp ruby /usr/local/bin 
```

3) 安装rubygem:version(1.8.16)

```bash
# rubygems-1.8.16.tgz  
cd /path/gem  
sudo ruby setup.rb  
sudo cp bin/gem /usr/local/bin  
```

4) 安装gem-redis:version(3.0.0)

```bash
gem install redis --version 3.0.0  
#由于源的原因，可能下载失败，就手动下载下来安装  
#download地址:http://rubygems.org/gems/redis/versions/3.0.0  
gem install -l /data/soft/redis-3.0.0.gem
```

5) 安装redis-cluster

```bash
cd /path/redis  
make  
sudo cp /opt/redis/src/redis-server /usr/local/bin  
sudo cp /opt/redis/src/redis-cli /usr/local/bin  
sudo cp /opt/redis/src/redis-trib.rb /usr/local/bin  
```

### 2 配置redis cluster

#### 1) redis配置文件结构:

![redis配置文件结构](/img/v5m7943ukizxeo0h81np2wjydsc6rlgbfatq.jpg "")

使用包含(include)把通用配置和特殊配置分离,方便维护.

#### 2)redis通用配置.

```bash
#GENERAL  
daemonize no  
tcp-backlog 511  
timeout 0  
tcp-keepalive 0  
loglevel notice  
databases 16  
dir /opt/redis/data  
slave-serve-stale-data yes  
#slave只读  
slave-read-only yes  
#not use default  
repl-disable-tcp-nodelay yes  
slave-priority 100  
#打开aof持久化  
appendonly yes  
#每秒一次aof写  
appendfsync everysec  
#关闭在aof rewrite的时候对新的写操作进行fsync  
no-appendfsync-on-rewrite yes  
auto-aof-rewrite-min-size 64mb  
lua-time-limit 5000  
#打开redis集群  
cluster-enabled yes  
#节点互连超时的阀值  
cluster-node-timeout 15000  
cluster-migration-barrier 1  
slowlog-log-slower-than 10000  
slowlog-max-len 128  
notify-keyspace-events ""  
hash-max-ziplist-entries 512  
hash-max-ziplist-value 64  
list-max-ziplist-entries 512  
list-max-ziplist-value 64  
set-max-intset-entries 512  
zset-max-ziplist-entries 128  
zset-max-ziplist-value 64  
activerehashing yes  
client-output-buffer-limit normal 0 0 0  
client-output-buffer-limit slave 256mb 64mb 60  
client-output-buffer-limit pubsub 32mb 8mb 60  
hz 10  
aof-rewrite-incremental-fsync yes  
```

#### 3)redis特殊配置.

```bash
#包含通用配置  
include /opt/redis/redis-common.conf  
#监听tcp端口  
port 6379  
#最大可用内存  
maxmemory 100m  
#内存耗尽时采用的淘汰策略:  
# volatile-lru -> remove the key with an expire set using an LRU algorithm  
# allkeys-lru -> remove any key accordingly to the LRU algorithm  
# volatile-random -> remove a random key with an expire set  
# allkeys-random -> remove a random key, any key  
# volatile-ttl -> remove the key with the nearest expire time (minor TTL)  
# noeviction -> don't expire at all, just return an error on write operations  
maxmemory-policy allkeys-lru  
#aof存储文件  
appendfilename "appendonly-6379.aof"  
#不开启rdb存储,只用于添加slave过程  
dbfilename dump-6379.rdb  
#cluster配置文件(启动自动生成)  
cluster-config-file nodes-6379.conf  
#部署在同一机器的redis实例，把auto-aof-rewrite搓开，因为cluster环境下内存占用基本一致.  
#防止同意机器下瞬间fork所有redis进程做aof rewrite,占用大量内存(ps:cluster必须开启aof)  
auto-aof-rewrite-percentage 80-100  
```

### 3 cluster 操作

cluster集群相关命令,更多redis相关命令见文档:   
http://redis.readthedocs.org/en/latest/

```bash
集群  
CLUSTER INFO 打印集群的信息  
CLUSTER NODES 列出集群当前已知的所有节点（node），以及这些节点的相关信息。  
节点  
CLUSTER MEET <ip> <port> 将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。  
CLUSTER FORGET <node_id> 从集群中移除 node_id 指定的节点。  
CLUSTER REPLICATE <node_id> 将当前节点设置为 node_id 指定的节点的从节点。  
CLUSTER SAVECONFIG 将节点的配置文件保存到硬盘里面。  
槽(slot)  
CLUSTER ADDSLOTS <slot> [slot ...] 将一个或多个槽（slot）指派（assign）给当前节点。  
CLUSTER DELSLOTS <slot> [slot ...] 移除一个或多个槽对当前节点的指派。  
CLUSTER FLUSHSLOTS 移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。  
CLUSTER SETSLOT <slot> NODE <node_id> 将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。  
CLUSTER SETSLOT <slot> MIGRATING <node_id> 将本节点的槽 slot 迁移到 node_id 指定的节点中。  
CLUSTER SETSLOT <slot> IMPORTING <node_id> 从 node_id 指定的节点中导入槽 slot 到本节点。  
CLUSTER SETSLOT <slot> STABLE 取消对槽 slot 的导入（import）或者迁移（migrate）。  
键  
CLUSTER KEYSLOT <key> 计算键 key 应该被放置在哪个槽上。  
CLUSTER COUNTKEYSINSLOT <slot> 返回槽 slot 目前包含的键值对数量。  
CLUSTER GETKEYSINSLOT <slot> <count> 返回 count 个 slot 槽中的键。  
```

### 4 redis cluster 运维操作

#### 1)初始化并构建集群

1) 启动集群相关节点（必须是空节点,beta3后可以是有数据的节点）,指定配置文件和输出日志

```bash
redis-server /opt/redis/conf/redis-6380.conf > /opt/redis/logs/redis-6380.log 2>&1 &  
redis-server /opt/redis/conf/redis-6381.conf > /opt/redis/logs/redis-6381.log 2>&1 &  
redis-server /opt/redis/conf/redis-6382.conf > /opt/redis/logs/redis-6382.log 2>&1 &  
redis-server /opt/redis/conf/redis-7380.conf > /opt/redis/logs/redis-7380.log 2>&1 &  
redis-server /opt/redis/conf/redis-7381.conf > /opt/redis/logs/redis-7381.log 2>&1 &  
redis-server /opt/redis/conf/redis-7382.conf > /opt/redis/logs/redis-7382.log 2>&1 &  
```

(2):使用自带的ruby工具(redis-trib.rb)构建集群

```bash
#redis-trib.rb的create子命令构建  
#--replicas 则指定了为Redis Cluster中的每个Master节点配备几个Slave节点  
#节点角色由顺序决定,先master之后是slave(为方便辨认,slave的端口比master大1000)  
redis-trib.rb create --replicas 1 10.10.34.14:6380 10.10.34.14:6381 10.10.34.14:6382 10.10.34.14:7380 10.10.34.14:7381 10.10.34.14:7382  
```

(3):检查集群状态

```bash
#redis-trib.rb的check子命令构建  
#ip:port可以是集群的任意节点  
redis-trib.rb check 10.10.34.14:6380  
```

最后输出如下信息,没有任何警告或错误，表示集群启动成功并处于ok状态

```bash
[OK] All nodes agree about slots configuration.  
>>> Check for open slots...  
>>> Check slots coverage...  
[OK] All 16384 slots covered.  
```

#### 2) 添加新master节点

添加一个master节点:创建一个空节点（empty node），然后将某些slot移动到这个空节点上,这个过程目前需要人工干预

a) 根据端口生成配置文件(ps:establish_config.sh是我自己写的输出配置脚本)

```bash
sh establish_config.sh 6386 > conf/redis-6386.conf  
```

b) 启动节点

```bash
redis-server /opt/redis/conf/redis-6386.conf > /opt/redis/logs/redis-6386.log 2>&1 &
``` 

c) 加入空节点到集群
add-node  将一个节点添加到集群里面， 第一个是新节点ip:port, 第二个是任意一个已存在节点ip:port

```bash
redis-trib.rb add-node 10.10.34.14:6386 10.10.34.14:6381  
```

node:新节点没有包含任何数据， 因为它没有包含任何slot。新加入的加点是一个主节点， 当集群需要将某个从节点升级为新的主节点时， 这个新节点不会被选中，同时新的主节点因为没有包含任何slot，不参加选举和failover。

d) 为新节点分配slot

```bash
redis-trib.rb reshard 10.10.34.14:6386  
#根据提示选择要迁移的slot数量(ps:这里选择500)  
How many slots do you want to move (from 1 to 16384)? 500  
#选择要接受这些slot的node-id  
What is the receiving node ID? f51e26b5d5ff74f85341f06f28f125b7254e61bf  
#选择slot来源:  
#all表示从所有的master重新分配，  
#或者数据要提取slot的master节点id,最后用done结束  
Please enter all the source node IDs.  
Type 'all' to use all the nodes as source nodes for the hash slots.  
Type 'done' once you entered all the source nodes IDs.  
Source node #1:all  
#打印被移动的slot后，输入yes开始移动slot以及对应的数据.  
#Do you want to proceed with the proposed reshard plan (yes/no)? yes  
#结束  
```

#### 3) 添加新的slave节点

a) 前三步操作同添加master一样   
b) 第四步:redis-cli连接上新节点shell,输入命令:cluster replicate 对应master的node-id

```bash
cluster replicate 2b9ebcbd627ff0fd7a7bbcc5332fb09e72788835 
```

注意:在线添加slave 时，需要bgsave整个master数据，并传递到slave，再由 slave加载rdb文件到内存，rdb生成和传输的过程中消耗Master大量内存和网络IO,以此不建议单实例内存过大，线上小心操作。
例如本次添加slave操作产生的rdb文件

```bash
-rw-r--r-- 1 root root  34946 Apr 17 18:23 dump-6386.rdb  
-rw-r--r-- 1 root root  34946 Apr 17 18:23 dump-7386.rdb  
```

#### 4) 在线reshard 数据:

对于负载/数据不均匀的情况，可以在线reshard slot来解决,方法与添加新master的reshard一样，只是需要reshard的master节点是已存在的老节点.

#### 5) 删除一个slave节点

```bash
#redis-trib del-node ip:port '<node-id>'  
redis-trib.rb del-node 10.10.34.14:7386 'c7ee2fca17cb79fe3c9822ced1d4f6c5e169e378'  
```

#### 6)删除一个master节点

a) 删除master节点之前首先要使用reshard移除master的全部slot,然后再删除当前节点(目前redis-trib.rb只能把被删除master的slot对应的数据迁移到一个节点上)

```bash
#把10.10.34.14:6386当前master迁移到10.10.34.14:6380上  
redis-trib.rb reshard 10.10.34.14:6380  
#根据提示选择要迁移的slot数量(ps:这里选择500)  
How many slots do you want to move (from 1 to 16384)? 500(被删除master的所有slot数量)  
#选择要接受这些slot的node-id(10.10.34.14:6380)  
What is the receiving node ID? c4a31c852f81686f6ed8bcd6d1b13accdc947fd2 (ps:10.10.34.14:6380的node-id)  
Please enter all the source node IDs.  
  Type 'all' to use all the nodes as source nodes for the hash slots.  
  Type 'done' once you entered all the source nodes IDs.  
Source node #1:f51e26b5d5ff74f85341f06f28f125b7254e61bf(被删除master的node-id)  
Source node #2:done  
#打印被移动的slot后，输入yes开始移动slot以及对应的数据.  
#Do you want to proceed with the proposed reshard plan (yes/no)? yes  
```

b) 删除空master节点


```bash
redis-trib.rb del-node 10.10.34.14:6386 'f51e26b5d5ff74f85341f06f28f125b7254e61bf'  
```

## 三 redis cluster 客户端(Jedis)

### 1 客户端基本操作使用

```Javascript
private static BinaryJedisCluster jc;  
  static {  
       //只给集群里一个实例就可以  
        Set<HostAndPort> jedisClusterNodes = new HashSet<HostAndPort>();  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 6380));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 6381));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 6382));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 6383));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 6384));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 7380));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 7381));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 7382));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 7383));  
        jedisClusterNodes.add(new HostAndPort("10.10.34.14", 7384));  
        jc = new BinaryJedisCluster(jedisClusterNodes);  
    }  
@Test  
public void testBenchRedisSet() throws Exception {  
        final Stopwatch stopwatch = new Stopwatch();  
        List list = buildBlogVideos();  
        for (int i = 0; i < 1000; i++) {  
            String key = "key:" + i;  
            stopwatch.start();  
            byte[] bytes1 = protostuffSerializer.serialize(list);  
            jc.setex(key, 60 * 60, bytes1);  
            stopwatch.stop();  
        }  
        System.out.println("time=" + stopwatch.toString());  
    }
```

### 2 redis-cluster客户端的一些坑.

1) cluster环境下slave默认不接受任何读写操作，在slave执行readonly命令后，可执行读操作   
2) client端不支持多key操作(mget,mset等)，但当keys集合对应的slot相同时支持mget操作见:hash_tag   
3) 不支持多数据库，只有一个db，select 0。   
4) JedisCluster 没有针对byte[]的API，需要自己扩展(附件是我加的基于byte[]的BinaryJedisCluster  api)   
目前"Jedis-3.0.0-SNAPSHOT"已支持BinaryJedisCluster和基于hash_tag的mget操作.


*参考文档:*

1. [http://redis.io/topics/cluster-spec](http://redis.io/topics/cluster-spec)   
2. [http://redis.io/topics/cluster-tutorial](http://redis.io/topics/cluster-tutorial)
3. [ruby安装](https://my.oschina.net/u/1449160/blog/260764)
