# 关于利用shadowsock科学上网及如何使得LINUX科学上网

!> 本章节仅供学习使用，私搭VPN违法,后果自负。

## 1.VPS服务器的购买和使用,SS服务端的搭建,可以参考教程
> https://github.com/zhaoweih/Shadowsocks-Tutorial

## 2.图形化界面的使用
> 图形化界面的使用也可以参考如上教程，写的还是非常详细的，安卓下一个APK，WINDOWS下个EXE 完事直接用就行了

## 3.linux在无图形化界面的时候应当如何安装并且使用客户端
> 这里以centos为例,使用shadowsocks-libev客户端插件

```shell
yum install shadowsocks-libev
```
>这里遇到YUM源有问题的自行搜索解决，下面在指定目录下放一个配置文件结构如下

```json
{
    "server": "shadowsock服务端的IP",
    "server_port": shadowsock服务端 端口,
    "local_port": shadowsock客户端提供的端口,
    "local_address": "127.0.0.1",
    "password": "shadowsock服务端的密码",
    "method": "加密方式",
    "mode": "tcp_and_udp",
    "timeout": 300,
    "fast_open":false,
    "workers":3
}
```

> 启动，/home/shadowsocks.json为自己设置的配置文件路径

```shell
sslocal -c /home/shadowsocks.json --pid-file /var/run/shadowsocks-libev.pid -d start
```
>下一步配置代理

```
# 安装 privoxy
yum install privoxy

# 修改 privoxy 配置文件 /etc/privoxy/config 8118是代理的监听端口，1080是shadowsock客户端提供的端口
listen-address 127.0.0.1:8118
forward-socks5t / 127.0.0.1:1080 .

# 启动 privoxy 服务
systemctl enable privoxy
systemctl start privoxy
systemctl status privoxy

# 设置代理环境变量
echo -e "export http_proxy=http://127.0.0.1:8118" >> /etc/profile
echo -e "export https_proxy=http://127.0.0.1:8118" >> /etc/profile
source /etc/profile

# 测试
curl www.google.com

# 去除代理，把 /etc/profile 里的配置注释即可。

```
> 至此，应当linux也可以科学上网了