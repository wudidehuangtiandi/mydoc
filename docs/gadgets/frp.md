# 免费FRP内网穿透服务

*1、frp是一个高性能的反向代理应用，可以帮助您轻松地进行内网穿透，对外网提供服务， 支持tcp, udp, http, https等协议类型，并且web服务支持根据域名进行路由转发。*
*2、frp内网穿透主要用于没有公网IP的用户，实现远程桌面、远程控制路由器、 搭建的WEB、FTP、SMB服务器被外网访问、远程查看摄像头、调试一些远程的API（比如微信公众号，企业号的开发）等。*



## 以下为TD门户搭建过程种使用到的FRP技术记录



首先根据不同的操作系统下载所需要的版本

[下载地址](https://github.com/fatedier/frp/releases)

本文以`frp_0.37.1_linux_386`为例，对应的压缩包为`frp_0.38.0_linux_386.tar.gz`

该包中同时包含客户端与服务端，基础目录如下图所示



![avatar](https://picture.zhanghong110.top/docsify/16390328581787.png)

### 1.服务端搭建



> 搭建服务端首先需要一台有公网IP的云跳板机，这里推荐腾讯云轻量级服务器，价格便宜高带宽



**下列方式仅在跳板机与本地服务器都为centos时保证有效**

首先服务端应当启动`frps`,对应的配置文件为`frps.ini` 配置文件如下

```
[common]
bind_port = 7000
vhost_http_port = 80
vhost_https_port = 443
```

这里需要解释下，7000 为服务端和客户端交互的端口 ，`vhost_http_port`和`vhost_https_port`为非必要配置对应的服务端如果仅有`tcp`配置项则无需这两个参数，还有其它的更多配置文件可以参考官网。这里暴露的所有端口，都需要跳板机防火墙开放。

启动命令如下,路径仅供参考

```
nohup /u01/frp/frp_0.37.1_linux_386/frps -c /u01/frp/frp_0.37.1_linux_386/frps.ini >output 2>&1 &
```

> 由于对应的跳板机可能会有重启的需求，可以将该命令加到 `/etc/rc.d/rc.local`的最后一行，用以实现开机自启动



### 2.客户端的搭建

客户端应当启动`frpc`,对应的配置文件为`frpc.ini`配置文件如下

```
[common]
server_addr = 1.116.91.244
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000

[nginx]
type = tcp
local_ip = 127.0.0.1
local_port = 80
remote_port = 80

[nacos]
type = tcp
local_ip = 127.0.0.1
local_port = 8848
remote_port = 8848

[nacos-grpc]
type = tcp
local_ip = 127.0.0.1
local_port = 9848
remote_port = 9848


[minio]
type = tcp
local_ip = 127.0.0.1
local_port = 9000
remote_port = 9000

[minio_api]
type = tcp
local_ip = 127.0.0.1
local_port = 9001
remote_port = 9001

[tomcat]
type = tcp
local_ip = 127.0.0.1
local_port = 8080
remote_port = 8080

[mysql]
type = tcp
local_ip = 127.0.0.1
local_port = 3306
remote_port = 3306

[redis]
type = tcp
local_ip = 127.0.0.1
local_port = 6379
remote_port = 6379

[https]
type = https
custom_domains = picture.zhanghong110.top

plugin = https2http
plugin_local_addr = 127.0.0.1:9001

# HTTPS 证书相关的配置
plugin_crt_path = ./picture.zhanghong110.top_bundle.crt
plugin_key_path = ./picture.zhanghong110.top.key
plugin_host_header_rewrite = 127.0.0.1
plugin_header_X-From-Where = frp

```

这里对该文件作下解读，[common]对应服务端的地址`server_port`需要与刚才的服务端`bind_port`相对应，后面就可以开始配置我们的正常的穿透功能，如果手中没有可用域名，这里推荐使用`tcp`实现，此时服务端不应当配置`vhost_http_port`和`vhost_https_port`。

`tcp`配置项关键标签`local_ip`：本地ip,`local_port`：本地端口, `remote_port`：这个项目配置后，客户端一旦启动则服务端也会跟着监听这个端口，（这里值得一提的是，本机此时并不会监听该端口，而是由对应的服务来监听）此时需要开放服务端对应端口，才能够实现顺利的穿透。本配置文件实现了包括文件下载，数据库及服务的各种穿透。



`type`为`http`或者`https`时，要求必须拥有域名，此时监听的端口就不由客户端实现，而是在服务端配置好，且不可重复，通过二级域名的变化来实现多个服务穿透。

本文仅介绍`https`,

`https`有时候目标服务需要`https`,`frp`也能够顺利的实现配置如上图所示。`custom_domains`为你需要使用`https`的域名，采用`https2http`插件转发到`plugin_local_addr`所标注的接口上，SSL证书路径加以配置即可。需要注意的是，`custom_domains`的域名和证书必须匹配。

证书问题检测可使用以下站点

[whynopadlock](https://www.whynopadlock.com/)



由于客户端启动的环境通常在外网，当frp服务更新后通常需要重新启动，建议为系统添加自动执行脚本,具体作用为杀死原进程，启动新的服务。

```
PID=`ps -ef | grep frp | grep -v grep | awk '{print $2}'`
if [ -z "$PID" ]
then
    echo Application is already stopped
else
    echo kill $PID
    kill -9 $PID
    nohup /u01/frp/frp_0.37.1_linux_386/frpc -c /u01/frp/frp_0.37.1_linux_386/frpc.ini >output 2>&1 &
fi
```

该脚本需要使用`chmod u+x  xxx`授权后才可使用。



> 这里要做个提示，类似[common]这种头，每个穿透的配置项不能重复