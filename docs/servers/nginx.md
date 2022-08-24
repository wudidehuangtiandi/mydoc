# NGINX的配置及优化

>环境为centos7.6及nginx1.21.6

## 1.配置文件分类引入

> 通常情况下，一个nginx可能会配置一个或者多个业务，这块内容nginx自带，我们讲解下

我们这边默认采用docker配置，此时需要映射两个路径的配置数据卷，可以看到默认地址位于`/etc/nginx/nginx.conf`及`/etc/nginx/conf.d`

```
-v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro  \
-v /data/nginx/conf.d:/etc/nginx/conf.d:ro  \
```

我们打开配置文件

`nginx.d`，里面默认有个`default.conf`

```
server {
    listen       8080;
    listen  [::]:8080;
    server_name  localhost;


    location / {
        root   /usr/share/nginx/html/dist;
        index  index.html index.htm;
    }

 
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

  
}
```

`nginx.conf`

```

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    client_max_body_size 60m;

    include /etc/nginx/conf.d/*.conf;
}
```

稍微观察下我们不难发现，`nginx.conf`中通过` include /etc/nginx/conf.d/*.conf;`引入了各个配置文件，一般来说，非通用的业务只需要我们在`conf.d`中定义匹配的配置文件即可。

## 2.nginx反向代理

当我们一台`Nginx`需要向多台应用服务器做出分发的时候可以使用如下配置，新建一个xxx.conf

```

	upstream backend{
		 server 127.0.0.1:8080       weight=1;
		 server 127.0.0.1:8081       weight=1;
		 server 127.0.0.1:8082       weight=1;
		}
		

   server {
         listen       80;
         server_name  hpv xxx.xxx.xxx.xxx;
	  
         location / {
            root   /xxx/xxx/xxx/xxx/xxx/xxx;
                	try_files $uri $uri/ /index.html;
            index  index.html index.htm;
         }	
	
		 location /xxx/xxxx-xxx/ {
		 proxy_set_header Host $http_host;
		 proxy_set_header X-Real-IP $remote_addr;
		 proxy_set_header REMOTE-HOST $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        proxy_pass http://backend/;
		 }
		 
         error_page   500 502 503 504  /50x.html;
         location = /50x.html {
         root   /usr/share/nginx/html;
    }
}

```

如上图所示，我们可以利用upstream做到一对多的分发。

## 3.nginx简单性能优化

> 对nginx做出简单的性能优化可以使得其并发能力大大提高

由于时通用的配置，我们对`nginx.conf`做出修改

```shell
user  nginx; #定义nginx的启动用户，不建议使用root
worker_processes  8; #定位为cpu的内核数量,根据CPU线程数配置。不过这值最多也就是8，8个以上也就没什么意义了，想继续提升性能只能参考下面一项配置 
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;#此项配置为开启多核CPU，对你先弄提升性能有很大帮助nginx默认是不开启的,1为开启，0为关闭，因此先开启第一个倒过来写

#第一位0001（关闭第四个、关闭第三个、关闭第二个、开启第一个）
#第二位0010（关闭第四个、关闭第三个、开启第二个、关闭第一个）
#第三位0100（关闭第四个、开启第三个、关闭第二个、关闭第一个）
#后面的依次类推。那么如果是16核或者8核cpu，就注意为00000001、00000010、00000100，总位数与cpu核数一样。

pid        /var/run/nginx.pid;
error_log  /var/log/nginx/error.log notice; #错误日志及日志登记设置

events {
    use epoll; #客户端线程轮询方法、内核2.6版本以上的建议使用epoll
    worker_connections 65535; #设置一个worker可以打开的最大连接数
    multi_accept on; #告诉nginx收到一个新连接通知后接受尽可能多的连接。
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # access_log  /var/log/nginx/access.log  main;
	access_log off; #设置nginx是否将存储访问日志，关闭这个选项可以让读取磁盘IO操作更快

    server_names_hash_bucket_size 128; 
    client_header_buffer_size 32k;
    large_client_header_buffers 4 32k;
    client_max_body_size 8m; #上传文件大小限制。
    
    sendfile on; 　#开启sendfile（）函数，sendfile可以再磁盘和tcp socket之间互相copy数据。
    tcp_nopush on; #告诉nginx在数据包中发送所有头文件，而不是一个一个的发
    keepalive_timeout 30;  #keepalive 超时时间。
    tcp_nodelay on;  #告诉nginx不要缓存数据，而是一段一段的发送--当需要及时发送数据时，就应该给应用设置这个属性，这样发送一小块数据信息时就不能立即得到     #返回值。
    proxy_intercept_errors on;
    
    #fastcgi相关配置，不开启的话不会用到
    fastcgi_intercept_errors on;
    fastcgi_connect_timeout 1300;
    fastcgi_send_timeout 1300;
    fastcgi_read_timeout 1300;
    fastcgi_buffer_size 512k;
    fastcgi_buffers 4 512k;
    fastcgi_busy_buffers_size 512k;
    fastcgi_temp_file_write_size 512k;
 
    #链接相关配置
    proxy_connect_timeout      20s; #链接超时时间
    proxy_send_timeout         30s; #发送超时时间
    proxy_read_timeout         30s; #读取超时时间

    
    # 开启gzip压缩，会大量减少我们的发数据的量
    gzip on;
    # 不压缩临界值，大于1K的才压缩，一般不用改
    gzip_min_length 1k;
    # 压缩缓冲区
    gzip_buffers 16 64K;
    # 压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
    gzip_http_version 1.1;
    # 压缩级别，1-10，数字越大压缩的越好，时间也越长
    gzip_comp_level 5;
    # 进行压缩的文件类型
    gzip_types text/plain application/x-javascript text/css application/xml application/javascript;
    # 跟Squid等缓存服务有关，on的话会在Header里增加"Vary: Accept-Encoding"
    gzip_vary on;
    # IE6对Gzip不怎么友好，不给它Gzip了
    gzip_disable "MSIE [1-6]\.";
    

    include /etc/nginx/conf.d/*.conf;
}
```

