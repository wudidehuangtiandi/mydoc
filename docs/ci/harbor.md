# Harbor搭建

> Harbor是由VMware公司开源，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、自我注册、镜像复制和中文支持等功能，具有web管理功能，有了它之后能够很方便的管理容器镜像，搭配Jenkins使用很是方便。它基于docker官方提供的仓库镜像服务 `registry` 二次开发， 也是企业级镜像仓库中比较常用的解决方案之一。

首先，它支持在线下载和离线下载，但是都是解压的方式，我们去官网下载个离线包

[地址](https://github.com/goharbor/harbor/releases)

![avatar](https://picture.zhanghong110.top/docsify/16532900317219.png)

解压安装包,这边要注意使用对应的版本

- 在线安装程序：

  ```sh
  tar xzvf harbor-online-installer-version.tgz
  ```

- 离线安装程序：

  ```sh
  tar xzvf harbor-offline-installer-version.tgz
  ```

我们进图目录下，修改配置文件`harbor.yml`

首先要修改hostname为机器地址，使用的是http，所以https部分需要注掉。没有搭建自己的数据库，所以使用了默认的数据库，改下默认密码。部分配置文件如下。

```yaml
# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: 1.116.91.244

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 89

# https related config
https:
  # https port for harbor, default is 443
  # port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
# external_url: https://reg.mydomain.com:8433

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: xxxxx #改下默认密码
```



!>机器需要安装`docker`以及`docker-compose`。而且要对应好版本

使用docker version命令查看docker版本，不清楚docker怎么下载的参见本文档docker部分，此处是`20.10.14`

Harbor中依赖类似redis、mysql、pgsql等很多存储系统，所以它需要编排很多容器协同起来工作，因此VMWare Harbor在部署和使用时，需要借助于Docker的单机编排工具( Docker compose)来实现。您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。[官方地址](https://github.com/docker/compose/releases)

```sh
sudo curl -L https://get.daocloud.io/docker/compose/releases/download/v2.5.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

再授权下,然后查下版本看看是否成功。

```sh
chmod +x /usr/local/bin/docker-compose
```

```sh
[root@centerm docker-compose]# docker-compose --version
Docker Compose version v2.5.1
```



前置工作做好后我们就可以回到刚才harbor的根目录执行`./install.sh`了

经过漫长的等待,成功以后

```
[root@centerm harbor]# docker-compose ps
NAME                COMMAND                  SERVICE             STATUS               PORTS
harbor-core         "/harbor/harbor_core"    core                running (healthy)    
harbor-db           "/docker-entrypoint.…"   postgresql          running (healthy)    5432/tcp
harbor-jobservice   "/harbor/harbor_jobs…"   jobservice          running (starting)   
harbor-log          "/bin/sh -c /usr/loc…"   log                 running (healthy)    127.0.0.1:1514->10514/tcp
harbor-portal       "nginx -g 'daemon of…"   portal              running (healthy)    8080/tcp
nginx               "nginx -g 'daemon of…"   proxy               running (starting)   0.0.0.0:89->8080/tcp, :::89->8080/tcp
redis               "redis-server /etc/r…"   redis               running (healthy)    6379/tcp
registry            "/home/harbor/entryp…"   registry            running (healthy)    5000/tcp
registryctl         "/home/harbor/start.…"   registryctl         running (healthy)
```



安装结束之后，可以通过ip地址访问Harbor镜像仓库，使用默认的账号和密码（admin/密码为配置文件中自己设置的),重置密码比较麻烦，这些镜像默认的映射人间在宿主机`/data`下（可以在初始化时在`harbor.yml`中的`data_volume: /data`修改），会有影响。如果密码不对，进入PG库修改也可能不行，原因就在此。

!>注意，由于docker拉取镜像默认使用https，这边为了避免以后麻烦，请尽量使用Https去安装

这边如果是公司内网，可以自签名证书，可以参考[这个地址](https://blog.csdn.net/networken/article/details/107502461)配置，我这边的话用公网域名。

我们在腾讯云下载 `xxx.zhanghong110.top`的nginx证书

进入服务器data路径下新建cert目录，然后么新建把

crt 和key结尾的两个文件复制进去，然后用以下命令解析,得到cert结尾的文件

```shell
openssl x509 -inform PEM -in xxx.zhanghong110.top_bundle.crt -out xxx.zhanghong110.top.cert
```

将服务器证书，密钥和CA文件复制到Harbor主机上的Docker证书文件夹中。您必须首先创建适当的文件夹

```shell
mkdir -p /etc/docker/certs.d/xxx.zhanghong110.top/
```

然后把刚刚生成的cert结尾的文件和key结尾的文件粘进去，xxx.zhanghong110.top_bundle.crt这个也要粘进去，并且重命名把_bundle去掉

然后修改配置文件`harbor.yml`,这边crt文件记得改下名要一致。

```shell
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /data/cert/xxx.zhanghong110.top.crt
  private_key: /data/cert/xxx.zhanghong110.top.key
```

配置文件里还有这个hostname一定要换成域名否则docker login还是会报http异常

```shell
hostname: xxx.zhanghong110.top
```

运行prepare脚本以启用HTTPS。
Harbor将nginx实例用作所有服务的反向代理。您可以使用prepare脚本来配置nginx为使用HTTPS。该prepare在harbor的安装包，在同级别的install.sh脚本。

```shell
./prepare
```

如果Harbor正在运行，请停止并删除现有实例。
您的镜像数据保留在文件系统中，因此不会丢失任何数据。

```shell
docker-compose down -v
```

重启harbor：

```shell
docker-compose up -d
```



如果想要自签名的方式配置(仅适用于公司内部)可以参考[这个链接](https://blog.csdn.net/networken/article/details/107502461)

> 我们再次访问，即可https访问了。