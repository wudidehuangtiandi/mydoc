# 高可用的redis集群搭建

## 1.Redis-cluster集群的搭建及使用

Redis支持三种集群方案

-  主从复制模式
-  Sentinel（哨兵）模式
-  Cluster模式

> 本次着重学习 Redis-cluster集群搭建

我们首先准备三台虚拟机，环境为centos7.6

每台机上安装docker。

redis版本为6.2.6

> 目前redis仅支持docker使用host模式来与 Redis Cluster 兼容

1.安装docker，可以参考本文档的docker模块

2.我们进入三台机器的/home目录下分别建立redis,redisslave。

3.这两个文件夹分别建立conf和data目录，conf下建立redis.conf文件夹，注意是文件夹

4.在redis.conf文件夹内建立配置文件redis.conf

> 原始的配置文件,[地址](https://github.com/redis/redis/blob/unstable/redis.conf)注意不同版本区别,这个配置文件可能有一些问题

在继续之前，我们先介绍一下文件中 Redis Cluster 引入的配置参数`redis.conf`。有些会很明显，有些会随着您继续阅读而变得更加清晰。

- **cluster-enabled`<yes/no>`**：如果是，则在特定 Redis 实例中启用 Redis Cluster 支持。否则，实例将像往常一样作为独立实例启动。
- **cluster-config-file`<filename>`**：请注意，尽管有此选项的名称，但这不是用户可编辑的配置文件，而是 Redis Cluster 节点在每次发生更改时自动持久化集群配置（基本上是状态）的文件，为了能够在启动时重新读取它。该文件列出了集群中的其他节点、它们的状态、持久变量等内容。由于某些消息接收，此文件通常会被重写并刷新到磁盘上。
- **cluster-node-timeout`<milliseconds>`**：Redis 集群节点不可用的最长时间，而不被视为失败。如果主节点在超过指定的时间内无法访问，它将由其副本进行故障转移。此参数控制 Redis Cluster 中的其他重要内容。值得注意的是，在指定时间内无法到达大多数主节点的每个节点都将停止接受查询。
- **cluster-slave-validity-factor`<factor>`**：如果设置为零，则副本将始终认为自己有效，因此将始终尝试故障转移主节点，而不管主节点和副本之间的链接保持断开连接的时间长短。如果该值为正，则计算最大断开时间作为*节点超时*值乘以此选项提供的因子，如果节点是副本，则如果主链接断开连接的时间超过指定的时间，它将不会尝试启动故障转移。例如，如果节点超时设置为 5 秒，有效性因子设置为 10，则与主节点断开连接超过 50 秒的副本将不会尝试对其主节点进行故障转移。请注意，如果没有能够对其进行故障转移的副本，则任何非零值都可能导致 Redis 集群在主故障后不可用。在这种情况下，只有当原始主节点重新加入集群时，集群才会恢复可用。
- **cluster-migration-barrier`<count>`**：master 将保持连接的最小副本数，以便另一个副本迁移到不再被任何副本覆盖的 master。有关更多信息，请参阅本教程中有关副本迁移的相应部分。
- **cluster-require-full-coverage`<yes/no>`**：如果设置为 yes，默认情况下，如果某个百分比的键空间未被任何节点覆盖，集群将停止接受写入。如果该选项设置为 no，即使只能处理有关键子集的请求，集群仍将提供查询服务。
- **cluster-allow-reads-when-down`<yes/no>`**：如果设置为 no，默认情况下，当集群被标记为失败时，Redis 集群中的节点将停止提供所有流量，或者当节点无法到达时达到法定人数或未达到完全覆盖时。这可以防止从不知道集群更改的节点读取可能不一致的数据。可以将此选项设置为 yes 以允许在故障状态期间从节点读取，这对于希望优先考虑读取可用性但仍希望防止写入不一致的应用程序很有用。当使用只有一个或两个分片的 Redis 集群时，也可以使用它，因为它允许节点在主节点失败但无法自动故障转移时继续提供写入服务。

> 一般来说，主 Redis 节点会处理 Clients 的读写操作，而从节点只处理读操作。

下面是一个最低限度的集群配置文件

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

5.我们将该配置文件从port7000开始依次覆盖各个redis.conf。

A 7000 7001

B 7002 7003

C 7004 7005

启动命令如下

```
docker run \
--network host \
--name redis \
-v /home/redis/data:/data \
-v /home/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf/redis.conf

docker run \
--network host \
--name redisslave \
-v /home/redisslave/data:/data \
-v /home/redisslave/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf/redis.conf
```

6.然后我们任意找一台机器进入

```
docker exec -it redis /bin/bash
```

输入命令

```
redis-cli --cluster create 192.168.191.99:7000 192.168.191.99:7001 \
192.168.191.100:7002 192.168.191.100:7003 192.168.191.101:7004 192.168.191.101:7005 \
--cluster-replicas 1
```

> IP为自己得机器得IP 即可贯通，注意需要防火墙开通对应端口

我们得到以下日志

```
M: 70e6f056825c68aaaa4bd12edb3ec03da85c85c6 192.168.191.99:7000
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 76ad8d3f064f0737e26ac1e208790b4b9969c68e 192.168.191.99:7001
   slots: (0 slots) slave
   replicates 79859d19b35bd83d1bd804d10e79d7ebe7298066
M: 0fee1355fa58624072e1e4f4132763869c7fbfdc 192.168.191.100:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 79859d19b35bd83d1bd804d10e79d7ebe7298066 192.168.191.101:7004
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: a2ec84ba7e80a3b59805a0161bd87c25bb00fd55 192.168.191.101:7005
   slots: (0 slots) slave
   replicates 0fee1355fa58624072e1e4f4132763869c7fbfdc
S: 9f8cf56f7ebddad614d006d93141d054244f2b2c 192.168.191.100:7003
   slots: (0 slots) slave
   replicates 70e6f056825c68aaaa4bd12edb3ec03da85c85c6
```

我们可以发现主从自动分配了，可以手动指定主从，但是当出问题后又会切换所以意义不大。但是主从的数量是稳定的

我们任意关闭主节点docker可以发现从节点顺利接替升级。

> 在docker的redis下，我们可以查通过以下命令对集群作出一些操作

```
首先登录redis（此处下文中的进入容器中redi命令行指的就是这里）
redis-cli -c -p 7000 -h 192.168.191.99
之后
集群
cluster info ：打印集群的信息
cluster nodes ：列出集群当前已知的所有节点（ node），以及这些节点的相关信息。
节点
cluster meet <ip> <port> ：将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。
cluster forget <node_id> ：从集群中移除 node_id 指定的节点。
cluster replicate <master_node_id> ：将当前从节点设置为 node_id 指定的master节点的slave节点。只能针对slave节点操作。
cluster saveconfig ：将节点的配置文件保存到硬盘里面。
槽(slot)
cluster addslots <slot> [slot ...] ：将一个或多个槽（ slot）指派（ assign）给当前节点。
cluster delslots <slot> [slot ...] ：移除一个或多个槽对当前节点的指派。
cluster flushslots ：移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。
cluster setslot <slot> node <node_id> ：将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给
另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。
cluster setslot <slot> migrating <node_id> ：将本节点的槽 slot 迁移到 node_id 指定的节点中。
cluster setslot <slot> importing <node_id> ：从 node_id 指定的节点中导入槽 slot 到本节点。
cluster setslot <slot> stable ：取消对槽 slot 的导入（ import）或者迁移（ migrate）。
键
cluster keyslot <key> ：计算键 key 应该被放置在哪个槽上。
cluster countkeysinslot <slot> ：返回槽 slot 目前包含的键值对数量。
cluster getkeysinslot <slot> <count> ：返回 count 个 slot 槽中的键 。
```

> 我们可以尝试停止一个master的docker容器，可以发现对应的slave接管了它的工作

下面我们来学一下如何扩容，我们新建两个docker容器，分别叫redis2及redisslave2,占用机器A的7008及7009端口

```

docker run \
--network host \
--name redis2 \
-v /home/redis2/data:/data \
-v /home/redis2/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf/redis.conf

docker run \
--network host \
--name redisslave2 \
-v /home/redisslave2/data:/data \
-v /home/redisslave2/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf/redis.conf
```

!>注意，这边由于redis的nodes.conf会持久化到磁盘，所以data文件请不要复制照抄，否则会出现一台机器上的多个redis互相顶替的情况。

启动后我们进入A机器的redis容器,登录redis后,使用以下命令加入redis集群，之后cluster nodes查看下状态

```
CLUSTER MEET 192.168.191.99 7008
CLUSTER MEET 192.168.191.99 7009
```

可见如下

```
79859d19b35bd83d1bd804d10e79d7ebe7298066 192.168.191.101:7004@17004 master - 0 1644977247355 8 connected 10923-16383
76ad8d3f064f0737e26ac1e208790b4b9969c68e 192.168.191.99:7001@17001 slave 79859d19b35bd83d1bd804d10e79d7ebe7298066 0 1644977248901 8 connected
8e61ee80d2aa761b22e834797adb4220d27b9b20 192.168.191.99:7009@17009 master - 0 1644977247000 10 connected
f6a5503a891984ab4eb867c78e1578f6aad82488 192.168.191.99:7008@17008 master - 0 1644977247562 0 connected
9f8cf56f7ebddad614d006d93141d054244f2b2c 192.168.191.100:7003@17003 myself,master - 0 1644977246000 9 connected 0-5460
0fee1355fa58624072e1e4f4132763869c7fbfdc 192.168.191.100:7002@17002 master - 0 1644977248389 3 connected 5461-10922
70e6f056825c68aaaa4bd12edb3ec03da85c85c6 192.168.191.99:7000@17000 slave 9f8cf56f7ebddad614d006d93141d054244f2b2c 0 1644977248000 9 connected
a2ec84ba7e80a3b59805a0161bd87c25bb00fd55 192.168.191.101:7005@17005 slave 0fee1355fa58624072e1e4f4132763869c7fbfdc 0 1644977248000 3 connected
```

我们可以看到7008成功作为master加入到了集群，但是此时它还没有分片。

> 新节点刚开始都是主节点状态，由于没有负责的槽，所以不能接受任何读写操作，后续我们就给他迁移槽和填充数据。

这里需要了解以下redis集群的数据分片机制，简单来介绍下

Redis Cluster 采用虚拟哈希槽分区，所有的键根据哈希函数映射到 0 ~ 16383 整数槽内，计算公式：slot = CRC16(key) & 16383。每一个节点负责维护一部分槽以及槽所映射的键值数据。

**Redis 虚拟槽分区的特点**

- 解耦数据和节点之间的关系，简化了节点扩容和收缩难度。
- 节点自身维护槽的映射关系，不需要客户端或者代理服务维护槽分区元数据
- 支持节点、槽和键之间的映射查询，用于数据路由，在线集群伸缩等场景。

我们计算下从三个Master到四个master，每个Master应当分出大约1365个槽位给目标

我们exit退到容器命令行，重复三次以下命令（从不同的master）分出指定槽位给7008

```
redis-cli --cluster reshard <host>:<port> --cluster-from <node-id> --cluster-to <node-id> --cluster-slots <number of slots> --cluster-yes
如：
redis-cli --cluster reshard 192.168.191.99:7008 --cluster-from 79859d19b35bd83d1bd804d10e79d7ebe7298066 --cluster-to f6a5503a891984ab4eb867c78e1578f6aad82488 --cluster-slots 1365 --cluster-yes
```

我们再次进入容器中的redis，cluster nodes可以看到如下内容

```
79859d19b35bd83d1bd804d10e79d7ebe7298066 192.168.191.101:7004@17004 master - 0 1644977949167 8 connected 12288-16383
76ad8d3f064f0737e26ac1e208790b4b9969c68e 192.168.191.99:7001@17001 slave 79859d19b35bd83d1bd804d10e79d7ebe7298066 0 1644977949000 8 connected
8e61ee80d2aa761b22e834797adb4220d27b9b20 192.168.191.99:7009@17009 master - 0 1644977948074 10 connected
f6a5503a891984ab4eb867c78e1578f6aad82488 192.168.191.99:7008@17008 master - 0 1644977948556 11 connected 0-1364 5461-6825 10923-12287
9f8cf56f7ebddad614d006d93141d054244f2b2c 192.168.191.100:7003@17003 myself,master - 0 1644977947000 9 connected 1365-5460
0fee1355fa58624072e1e4f4132763869c7fbfdc 192.168.191.100:7002@17002 master - 0 1644977950225 3 connected 6826-10922
70e6f056825c68aaaa4bd12edb3ec03da85c85c6 192.168.191.99:7000@17000 slave 9f8cf56f7ebddad614d006d93141d054244f2b2c 0 1644977949000 9 connected
a2ec84ba7e80a3b59805a0161bd87c25bb00fd55 192.168.191.101:7005@17005 slave 0fee1355fa58624072e1e4f4132763869c7fbfdc 0 1644977948609 3 connected
```

可以发现7008小老弟已经被分到了三个槽位。

下面我们进入 7009，用以下命令给7008分配一个小弟

```
redis-cli -c -p 7009 -h 192.168.191.99 

cluster replicate f6a5503a891984ab4eb867c78e1578f6aad82488
```

然后我们cluster nodes看下，发现7009已经成功作为7008的随从了。

```
76ad8d3f064f0737e26ac1e208790b4b9969c68e 192.168.191.99:7001@17001 slave 79859d19b35bd83d1bd804d10e79d7ebe7298066 0 1644978443534 8 connected
f6a5503a891984ab4eb867c78e1578f6aad82488 192.168.191.99:7008@17008 master - 0 1644978443431 11 connected 0-1364 5461-6825 10923-12287
9f8cf56f7ebddad614d006d93141d054244f2b2c 192.168.191.100:7003@17003 master - 0 1644978444577 9 connected 1365-5460
70e6f056825c68aaaa4bd12edb3ec03da85c85c6 192.168.191.99:7000@17000 slave 9f8cf56f7ebddad614d006d93141d054244f2b2c 0 1644978443000 9 connected
79859d19b35bd83d1bd804d10e79d7ebe7298066 192.168.191.101:7004@17004 master - 0 1644978444000 8 connected 12288-16383
0fee1355fa58624072e1e4f4132763869c7fbfdc 192.168.191.100:7002@17002 master - 0 1644978443534 3 connected 6826-10922
a2ec84ba7e80a3b59805a0161bd87c25bb00fd55 192.168.191.101:7005@17005 slave 0fee1355fa58624072e1e4f4132763869c7fbfdc 0 1644978444474 3 connected
8e61ee80d2aa761b22e834797adb4220d27b9b20 192.168.191.99:7009@17009 myself,slave f6a5503a891984ab4eb867c78e1578f6aad82488 0 1644978444000 11 connected
```

> 至此我们能大致了解如何扩容。

至于缩容，原理是相似的，主节点缩容前需要将自己负责的槽迁移给其它节点，只要有一个槽位未被分配，集群就不可用

之后可以使用以下命令使得节点从集群中移除即可，这里就不做演示了。

```
cluster forget <node_id>
```

集群的操作远不止此，我们开发人员到这里就差不多了，之后如果还有什么需要操作的地方再行补充。

> 接下来我们来研究下如何使用图形化工具监控及java代码如何接入

首先集群监控工具

![avatar](https://picture.zhanghong110.top/docsify/16449807379231.png)

点击这里就相当于链接了整个集群

可以看到我们配置的99实际上先连上了100的master

![avatar](https://picture.zhanghong110.top/docsify/1644981087714.png)

![avatar](https://picture.zhanghong110.top/docsify/16449811045380.png)



> 我们再来看java端,连接池我们采用`springboot`默认的撸菜来实现

引入如下依赖，默认撸菜连接池，也可以自己用其他的连接池，配置如下图所示

```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

![avatar](https://picture.zhanghong110.top/docsify/16449902794885.png)

启动类中稍微做一下序列化配置，这边为了演示直接放在启动类里了

![avatar](https://picture.zhanghong110.top/docsify/16449904827027.png)

简单写个测试接口自增

![avatar](https://picture.zhanghong110.top/docsify/16449905964238.png)

我们用工具查一下，可以发现没有问题，数据被分配到了100上的7002

![avatar](https://picture.zhanghong110.top/docsify/16449907203996.png)

我们回到服务器去验证一下

随便挑一台，这次我们用C服务器也就是101

```
docker exec -it redis /bin/bash

redis-cli -c -p 7005 -h 192.168.191.101
```

进去了查一下test这个key被分配到了哪个节点

```
cluster keyslot test
```

得到结果

```
(integer) 6918
```

可见槽位是6918

我们用以下命令再查一下

```
cluster nodes
```

得到结果

```
a2ec84ba7e80a3b59805a0161bd87c25bb00fd55 192.168.191.101:7005@17005 myself,slave 0fee1355fa58624072e1e4f4132763869c7fbfdc 0 1644990097000 3 connected
9f8cf56f7ebddad614d006d93141d054244f2b2c 192.168.191.100:7003@17003 master - 0 1644990098622 9 connected 1365-5460
f6a5503a891984ab4eb867c78e1578f6aad82488 192.168.191.99:7008@17008 slave 8e61ee80d2aa761b22e834797adb4220d27b9b20 0 1644990098000 12 connected
76ad8d3f064f0737e26ac1e208790b4b9969c68e 192.168.191.99:7001@17001 slave 79859d19b35bd83d1bd804d10e79d7ebe7298066 0 1644990097140 8 connected
0fee1355fa58624072e1e4f4132763869c7fbfdc 192.168.191.100:7002@17002 master - 0 1644990098519 3 connected 6826-10922
70e6f056825c68aaaa4bd12edb3ec03da85c85c6 192.168.191.99:7000@17000 slave 9f8cf56f7ebddad614d006d93141d054244f2b2c 0 1644990097409 9 connected
79859d19b35bd83d1bd804d10e79d7ebe7298066 192.168.191.101:7004@17004 master - 0 1644990098519 8 connected 12288-16383
8e61ee80d2aa761b22e834797adb4220d27b9b20 192.168.191.99:7009@17009 master - 0 1644990097515 12 connected 0-1364 5461-6825 10923-12287
```

可见确实被分配到了7002的master

我们最后直接在服务器的redis里get下

```
get test
```

得到结果，与工具一致

```
-> Redirected to slot [6918] located at 192.168.191.100:7002
"68"
```

至此，一个高可用的redis-cluster集群的搭建及使用方式就明确了



> 后期我们还要去增加密码和设置一些配置，以及明确哪些操作会降低效率，之后如果有需要可以自行了解，本文不在赘述。



## 2.利用redis实现分布锁

> 在很多场景中，为了保证数据的最终一致性，我们可能会用到分布式锁。分布式锁通常有如下几种实现方式

```
基于数据库实现分布式锁；
基于缓存（Redis等）实现分布式锁；
基于Zookeeper实现分布式锁；
```

在澳洋HPV项目中使用到了基于redis实现的分布式锁，这边记录下

首先为何选型redis来实现分布式锁

（1）Redis有很高的性能；
（2）Redis命令对此支持较好，实现起来比较方便

redis实现分布式锁主要用到了它的`setnx`命令，`setnx`是`SET if not exists`(如果不存在，则 SET)的简写。

```
127.0.0.1:6379> setnx lock value1 #在键lock不存在的情况下，将键key的值设置为value1
(integer) 1
127.0.0.1:6379> setnx lock value2 #试图覆盖lock的值，返回0表示失败
(integer) 0
127.0.0.1:6379> get lock #获取lock的值，验证没有被覆盖
"value1"
127.0.0.1:6379> del lock #删除lock的值，删除成功
(integer) 1
127.0.0.1:6379> setnx lock value2 #再使用setnx命令设置，返回0表示成功
(integer) 1
127.0.0.1:6379> get lock #获取lock的值，验证设置成功
"value2"
```

上面这几个命令就是最基本的用来完成分布式锁的命令。

加锁：使用`setnx key value`命令，如果key不存在，设置value(加锁成功)。如果已经存在lock(也就是有客户端持有锁了)，则设置失败(加锁失败)。

解锁：使用`del`命令，通过删除键值释放锁。释放锁之后，其他客户端可以通过`setnx`命令进行加锁。

key的值可以根据业务设置，比如是用户中心使用的，可以命令为`USER_REDIS_LOCK`，value可以使用uuid保证唯一，用于标识加锁的客户端。保证加锁和解锁都是同一个客户端。

> redis分布式锁实现起来主要会涉及以下几个问题

1.如何避免死锁？

```
解决方式是给key增加过期时间，超时自动解锁即可
```

2.锁被其它客户端释放了怎么办

```
解决方式是加锁时设置唯一标识，比如线程ID或者UUID
```

3.释放锁如何保证原子性

```
在这里，释放锁的时候会产生一个问题，即判断锁是否为自己所有，使用了GET+DEL会涉及到原子性问题
采用的解决方案是使用lua脚本，让redis来执行，由于redis处理每个请求都是单线程执行的，所lua脚本执行会是一个原子操作
```



!>值得注意的是，在加锁的环节由于redis 2.6.12后可以使用 `SET lock 1 EX 10 NX` 命令，故而是原子性操作，这个在redisTemplate中使用`setIfAbsent`可以完美解决



下面我们看一下具体的代码实现

```java
package com.ay.utils;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Component;

import java.util.Collections;
import java.util.concurrent.TimeUnit;

/**
 * 基于setnx实现redis分布式锁
 * <p>
 * by gc_snow
 */
@Component
public class RedisLockUtil {

    private static final DefaultRedisScript<Long> redisScript;
    private static final Long RELEASE_SUCCESS = 1L;

    /**
     * lua脚本,用以保证这条命令的原子性
     */
    static {
        DefaultRedisScript defaultRedisScript = new DefaultRedisScript();
        defaultRedisScript.setResultType(Long.class);
        defaultRedisScript.setScriptText("if (redis.call('get', KEYS[1]) == ARGV[1]) then return redis.call('del', KEYS[1]) end return 0 ");
        redisScript = defaultRedisScript;
    }

    @Autowired
    private StringRedisTemplate redisTemplate;
    private long timeout = 5000;

    /**
     * 上锁
     *
     * @param key   锁标识（锁名称）同一个锁名称并发则失败,名称不同的锁不会互相阻塞。
     * @param value 线程标识
     * @return 上锁状态
     */
    public boolean lock(String key, String value) throws InterruptedException {
        long start = System.currentTimeMillis();
        //非阻塞执行一次，
        while (true) {
            //检测是否超时
            if (System.currentTimeMillis() - start > timeout) {
                return false;
            }
            //执行set命令,setIfAbsent设置锁原子性保证及过期时间增加保证不会死锁
            Boolean absent = redisTemplate.opsForValue().setIfAbsent(key, value, timeout, TimeUnit.MILLISECONDS);//1
            //是否成功获取锁，防止自动拆箱造成问题
            if (absent) {
                return true;
            }
            return false;

        }
    }

    /**
     * 解锁
     *
     * @param key   锁标识
     * @param value 线程标识
     * @return 解锁状态
     */
    public boolean unlock(String key, String value) {
        //使用Lua脚本：先判断是否是自己设置的锁，再执行删除
        Long result = redisTemplate.execute(redisScript, Collections.singletonList(key), value);
        //返回最终结果
        return RELEASE_SUCCESS.equals(result);
    }

}
```



> 至此我们应当明白如何利用redis实现一个分布式锁