`centos`内核优化,替换`/etc/sysctl.conf`

```shell
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).

fs.file-max = 999999
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 10240 87380 12582912
net.ipv4.tcp_wmem = 10240 87380 12582912
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 40960
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.ip_local_port_range = 1024 65000
```

> 之后使得配置立刻生效  `/sbin/sysctl -p`

修改 `/etc/security/limits.conf`后保存

```
*       soft    nproc   102400
*       hard    nproc   102400
*       soft    nofile  102400
*       hard    nofile  102400
```

> 上面这个修改如果重启了需要用`ulimit -a`看下有么生效，没有的话自行百度解决，关联到最大可打开的句柄数。

> 以上配置经过实战验证，单机4核8线程2.4GHZ主频 8G内存可以轻松达到5K+并发无压力

## 4.错误排查

- 类型1: upstream timed out
- 类型2: connect() failed
- 类型3: no live upstreams
- 类型4: upstream prematurely closed connection
- 类型5: 104: Connection reset by peer
- 类型6: client intended to send too large body
- 类型7: upstream sent no valid HTTP/1.0 header



**详细说明**

| 类型 | 错误日志                                                     | 原因                                                         | 解决办法                                                     |
| :--- | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 类型 | 错误日志                                                     | 原因                                                         | 解决办法                                                     |
| 1    | upstream timed out (110: Connection timed out) while connecting to upstream | nginx与upstream建立tcp连接超时，nginx默认连接建立超时为200ms | 排查upstream是否能正常建立tcp连接                            |
| 1    | upstream timed out (110: Connection timed out) while reading response header from upstream | nginx从upstream读取响应时超时，nginx默认的读超时为20s，读超时不是整体读的时间超时，而是指两次读操作之间的超时，整体读耗时有可能超过20s | 排查upstream响应请求为什么过于缓慢                           |
| 2    | connect() failed (104: Connection reset by peer) while connecting to upstream | nginx与upstream建立tcp连接时被reset                          | 排查upstream是否能正常建立tcp连接                            |
| 2    | connect() failed (111: Connection refused) while connecting to upstream | nginx与upstream建立tcp连接时被拒                             | 排查upstream是否能正常建立tcp连接                            |
| 3    | no live upstreams while connecting to upstream               | nginx向upstream转发请求时发现upstream状态全都为down          | 排查nginx的upstream的健康检查为什么失败                      |
| 4    | upstream prematurely closed connection                       | nginx在与upstream建立完tcp连接之后，试图发送请求或者读取响应时，连接被upstream强制关闭 | 排查upstream程序是否异常，是否能正常处理http请求             |
| 5    | recv() failed (104: Connection reset by peer) while reading response header from upstream | nginx从upstream读取响应时连接被对方reset                     | 排查upstream应用已经tcp连接状态是否异常                      |
| 6    | client intended to send too large body                       | 客户端试图发送过大的请求body，nginx默认最大允许的大小为1m，超过此大小，客户端会受到http 413错误码 | 调整请求客户端的请求body大小；调大相关域名的nginx配置：client_max_body_size； |
| 7    | upstream sent no valid HTTP/1.0 header                       | nginx不能正常解析从upstream返回来的请求行                    | 排查upstream http响应异常                                    |







------

nginx日志错误日志说明

