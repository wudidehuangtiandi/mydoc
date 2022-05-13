# GIT的基本使用及配置

> 为了解决`git`及`github`,`gitlab`,`gitlee`,集成小乌龟，`idea`及`vscode`时的问题，故而记录下具体的研究情况。

## 1.GIT的安装及配置

首先下载GIT，去官网搞一个地址如下

[下载地址](https://git-scm.com/)

![avatar](https://picture.zhanghong110.top/docsify/16524080655241.png)

点击下载

![avatar](https://picture.zhanghong110.top/docsify/16524083881135.png)



下载完后我们一直下一步即可，注意这边要选择下你要装的地方，然后这个命令框如果想用小乌龟也可以不装。

![avatar](https://picture.zhanghong110.top/docsify/16524088118785.png)

下面就一直下一步就好了,采用默认的就好了不用管这些了，有兴趣的也可以去看下各项配置的作用，还是比较多的。

> 至此，我们已经完成了git的安装下载及自动配置，通过在指定文件夹使用CMD打开的命令窗口即可直接操作GIT了

在安装好git后、使用git前，需要给git配置用户名和邮箱

我们可能会有疑问

1、为什么要配置用户名和邮箱？

因为Git是分布式版本控制系统，所以，每个机器都必须自报家门：你的名字和Email地址（名字和邮箱都不会进行验证），这样远程仓库才知道哪次提交是由谁完

成的。你也许会担心，如果有人故意冒充别人怎么办？这个不必担心，首先我们相信大家都是善良无知的群众，其次，真的有冒充的也是有办法可查的。

2、配置的用户名和邮箱对push代码到远程仓库有什么影响？

首先，配置的用户名和邮箱对push代码到远程仓库时的身份验证没有作用，即不用他们进行身份验证；他们仅仅会出现在远程仓库的commits里。

其次，按正常操作来说，你应该配置你的真实用户名和邮箱，这样一来在远程仓库的commits里可以看到哪个操作是你所为




```
git config --global user.name "username"  
git config --global user.email "email"
```



## 2.小乌龟的安装及汉化

下面安装小乌龟，这里面包含了小乌龟各种语言的汉化包，同时，小乌龟也可以解决SVN的图形化问题，一样的操作。

[小乌龟下载地址](https://tortoisegit.org/download/)

这个安装没啥内容，选择好对应的GIT安装地址即可

## 3.IDEA及VSCODE集成

IDEA的集成，我们直接选择下`git.exe`的路径即可

![avatar](https://picture.zhanghong110.top/docsify/16524116171735.png)





VSCODE集成我们



## 4.GIT的配置文件

> 我们这里主要解决GIT配置账号密码及SSH配置及多个客户端比如GITHUB,GITLAB,GITLEE同时使用的问题

