# aMYSQL

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

## 二.条件位置和执行顺序

> 我们可能经常这种left join，inner join这种连接，却发现居然只是模糊的认识。

那么，当条件跟在`where`后面和条件在`and`后面时有什么区别呢？

我们使用`m` 和 `s`两张表做个演示，内容如下

`m`:

![avatar](https://picture.zhanghong110.top/docsify/16621670659303.png)

`n`:

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

!>在实际操作过程中，有一个这样的场景，一条有十几万条记录的主表，`left join`查询一条五百万数据的表，`on`项都加了`b+`索引当主表加上`where`条件匹配少数列时查询成功，否则就慢查询，查询失败。



由此我们思考下，当如果主从表先笛卡尔积，再筛选，就是十几万条和几百万条的积，再做筛选，这样就是不加条件，筛选失败，显然`mysql`内部不是这样筛选的。