| 错误信息                                                     | 错误说明                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| 错误信息                                                     | 错误说明                                                     |
| "upstream prematurely（过早的） closed connection"           | 请求uri的时候出现的异常，是由于upstream还未返回应答给用户时用户断掉连接造成的，对系统没有影响，可以忽略 |
| "recv() failed (104: Connection reset by peer)"              | （1）服务器的并发连接数超过了其承载量，服务器会将其中一些连接Down掉； （2）客户关掉了浏览器，而服务器还在给客户端发送数据； （3）浏览器端按了Stop |
| "(111: Connection refused) while connecting to upstream"     | 用户在连接时，若遇到后端upstream挂掉或者不通，会收到该错误   |
| "(111: Connection refused) while reading response header from upstream" | 用户在连接成功后读取数据时，若遇到后端upstream挂掉或者不通，会收到该错误 |
| "(111: Connection refused) while sending request to upstream" | Nginx和upstream连接成功后发送数据时，若遇到后端upstream挂掉或者不通，会收到该错误 |
| "(110: Connection timed out) while connecting to upstream"   | nginx连接后面的upstream时超时                                |
| "(110: Connection timed out) while reading upstream"         | nginx读取来自upstream的响应时超时                            |
| "(110: Connection timed out) while reading response header from upstream" | nginx读取来自upstream的响应头时超时                          |
| "(110: Connection timed out) while reading upstream"         | nginx读取来自upstream的响应时超时                            |
| "(104: Connection reset by peer) while connecting to upstream" | upstream发送了RST，将连接重置                                |
| "upstream sent invalid header while reading response header from upstream" | upstream发送的响应头无效                                     |
| "upstream sent no valid HTTP/1.0 header while reading response header from upstream" | upstream发送的响应头无效                                     |
| "client intended to send too large body"                     | 用于设置允许接受的客户端请求内容的最大值，默认值是1M，client发送的body超过了设置值 |
| "reopening logs"                                             | 用户发送kill -USR1命令                                       |
| "gracefully shutting down",                                  | 用户发送kill -WINCH命令                                      |
| "no servers are inside upstream"                             | upstream下未配置server                                       |
| "no live upstreams while connecting to upstream"             | upstream下的server全都挂了                                   |
| "SSL_do_handshake() failed"                                  | SSL握手失败                                                  |
| "SSL_write() failed (SSL:) while sending to client"          |                                                              |
| "(13: Permission denied) while reading upstream"             |                                                              |
| "(98: Address already in use) while connecting to upstream"  |                                                              |
| "(99: Cannot assign requested address) while connecting to upstream" |                                                              |
| "ngx_slab_alloc() failed: no memory in SSL session shared cache" | ssl_session_cache大小不够等原因造成                          |
| "could not add new SSL session to the session cache while SSL handshaking" | ssl_session_cache大小不够等原因造成                          |
| "send() failed (111: Connection refused)"                    |                                                              |



!>这边讲一个实际生产环境时遇到的一个问题，请求通过单位DMZ区的NGINX后转发到内网服务器，并发高的时候DMZ区的NGINX报`upstream timed out (110: Connection timed out) while connecting to upstream`，经过排查发现并发高的时候ping 延迟会突破200ms导致。这个问题暂时还在排查中。目前情况，单独压测内网服务器和dmz区服务器都能够承受8000+的并发。

# NGINX设置简单密码及浏览器判断

> nginx可以设置简单的密码以供访问时做一定的限制

1.我们需要装个插件生成个密码文件。我们这采用`httpd-tools`插件来完成

第一步，安装插件

```shell
yum -y install httpd-tools
```

2.第二部，利用该插件生成一个密码文件,第一个参数为文件名（不写具体默认是当前路径下），第二个参数为密码文件，需要以.db结尾

```shell
htpasswd -c xxxx xxxx
```

3.配置nginx

```shell
        location /jl {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            auth_basic "请输入验证信息"; #这里是验证时的提示信息,只有部分浏览器支持 
       	    auth_basic_user_file /etc/nginx/conf.d/jl.db; #刚才生成的第二个参数里的账号密码
        }
```

# NGINX判断不同的来源并分发到不同页面

> 有时我们有这种需求,这边贴一个完整的nginx配置，然后讲解下

```
server {
        listen       80;
        server_name  td;

        #先定义一个变量 赋值为空

        set $flag 0;

        #用 http_user_agent 判断是否微信浏览器访问

        if ($http_user_agent ~ "MicroMessenger|^$" ){

            set $flag "${flag}1";

        }

        location /jl {
             if ($flag = "01"){
                rewrite ^/ /jlwxfw/index.html;
             }

            root   /usr/share/nginx/html;
            index  index.html index.htm;
            auth_basic "请输入验证信息"; #这里是验证时的提示信息 
       	    auth_basic_user_file /etc/nginx/conf.d/jl.db;

        }

        location /jlwxfw {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

这个场景为判断腾讯微信的浏览器，如果是，则转发到`/jlwxfw/index.html`这个路径下然后我们监听下这个路径，配另一个页面即可。这里有个if判断，可以参考下。