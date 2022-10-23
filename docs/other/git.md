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

小乌龟这边需要修改下设置，将网络下面的SSH客户端设置为Git根目录下的 git\usr\bin 中的ssh.exe

这样以后再也不用输入密码了，就可以使用SSH了。不通的网站用不用的邮箱即可，用相同的也可以。

## 5.git及一些常用操作

以下操作前提是需要设置账户身份即

```
git config --global user.name "xx"
git config --global user.email "邮箱"
```

创建一个新仓库推送到git仓库，`github`,`gitlab`,`gitlee`都略有不同，但是大致意思一致，我们以`gitlab`为例子

```
创建新的存储库
git clone ssh://git@172.23.112.16:8022/gechao/git.git
cd git
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master

将本地仓库推送到已存在的地址
cd existing_folder
git init
git remote add origin ssh://git@172.23.112.16:8022/gechao/git.git
git add .
git commit -m "Initial commit"
git push -u origin master

推送到一个已经存在的库
cd existing_repo
git remote rename origin old-origin
git remote add origin ssh://git@172.23.112.16:8022/gechao/git.git
git push -u origin --all
git push -u origin --tags
```

> 下面是一些日常生产过程中可能需要的操作

1.同步线上的分支情况

```
git remote update origin --prune
```

这个操作在idea上也可以用这个按钮来实现

