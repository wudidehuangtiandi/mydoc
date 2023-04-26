# jenkins的部署运用

> 本文部署的jenkins也只是不使用docker的部署方式，官网有基于docker的部署方式并且是中文版的

[官网安装地址](https://www.jenkins.io/zh/doc/book/installing/)

!>docker 安装的注意事项,使用jenkins/jenkins镜像，不要使用官网推荐那个，会有各种奇怪问题，参考安装命令

```shell
docker run \
  --name jenkins \
  -u root \
  -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /home/jenkins:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkinsci/blueocean
```

## jenkins的非docker的正确安装

```shell
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



> 必要插件的安装，在我们配置全局工具及部署项目前我们需要安装一些必要的插件，包括git相关,nodejs,jdk,maven,gradle等



## jenkins部署前端项目



1.在`全局工具配置`中添加一个nodejs，切换源可以在Global 那个框种填写 npm --registry=https://registry.npm.taobao.org![avatar](https://picture.zhanghong110.top/docsify/16393781064243.png)



2.在系统配置中，配置SSH SERVER ，输入名字，地址，账户，密码，端口点击保存,这一步如果需要SSH通道，可以直接在高级-KEY种输入，注意这里只支持前缀为-----BEGIN RSA PRIVATE KEY-----的私钥

![avatar](https://picture.zhanghong110.top/docsify/16393785227014.png)

![avatar](https://picture.zhanghong110.top/docsify/16393786103793.png)

> 至此，准备工作完成，接下来我们开始配置项目

3.新建项目，输入名字，构建一个自由风格的项目

![avatar](https://picture.zhanghong110.top/docsify/16393789757698.png)

> 本项目以gitlab的项目自动部署为主

4.添加对应的仓库地址及用户名密码

![avatar](https://picture.zhanghong110.top/docsify/16393792775182.png)

5.构建触发器，这里忘记要不要改了，这个项目是这么配置的，可以参考下

![avatar](https://picture.zhanghong110.top/docsify/16393794104230.png)

6.构建环境配置，这里也忘记了，参考下这个项目的配置即可

![avatar](https://picture.zhanghong110.top/docsify/16393796203307.png)

7.构建，分为两步，

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

> 到步骤7前端项目就部署完毕了，下面我们讲解下gitlab和配置和前端项目中该如何打包

8.前端项目打包

由于需要将压缩包发送到目标服务器，因此前端项目需要配置打包插件`filemanager-webpack-plugin`。

这个插件不同的版本配置有所区别，我们以`^2.0.5`为例，在`vue.config.js`里导出的根目录下增加如下配置,作用为删除上一个压缩包并且打一个压缩包。

```shell
  configureWebpack: {
    plugins: [
      　new FileManagerPlugin({
        　　   onEnd: {
              　　 delete: [
                　　 path.resolve(__dirname, './dist/dist.zip'),
             　　  ],
              　　 archive: [
                　　   {source: path.resolve(__dirname, './dist/'), destination:  path.resolve(__dirname, './dist/dist.zip')},
               　　]
          　　 }
       　　})
    ],
    name: name,
    resolve: {
      alias: {
        '@': resolve('src')
      }
    }
  },
```

9.gitlab的配置

在构建触发器部分，有如下两张图，一张是点击高级后的配置，可以产生TOKEN还有一张是用来演示gitlab应该配置什么样子的链接

![avatar](https://picture.zhanghong110.top/docsify/16395472479680.png)



![avatar](https://picture.zhanghong110.top/docsify/16395473613800.png)

gitlab中填入以下属性，注意选择的分支需要和Jenkins的一致。当发起推送请求时就会触发jenkins的流程

![avatar](https://picture.zhanghong110.top/docsify/16395474676011.png)



> 至此，一个基本的前端项目就可以通过jenkins完成自动发布了，由于前端是直接由nginx转发，所以前端脚本只需要替换文件



## jenkins部署后端项目

> 下面我们来介绍jenkins如何自动部署java后端项目

1.配置全局工具，这边docker安装的jenkins如果使用推荐的插件则无需配置JDK。

>JDK，选一个版本后直接自动安装即可，SSH的话配置和前端一样，在系统配置中设置，这两个就不放图片了

maven,如下如所示在jenkins所在机器上放置maven所需配置文件（如果有私服需求），然后jenkins做出对应的配置

![avatar](https://picture.zhanghong110.top/docsify/16395490262370.png)

对应的下方选择一个maven版本安装即可

![avatar](https://picture.zhanghong110.top/docsify/16395492552419.png)

> 这里的gitlab设置和前端完全一样，jenkins构建环境之前的配置也和前端完全一样

2.构建，在构建环境中的Build Steps中配置

这一步区别在于首先需要设置打包命令，注意这个的话是忽略mvn的，正常时`mvn clean install package -Dmaven.test.skip=true`,然后下main设置正确配置文件即可。

![avatar](https://picture.zhanghong110.top/docsify/1639549728.png)

3.构建后操作，这步的目的就是把包发送到目标服务器并且执行命令启动

其前缀路径意义可以参考前端，完全一摸一样，如果没有的可以点击构建选项下的增加构建步骤，注意这边包路径是项目路径下第二个层级开始的。最后一行用来执行docker命令

![avatar](https://picture.zhanghong110.top/docsify/16395507121046.png)

> 最后也是配置webhook，这样一个后端项目就部署完毕了

