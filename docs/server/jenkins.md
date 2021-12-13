# jenkins的部署运用

> 本文部署的jenkins也只是不使用docker的部署方式，官网有基于docker的部署方式并且是中文版的

[官网安装地址](https://www.jenkins.io/zh/doc/book/installing/)

## jenkins的非docker的正确安装

```
  sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo
  sudo rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
  yum install epel-release # repository that provides 'daemonize'
  yum install java-11-openjdk-devel  有JDK环境可以跳过这步
  yum install jenkins
  
  修改启动用户为root：
  vim /etc/sysconfig/jenkins

  JENKINS_USER="root"
  
  启动jenkins并加入开机启动
  systemctl start jenkins
  systemctl enable jenkins

  启动前查看该目录是空的，启动后会生成相应的文件
  ll /var/lib/jenkins/


  登录web页面进行安装：http://ip:port　　(默认端口8080)


  根据界面的提示信息去服务端查看密码并输入,这边查看到的是默认密码，可以点击右上角用户修改
  cat /var/lib/jenkins/secrets/initialAdminPassword 

  新手入门点右上角叉掉，就可以开始使用了

  修改为中文，下个local插件， 重启jenkins的方法 ，浏览器输入地址http://localhost:8080/restart 即可

  需要装MAVWN SSH GIT GITLAB NODEJS等插件

  修改时间 ：打开jenkins的【系统管理】---> 【脚本命令行】，在命令框中输入一下命令【时间时区设为 亚洲上海】： 
  System.setProperty('org.apache.commons.jelly.tags.fmt.timeZone', 'Asia/Shanghai')

```



## jenkins部署前端项目



1.插件管理中安装nodeJs



![avatar](https://picture.zhanghong110.top/docsify/16393769645656.png)



2.在`全局工具配置`中添加一个nodejs![avatar](https://picture.zhanghong110.top/docsify/16393781064243.png)



3.配置SSH SERVER ，输入名字，地址，账户，密码，端口点击保存

![avatar](https://picture.zhanghong110.top/docsify/16393785227014.png)

![avatar](https://picture.zhanghong110.top/docsify/16393786103793.png)

> 至此，准备工作完成，接下来我们开始配置项目

4.新建项目，输入名字，构建一个自由风格的项目

![avatar](https://picture.zhanghong110.top/docsify/16393789757698.png)

> 本项目以gitlab的项目自动部署为主

5.添加对应的仓库地址及用户名密码

![avatar](https://picture.zhanghong110.top/docsify/16393792775182.png)

6.构建触发器，这里忘记要不要改了，这个项目是这么配置的，可以参考下

![avatar](https://picture.zhanghong110.top/docsify/16393794104230.png)

7.构建环境配置，这里也忘记了，参考下这个项目的配置即可

![avatar](https://picture.zhanghong110.top/docsify/16393796203307.png)

8.构建，分为两步，

第一步是指源码拉过来后在jenkins环境中所作的命令，主要作用是构建出项目的打包内容，可以根据所需修改脚本

```
echo $PATH
node -v 
npm install
rm -rf ./dist/*
npm run build:prod
```

![avatar](https://picture.zhanghong110.top/docsify/16393797165329.png)

第二步将打包好的项目发送到目标服务器，这边需要讲解几点，Source files和Remove prefix是指在，jenkins机的源码打包根目录下和需要移除的前缀，

Remote directory和Exec command是指目标服务器的路径和在该路径下执行的脚本，这里要注意此处Remote directory是在SSH配置的前缀下的路径。

```
cd /u01/ycrh/front/dist
unzip -o dist.zip -d .
echo "success"
```

![avatar](https://picture.zhanghong110.top/docsify/16393801154093.png)

> 到步骤8前端项目就部署完毕了，下面我们讲解下gitlab和配置和前端项目中该如何打包