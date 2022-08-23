# nodejs多版本兼容及npm下载位置更改

# 1.卸载老版本

node的卸载，直接在windows的卸载里就可以去掉了，但是需要同时删除npm及一些下载的依赖。

默认位置为

```
C:\Users{User}\AppData\Roaming\npm（或%appdata%\npm）
C:\Users{User}\AppData\Roaming\npm-cache（或%appdata%\npm-cache）
```

# 2.安装新版本

[官网](https://nodejs.org/zh-cn/)下个长期维护版，然后直接无脑下一步即可。

# 3.npm默认源切换

```shell
#注意它将只会更改npm的源而不会更改之后下载的如yarn,pnpm的源
#切换淘宝镜像
npm config set registry https://registry.npm.taobao.org
#查看
npm get registry
```

# 4.切换npm默认存储位置

```shell
1、npm 全局安装地址
npm config get prefix

2、npm 缓存位置
npm config get cache
```

```shell
1、设置npm全局安装的地址
npm config set prefix "E:\npmrepository\nodejs\node_global"

2、npm缓存位置设置
npm config set cache "E:\npmrepository\nodejs\node_cache"
```

设置环境变量

```shell
#系统变量增加
NODE_PATH = E:\npmrepository\nodejs\node_global\node_modules
#用户变量PATH中增加
E:\npmrepository\nodejs\node_global
```

# 5.yarn,pnpm切换默认存储位置

此时安装其它包管理工具，依然会缓存到C盘, 依然需要修改镜像为国内，切换的方式是一样的。这里给出yarn及pnpm的配置demo

```shell
#安装yarn
npm install --global yarn

#切换淘宝源
yarn config set registry https://registry.npm.taobao.org/

#查看全局global bin位置yarn global bin
#如果我们改了npm的则这里不用改，会在E:\npmrepository\nodejs\node_global\bin下
#其它需要自行替换到指定的位置(nvm也不会自行更改)
yarn config set prefix "E:\yarnrepository\global\node_global\bin"

#查看全局安装位置yarn global dir
#修改全局安装包的位置
yarn config  set global-folder "E:\yarnrepository\global"，

#查看缓存位置yarn cache dir
#修改缓存的位置
yarn config set cache-folder "E:\yarnrepository\cache"

#完事之后查一下，就可以愉快的缓存了，实测yarn缓存的文件都在E盘中了而不会去到C盘
yarn config list
```

```shell
#安装pnpm
npm install -g pnpm

#查看源
pnpm config get registry 

#切换淘宝源
pnpm config set registry https://registry.npmmirror.com/

#pnpm的配置文件和npm共享都是.npmrc
#pnpm全局仓库路径(类似 .git 仓库)
pnpm config set store-dir E:\pnpmrepository\.pnpm-store

#pnpm全局安装路径
pnpm config set global-dir "E:\pnpmrepository\pnpm\pnpm-global" 

#pnpm全局bin路径
pnpm config set global-bin-dir "E:\pnpmrepository"

# pnpm全局缓存路径
pnpm config set cache-dir "E:\pnpmrepository\pnpm\cache

#获取的时候set换成get即可
```

!>这里要注意下pnpm和nodejs对应关系

| Node.js    | pnpm 4 | pnpm 5 | pnpm 6 | pnpm 7 |
| ---------- | ------ | ------ | ------ | ------ |
| Node.js 10 | ✔️      | ✔️      | ❌      | ❌      |
| Node.js 12 | ✔️      | ✔️      | ✔️      | ❌      |
| Node.js 14 | ✔️      | ✔️      | ✔️      | ✔️      |
| Node.js 16 | ?️      | ?️      | ✔️      | ✔️      |
| Node.js 18 | ?️      | ?️      | ✔️      | ✔️      |

# 6.nodejs版本共存

> 我们常常会遇到某些时候由于Node版本过高无法成功配置低版本的项目，此时需要版本共存

可以采用[`nvm`](https://www.cnblogs.com/gaozejie/p/10689742.html)来管理版本,[下载地址](https://github.com/coreybutler/nvm-windows/releases),下载最新的nvm-setup.zip解压下,运行exe,运行后需要选择nvm的位置及nodejs安装的位置，建议选择非C盘，这样就可以自动指定环境变量。

!>在此之前最好将上述1~5安装的都卸了

安装完后重启即可使用命令

```shell
#版本查看
nvm v 

#安装指定Nodejs版本
nvm install 16.17.0

#查看可下载的全部版本(建议去官网看，这个长度有限)
nvm ls available

#查看全部下载的node版本
nvm ls

#使用某版本的node(需要使用管理员运行cmd)
nvm use 16.17.0

#此时即可完成动态切换
```

> 这时候我们需要思考几个问题，切换的node自带的npm的各种缓存啊，包的路径在哪里。不同版本的node安装的工具是否共享，工具的位置又是否需要配置

首先我们切换下淘宝镜像，切换版本，可以发现，两个不同版本的node,npm使用了同一个切换过的源，由此可以推论它们使用相同配置文件

然后我们在16的Node版本中全局安装yarn,切回12。使用yarn,发现不行，可以再次推测出，他们的下载的依赖互相独立。

我们切到安装nvm命令的两个文件夹，可以发现，nvm文件夹下拥有不同版本的nodejs安装的包,每当切换版本的时候，它会将其中内容复制（全量替换）到nodejs安装文件中。这两个文件夹在nvm目录的settings.txt下设置（也就是安装的时候配置的位置）。



故而我们可以得到如下结论：

1.不同版本的nodejs，npm使用的配置文件都是`C:\Users{User}\.npmrc`中

2.不同版本的npm 全局安装地址都在nvm安装时就设定好了，切换版本时会全量替换，可以使用`npm config get prefix`查看,配置文件地址为nvm目录的`settings.txt`

3.npm缓存会放在C盘中的固定位置，我们可以自行设置比如`npm config get cache`,这样缓存就会同步放在这个位置,不同版本切换的时候建议清空

`npm cache clean --force`



> 至于yarn和pnpm这么处理

我们可以想到yarn和pnpm的配置文件全局都是引用的C盘的，故而切换版本后除了要重新Install下（注意不同版本的Node要装的版本相同）之后不同node下的yarn和pnpm将共享配置文件，当然上面的4~5两步依然需要配置。