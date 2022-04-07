# Centos常用操作

> 以下所有命令适用于centos7.x

## 1.常用命令

```shell
#显示文件列表
#ll等同于ls -l 表示显示当前目录下文件详细信息
ll  

#显示包括隐藏文件在内
ll -a  

#复制(将当前目录dist复制到tmp目录下，这边要注意如果dist是文件夹,而目标目录下有该名称的文件夹是不能覆盖过去的)
cp ./dist /tmp

#剪切,和复制同理
mv ./dist /tmp

#查看当前路径
pwd

#进入某文件夹
cd xxx

#编辑某配置文件
vim xxx
#进入后按s编辑模式，然后esc退出编辑  :wq保存  :q正常退出

#查看日志
tail -f xxxx.log

#查看执行过的命令历史
history

#查看内存占用,以M为单位
free -m

#ip查询
ip address

#查看进程（动态）
top
top -c #可以查看进程具体路径，可以直接带上以下命令
#这是个可交互命令
#P：根据CPU使用大小进行排序。
#T：根据时间、累计时间排序。
#q：退出top命令。
#m：切换显示内存信息。
#t：切换显示进程和CPU状态信息。
#c：切换显示命令名称和完整命令行。
#M：根据使用内存大小进行排序。
#W：将当前设置写入~/.toprc文件中。这是写top配置文件的推荐方法。

#查看进程（静态）
ps -ef
ps aux
#两者没太大差别，讨论这个问题，要追溯到Unix系统中的两种风格，System Ｖ风格和BSD 风格，ps aux最初用到Unix Style中，而ps -ef被用在System V Style中，两者输出略有不同。现在的大部分Linux系统都是可以同时使用这两种方式的。
#当需要查看某服务比如frp时，以下两者都可以，| 符号，是个管道符号，表示ps 和 grep 命令同时执行，grep 命令是查找（Global Regular Expression
#Print），能使用正则表达式搜索文本，然后把匹配的行显示出来；
#这个命令还有一些参数可以自行研究
ps -ef | grep frp
ps aux | grep frp

#查看端口下的进程,比如8080
ss -lntpd | grep :8080
#显示所有打开的的端口ss -l，显示所有tcp链接和进程ss -t -a
#查看所有Ip4协议也可以用 lsof -i 4 查看所有TCP链接 lsof -i tcp 查看端口使用lsof -i :80这些ss也能实现
#ss比netstat更加高效，当链接数量很大时使用ss -atr | wc -l 查看连接数会比 netstat -at | wc -l快很多而且ss能够显示更多更详细的有关TCP和连接状态的信息
#ss lsof netstat者三个命令比较类似 

#杀死某进程
kill -9 xxx(pid)

#防火墙的开启、关闭、禁用命令
#（1）设置开机启用防火墙：
 systemctl enable firewalld.service
#（2）设置开机禁用防火墙：
 systemctl disable firewalld.service
#（3）启动防火墙：
 systemctl start firewalld
#（4）关闭防火墙：
 systemctl stop firewalld
#（5）检查防火墙状态：
 systemctl status firewalld
#二、使用firewall-cmd配置端口
#（1）查看防火墙状态：
firewall-cmd --state
#（2）重新加载配置：
firewall-cmd --reload
#（3）查看开放的端口：
firewall-cmd --list-ports
#（4）开启防火墙端口：
firewall-cmd --zone=public --add-port=9200/tcp --permanent
#命令含义：
–zone #作用域
–add-port=9200/tcp #添加端口，格式为：端口/通讯协议
–permanent #永久生效，没有此参数重启后失效
#注意：添加端口后，必须用命令firewall-cmd --reload重新加载一遍才会生效
#（5）关闭防火墙端口：
firewall-cmd --zone=public --remove-port=9200/tcp --permanent

#给.sh增加可执行权限
#chmod是权限管理命令change the permissions mode of a file的缩写。。
#u代表所有者，
#x代表执行权限。
#+ 表示增加权限。
chmod u+x *.sh 
#就表示对当前目录下的*.sh文件的所有者增加可执行权限。。。

#给其他用户授权,谨慎使用
#读取权限 r = 4
#写入权限 w = 2
#执行权限 x = 1
#775 这三个数字代表拥有者，组用户，其他用户的权限。
chmod +r 775 
chmod +r 777

#后台运行
#不产生自生日志，采用服务中配置的日志产生方式
nohup java -jar ./xxx.jar --spring.profiles.active=prod >/dev/null 2>&1 & 
#产生自生日志,可以看到>和>>并没有太多区别
nohup java -jar ./xxx.jar --spring.profiles.active=prod >> /usr/local/node/output.log 2>&1 &
#标准输入(stdin)	0	< 或 <<	System.in	/dev/stdin -> /proc/self/fd/0 -> /dev/pts/0
#标准输出(stdout)	1	>, >>, 1> 或 1>>	System.out	/dev/stdout -> /proc/self/fd/1 -> /dev/pts/0
#标准错误输出(stderr)	2	2> 或 2>>	System.err	/dev/stderr -> /proc/self/fd/2 -> /dev/pts/0

#Systemd 统一管理所有 Unit 的启动日志。带来的好处就是，可以只用journalctl一个命令，查看所有日志（内核日志和应用日志）。日志的配置文
#件/etc/systemd/journald.conf
#查看最后20条系统日志信息
journalctl -n 20

#1、/var/log/secure 记录登录系统存取数据的文件(例如:pop3，ssh，telnet，ftp等都会记录在此);
#2、/ar/log/btmp 记录登录信息记录，被编码过，所以必须以lastb解析;
#lastb | awk ‘{ print $3}’ | sort | uniq -c | sort -nr | more
#Copy
#3、/var/log/message 几乎所有的开机系统发生的错误都会在此记录;
#4、/var/log/boot.log 记录一些开机或者关机启动的一些服务显示的启动或者关闭的信息;
#5、/var/log/maillog 记录邮件的存取和往来;
#6、/var/log/cron 用来记录crontab(定时任务)这个服务的内容;
#7、/var/log/lastlog 记录每个用户最后的登录信息;
#8、/var/log/btmp 记录错误的登录尝试;
#9、/var/log/dmesg 内核日志;
#10、/var/log/yum.log 使用yum安装的软件包信息
#清除登录信息日志解决被SSH扫描暴力破解后日志过大的问题 cat /dev/null > /var/log/btmp

#清除内存中cache中缓存信息
# 仅清除页面缓存（PageCache）
sync; echo 1 > /proc/sys/vm/drop_caches 
#清除目录项和inode
sync; echo 2 > /proc/sys/vm/drop_caches 
#清除页面缓存，目录项和inode
sync; echo 3 > /proc/sys/vm/drop_caches 

#查看所有用户
cat /etc/passwd
#切换用户
su xxxx
```

待续