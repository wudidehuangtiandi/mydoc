# MYSQL

## 一.mysql集群

> mysql集群主要有两种搭建方式

**replication:** 异步复制，速度快，无法保证数据的一致性。
**pxc:** 同步复制，速度慢，多个集群之间是事务提交的数据一致性强。

![avatar](https://picture.zhanghong110.top/docsify/msqb12332112345677222.png)

> 在很多情况下，数据不同步可能是我们无法接受的，所以我们主要研究第二种方案。如果需要前者可以去这个链接看下，比较全[地址](https://www.xdx97.com/article/712671146654302208)

PXC全称是Percona XtraDB Cluster 是著名的Mysql咨询公司Percona 出品的免费的Mysql集群产品，它基于Galera技术。我们只要知道这玩意屌的一比就行了，是目前实现集群的最佳方案。

!>PXC集群节点不是越大越好，随着规模变大，效率会变差。并且同步速度取决于集群中性能最差的节点，然后的话只有InnoDB引擎才会被同步，其它的引擎建议使用replication。

> PXC至少需要三台机器来完成，否则会出现脑裂。所谓脑裂就是说能正常链接的节点必须超过半数，否则假如两台机器，当链接中断后就可能出现链接两边的服务访问的mysql库数据不一致的情况。

下面我们来演示下如何装一个`pxc`集群，我们借助了`docker`环境,系统环境为`centos7.6`

前期准备

`centos7.6`需要关闭防火墙，或者开启某些需要的端口；`pxc`会自带`mysql`，版本是对应一致的，所以机子上不需要`mysql`；最好关闭`SELINUX`，``centos7.6``自带的安全增强。

注意这些配置，三台机子上都要操作。

开放`pxc`所需端口

| 端口 | 功能                     |
| :--- | :----------------------- |
| 3306 | mysql数据库              |
| 4567 | pxc cluster 相互通讯端口 |
| 4444 | sst全量传输              |
| 4568 | ist增量传输              |

我们还会用到`docker`集群工具`swarm`,`swarm`也需要一些端口的开放，当然如果你是关闭防火墙就无需多言

| 端口 | 功能         |
| :--- | :----------- |
| 2377 | 用于集群通信 |
| 4789 | 容器覆盖网络 |
| 7946 | 容器网络发现 |

```shell
#我们先拉镜像，拉一个5.7的版本
docker pull percona/percona-xtradb-cluster:5.7.21

#使用它创建一个镜像命名为pxc
docker tag percona/percona-xtradb-cluster:5.7.21 pxc

#下面操作swam,我们挑选一台机器作为swarm主节点
docker swarm init
#输出以下内容
Swarm initialized: current node (kggw0pf8otojdajagtf3m9h9m) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-29pm3zuvlp0zxvlish4yn4dsfwmvg8whhk35h1f1cmzqwou92a-027ra3kkpjuwdlt2mertmi1rm 192.168.191.99:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

#其它两个节点加入，使用它打出的日志
docker swarm join --token SWMTKN-1-29pm3zuvlp0zxvlish4yn4dsfwmvg8whhk35h1f1cmzqwou92a-027ra3kkpjuwdlt2mertmi1rm 192.168.191.99:2377
#输出以下内容
This node joined a swarm as a worker.

#这样我们的swarm便搭建完成，我们回到主机看一下
[root@redismaster1 ~]# docker node ls
ID                            HOSTNAME       STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
kggw0pf8otojdajagtf3m9h9m *   redismaster1   Ready     Active         Leader           20.10.12
bpdshnp2fmicf018m642upx15     redismaster2   Ready     Active                          20.10.12

#下面我们在主机创建一个虚拟网络，类型为overlay，可能没接触过，是docker提供的跨主机网络方案
docker network create -d overlay --attachable swarm_mysql

#查看下
[root@redismaster1 ~]# docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
2g8rkp2klu0d   swarm_mysql       overlay   swarm

#如果你是mysql8.0+且没有使用相同的证书，那么你肯定会遇到一个ssl相关的错误
#“error:0407008A:rsa routines:RSA_padding_check_PKCS1_type_1:invalid padding”
#这是因为8.0后，是ssl来连接，三台机子，就必须保持密钥的一致性才可以通信。
#这是 官方的解决方案https://www.percona.com/doc/percona-xtradb-cluster/LATEST/install/docker.html ，生成证书，大家使用同一套。
#可以看下这个文档https://www.jb51.net/article/206277.htm

#三台机器创建下文件路径
cd /home
#mysql自定义配置文件
mkdir -m 777 pxc_config
#数据
mkdir -m 777 pxc_data

#开始创建集群，主节点（防火墙刚关需要重启docker）
docker run -d -p 3306:3306 --net=swarm_mysql \
-e MYSQL_ROOT_PASSWORD=test123 \
-e CLUSTER_NAME=pxc_cluster \
-e XTRABACKUP_PASSWORD=test123 \
-v /home/pxc_data:/var/lib/mysql \
-v /home/pxc_config/:/etc/percona-xtradb-cluster.conf.d \
--privileged --name=pxc1 pxc

#解读下
#docker run -d 
#-p 3306:3306 3306端口映射
#--net=swarm_mysql 虚拟网络名字
#-e MYSQL_ROOT_PASSWORD=test123 数据库初始密码
#-e CLUSTER_NAME=pxc_cluster 集群名字
#-e XTRABACKUP_PASSWORD=test123 备份密码
#-v /home/pxc:/var/lib/mysql pxc路径映射 
#-v /home/pxc/config/:/etc/percona-xtradb-cluster.conf.d mysql配置文件路径映射
#--privileged 给予权限
#--name=pxc1 pxc

#创建两个从节点,加入主节点
docker run -d -p 3306:3306 --net=swarm_mysql \
-e MYSQL_ROOT_PASSWORD=test123 \
-e CLUSTER_NAME=pxc_cluster \
-e XTRABACKUP_PASSWORD=test123 \
-v /home/pxc_data:/var/lib/mysql \
-v /home/pxc_config/:/etc/percona-xtradb-cluster.conf.d \
-e CLUSTER_JOIN=pxc1 \
--privileged --name=pxc2 pxc

docker run -d -p 3306:3306 --net=swarm_mysql \
-e MYSQL_ROOT_PASSWORD=test123 \
-e CLUSTER_NAME=pxc_cluster \
-e XTRABACKUP_PASSWORD=test123 \
-v /home/pxc_data:/var/lib/mysql \
-v /home/pxc_cert:/cert \
-v /home/pxc_config/:/etc/percona-xtradb-cluster.conf.d \
-e CLUSTER_JOIN=pxc1 \
--privileged --name=pxc2 pxc
```

至此我们的一个三点集群就安装好了,我们测试下,可以发现，可以完美同步。

![avatar](https://picture.zhanghong110.top/docsify/16537236879261.png)

我们来看下这个集群部分`pxc`集群的部分参数的查看及配置

随便连一个数据库。

```sql
--查看和pxc相关得参数
show status like "%WSREP%"
```
节点信息

![avatar](https://picture.zhanghong110.top/docsify/16537247794795.png)

队列信息

![avatar](https://picture.zhanghong110.top/docsify/16537249615603.png)

流控信息

这边要注意wsrep_flow_control_interval	[ 173, 173 ]，流控最小值和最大值都是173，当达队列积压值达到后会很坑爹，就会触发流控导致拒绝其它同步请求。这些东西可以在创建时得配置文件中优化，这边先略过，以后有真的需求得时候可以优化。值得一提得是，当节点加入时需要先手动同步下数据库，避免全量同步。

![avatar](https://picture.zhanghong110.top/docsify/16537251385801.png)

> 这个集群初步得搭建就到这里了，实际过程中应该多少还有些问题，我们遇到了再进行处理

## 二.条件位置和连接查询执行顺序

> 我们可能经常这种left join，inner join这种连接，却发现居然只是模糊的认识。

那么，当条件跟在`where`后面和条件在`and`后面时有什么区别呢？

我们使用`m` 和 `s`两张表做个演示，内容如下

`m`:

![avatar](https://picture.zhanghong110.top/docsify/16621670659303.png)

`s`:

![avatar](https://picture.zhanghong110.top/docsify/16621640883680.png)

这样我们使用如下语句对比结果

1.首先是`left join` 

> 我们对a表做对比

```sql
SELECT * FROM `m` a  left join `s` b  on a.id = b.mid  where a.delete=0 order by a.id 
```

结果为

![avatar](https://picture.zhanghong110.top/docsify/16621671749751.png)

可以看到。测试数据2应为`delete=0`完全没有显示。这种是`left join`正常筛选左表的方式。



如果我们在`on`上对`a`表进行操作

```sql
SELECT * FROM `m` a  left join `s` b  on a.id = b.mid and a.delete=0 order by a.id 
```

结果为

![avatar](https://picture.zhanghong110.top/docsify/16621672401373.png)

正常如果不加 条件 `and a.delete=0 ` ，结果如下

![avatar](https://picture.zhanghong110.top/docsify/16621672768405.png)

一般查资料我们会得到`left join` 如果在`on`上加上左表条件将会不生效，实际我们发现，结果是左表记录都会被查出，但是左表不符合条件的列将不参与链接，导致其对应的右表数据为`null`



我们接下来试下对b表做条件

我们还是先用`where`做条件

```sql
SELECT * FROM `m` a  left join `s` b  on a.id = b.mid where b.delete= 0 order by a.id 
```

![avatar](https://picture.zhanghong110.top/docsify/16621674595800.png)

如果加在`on`后面

```sql
SELECT * FROM `m` a  left join `s` b  on a.id = b.mid and b.delete= 0 order by a.id
```

![avatar](https://picture.zhanghong110.top/docsify/16621681197181.png)

可以看出实际上`where`是链接后筛选了，由于`left join`的关系，我们关注的是`a`表，想要`a`表即使有记录未匹配到`b`表也可以被查询到。但是筛选的时候我们就等于没有了这一项。所以我们将`b`表条件放在`on`中比较合理，此时该条件只会影响`b`表中数据。



!>right join 也同理

2.下面我们对比下 `join (inner join)`

> 首先我们还是先对a表做操作

```sql
SELECT * FROM `m` a   join `s` b  on a.id = b.mid  where a.delete=0 order by a.id 

SELECT * FROM `m` a   join `s` b  on a.id = b.mid and a.delete=0 order by a.id 
```

这两条语句结果都是

![avatar](https://picture.zhanghong110.top/docsify/16621699706344.png)

可以看到，无论是先筛选再参与来链接还是先链接再筛选，对于`inner join`来说都是一样的。

下面对`b`表做操作

```sql
SELECT * FROM `m` a   join `s` b  on a.id = b.mid where b.delete= 0 order by a.id 

SELECT * FROM `m` a   join `s` b  on a.id = b.mid and b.delete= 0 order by a.id
```

![avatar](https://picture.zhanghong110.top/docsify/16621708432405.png)

可以发现对于`b`表来说结果还都是一样的。

可知这个条件仅对外链查询有影响。

> 由此我们可以获得如下结论

1.`where`和`on`的条件位置仅对外链查询有影响

2.内联查询条件位置不影响

3.外链查询对链接主表操作时应当将条件放在`where`中，如果放在`on`中，则不符合条件的不参与链接，但是依然会被查询出来。

4.外链查询对从表操作时应当将条件放在`on`中，否则如果放在`where`中外链查询将会转换为和内链查询一样的作用，当放在`on`中时将会筛选从表而不影响主表。



!>场景思考：

在实际操作过程中，有一个这样的场景，一条有十几万条记录的主表，`left join`查询一条五百万数据的表，`on`项都加了`b+`索引当主表加上`where`条件匹配少数列时查询成功，否则就慢查询，查询失败。



由此我们思考下，当如果主从表先做积，再筛选，就是十几万条和几百万条的积，再做筛选，这样就是不加条件，筛选失败，显然`mysql`内部不是这样筛选的。



这里我们详细描述下这个场景

A表 十万条数据

B表 五百万条数据

C表 八万条数据

A左联B左联C ，所有的链接条件及where条件均带有B+索引；

 当给出A的where条件，此时`explain`下可以发现，此时A,B,C表均使用索引，遍历的rows均在50以下，一秒即可查询出结果。



实验一：

1 当A条件不给，A直接左联C，费时18秒，可以查询出结果。`explain`A为全表查询，C为索引查询。

2 当A条件不给，A直接左联B，无法查询出结果。`explain`A为全表查询，B为索引查询。

下面我们增加一组对比

实验二：

上面 1，2两步骤，分别都给A加上where条件，`explain`下再看，它两可以发现都能够非常快速的查到结果，分别耗时0.07秒和0.1秒

我们将这两个带where步骤命名为3和4

实验三：

此时我们在做一个操作，去除where条件下的索引，再执行一次。 `explain`A为全表查询，此时分别耗时0.8和0.9秒就查出了结果。

> 至此我们基本可以看出，在保证链接字段带有索引的情况下，影响点有三个，一是主表的数据范围，二是子表数据量，三是主表筛选字段索引

由于A表的筛选字段是否索引在这次关联查询中影响较小，基本等同于A表单表查询的速度。我们可以简单推测，若是将ABC表先做`left join`，再进行`where`操作，显然不可能是这样的结果，这样第一A表的where字段索引应当影响较大，第二数据量会非常大应当难以查询出来。



讲了这么多，结论是啥，我们可以看下[官网](https://dev.mysql.com/doc/refman/5.7/en/condition-filtering.html),

```
在没有条件过滤的情况下，表的前缀行数基于 WHERE子句根据优化器选择的访问方法选择的估计行数。条件过滤使优化器能够使用 WHERE访问方法未考虑的子句中的其他相关条件，从而改进其前缀行计数估计。
```

可以看到`mysql`确实会根据`where`条件优先过滤。这个值在`explain`中的`filter`值中可以体现。所以，链接查询确实不是和网上很多人写的一样简单的直接先做积再筛选的。

## 三.锁机制


首先对mysql锁进行划分：

1. 按照锁的粒度划分：行锁、表锁、页锁
2. 按照锁的使用方式划分：共享锁、排它锁（悲观锁的一种实现）
3. 还有两种思想上的锁：悲观锁、乐观锁。
4. InnoDB中有几种行级锁类型：Record Lock、Gap Lock、Next-key Lock

> MySQL的锁机制最显著的特点是不同的存储引擎支持不同的锁机制。比如，MyISAM和MEMORY存储引擎采用的是表级锁（table-level locking）；BDB存储引擎采用的是页面锁（page-level locking），但也支持表级锁；InnoDB存储引擎既支持行级锁（row-level locking），也支持表级锁，但默认情况下是采用行级锁。

行级锁是Mysql中锁定粒度最细的一种锁，表示只针对当前操作的行进行加锁。**行级锁能大大减少数据库操作的冲突。其加锁粒度最小，但加锁的开销也最大。有可能会出现死锁的情况**。 行级锁按照使用方式分为共享锁和排他锁。



**共享锁用法（S锁 读锁）：**
若事务T对数据对象A加上S锁，则事务**T可以读A但不能修改A**，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。

```sql
select ... lock in share mode;
```

共享锁就是允许多个线程同时获取一个锁，一个锁可以同时被多个线程拥有。



**排它锁用法（X 锁 写锁）：**
 若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。这保证了其他事务在T释放A上的锁之前不能再读取和修改A。

```sql
select ... for update
```

排它锁，也称作独占锁，一个锁在某一时刻只能被一个线程占有，其它线程必须等待锁被释放之后才可能获取到锁



!>MySQL的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如 MyISAM引擎就不支持行锁。不支持行锁意味着并发控制只能使用表锁，对于这种引擎的表，同 一张表上任何时刻只能有一个更新在执行，这就会影响到业务并发度。InnoDB是支持行锁的， 这也是MyISAM被InnoDB替代的重要原因之一。



> 表锁和页锁比较类似，字面意思，这边先不介绍了



**乐观锁和悲观锁**

在关系数据库管理系统里，悲观并发控制（又名“悲观锁”，Pessimistic Concurrency Control，缩写“PCC”）是一种并发控制的方法。它可以阻止一个事务以影响其他用户的方式来修改数据。如果一个事务执行的操作对某行数据应用了锁，那只有当这个事务把锁释放，其他事务才能够执行与该锁冲突的操作。悲观并发控制主要用于数据争用激烈的环境，以及发生并发冲突时使用锁保护数据的成本要低于回滚事务的成本的环境中。

> 悲观锁

悲观锁，正如其名，它指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度(悲观)，因此，在整个数据处理过程中，将数据处于锁定状态。 悲观锁的实现，往往依靠数据库提供的锁机制 （也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据

mysql中实现悲观锁的具体流程：

在对任意记录进行修改前，先尝试为该记录加上排他锁（exclusive locking）
如果加锁失败，说明该记录正在被修改，那么当前查询可能要等待或者抛出异常。 具体响应方式由开发者根据实际需要决定。
如果成功加锁，那么就可以对记录做修改，事务完成后就会解锁了。
其间如果有其他对该记录做修改或加排他锁的操作，都会等待我们解锁或直接抛出异常。

总而言之就是一句话：mysql中悲观锁的实现是通过排他锁来实现的

```mysql
1.开始事务
begin;/begin work;/start transaction; (三者选一就可以)
2.查询出商品信息
select ... for update;(这里是使用的行锁的排他锁)
4.提交事务
commit;/commit work;
```

**悲观锁的优点和不足：**
悲观锁实际上是采取了“先取锁在访问”的策略，为数据的处理安全提供了保证，但是在效率方面，由于额外的加锁机制产生了额外的开销，并且增加了死锁的机会。并且降低了并发性；当一个事务加锁一行数据的时候，其他事务必须等待该事务提交之后，才能操作这行数据。



> 乐观锁

在关系数据库管理系统里，乐观并发控制（又名“乐观锁”，Optimistic Concurrency Control，缩写“OCC”）是一种并发控制的方法。它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。如果其他事务有更新的话，正在提交的事务会进行回滚。

乐观锁（ Optimistic Locking ） 相对悲观锁而言，乐观锁假设认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则让返回用户错误的信息，让用户决定如何去做。

相对于悲观锁，在对数据库进行处理的时候，乐观锁并不会使用数据库提供的锁机制。一般的实现乐观锁的方式就是记录数据版本。

mysql实现乐观锁一般来说有2种方式：
1.使用数据版本（Version）记录机制实现，这是乐观锁最常用的一种实现方式。
一般是通过为数据库表增加一个数字类型的 “version” 字段来实现。当读取数据时，将version字段的值一同读出，数据每更新一次，对此version值加一。
当提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，就进行更新操作，否则认为是过期数据，正在提交的事务会进行回滚。
2.第二种实现方式和第一种差不多，同样是在需要乐观锁控制的table中增加一个字段，名称无所谓，字段类型使用时间戳（timestamp）, 和上面的version类似，也是在更新提交的时候检查当前数据库中数据的时间戳和自己更新前取到的时间戳进行对比，如果一致就更新，否则就是版本冲突。

乐观锁的优点和不足：
 乐观并发控制相信事务之间的数据竞争(data race)的概率是比较小的，因此尽可能直接做下去，直到提交的时候才去锁定，所以不会产生任何锁和死锁。但如果直接简单这么做，还是有可能会遇到不可预期的结果，例如两个事务都读取了数据库的某一行，经过修改以后写回数据库，这时就遇到了问题。

```sql
select version from xxx where id = xx;

update xxx set version=version+1 where id = xx and version = 上面查出来的version
```

乐观并发控制相信事务之间的数据竞争(data race)的概率是比较小的，因此尽可能直接做下去，直到提交的时候才去锁定，所以不会产生任何锁和死锁。但如果直接简单这么做，还是有可能会遇到不可预期的结果，例如两个事务都读取了数据库的某一行，经过修改以后写回数据库，这时就遇到了问题。



> InnoDB的特性

1.在不通过索引条件查询的时候，InnoDB使用的确实是表锁（锁的是整张表）！
2.由于 MySQL 的行锁是针对索引加的锁,不是针对记录加的锁,所以虽然是访问不同行的记录,但是如果是使用相同的索引键,是会出现锁冲突的。
3.当表有多个索引的时候,不同的事务可以使用不同的索引锁定不同的行,另外,不论 是使用主键索引、唯一索引或普通索引,InnoDB都会使用行锁来对数据加锁。
4.即便在条件中使用了索引字段,但是否使用索引来检索数据是由 MySQL 通过判断不同 执行计划的代价来决定的,如果 MySQL 认为全表扫效率更高,比如对一些很小的表,它 就不会使用索引,这种情况下 InnoDB 将使用表锁,而不是行锁。因此,在分析锁冲突时, 别忘了检查SQL 的执行计划（explain查看）,以确认是否真正使用了索引。



## 四.存储过程

> 此处记录生产过程中常见的导入数据的几个存储过程

1.将`excel`导入`mysql`,场景1 我们需要将用户的表导入到我们的系统中，但是有个字段用户存的是中文，我们必须去我们的另一张表动态获取

```sql
CREATE DEFINER=`root`@`%` PROCEDURE `import_house`()
BEGIN
-- 	一二三四级网格名，房屋信息为推测，不准确
	DECLARE
		f_gdn VARCHAR ( 32 );
	DECLARE
		w_gdn VARCHAR ( 32 );
	DECLARE
		t_gdn VARCHAR ( 32 );
	DECLARE
		fo_gdn VARCHAR ( 32 );
	DECLARE
		house_number VARCHAR ( 32 );
	DECLARE
		homeowner_name VARCHAR ( 32 );
	DECLARE
		homeowner_idcard VARCHAR ( 32 );
	DECLARE
		homeowner_phone VARCHAR ( 32 );
	DECLARE
		g_id BIGINT;-- 没有boolean
	DECLARE
		done INT DEFAULT FALSE;
	DECLARE
		cur CURSOR FOR SELECT DISTINCT
		一级网格名称,二级网格名称,三级网格名称,四级网格名称,门牌号,户主姓名,身份证号码,手机号码 
	FROM
		`人口信息表` 
	WHERE
		与户主关系 = '户主';
	DECLARE
		CONTINUE HANDLER FOR NOT FOUND 
		SET done = TRUE;
	OPEN cur;
	read_loop :
	LOOP
			FETCH cur INTO f_gdn,
			w_gdn,
			t_gdn,
			fo_gdn,
			house_number,
			homeowner_name,
			homeowner_idcard,
			homeowner_phone;
		IF
			done THEN
				LEAVE read_loop;
			
		END IF;
		SELECT
			@g_id := grid_id 
		FROM
			sys_grid 
		WHERE
			grid_name = fo_gdn 
			AND parent_id = ( SELECT grid_id FROM sys_grid WHERE grid_name = t_gdn );
		
--   打印
-- 	 SELECT
-- 		 concat( 'grid_id is ', g_id );
			
		INSERT INTO csgm_house ( grid_id, house_number, homeowner_name, homeowner_idcard, homeowner_phone, house_status, create_time, update_time, del_flag )
		VALUES
			( @g_id, house_number, homeowner_name, homeowner_idcard, homeowner_phone, '1', now(), now(), '0' );
		COMMIT;
		
	END LOOP;
	CLOSE cur;

END
```

2.将`excel`导入`mysql`,场景2 我们要从刚才导入的表中查询一个字段，且这次游标中有大量字段，且有很多字段需要动态匹配，包括字符串和日期

```sql
CREATE DEFINER=`root`@`%` PROCEDURE `import_popution`()
BEGIN-- 	一二三四级网格名
	DECLARE
		f_gdn VARCHAR ( 32 );
	DECLARE
		w_gdn VARCHAR ( 32 );
	DECLARE
		t_gdn VARCHAR ( 32 );
	DECLARE
		fo_gdn VARCHAR ( 32 );
	DECLARE
		mph VARCHAR ( 32 );
	DECLARE
		hzxm VARCHAR ( 32 );
	DECLARE
		xjd VARCHAR ( 300 );
	DECLARE
		xm VARCHAR ( 32 );
	DECLARE
		sfzh VARCHAR ( 32 );
	DECLARE
		xb VARCHAR ( 32 );
	DECLARE
		yhzgx VARCHAR ( 32 );
	DECLARE
		mz VARCHAR ( 32 );
	DECLARE
		jg VARCHAR ( 32 );
	DECLARE
		hyzk VARCHAR ( 32 );
	DECLARE
		csrq VARCHAR ( 32 );
	DECLARE
		gzdw VARCHAR ( 32 );
	DECLARE
		whcd VARCHAR ( 32 );
	DECLARE
		zzmm VARCHAR ( 32 );
	DECLARE
		sjhm VARCHAR ( 32 );
	DECLARE
		rklx VARCHAR ( 32 );
	DECLARE
		gllb VARCHAR ( 32 );
	DECLARE
		byzk VARCHAR ( 32 );
	DECLARE
		ylbx VARCHAR ( 32 );
	DECLARE
		jkzk VARCHAR ( 32 );
	DECLARE
		yfdx VARCHAR ( 32 );
	DECLARE
		clxx VARCHAR ( 32 );
	DECLARE
		rks VARCHAR ( 32 );
	DECLARE
		nl VARCHAR ( 32 );
	DECLARE
		zp VARCHAR ( 32 );
	DECLARE
		zxsj VARCHAR ( 32 );
	DECLARE
		zxxx VARCHAR ( 32 );
	DECLARE
		djr VARCHAR ( 32 );
	DECLARE
		djrq VARCHAR ( 32 );
	DECLARE
		bz VARCHAR ( 32 );
	DECLARE
		dwbh VARCHAR ( 32 );
	DECLARE
		fzxm VARCHAR ( 32 );
	DECLARE
		yzd VARCHAR ( 32 );
	DECLARE
		khh VARCHAR ( 32 );
	DECLARE
		yhzh VARCHAR ( 32 );
	DECLARE
		zdaz VARCHAR ( 32 );
	DECLARE
		azwh VARCHAR ( 32 );
	DECLARE
		shjz VARCHAR ( 32 );
	DECLARE
		zxck VARCHAR ( 32 );
	DECLARE
		qck VARCHAR ( 32 );
	DECLARE
		fcb VARCHAR ( 32 );
	DECLARE
		ymjz VARCHAR ( 32 );
	DECLARE
		g_id BIGINT;
	DECLARE
		h_id BIGINT;
	DECLARE
		done INT DEFAULT FALSE;
	DECLARE
		cur CURSOR FOR SELECT DISTINCT
		* 
	FROM
		`人口信息表`;
	DECLARE
		CONTINUE HANDLER FOR NOT FOUND 
		SET done = TRUE;
	OPEN cur;
	read_loop :
	LOOP
			FETCH cur INTO f_gdn,
			w_gdn,
			t_gdn,
			fo_gdn,
			mph,
			hzxm,
			xjd,
			xm,
			sfzh,
			xb,
			yhzgx,
			mz,
			jg,
			hyzk,
			csrq,
			gzdw,
			whcd,
			zzmm,
			sjhm,
			rklx,
			gllb,
			byzk,
			ylbx,
			jkzk,
			yfdx,
			clxx,
			rks,
			nl,
			zp,
			zxsj,
			zxxx,
			djr,
			djrq,
			bz,
			dwbh,
			fzxm,
			yzd,
			khh,
			yhzh,
			zdaz,
			azwh,
			shjz,
			zxck,
			qck,
			fcb,
			ymjz;
		IF
			done THEN
				LEAVE read_loop;
			
		END IF;-- 查询网格ID 房屋由于未精确提供暂时关联不上
		SELECT
			@g_id := grid_id 
		FROM
			sys_grid 
		WHERE
			grid_name = fo_gdn 
			AND parent_id = ( SELECT grid_id FROM sys_grid WHERE grid_name = t_gdn );
		SELECT
			@h_id := house_id 
		FROM
			csgm_house 
		WHERE
			grid_id = @g_id 
			AND house_number = mph 
			AND homeowner_name = hzxm;

		INSERT INTO csgm_population (
			grid_id,
			house_id,
			house_number,
			house_name,
			current_residence,
			NAME,
			idcard,
			gender,
			with_householder_relationship,
			nation,
			native_place,
			marital_status,
			birth_date,
			work_unit,
			culture_degree,
			political_outlook,
			phone,
			population_type,
			account_type,
			management_category,
			military_service_status,
			endowment_insurance,
			medical_insurance,
			health_status,
			religious_figures,
			family_planning_object,
			special_care_object,
			car_information,
			poor_status,
			people_total,
			age,
			photo,
			logout_time,
			logout_options,
			registrant,
			registration_date,
			unit_number,
			homeowner_name,
			movein_time,
			moveout_time,
			reason,
			photo_file,
			place_origin,
			bank_deposit,
			bank_number,
			land_requisition_place,
			place_document_number,
			social_assistance,
			bicycle_parking,
			car_parking,
			fu_cun_bao,
			vaccination,
			create_by,
			create_time,
			update_by,
			update_time,
			remark,
			del_flag,
			fixed_phone,
			audit_status,
			auditor,
			audit_time 
		)
		VALUES
			(
				@g_id,
				@h_id,
				mph,
				hzxm,
				xjd,
				xm,
				sfzh,
			CASE
					
					WHEN xb = '男' THEN
					0 
					WHEN xb = '女' THEN
					1 
				END,
			CASE
					
					WHEN yhzgx = '户主' THEN
					1 
					WHEN yhzgx = '父亲' THEN
					2 
					WHEN yhzgx = '父' THEN
					2 
					WHEN yhzgx = '母亲' THEN
					3 
					WHEN yhzgx = '母' THEN
					3 
					WHEN yhzgx = '妻子' THEN
					4 
					WHEN yhzgx = '妻' THEN
					4 
					WHEN yhzgx = '妻 子' THEN
					4 
					WHEN yhzgx = '媳妇' THEN
					4 
					WHEN yhzgx = '配偶' THEN
					4 
					WHEN yhzgx = '丈夫' THEN
					5 
					WHEN yhzgx = '儿子' THEN
					6 
					WHEN yhzgx = '女儿' THEN
					7 
					WHEN yhzgx = '儿媳' THEN
					8 
					WHEN yhzgx = '孙子' THEN
					9 
					WHEN yhzgx = '孙女' THEN
					10 
					WHEN yhzgx = '重孙' THEN
					11 
					WHEN yhzgx = '孙媳' THEN
					12 ELSE 99 
				END,
				CASE
					
					WHEN mz = '汉' THEN
					1 
					WHEN mz = '汉族' THEN
					1 
					WHEN mz = '维吾尔族' THEN
					4 
				END,
				jg,
						CASE
					
					WHEN hyzk = '已婚' THEN
					2 
				END,
			CASE
					
					WHEN csrq IS NOT NULL THEN
					str_to_date( csrq, '%Y/%m/%d' ) ELSE NULL 
				END,
				gzdw,
			CASE
					
					WHEN whcd = '小学' THEN
					1 
					WHEN whcd = '初中' THEN
					2 
					WHEN whcd = '中专' THEN
					3 
					WHEN whcd = '高中' THEN
					4 
					WHEN whcd = '专科' THEN
					5 
					WHEN whcd = '大学专科' THEN
					5 
					WHEN whcd = '本科' THEN
					6 
					WHEN whcd = '大学本科' THEN
					6 
					WHEN whcd = '硕士' THEN
					7 
					WHEN whcd = '硕士研究生' THEN
					7 
					WHEN whcd = '博士' THEN
					8 
					WHEN whcd = '博士研究生' THEN
					8 
					WHEN whcd = '博士后' THEN
					9 ELSE 10 
				END,
			CASE
					
					WHEN zzmm = '党员' THEN
					3 
				END,
				sjhm,
			CASE
					
					WHEN rklx = '常住人口' THEN
					1 
					WHEN rklx = '流动人口' THEN
					2 
				END,
				"",
				gllb,
			CASE
					
					WHEN byzk = '已服兵役' THEN
					1 
				END,
				"",
			CASE
					
					WHEN ylbx = '职工医保' THEN
					2 
				END,
			CASE
					
					WHEN jkzk = '健康' THEN
					1 
				END,
				"",
				"",
				yfdx,
			CASE
					
					WHEN jkzk = '有车' THEN
					1 
				END,
				"",
				rks,
				nl,
				zp,
			CASE
					
					WHEN zxsj IS NOT NULL THEN
					str_to_date( zxsj, '%Y/%m/%d' ) ELSE NULL 
				END,
				zxxx,
				djr,
			CASE
					
					WHEN djrq IS NOT NULL THEN
					str_to_date( djrq, '%Y/%m/%d' ) ELSE NULL 
				END,
				dwbh,
				fzxm,
				NULL,
				NULL,
				"",
				"",
				yzd,
				khh,
				yhzh,
			CASE
					
					WHEN zdaz = 'False' THEN
					0 
					WHEN zdaz = 'True' THEN
					1 
				END,
				azwh,
			CASE
					
					WHEN shjz = 'False' THEN
					0 
				END,
				zxck,
				qck,
			CASE
					
					WHEN fcb = 'False' THEN
					0
					WHEN zdaz = 'True' THEN
					1
				END,
			CASE
					
					WHEN fcb = '已接种' THEN
					2 
				END,
				"",
				now(),
				"",
				now(),
				bz,
				"0",
				"",
				"2",
				"",
				NOW() 
			);
		COMMIT;
		
	END LOOP;
	CLOSE cur;

END
```

3.将`excel`导入`mysql`,场景3 我们要根据我们查询到的表数据去动态更新我们的表

```sql
CREATE DEFINER=`root`@`%` PROCEDURE `import`()
BEGIN
	DECLARE
		l_id VARCHAR ( 32 );
	DECLARE
		code VARCHAR ( 64 );
	DECLARE
		done INT DEFAULT FALSE;
	DECLARE
		cur CURSOR FOR SELECT DISTINCT
		a.BIS_ID id 
	FROM
		base_logistics_drug a
		LEFT JOIN base_drug_map b ON a.BIS_ID = b.LOGISTICS_DRUG_ID 
		AND b.END_TIME IS NULL 
		AND b.DELETED = "0" 
		AND a.END_TIME IS NULL 
		AND a.DELETED = "0" 
	WHERE
		b.sys_org_code IS NOT NULL;
	DECLARE
		CONTINUE HANDLER FOR NOT FOUND 
		SET done = TRUE;
	OPEN cur;
	read_loop :
	LOOP
			FETCH cur INTO l_id;
		IF
			done THEN
				LEAVE read_loop;		
		END IF;
		
			SELECT
				@code:=concode 
			FROM
				(
				SELECT
					m.id,
					GROUP_CONCAT( m.CODE ) concode 
				FROM
					(
					SELECT DISTINCT
						a.BIS_ID id,
						b.sys_org_code CODE 
					FROM
						base_logistics_drug a
						LEFT JOIN base_drug_map b ON a.BIS_ID = b.LOGISTICS_DRUG_ID 
						AND b.END_TIME IS NULL 
						AND b.DELETED = "0" 
						AND a.END_TIME IS NULL 
						AND a.DELETED = "0" 
					WHERE
						b.sys_org_code IS NOT NULL 
					) m 
				GROUP BY
					m.id 
				) list 
			WHERE
				list.id = l_id;
						
			UPDATE base_logistics_drug set map_dept = @code where  BIS_ID=l_id and END_TIME is null and DELETED ='0';
			
			commit;
	
		END LOOP;
CLOSE cur;
END
```

