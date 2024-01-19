# acme 的使用

> 这个开源的软件可以自动生成SSL证书及自动续期

[官方地址](https://github.com/acmesh-official/acme.sh)

用法：

1.安装，邮箱输自己的，不然后面会报错

```
curl https://get.acme.sh | sh -s email=my@example.com
```

2.有两种生成证书的方式http和dns

我们只介绍dns的方式

dns 方式的真正强大之处在于可以使用域解析商提供的 api 自动添加 txt 记录完成验证。

> **acme.sh**目前支持 cloudflare, dnspod, cloudxns, godaddy 以及 ovh 等十种解析商的自主集成。

我们需要去对应的平台去生成DNS API密钥，然后去配置acme.sh的配置文件，具体配置方式地址

> https://github.com/acmesh-official/acme.sh/wiki/dnsapi

`acme.sh`默认的安装地址是 root下的.acme.sh/，是个隐藏文件，我们进入后里面有个`account.conf`

比如阿里云，在这里面加上，把上面和平台申请的API 加在头上即可

```
export Ali_Key="xxx"
export Ali_Secret="xxx"
```

完事之后我们去生成证书，这个命令可dns_ali这边会有变化，这个是阿里云的，具体可以去上面那个链接看下

```
./acme.sh --issue --dns dns_ali -d xxxx.cn
```
也可以直接-d *.xxxx.cn 生成一个泛用的，注意这边有可能会超时应为GFW的缘故有的可能会失败 可以试下 `--server letsencryp`
这个搞完在`.acme.sh`下就会生成证书了，但是我们需要去复制到别的文件中去，用`--install-cert`命令去复制到指需要的地址即可

```
acme.sh --install-cert -d bunnyfit.cn \
--key-file       /home/nginx/html/key.pem  \
--fullchain-file /home/nginx/html/cert.pem \
--reloadcmd     "docker restart nginx" \
```

