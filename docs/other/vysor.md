# 利用电脑控制手机的技巧

> 本文只展示安卓机，不过也适用于苹果，自己参考文档

前置条件，安卓机多次点击操作系统版本号打开开发者模式，完事后进入开发者选项，打开USB调试

## vysor

这个软件可以用来控制手机端，支持有线和无线操作[官网](https://www.vysor.io/)下载，支持windiws linux mac及浏览器。

完事之后windows的话可能需要安装ADB

```
ADB是Android Debug Bridge的缩写，它是一个Android SDK中的工具，用于连接Android设备（手机）和PC端，方便调试设备或调试开发的Android APP。

ADB是一个客户端-服务器端程序，其中客户端是你用来操作的电脑，服务器端是Android设备。电脑上需要安装客户端，客户端包含在SDK里。设备上不需要安装，只需要在手机上打开USB调试（4.0：设备－开发人员选项）adb配置。

ADB具有多种功能，如安装卸载APK、拷贝推送文件、查看设备硬件信息、查看应用程序占用资源、在设备执行Shell命令等。
```

ADB安装路径如下：

Windows版本：https://dl.google.com/android/repository/platform-tools-latest-windows.zip
Mac版本：https://dl.google.com/android/repository/platform-tools-latest-darwin.zip
Linux版本：https://dl.google.com/android/repository/platform-tools-latest-linux.zip

下载解压了以后配置一个环境变量 增加adb 到路径platform-tools下完事了在path下增加%adb% 即可

此时使用数据线链接到手机会引导你手机下载vysor，完事之后就能控制了

如果需要无线的需要手机和电脑处于同一网络环境下，首先电脑黑窗口 `adb version`查看下adb有无正确配置

此时链接数据线的情况下使用命令`adb devices`可以查看此时的链接，应该有一个，完事之后输入命令`adb tcpip 5555`可以打开一个5555的端口，在`adb connect 手机IP:5555` 此时显示connected即可

此时上面应该会出现两个链接，拔掉数据线，仅存一个链接就是你的无线远程链接了。点进去就能愉快的双向操作了。

其实使用软件上的connect network or shared device,不用输入端口，一开始也不用线，直接链接手机也可以，connect network or shared device就相当于上面这些操作

下面几个常用命令

查看链接`adb devices`

重启服务`adb start-srever`

杀死服务`adb kill-server`

!>注意，此教程仅适用于安卓，苹果请参照vysor右侧IOS教程操作





