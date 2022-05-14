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

首先，配置的用户名和邮箱对push代码到远程仓库时的身份验证没有作用，即不用他们进行身份验证；他们仅仅会出现在远程仓库的commits里。（仅仅用于在git历史中显示）

其次，按正常操作来说，你应该配置你的真实用户名和邮箱，这样一来在远程仓库的commits里可以看到哪个操作是你所为

全局配置只要使用如下命令即可


```
git config --global user.name "username"  
git config --global user.email "email"
```

这个配置文件有如下优先级，可知上述命令增加的是第二级的配置文件。

系统配置文件，供所有用户的所有仓库使用，位置${git_home}/etc/gitconfig
用户配置文件，供某一用户的所有仓库使用，位置C:/users/${user_home}/.gitconfig
仓库配置文件，供某一用户的某一仓库使用，位置${自己的git仓库}/.git/config

## 2.小乌龟的安装及汉化

下面安装小乌龟，这里面包含了小乌龟各种语言的汉化包，同时，小乌龟也可以解决SVN的图形化问题，一样的操作。

[小乌龟下载地址](https://tortoisegit.org/download/)

这个安装没啥内容，选择好对应的GIT安装地址即可

## 3.IDEA及VSCODE集成

IDEA的集成，我们直接选择下`git.exe`的路径即可

![avatar](https://picture.zhanghong110.top/docsify/16524116171735.png)

VSCODE就更加简单了，默认会给我们增加git版本管理工具，安装GIT后会自动匹配。

![avatar](https://picture.zhanghong110.top/docsify/16524313747761.png)



## 4.GIT的配置文件

> 我们这里主要解决GIT配置账号密码及SSH配置及多个客户端比如GITHUB,GITLAB,GITLEE同时使用的问题

以上操作完成后我们就可以正常的使用https拉取代码及提交，此时我们提交代码需要每次都输入账号及密码

我们可以用以下设置来

设置记住密码（默认15分钟）：
`git config –global credential.helper cache`
如果想自己设置时间，可以这样做：
`git config credential.helper ‘cache –timeout=3600’`
这样就设置一个小时之后失效
长期存储密码：
`git config –global credential.helper store`

这么设置后你下次提交完代码，就会在一定时间内保存你的账号密码。

!>但是缺点也很明显，假如你不同的项目使用了不同的仓库，则会造成密码错误。而且例如`github`不支持登录时的账号密码，需要用生成的TOKEN作为密码

这边自己的仓库的话还是推荐还是使用SSH来完成代码的下拉及提交，我们只需要设置SSH即可区分不同的仓库

生成`SSHkey`

> ssh-keygen -t rsa -C '邮箱'

一直确定直到结束；根据日志信息里面的 SSH KEY 存储路径找到 .ssh/id_rsa.pub 文件

复制 .ssh/id_rsa.pub 文件内容（公钥）
打开 git 网站，右上角用户头像，点击 settings，左侧菜单 SSH KEYS，将文件内容复制到 key 里 添加就可以了

这样以后再也不用输入密码了，就可以使用SSH了。不通的网站用不用的邮箱即可，用相同的也可以。