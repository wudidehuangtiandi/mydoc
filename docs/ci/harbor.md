# Harbor 搭建

> Harbor 是由 VMware 公司开源，它包括权限管理(RBAC)、LDAP、日志审核、管理界面、自我注册、镜像复制和中文支持等功能，具有 web 管理功能，有了它之后能够很方便的管理容器镜像，搭配 Jenkins 使用很是方便。它基于 docker 官方提供的仓库镜像服务 `registry` 二次开发， 也是企业级镜像仓库中比较常用的解决方案之一。

首先，它支持在线下载和离线下载，但是都是解压的方式，我们去官网下载个离线包(我这边不知道为啥阴差阳错下的版本老了，新的下 2.5 之后的版本，不过流程都一样)

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

首先要修改 hostname 为机器地址，使用的是 http，所以 https 部分需要注掉。没有搭建自己的数据库，所以使用了默认的数据库，改下默认密码。部分配置文件如下。

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

使用 docker version 命令查看 docker 版本，不清楚 docker 怎么下载的参见本文档 docker 部分，此处是`20.10.14`

Harbor 中依赖类似 redis、mysql、pgsql 等很多存储系统，所以它需要编排很多容器协同起来工作，因此 VMWare Harbor 在部署和使用时，需要借助于 Docker 的单机编排工具( Docker compose)来实现。您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。[官方地址](https://github.com/docker/compose/releases)

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

```sh
#不行的话可以试下加上
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
#再执行
docker-compose --version
```

前置工作做好后我们就可以回到刚才 harbor 的根目录执行`./install.sh`了

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

安装结束之后，可以通过 ip 地址访问 Harbor 镜像仓库，使用默认的账号和密码（admin/密码为配置文件中自己设置的),重置密码比较麻烦，这些镜像默认的映射人间在宿主机`/data`下（可以在初始化时在`harbor.yml`中的`data_volume: /data`修改），会有影响。如果密码不对，进入 PG 库修改也可能不行，原因就在此。

!>注意，由于 docker 拉取镜像默认使用 https，这边为了避免以后麻烦，请尽量使用 Https 去安装

我们在腾讯云下载 `xxx.zhanghong110.top`的 nginx 证书

进入服务器 data 路径下新建 cert 目录，然后么新建把

crt 和 key 结尾的两个文件复制进去，然后用以下命令解析,得到 cert 结尾的文件

```shell
openssl x509 -inform PEM -in xxx.zhanghong110.top_bundle.crt -out xxx.zhanghong110.top.cert
```

将服务器证书，密钥和 CA 文件复制到 Harbor 主机上的 Docker 证书文件夹中。您必须首先创建适当的文件夹

```shell
mkdir -p /etc/docker/certs.d/xxx.zhanghong110.top/
```

然后把刚刚生成的 cert 结尾的文件和 key 结尾的文件粘进去，xxx.zhanghong110.top_bundle.crt 这个也要粘进去，并且重命名把\_bundle 去掉

然后修改配置文件`harbor.yml`,这边 crt 文件记得改下名要一致。

```shell
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /data/cert/xxx.zhanghong110.top.crt
  private_key: /data/cert/xxx.zhanghong110.top.key
```

配置文件里还有这个 hostname 一定要换成域名否则 docker login 还是会报 http 异常

```shell
hostname: xxx.zhanghong110.top
```

运行 prepare 脚本以启用 HTTPS。
Harbor 将 nginx 实例用作所有服务的反向代理。您可以使用 prepare 脚本来配置 nginx 为使用 HTTPS。该 prepare 在 harbor 的安装包，在同级别的 install.sh 脚本。

```shell
./prepare
```

如果 Harbor 正在运行，请停止并删除现有实例。
您的镜像数据保留在文件系统中，因此不会丢失任何数据。

```shell
docker-compose down -v
```

重启 harbor：

```shell
docker-compose up -d
```

> 我们再次访问，即可 https 访问了。

