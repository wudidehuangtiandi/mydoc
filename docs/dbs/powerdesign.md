# PowerDesign的简单使用

> 以下内容均基于powerdesign16

## 1.数据库逆向工程

首先正常创建我们应该

![avatar](https://picture.zhanghong110.top/docsify/16672954885214.png)

然后点击`PhysicalDataModel_1`同层级，创建`table`,可以直接这样创建或者右边拖进来也可

![avatar](https://picture.zhanghong110.top/docsify/1667295858955.png)

然后此时加入我们已经有数据库，则可开始逆向工程 点击`file/Reverse Engineer/Database`选择合适的数据库，下一步

![avatar](https://picture.zhanghong110.top/docsify/16672960148443.png)

这里需要我们安装`ODBC`驱动[下载地址](https://picture.zhanghong110.top/docsify/mysql-connector-odbc-8.0.31-win32.msi)或者[官网下载](https://dev.mysql.com/downloads/connector/odbc/)，注意位数需要和`powerdesign`需要一致,我们再电脑开始菜单中输入ODBC，打开`ODBC Data Sources (32-bit)`，添加一个`mysql ODBC8.0 unicode driver`之后 填入参数，如下

![avatar](https://picture.zhanghong110.top/docsify/16672966189806.png)



回到`powerdesign`,我们选择刚才添加的数据源

![avatar](https://picture.zhanghong110.top/docsify/16672967677436.png)

这里有三步，先取消全选，再切换到需要的库，再全选，点OK即可。

![avatar](https://picture.zhanghong110.top/docsify/16672968554668.png)

此时生成的库还有些许问题，我们可以看到，没有将注释行英文搞下来。我们再操作下

![avatar](https://picture.zhanghong110.top/docsify/1667296973501.png)



点击`tools/ExecuteCommands/Edit Run script`执行如下脚本

```vb
 Option   Explicit     
    ValidationMode   =   True     
    InteractiveMode   =   im_Batch  
    Dim blankStr  
    blankStr   =   Space(1)  
    Dim   mdl   '   the   current   model    
        
    '   get   the   current   active   model     
    Set   mdl   =   ActiveModel     
    If   (mdl   Is   Nothing)   Then     
          MsgBox   "There   is   no   current   Model "     
    ElseIf   Not   mdl.IsKindOf(PdPDM.cls_Model)   Then     
          MsgBox   "The   current   model   is   not   an   Physical   Data   model. "     
    Else     
          ProcessFolder   mdl     
    End   If    
        
    Private   sub   ProcessFolder(folder)     
    On Error Resume Next    
          Dim   Tab   'running     table     
          for   each   Tab   in   folder.tables     
                if   not   tab.isShortcut   then     
                      tab.name   =   tab.comment    
                      Dim   col   '   running   column     
                      for   each   col   in   tab.columns     
                      if col.comment = "" or replace(col.comment," ", "")="" Then  
                            col.name = blankStr  
                            blankStr = blankStr & Space(1)  
                      else    
                            col.name = col.comment     
                      end if    
                      next     
                end   if     
          next    
        
          Dim   view   'running   view     
          for   each   view   in   folder.Views     
                if   not   view.isShortcut   then     
                      view.name   =   view.comment     
                end   if     
          next    
        
          '   go   into   the   sub-packages     
          Dim   f   '   running   folder     
          For   Each   f   In   folder.Packages     
                if   not   f.IsShortcut   then     
                      ProcessFolder   f     
                end   if     
          Next     
    end   sub    
```

此时我们发现，略有问题，这CODE列没了

![avatar](https://picture.zhanghong110.top/docsify/16672975141475.png)

点击菜单`tools/Display Preferences`,选择`table`点击`Advanced`

![avatar](https://picture.zhanghong110.top/docsify/16672971361419.png)

选到`columns`点击这个按钮

![avatar](https://picture.zhanghong110.top/docsify/1667297583302.png)

勾上`code`即可

![avatar](https://picture.zhanghong110.top/docsify/1667297690208.png)

这样我们就完成了一次反向生成

![avatar](https://picture.zhanghong110.top/docsify/16672977465930.png)



## 2.根据pd生成数据库

我们点击`database/Generate Datebase`这时候我们点击应用，注意要确保filename在数据库中存在，选择的话需要选的我们需要的数据源（ODBC中配置的）

![avatar](https://picture.zhanghong110.top/docsify/16673528104322.png)

之后它会在指定目录生成一个SQL文件，还会弹出一个运行文件，我们`run`即可，可以发现数据库中成功生成了

![avatar](https://picture.zhanghong110.top/docsify/16673529375998.png)

> 单表生成直接运行sql去生成

![avatar](https://picture.zhanghong110.top/docsify/16673530931369.png)