![avatar](https://picture.zhanghong110.top/docsify/16526614627542.png)

2.当你代码写在某分支上，现在想要拉取某个其它分支的代码，比如你在自己的dev分支开发，此时想要拉取主分支上的代码

```
1.先要提交到本地或者push到自己分支

2.git checkout master，切刀想要拉取代码的分支，这边要注意，开发过工具建议使用工具的按钮切换，否则有无法实时检测你分支的情况出现。
切过去了Update下目标分支，保证代码最新。

3.checkout回自己的分支，然后 git merge origin/master ，将目标分支与自己分支合并，解决冲突即可，同样工具中建议使用提供的按钮，如下图所示
```

![avatar](https://picture.zhanghong110.top/docsify/16526659072845.png)

> vscode的操作方式，直接切换远程分支目标分支即可，或者切换到本地的目标分支，update下代码。之后切回自己分支。选择合并代码，选下目标分支即可

![avatar](https://picture.zhanghong110.top/docsify/16526674635520.png)

![avatar](https://picture.zhanghong110.top/docsify/16526676093777.png)

![avatar](https://picture.zhanghong110.top/docsify/16526677992624.png)

> 下面稍微讲一下`rebase`

```
1.拉公共分支最新代码的时候使用rebase，也就是git pull -r或git pull --rebase。这样的好处很明显，我用rebase拉代码下来，但有个缺点就是rebase以后我就不知道我的当前分支最早是从哪个分支拉出来的了，因为基底变了嘛。
2.往公共分支上合代码的时候，使用merge。如果使用rebase，那么其他开发人员想看主分支的历史，就不是原来的历史了，历史已经被你篡改了。举个例子解释下，比如张三和李四从共同的节点拉出来开发，张三先开发完提交了两次然后merge上去了，李四后来开发完如果rebase上去（注意李四需要切换到自己本地的主分支，假设先pull了张三的最新改动下来，然后执行<git rebase 李四的开发分支>，然后再git push到远端），则李四的新提交变成了张三的新提交的新基底，本来李四的提交是最新的，结果最新的提交显示反而是张三的，就乱套了。
3.正因如此，大部分公司其实会禁用rebase，不管是拉代码还是push代码统一都使用merge，虽然会多出无意义的一条提交记录“Merge … to …”，但至少能清楚地知道主线上谁合了的代码以及他们合代码的时间先后顺序
```

 3.仓库版本要回滚到某个`commit`,我们以`gitlab`为例子

!>此操作会撤销之后的所有提交，慎重使用

```
进入项目目录下
git log 一直回车。
找到递交的commit号，比如 c0b0d7ff52c47a2135cb838aba2ea657db8185c0
```

或者可以点下图这个按钮复制，就是钱买你那个ID的全部

![avatar](https://picture.zhanghong110.top/docsify/16526693606477.png)

```
然后找到需要的回滚到的操作git reset --hard 8c67eebf3e95ff31417f256d09832afab4b3b865

然后推送 git push -f -u origin dev 到这个分支即可 -f 代表强制推送，不然你切回到某个版本后提交，它会提示你需要Update
```



!>注意，有的时候`master`分支会被保护如果要回退需要解除保护。

```
我们以gitlab为例子如下图所示，分配主分支可推送角色时还需要在这设置的。
```

![avatar](https://picture.zhanghong110.top/docsify/16526703273615.png)



4.vscode同一分支提交代码与冲突解决

> 正常来说我们应当各自使用自己的分支，然后定期统一发起合并请求来合并自己写入的代码，由版本管理者来解决冲突。
>
> 第二种情况当多人开发同一分支时，此时如果我们一个人写代码，在不建分支的情况下则不会产生冲突，我们自然可以顺利的迭代版本，而多人开发又不建分支，则需要自行解决版本合并时的各种问题，我们就以这个场景，以`vscode`为例来演示下

我们新建一个demo项目，其中有test1和test2两个文件，内容分别写了1和2

![avatar](https://picture.zhanghong110.top/docsify/16665216369540.png)

此时我们修改test1中的内容为111（vscode都是开启自动保存的前提下）

![avatar](https://picture.zhanghong110.top/docsify/16665218708106.png)

我们可以看到，版本管理中多了一个test1的更改。

![avatar](https://picture.zhanghong110.top/docsify/16665219698357.png)



此时我们可以看到有提交，提交和推送，提交和同步这几个按钮，提交即所谓的暂存是一样的。即将代码先保存到本地库中，提交(commit)不会和远程的代码发成任何冲突，所以无论如何都会成功。

为了模拟冲突的情况下，推送和同步的区别，我们在提交之前先打开`github`先去修改下test1中的内容为123456并提交，如下图所示。

![avatar](https://picture.zhanghong110.top/docsify/16665222277558.png)



> 当只有一个分支又有几人合作时，正确的做法，在每次写代码前，都需要拉取代码，以尽量保证自己的代码和线上的一致性，但是这并不能保证我们提交时线上的代码和当前的没有代差，所以如果动过相同的模块，冲突几乎无法避免。

此时test1文件线上的版本和本地库的版本不一致。

我们分别测试 `提交`，`直接提交和推送`，`直接提交和同步`

由于我们推送到线上的代码必定是提交暂存的代码，所以第一步肯定是提交。我们首先是直接点提交。

![avatar](https://picture.zhanghong110.top/docsify/16665236652759.png)

我们可以发现，十分的顺利。 此时我们尚未和线上代码发生关系，我们可以想到两种后续操作，即点三个点的推送 或者 点击提交后vscode提示的同步更改按钮。

![avatar](https://picture.zhanghong110.top/docsify/16665247315664.png)



> 到此处我们应该不难发现,其实你点了提交后还是会面对推送或者同步按钮，所以刚才我们可以直接提交并推送或者提交并同步即可。



我们首先演示 点击`同步更改`，这个按钮和刚才直接点`提交并推送一致`

![avatar](https://picture.zhanghong110.top/docsify/16665250098443.png)

我们发现vscode弹出冲突修改，让我们解决冲突。此时我们的正确操作是点击上方的这牌按钮，在选择采用哪一条前严禁去直接`删改文件`，全都选择过后才可以。

![avatar](https://picture.zhanghong110.top/docsify/16665251747215.png)

至于它的意思，字面意思采用当前分支（就是这处`<<<<<<<` ` >>>>>>>`之间），采用你本地的版本(1111)还是采用线上的版本(123456) 或者是两者都采用（将会混在一起，不推荐），最后一个按钮是对比视图的意思。这是细粒度的操作，也可以直接在合并文件上右击,全部采用某版本，注意若果是大家的改的东西都要采用，则需要细粒度的选择。

![avatar](https://picture.zhanghong110.top/docsify/1666525396965.png)

为了演示效果，我们这边就采用我们本地的版本，即1111 去覆盖123456 。解决完冲突我们需要点击加号去暂存这些更改，即将解决后的版本提交到本地的意思

![avatar](https://picture.zhanghong110.top/docsify/16665256376426.png)

冲突文件全部暂存后我们即可点击同步按钮上传。

![avatar](https://picture.zhanghong110.top/docsify/1666526893268.png)

可以发现，线上发生了一次合并，我们这一套操作下来就相当于分支合并，只不过我们在本地完成了合并后又上传了。

!>这个操作过程中有个小坑要注意,如果你完全采用自己的第一个提交的版本（产生冲突的提交版）一个字都不改的话，会出现You have not concluded your merge (MERGE_HEAD exists)

其标志性的情况就是点完暂存，`vscode`暂存的版本这一栏消失

解决的话可以使用

```
git merge --abort
git reset --merge
```

然后重新git pull，此时又会回到冲突解决界面，如果全部采用自己的版本，记得解决完，改点，不要完全一样，然后再暂存，同步即可。



> 上面这个操作中我们一直是点击了同步，那么我们如果解决完冲突不点同步，而是直接提交会发生什么呢？

这边不贴图了，这里解决完直接提交。此时点推送，和点直接同步一个效果。



我们回到开头，当我们在有冲突的情况下点直接和推送会发生什么呢？这边文字形容下，就不贴图了，这个操作效果和提交，然后推送一个样子。推送会直接报错并且不会让你解决冲突。

![avatar](https://picture.zhanghong110.top/docsify/16665294037847.png)

我们此时点取消，它将会再次让我们解决冲突。我们依然有可能会遇到上面说的那个小坑。

> 分析

其实git并没有同步这项功能，一个标准的流程包括 提交,拉取,推送。 拉取的时候可能会遇到冲突，此时解决以后再推送即可。

我们一直点同步和最后一步点提交这两方式，其实由于第一步时点了提交并同步，它完成了包括提交，拉取这一系列的动作，并且在这里遇到冲突让我们解决了，最后无论推送也好同步也罢都没问题了。

但是我们一开始使用提交并推送，由于缺少拉取功能，就不会提示你解决冲突，我们需要再点同步然后再解决冲突，暂存，再推送。

> 结论

综上，我们用`vscode`多人提交单一分支时可以简单采用提交并同步按钮，此时有冲突我们就解决，切记在选择完全部冲突处的选用版本时，不可修改代码，当所有冲突处都选完采用目标以后，如果提交的内容和解决冲突前的代码一模一样，会报（MERGE_HEAD exists）。解决方式就是用上面两个命令回退合并，然后pull，之后冲i性能解决冲突，稍微改下，和第一个提交的版本不同即可。



如果不想用同步按钮，请遵循提交，拉取，有冲突就解决，推送这个流程。