有的时候可能只是内网访问，可以生成自签名的证书[官网地址](https://goharbor.io/docs/2.0.0/install-config/configure-https/)

!> 注意自签名的证书存在局限性，完成后如果需要从其他服务器登录 harbor 需要将证书复制到其他服务器上才可以。下面域名我们用`harbor.project.com`代替

```shell
#1.创建CA的私钥
openssl genrsa -out ca.key 4096

#2.创建CA证书
#X.509 证书由多个字段组成。该 Subject领域是与本教程最相关的领域之一。它给出了证书所属客户端的 DName。DName 是赋予 X.500 目录对象的唯一名称。它由许多称为相对可分辨名称 (RDN) 的属性值对组成。一些最常见的RDN及其解释如下：
#CN: 通用名
#OU： 组织单位
#O： 组织
#L: 地方
#S: 州或省名
#C： 国家的名字
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=UK/ST=Wales/L=Cardiff/O=Cardiff University/OU=Headquarter/CN=project.com" \
 -key ca.key \
 -out ca.crt

#3.生成CA的私钥和证书后，需要生成Harbor的私钥和证书：
openssl req -sha512 -new \
    -subj "/C=UK/ST=Wales/L=Cardiff/O=Cardiff University/OU=Headquarter/CN=harbor.project.com" \
    -key harbor.project.com.key \
    -out harbor.project.com.csr

#4.配置x509 v3拓展文件，配置该文件的目的是帮助生成符合主题备用名称 (SAN) 和 x509 v3 的证书扩展要求的证书文件。其中，SAN 或主题备用名称是一种结构化方式，用于指示受证书保护的所有域名和 IP 地址。被视为 SAN 的项目的简短列表中包括子域和 IP 地址。该文件的格式如下：
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.project.com
DNS.2=harbor.project
DNS.3=harbor
EOF

#6.配置完成后，使用该文件和openssl为Harbor生成证书：
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.project.com.csr \
    -out harbor.project.com.crt

#7.接下来，我们需要给Harbor容器添加证书。由于在部署容器时使用了卷映射，所以我们直接将Harbor的私钥和证书拷贝到宿主机的/data/cert目录下即可：
cp harbor.project.com.crt /data/cert
cp harbor.project.com.key /data/cert
#如果目录不存在先要 mkdir /data/cert

#8.完成后，转换harbor.project.com.crt为harbor.project.cert, 供 Docker 使用。Docker 守护进程将.crt文件解释为 CA 证书，将.cert文件解释为Harbor证书。
openssl x509 -inform PEM -in harbor.project.com.crt -out harbor.project.com.cert

#9.创建存放验证Harbor容器证书的Docker的目录，并将Harbor的私钥，证书，以及CA的证书拷贝进去
mkdir -p /etc/docker/certs.d/harbor.project.com/
cp harbor.project.com.cert /etc/docker/certs.d/harbor.project.com/
cp harbor.project.com.key /etc/docker/certs.d/harbor.project.com/
cp ca.crt /etc/docker/certs.d/harbor.project.com/

#10.完成后，在解压后的Harbor目录下，找到如下的harbor.yml文件，并修改其中如下三处,如有需要密码也需要修改
hostname: harbor.project.com

certificate: /data/cert/harbor.project.com.crt
private_key: /data/cert/harbor.project.com.key

#11 去Harbor目录下 sudo ./install.sh即可

#12 docker login harbor.project.com -u admin -p Harbor12345

#13 此时，如果内网有DNS解析服务器则可以统一配置DNS解析，如果没有则登录的机器包括自生都需要修改host映射才可以

#14 如果用其他服务器的docker调用会发生x509:certificate signed by unknown authority，官方解决方案https://docs.docker.com/engine/security/certificates/
#从官方文档提示，客户端要使用tls与Harbor通信，使用的还是自签证书，那么必须建立一个目录：/etc/docker/certs.d
#在这个目录下建立签名的域名的目录，比如域名为harbor.project.com， 那么整个目录为: /etc/docker/certs.d/harbor.project.com, 然后把harbor的证书拷贝到这个目录即可。所需要拷贝的文件包括如下几个
#harbor.project.com.key
#harbor.project.com.csr
#harbor.project.com.crt
#harbor.project.com.cert
#拷贝完成后即可使用docker login harbor.project.com -u admin -p Harbor12345登录
```
