# 关于docsify的使用及部署



## 1.项目说明

仓库地址是https://github.com/wudidehuangtiandi/mydoc



## 2.本地运行此项目



1. 确保全局安装过`npm`的`docsify`插件，如果已安装过这步可以跳过

   ```
   npm i docsify-cli -g
   ```

   !> 如果报权限不够的错误需要用`sudo npm i docsify-cli -g`或者windows下管理员方式运行

2. 初始化项目

   ```
   docsify init ./docs
   ```

3. 在项目根目录下运行

   ```
   docsify serve docs
   ```

   

## 3.托管到gitHub



> 项目运行起来以后可以自己使用`nginx`部署，或者可以托管到`github`,`gitlab`或者`gitlee`上，本项目采用托管到`github`的方式，优点是相当的简单，托管的好处是上传即自动更新，`nginx`则需要配合`jenkins`来实现，由于`gitlab`需要配置额外的`runner`，所以权衡之下还是采用	`github`来部署。带来的缺点是不翻墙无法访问到。



1.登录`github`，创建一个仓库，起个名字

- ![avatar](https://picture.zhanghong110.top/docsify/20200105143404136.png)

 2.将本地创建好的`docsify`的项目推送到`github`

 3.使用`github pages`宫娥能建立站点，这一步相当简单，首先在仓库库选择settings选项

![avatar](https://picture.zhanghong110.top/docsify/20200105145128951.png)

4.选择分支及对应木目录，推荐使用默认目录，然后`github`就会在上方告诉你你应该访问的地址，至此，即可达到上传即更新。

![avatar](https://picture.zhanghong110.top/docsify/16389324572795.png)







