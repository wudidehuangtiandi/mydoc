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