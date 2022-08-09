# DOCKER的运用及一些项目的部署



> 本文仅记录目前为止的docker的一些运用及集成方式，不涉及dockerhub及dockerfile。主要为服务运用docker部署及管理。以后知识面有所扩充了可能会进行一些更新及改进。



*docker部署的好处，更方便的实现服务的管理，拥有隔离性及安全性。一键重启服务及和宿主机环境隔离是现阶段最大的优点，不过由于未使用dockerhub而丧失了其可移植性及版本管理的优点。*



## 一.docker的安装

> 仅针对centos7，也可以去官网看教程，要选择对应的centos的脚本

> 首先大部分机器需要源切换，前提yum安装了yum -y install yum-utils，然后yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

1.首先卸载旧版本的docker,使用以下命令

```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

> 国内yum源建议用阿里的镜像

2.安装*最新版本*的 `Docker Engine` 和` containerd`

```shell
sudo yum install docker-ce docker-ce-cli containerd.io
```

3.启动docker

```shell
sudo systemctl start docker
```

4.验证是否成功

```shell
sudo docker run hello-world
```



5.采用阿里云的镜像加速器

[地址](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)

配置对应的镜像加速器，不同的服务器命令不同具体可以参照官网给出的脚本配置。



*以下为非必须流程*

6.迁移对应的容器目录，有时候默认的路径容量可能较小，此时应当迁移到指定的目录，以centos7为例

```shell
停止docker服务
sudo systemctl stop docker 
复制到指定路径
cp -R /var/lib/docker/* /home/docker/lib/docker
修改配置文件(没有则创建)
etc/systemd/system/docker.service.d/docker-options.conf
增加以下内容
ExecStart=/usr/bin/dockerd  --graph=/home/docker/lib/docker
启动docker服务
sudo systemctl start docker
```

7.当需要配置开机启动时,采用以下命令即可

```shell
systemctl enable docker
```

## 二.docker的一些概念和命令



> 要使用docker，一些概念和命令是需要掌握的

1.镜像和容器

docker的整个生命周期有三部分组成：镜像（`image`）+容器（`container`）+仓库（`repository`）

docker镜像实际上是由一层一层的系统文件组成，这种层级的文件系统被称为`UnionFS( Union file system  统一文件系统)`，镜像可以基于dockerfile构建，dockerfile是一个描述文件，里面包含了若干条密令，每条命令都会对基础文件系统创建新的层次结构。

docker提供了一个很简单的机制来创建镜像或更新现有的镜像。用户甚至可以从其他人那里下载一个已经做好的镜像直接使用。（镜像是只读的，可以理解为静态文件）

​      docker利用容器来运行应用：docker容器是由docker镜像创建的运行实例。docker容器类似虚拟机，可以执行包含启动，停止，删除等。每个容器间是相互隔离的。容器中会运行特定的运用，包含特定应用的代码及所需的依赖文件。可以把容器看作一个简易版的linux环境（包含root用户权限，进程空间，用户空间和网络空间等）和运行在其中的应用程序。

仓库的话本文由于不涉及暂不介绍。

2.数据卷的概念

`Docker镜像`是有多层`只读文件`叠加而成，当运行起一个容器的时候，Docker会在制只读层上创建一个`读写层`。如果运行中的容器需要修改文件，那么并不会修改只读层的文件，只会把该文件复制到`读写层`然后进行修改，只读层的文件就被隐藏了。当删除了该容器之后，或者重启容器之后，之前对文件的更改会丢失，镜像的只读层以及容器运行是的“读写层”被称为`联合文件系统（Union File System）`
 为了实现容器与主机之间、容器与容器之间`共享文件`，容器中`数据的持久化`，将容器中的数据`备份`、`迁移`、`恢复`等,Docker加入了数据卷(volumes)机制。简单的讲，就是做了一个文件夹的实时共享，有点像局域网的文件共享。

数据卷的特点

(1)数据卷存在于宿主机的文件系统中，独立于容器，和容器的生命周期是分离的。

(2)数据卷可以是目录也可以是文件，容器可以利用数据卷与宿主机进行数据共享，实现了荣期间的数据共享和交换。

(3)容器启动初始化时，如果容器使用的镜像包含了数据，这些数据会拷贝到数据卷中。

(4)容器对数据卷的修改是实时进行的。

(5)数据卷的变化不会影响镜像的更新。数据卷是独立于联合文件系统，镜像是基于联合文件系统。镜像与数据卷之间不会相互影响

`bind mounts`、`Volumes`、和`tepfs mounts`三种方式，还有就是共享其他容器的数据卷，其中`tmpfs`是一种基于内存的临时文件系统。`tepfs mounts`数据不会存储在磁盘上，在次笔记中暂不做笔记。

3.容器的启动

```
docker run ...  -v 宿主机目录（文件）:容器内目录（文件）   注意事项：目录必须为绝对路径，如果不存在会自动创建，可以挂载多个数据卷
```

4.进入docker容器

```
进入 docker容器 docker exec -it 容器名 /bin/bash   , 退出exit
```

5.一些基本命令

```shell
#搜索镜像
docker search xxx

#拉取镜像
docker pull xxx

#查看镜像
docker images 或者 docker image ls

#打标签 tag后面分别是镜像ID 推送的网站 账号 版本号
docker tag [imageid] docker.io/[dockerhub账号]/xxx:v1.0.0

#推送
docker push docker.io/[dockerhub账号]/xxx:v1.0.0

# 查看日志 
docker logs 容器名(不是ID) --tail 1000

# 查看运行中的容器进程
docker ps

# 查看所有容器
docker ps -a

# 启动已配置过的容器
docker start container1

# 启动多个容器
docker start container1 container2

#一次性重启所有容器(效果不好，有的容器有启动顺序要求)
docker start $(docker ps -a | awk '{ print $1}' | tail -n +2)


# 重启
docker restart container1

# 停止
docker stop container1

# 停止多个
docker stop container1 container2 container3

# 删除
docker rm container1

# 查看日志
docker logs container1
```

## 三.一些常见的服务部署

1.nginx

```shell
#首先下载nginx镜像
docker pull nginx

#创建挂载的目录，我是放在/data/nginx里面，可自行更改
mkdir -p /data/nginx/conf #存放配置文件
mkdir -p /data/nginx/logs
mkdir -p /data/nginx/html
mkdir -p /data/nginx/conf.d

#因为不能挂载文件，只能挂载一个文件夹，所以我们要先创建一个测试test容器的nginx，然后复制配置文件到挂载的目录上
##启动测试容器
docker run --name test -d nginx

##复制配置文件
docker cp test:/etc/nginx/nginx.conf /data/nginx/conf/
docker cp test:/etc/nginx/conf.d/default.conf  /data/nginx/conf.d

##如果不知道配置文件在docker里面的目录位置,可以进去看一下
docker exec -it test /bin/bash

#然后运行你自己的nginx
docker run --name nginx --privileged -it -p 80:80 -v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro  \
-v /data/nginx/conf.d:/etc/nginx/conf.d:ro  \
-v /data/nginx/html:/usr/share/nginx/html:rw \
-v /etc/localtime:/etc/localtime \
-v /data/nginx/logs:/var/log/nginx -d nginx

#最后把我们的放到html文件夹解压,重启nginx即可

##在html文件夹解压我们上传的dist文件
unzip dist.zip

##重启nginx
docker restart b0ba

#注意，这时候配置文件虽然映射出来了但是路径依然是需要配置容器路径才能正常映射。

#下面是TD的真实配置脚本
docker run --name nginx --privileged -it -v /home/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro  \
-v /home/nginx/conf.d:/etc/nginx/conf.d:ro  \
-v /home/nginx/html:/usr/share/nginx/html:rw \
-v /etc/localtime:/etc/localtime \
--network host \
-v /home/nginx/logs:/var/log/nginx -d nginx

```



> ++++++++++++++++++++ HOST模式  --network host \  以下服务由于不想使得访问的时候再通过公网跳转，可以使用HOST模式共享宿主机的IP和端口这样就可以127.0.0.1了，这种情况下-P 或者-p这种桥接模式将会失效  ++++++++++++++++++++++++++



2.minio

```shell
docker pull minio/minio

#启动脚本：

#在home目录下新建data及config文件,新版的需要暴露两个端口一个是控制台一个是API

docker run -p 9000:9000 -p 9001:9001 --name minio \
  -v /etc/localtime:/etc/localtime \
  -v /home/minio/data:/data \
  -v /home/minio/config:/root/.minio \
  -d \
  -e "MINIO_ROOT_USER=xxxxx" \
  -e "MINIO_ROOT_PASSWORD=xxxxxx" \
   minio/minio server /data \
  --console-address ":9000" --address ":9001"
```



3.mysql

```shell
mysql 注意配置文件默认为空，有需要的话去官网搞一个 

docker run -p 3306:3306 --name mysql \
-v /home/mysql/data:/var/lib/mysql \
-v /home/mysql/conf:/etc/mysql \
-v /home/mysql/mysql-files:/var/lib/mysql-files/ \
-e TZ=Asia/Shanghai \
-e MYSQL_ROOT_PASSWORD=xxxxxx \
-d mysql \
--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --default-time_zone='+8:00' --lower_case_table_names=1
```



4.redis

```shell
redis 注意配置文件默认为空，需要去官网搞一个 。注意，用配置文件需要修改三处保护模式(protected-mode)修改为no。
把密码这行的注释解掉，修改为自己的密码 requirepass xxx 。 把bind 127.0.0.1注释掉。

docker run -p 6379:6379 --name redis -v /home/redis/data:/data \
-v /home/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis redis-server /etc/redis/redis.conf/redis.conf
```



5.nacos

```shell
docker pull nacos/nacos-server

需要新建配置文件,注意2.0以上版本需要暴露8848+1000的端口号用以客户端请求，集群还需要8848+1001用以服务间同步,
需要有配置文件,可以去解压了抄一个

docker  run \
--name nacos -d \
-p 8848:8848 \
-p 9848:9848 \
--privileged=true \
--restart=always \
-e JVM_XMS=256m \
-e JVM_XMX=256m \
-e MODE=standalone \
-e PREFER_HOST_MODE=hostname \
-e TIME_ZONE='Asia/Shanghai' \
-v /home/nacos/logs:/home/nacos/logs \
-v /home/nacos/conf/application.properties:/home/nacos/conf/application.properties \
nacos/nacos-server
```



6.下面展示几个服务的配置方式，具体为公司业财融合系统的脚本，可以作为参考

```shell
docker run -d \
       --name ry-geteway \
       -v /u01/ycrh/ry-gateway:/u01/ycrh/ry-gateway \
       -w /u01/ycrh/ry-gateway \
       -e SERVICE_NAME=ry-geteway \
       -e TZ="Asia/Shanghai" \
       --log-driver=syslog --log-opt syslog-address=tcp://172.23.112.18:4904  \
       --expose 8085 \
       -p 8085:8085 \
       openjdk:8 java -Xms256m -Xmx256m -XX:CompressedClassSpaceSize=128m -XX:MetaspaceSize=200m -XX:MaxMetaspaceSize=200m -Dfile.encoding=utf-8 -jar /u01/ycrh/ry-gateway/ruoyi-gateway.jar

这里稍微讲解下
--log-driver=syslog --log-opt syslog-address=tcp://172.23.112.18:4907  \d
这段的作用， ELK在DOCKER里需要特别配置 ，1.代码里需要暴露2.需要DOCKER暴露。3.需要LOGSTASH配置更换成一下这种

input {

  tcp {
    port => "4900"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4901"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4902"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4903"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4904"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4905"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4906"
    type => "syslog"
    codec => json_lines
  }
   tcp {
    port => "4907"
    type => "syslog"
    codec => json_lines
  }
  tcp {
    port => "4908"
    type => "syslog"
    codec => json_lines
  }

 beats {
   host => "127.0.0.1"
   port => "5044"
   add_field => {"appname" => "self-help"} 
 }
}

filter {
  #Only matched data are send to output. 
}

output {

  # For detail config for elasticsearch as output,
  # See: https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html

  # stdout {
  #  codec => rubydebug
  # }

   elasticsearch {
      action => "index"          #The operation on ES
      hosts  => "http://127.0.0.1:9200"   #ElasticSearch host, can be array.
      index => "log_%{[appname]}-%{+YYYY.MM.dd}"
   }
  
}
具体参考分布式里面的ELK的实现

//后面的服务
docker run -d \
       --name bf-system \
       -v /u01/ycrh/bf-system:/u01/ycrh/bf-system \
       -w /u01/ycrh/bf-system \
       -e SERVICE_NAME=bf-system \
       -e TZ="Asia/Shanghai" \
       --expose 9929 \
       --log-driver=syslog --log-opt syslog-address=tcp://172.23.112.18:4907  \
       -P \
       openjdk:8 java -Xms256m -Xmx256m -XX:CompressedClassSpaceSize=128m -XX:MetaspaceSize=200m -XX:MaxMetaspaceSize=200m -Dfile.encoding=utf-8 -jar /u01/ycrh/bf-system/aoyang-bf-system.jar

```

!> 以上脚本存在的问题在于，由于采用桥接模式问题，导致服务间的交互也需要从外网进行

下面演示下TD里部分脚本

```shell
采用HOST模式，不采用桥接模式，否则127.0.0.1不生效。
docker run -d \
       --name ry-gateway \
       -v /home/td/ry-gateway:/home/td/ry-gateway \
       -w /home/td/ry-gateway \
       -e SERVICE_NAME=ruoyi-gateway \
       -e TZ="Asia/Shanghai" \
       --network host \
       openjdk:8 java -Xms256m -Xmx256m -XX:CompressedClassSpaceSize=128m -XX:MetaspaceSize=200m -XX:MaxMetaspaceSize=200m -Dfile.encoding=utf-8 -jar /home/td/ry-gateway/ruoyi-gateway.jar


==服务==

docker run -d \
       --name ry-modules-system \
       -v /home/td/ry-modules-system:/home/td/ry-modules-system \
       -w /home/td/ry-modules-system \
       -e SERVICE_NAME=ry-modules-system \
       -e TZ="Asia/Shanghai" \
       --network host \
       openjdk:8 java -Xms256m -Xmx256m -XX:CompressedClassSpaceSize=128m -XX:MetaspaceSize=200m -XX:MaxMetaspaceSize=200m -Dfile.encoding=utf-8 -jar /home/td/ry-modules-system/ruoyi-modules-system.jar
```

> 由于采用了HOST模式，端口需要和宿主机不能冲突

7.rabbitmq

```shell
docker pull rabbitmq
docker pull rabbitmq:management 
需要注意的是，docker pull rabbitmq (镜像未配有控制台)，docker pull rabbitmq:management (镜像配有控制台)。

docker run -d --name rabbitmq \
--network host \
-v /home/rabbitmq/data:/var/lib/rabbitmq/mnesia \
-v /home/rabbitmq/conf:/etc/rabbitmq \
-v /home/rabbitmq/log:/var/log/rabbitmq \
-e RABBITMQ_DEFAULT_USER=wudidehuangtiandi \
-e RABBITMQ_DEFAULT_PASS=gc123456 \
rabbitmq:management 

打开控制台
docker exec -it rabbitmq rabbitmq-plugins enable rabbitmq_management
```



## 四.关于dockerfile及dockerhub

本文不作深入探讨，这块知识逻辑等待学习，记录下曾经接触过的部分内容

1.如何打包自己的镜像，以下为我的第一个dockerfile脚本

```dockerfile
FROM adoptopenjdk/openjdk11:alpine AS builder
MAINTAINER GC
ADD msa-registy-0.0.1-SNAPSHOT.jar  /usr/var/docker_test/eureka.jar
ENTRYPOINT ["nohup","java","-jar","/usr/var/docker_test/eureka.jar","&"]
```



> 2022/04/25更新

由于需要结合K8S部署项目，所以这边学习下如何这块内容

**dockerfile**



> 简单的dockerfile编写

```dockerfile
#简单的Dockerfile编写

#用于指定镜像
FROM alpine:latest
#在这个镜像下创建某目录（工作目录）
WORKDIR /app
#将宿主机文件拷贝到镜像,下面这句的命令是将src文件夹拷贝到/app目录下
COPY src/ /app
#在构建的时候运行的脚本，目录为工作目录下
RUN echo 321 >> 1.txt
#容器运行时才会执行的脚本,执行完了容器生命周期结束，所以这边设置为阻塞式。
CMD tail -f 1.txt
```



> 构建镜像

```shell
#表示在当前目录下构建镜像，名字为test
docker build -t test .

#docker build的一些其他参数
-f，--file
#指定 dockerfile 路径

docker build -f /path/to/a/Dockerfile .
#不指定的话，默认会读取上下文路径（  .  )下的 dockerfile

-t，--tag
#指定构建的镜像名和 tag
#docker build -t ubuntu-nginx:v1 . 
```



> 运行

```shell
#运行
docker run test
```



> 几个相似指令和一些别的指令

```dockerfile
#这个指令类似于COPY 区别是ADD 能够添加网络地址，推荐使用COPY。
ADD 

#和CMD类似，也是可以指定容器运行后的核心脚本
#和CMD都可以用JSON数组指定比如["cat","1.txt"]
#可能会存在两者同时使用的情况，会按照一下准则。ENTRYPOINT非JOSN则以ENTRYPOINT为准，CMD无效，
#如果ENTRYPOINT和CMD都是JSON则ENTRYPOINT+CMD拼接成shell。其它的情况都不太会遇到。
ENTRYPOINT

#指定暴露出来的端口
EXPOSE

#指定映射文件，以下命令会把容器的/a/b这个目录映射到宿主机的匿名卷
#EXPOSE和这个命令类似docker run -p -v 这两命令
VOLUME /a/b

#指定环境变量，会一直存在
#比如
#ENV A=10 
#CMD echo $A 
#就能打印10
ENV

#也是环境变量，只在构建时生效，运行时失效，可以类似这样指定 
#ARG B=11
#ENV A $B
#CMD echo $A
#就能打印出11，直接打印B则无效，应为CMD是运行时命令而不是构建时命令
#可以在构建时通过 docker build -t test --build-arg B=12 . 来将变量传入
ARG

#一般写在第二行，用来指定一些元数据信息 比如 k="v" 
#可以通过docker inspect来找到镜像信息
LABEL

#指定的指令在当前的镜像是不会运行的，如果另一个镜像是基于这个镜像的话则会运行
ONBUILD

#用于指定RUN CMD等指令运行时的用户身份，不指定是root
#USER  用户名:用户组   或
#USER  用户id:组id
USER

#指定信号名使得容器停止，很少使用
STOPSIGNAL

#检查容器健康状态，很少使用
HEALTHCHECK

#就是这个SHELL脚本是基于哪个SHELL进行的，LINUX就是默认下面这个
SHELL /bin/sh
```

