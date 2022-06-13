# gradle

## 1. gradle介绍

### 1.1 gradle简介

参考链接：https://blog.csdn.net/freekiteyu/article/details/80677361

Gradle :自动化构建项目的工具，用来帮助我们自动构建项目。就像我们在写 Java 项目的时候，如果没有构建工具，我们需要先执行 `javac`命令先将 Java 源码编译成 class 文件，然后再执行 jar 命令再把 class 文件打成 jar 包。有了构建工具我们只需要点一个按钮就可以了。

Java 的构建，经历了从 Ant -> Maven -> Gradle 的过程，每一次的进步，都是为了解决之前的工具带来的问题:

Ant：Ant 的功能虽然强大，但过于灵活，规范性不足，对目录结构及 build.xml 没有默认约定，且没有统一的项目依赖管理。
Maven：Maven 解决了规范性的问题，也顺带解决了依赖项统一管理的问题，但由于规范性太强，灵活性不足，pom.xml 采用 xml 结构，项目一大，Xml 就显得冗长。

Gradle综合了 Ant 和 Maven 的优点，吸收了 Ant 中 task 的思想，然后把 Maven 的目录规范以及仓库思想也融合了进来，但允许用户自由的修改默认的规范（如：可随意修改源码目录），配置文件则采用 Groovy 语言来书写，Groovy 是一门可编程语言，配置文件本身就可以视为一份源代码，并最终交由 Gradle 来处理执行。它不仅仅是一个构建工具，还是一个开发框架,它基于 Groovy 语言。 我们可以通过 Groovy 语言去写自己的Gradle 插件，也可以去编写指定的脚本去改变构建规则。

### 1.2 groovy language

参考链接1：https://blog.csdn.net/freekiteyu/article/details/80846149

参考链接2：https://developer.aliyun.com/article/885778

#### 1.2.1 简介

Groovy 同时本身是一种 `DSL`

> DSL（Domain Specific Language）是针对某一领域，具有受限表达性的一种计算机程序设计语言。 常用于聚焦指定的领域或问题，这就要求 DSL 具备强大的表现力，同时在使用起来要简单。说到DSL，大家也会自然的想到通用语言（如Java、C等）。
>
> 最常见的分类方法是按照DSL的实现途径来分类。马丁·福勒曾将DSL分为内部和外部两大类，他的分类法得到了绝大多数业界人士的认可和沿袭。内部与外部之分取决于DSL是否将一种现存语言作为宿主语言，在其上构建自身的实现。
>
> 内部DSL
>
> 也称内嵌式DSL。因为它们的实现嵌入到宿主语言中，与之合为一体。内部DSL将一种现有编程语言作为宿主语言，基于其设施建立专门面向特定领域的各种语义。例如：Kotlin DSL、Groovy DSL等；
>
> 外部DSL
>
> 也称独立DSL。因为它们是从零开始建立起来的独立语言，而不基于任何现有宿主语言的设施建立。外部DSL是从零开发的DSL，在词法分析、解析技术、解释、编译、代码生成等方面拥有独立的设施。开发外部DSL近似于从零开始实现一种拥有独特语法和语义的全新语言。构建工具make 、语法分析器生成工具YACC、词法分析工具LEX等都是常见的外部DSL。例如：正则表达式、XML、SQL、JSON、 Markdown等；

​        Groovy 程序运行时，首先被编译成 Java 字节码，然后通过 JVM 来执行，实际上，由于 Groovy Code 在真正执行的时候，已经变成了 Java 字节码， 因此 JVM 根本不知道自己运行的是 Groovy 代码

#### 1.2.2 基本语法

- 变量

```groovy
def variable = 1//不需要指定类型，不需要分号结尾
def int x = 1//也可以指定类型
```

- 函数

```groovy
//无需指定参数类型
String test(arg1, arg2) {
    return "hello"
}

//返回值也可以无类型
def test2(arg1, arg2) {
    return 1
}

def getResult() {
    "First Blood, Double Kill" // 如果这是最后一行代码，则返回类型为String
    1000 //如果这是最后一行代码，则返回类型为Integer
}

//函数调用，可以不加()
test a,b
test2 a,b
getResult()

```

- 字符串

``` 
//单引号包裹的内容严格对应Java中的String，不对$符号进行转义
def singleQuote='I am $ dolloar' //打印singleQuote时，输出I am $ dollar

def x = 1
def test = "I am $x" //打印test时，将输出I am 1
```

- 容器类

  Groovy中的容器类主要有三种： List(链表)、Map(键-值表)及Range(范围)。

```groovy
//List
// 元素默认为Object，因此可以存储任何类型
def aList = [5, 'test', true]
println aList.size  //结果为3
println aList[2]  //输出true
aList[10] = 8
println aList.size // 在index=10的位置插入元素后，输出11，即自动增加了长度
println aList[9] //输出null， 自动增加长度时，未赋值的索引存储null

//添加as关键字，并指定类型
def aList = [5, 'test', true] as int[]

//Map
def aMap = ['key1':1, "key2":'test', key3:true]

//读取元素
println aMap.key1    //结果为1
println aMap.key2    //结果为test
             //注意这种使用方式，key不用加引号

println aMap['key2'] //结果为test

//插入元素
aMap['key3'] = false
println aMap         //结果为[key1:1, key2:test, key3:false] 
                     //注意用[]持有key时，必须加引号

aMap.key4 = 'fun'    //Map也支持自动扩充
println aMap         //结果为[key1:1, key2:test, key3:false, key4:fun]

//Range
def aRange = 1..5
println aRange       // [1, 2, 3, 4, 5]

aRange = 1..<6       
println aRange       // [1, 2, 3, 4, 5]

println aRange.from  // 1
println aRange.to    // 5

println aRange[0]    //输出1
aRange[0] = 2        //抛出java.lang.UnsupportedOperationException

```

- 闭包

```groovy
//同样用def定义一个闭包
def aClosure = {
    //代码为具体执行时的代码
    println 'this is closure'
}

//像函数一样调用，无参数
aClosure() //将执行闭包中的代码，即输出'this is closure'

//下面这种写法也可以
//aClosure.call()

```

- 类

   Groovy 中定义类的方法及成员变量均默认是 `public` 的。

```groovy
package com.jeanboy.groovy

class Test {
    String mName
    String mTitle

    Test(name, title) {
        mName = name
        mTitle = title
    }

    def print() {
        println mName + ' ' + mTitle
    }
}

```

其他类引用

```groovy
import com.jeanboy.groovy.Test

def test = new Test('superman', 'hero')
test.print()
```

- 文件

```groovy
def targetFile = new File("/home/jeanboy/Desktop/file.txt")
//读文件的每一行
targetFile.eachLine { String oneLine ->
    println oneLine
}
def bytes = targetFile.getBytes()//返回文件对应的 byte()

```

### 1.3 dsl

参考链接：https://blog.csdn.net/freekiteyu/article/details/81066845

#### 1.3.1 简介

Gradle 是一个编译打包工具，但实际上它也是一个编程框架。Gradle 有自己的 API 文档，对应链接如下：

- [Gradle User Manual - 官方介绍文档](https://docs.gradle.org/current/userguide/userguide.html)

- [DSL Reference - API 文档](https://docs.gradle.org/current/dsl/)

因此，编写 Gradle 脚本时，我们实际上就是调用 Gradle 的 API 编程。

#### 1.3.2 基本组件

- Project

Gradle 中，每一个待编译的工程都是一个 Project。

- Task

每一个 Project 在构建的时候都包含一系列的 Task。

- Plugin

一个 Project 到底包含多少个 Task，在很大程度上依赖于编译脚本指定的插件。插件定义了一系列基本的 Task，并指定了这些 Task 的执行顺序。

整体来看，Gradle 作为一个框架，负责定义通用的流程和规则；根据不同的需求，实际的编译工作则通过具体的插件来完成。

#### 1.3.3 构建过程

```javascript
ProjectName
	|-app
		|-build
		|-lib
		|-src
		|-build.gradle
	|-library-test
		|-build.gradle
	|-gradle
		|-wrapper
	|-build.gradle	
	|-settings.gradle

```

#### 1.3.4 生命周期

- 初始化阶段

  读取项目根目录中 `setting.gradle` 中的 include 信息，决定有哪几个工程加入构建，并为每个项目创建一个 Project 对象实例

- 配置阶段

  执行所有 Project 中的 `build.gradle` 脚本，配置 Project 对象，一个对象由多个任务组成，此阶段也会去创建、配置 task 及相关信息。

- 运行阶段

  根据 Gradle 命令传递过来的 task 名称，执行相关依赖任务。task 的执行阶段。首先执行 `doFirst {}` 闭包中的内容，最后执行 `doLast {}` 闭包中的内容。

Gradle 基于 Groovy，Groovy 又基于 Java。所以，Gradle 执行的时候和 Groovy 一样，会把脚本转换成 Java 对象。Gradle 主要有三种对象，这三种对象和三种不同的脚本文件对应，在 Gradle 执行的时候，会将脚本转换成对应的对端：

Gradle 对象

当我们执行 gradle xxx 或者什么的时候，gradle 会从默认的配置脚本中构造出一个 Gradle 对象。在整个执行过程中，只有这么一个对象。Gradle 对象的数据类型就是 Gradle。我们一般很少去定制这个默认的配置脚本。

Project 对象

每一个 build.gradle 会转换成一个 Project 对象。

Settings 对象

显然，每一个 settings.gradle 都会转换成一个 Settings 对象。

> 对于其他 gradle 文件，除非定义了 class，否则会转换成一个实现了 Script 接口的对象。

#### 1.3.5 Task

Task 是 Gradle 中的一种数据类型，它代表了一些要执行或者要干的工作。不同的插件可以添加不同的 Task。每一个 Task 都需要和一个 Project 关联。

- 任务创建

```groovy
task hello {
    doLast {//doLast 可用 << 代替，不推荐此写法
        println "hello"//在 gradle 的运行阶段打印出来
    }
}

task hello {
    println "hello"//在 gradle 的配置阶段打印出来
}

```

task 中有一个 action list，task 运行时会顺序执行 action list 中的 action，doLast 或者 doFirst 后面跟的闭包就是一个 action，doLast 是把 action 插入到 list 的最后面，而 doFirst 是把 action 插入到 list 的最前面

- 任务依赖

当我们在 Android 工程中执行 `./gradlew build` 的时候，会有很多任务运行，因为 build 任务依赖了很多任务，要先执行依赖任务才能运行当前任务。

任务依赖主要使用 `dependsOn` 方法，如下所示：

```groovy
task A << {println 'Hello from A'}
task B << {println 'Hello from B'}
task C << {println 'Hello from C'}
B.dependsOn A	//执行 B 之前会先执行 A
C.dependsOn B	//执行 C 之前会先执行 B

```

另外，你也可以在 Task 的配置区中来声明它的依赖：

```groovy
task A << {println 'Hello from A'}
task B {
    dependsOn A
    doLast {
        println 'Hello from B'  
    }
}

```

mustRunAfter：

例如下面的场景，A 依赖 B，A 又同时依赖 C。但执行的结果可能是 B -> C -> A，我们想 C 在 B 之前执行，可以使用 mustRunAfter。

```groovy
task A << {println 'Hello from A'}
task B << {println 'Hello from B'}
task C << {println 'Hello from C'}
A.dependsOn B
A.dependsOn C
B.mustRunAfter C	//B 必须在 C 之后执行

```

finalizedBy：在 Task 执行完之后要执行的 Task。

- 增量构建

你在执行 Gradle 命令的时候，是不是经常看到有些任务后面跟着 [UP-TO-DATE]，这是怎么回事？

在 Gradle 中，每一个 Task 都有 inputs 和 outputs，如果在执行一个 Task时，如果它的输入和输出与前一次执行时没有发生变化，那么 Gradle 便会认为该 Task 是最新的，因此 Gradle 将不予执行，这就是增量构建的概念。

一个 Task 的 inputs 和 outputs 可以是一个或多个文件，可以是文件夹，还可以是 project 的某个 property，甚至可以是某个闭包所定义的条件。自定义 Task 默认每次执行，但通过指定 inputs 和 outputs，可以达到增量构建的效果。

- 依赖传递

Gradle 默认支持传递性依赖，比如当前工程依赖包A，包 A 依赖包 B，那么当前工程会自动依赖包 B。同时，Gradle 支持排除和关闭依赖性传递。

#### 1.3.6 常用命令

> $ gradle tasks // 查看根目录包含的 task
>
> $ gradle tasks -all // 查看根目录包含的所有 task
>
> $ gradle app:tasks //查看具体 Project 中的 task
>
> $ gradle projects //查看项目下所有的子 Project
>
> $ gradle build //构建项目

### 1.4 groovy与dsl的关系

```groovy
//project 的buildScript方法  实际返回的是一个ScriptHandler对象
buildScript {
    repositories {
         mavenCentral()
    }
}
//调用apply方法 参数是一个 map,调用时参数省略了括号
apply plugin: 'java'
//定义扩展属性(可选)
ext {
    foo="foo"
}
//定义局部变量(可选)
def bar="bar"

//修改项目属性(可选)
group 'pkaq'
version '1.0-SNAPSHOT'

//project 的repositories 方法  实际返回的是一个RepositoryHandler对象
repositories {
    jcenter()
}

//project 的dependencies 方法  实际返回的是一个DependencyHandler对象
dependencies {
    compile "cn.pkaq:ptj.tiger:+"
}

//调用Task task(String name, Closure configureClosure)方法
task printFoobar {
    println "${foo}__${bar}"
}

```

### 1.5 插件开发

参考文档：https://blog.csdn.net/freekiteyu/article/details/81353922

在学习中。。。

### 1.6 插件发布

参考文档：https://blog.csdn.net/freekiteyu/article/details/81449721

在学习中。。。

## 2. gradle使用

参考链接1：https://www.jianshu.com/p/724d1abc61a2

参考链接2：https://www.jianshu.com/p/7ccdca8199b8

### 2.1 引入插件

#### 高版本`plugins`

**必须在Top Level（父项目或子项目的gradle.build`顶级`），直接使用**

```groovy
plugins{
    id 'java'
    id 'org.springframework.boot' version '2.4.1'
}
```

注意： `java`是“核心插件”，而`org.springframework.boot`是“社区插件”（[Gradle插件中心](https://links.jianshu.com/go?to=https%3A%2F%2Fplugins.gradle.org%2F)），必须指定version

#### 低版本`apply`

1. 与buildscript结合：

   ```groovy
   buildscript {
       ext {
           springBootVersion = "2.3.3.RELEASE"
       }
       repositories {
           mavenLocal()
           maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
           jcenter()
       }
   //    此处引入插件
       dependencies {
           classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
       }
   }
   apply plugin: 'java' //核心插件，无需事先引入
   apply plugin: 'org.springframework.boot' //社区插件，需要事先引入，不必写版本号
   ```

2. 父项目的subprojects块中指定子项目中使用：

   ```groovy
   // 此处也可与buildscripts和apply plugin结合
   plugins {
       id 'java'
       id 'org.springframework.boot' version '2.4.2'
   }
   //所有的子项目，都应用了以下插件
   subprojects {
       apply plugin: 'java' //核心插件，无需事先引入
       apply plugin: 'org.springframework.boot' //社区插件，需要事先引入，不必写版本号
   }
   ```

3. 子项目中直接使用：

   ```groovy
   //这是子项目的位置
   apply plugin: 'java' //核心插件，无需事先引入
   apply plugin: 'org.springframework.boot' //社区插件，需要事先引入，不必写版本号
   ```

### 2.2 组成部分

#### 2.2.1 局部变量

参考链接：https://blog.csdn.net/yue530tomtom/article/details/79399417

**申明局部变量**

局部变量使用关键字 def 来声明，其只在声明它的地方可见 . 局部变量是 Groovy 语言的一个基本特性

```groovy
def var = "var_value"

task testLocalVar() {
    def varInner = "inner"
    println("local var's value:["+var+"]")
}

//println("local varInner's value:["+varInner+"]")
```

#### 2.2.2 属性定义

**定义扩展属性**

属性扩展允许将新属性添加到现有的域对象中。类似map可以存储键值对，所有的已知扩展有一个统一的类型”ext”,支持扩展的对象有projects, tasks, configurations, dependencies等。可以在运行时使用其他对象扩展的对象。

举例1：

```groovy
ext {
    extProperties="extProperties_value"
}

task printProperties << {
    println extProperties
}
```

结果

```groovy
:printProperties
extProperties_value
```

举例2：

```groovy
class MyExtension {
  String foo

  MyExtension(String foo) {
    this.foo = foo
  }
}

project.extensions.create('custom', MyExtension, "bar")

println project.custom instanceof MyExtension
println project.custom.foo == "bar"

project.custom {
  println foo == "bar"
  foo = "other"
}
println project.custom.foo == "other"

println project.custom instanceof ExtensionAware
project.custom.extensions.create("nested", MyExtension, "baz")
println project.custom.nested.foo == "baz"

println project.hasProperty("myProperty") == false
project.ext.myProperty = "myValue"

println project.myProperty == "myValue"
```

属性扩展的一个重要特性是，它的所有属性都通过具有扩展名的ExtensionAware对象来进行读取和写入

```groovy
project.ext.set("myProp", "myValue")
println project.myProp == "myValue"

project.myProp = "anotherValue"
println project.myProp == "anotherValue"
println project.ext.get("myProp") == "anotherValue"
```

属性扩展对象支持groovy属性语法。也就是说，可以通过扩展来读取属性

```groovy
//展示不同的赋值方式
//1、
project.ext {
  myprop = "a"
}
//读取值并判断(下同)
println project.myprop == "a"
println project.ext.myprop == "a"

//2、
project.myprop = "b"
println project.myprop == "b"
println project.ext.myprop == "b"
//3、
project.ext["otherProp"] = "a"
println project.otherProp == "a"
println project.ext["otherProp"] == "a"
```

#### 2.2.3 属性修改/指定

```groovy
//修改项目属性(可选)
group 'pkaq'
version '1.0-SNAPSHOT'
```

#### 2.2.4 定义仓库

```groovy
//定义仓库,当然gradle也可以使用各maven库 ivy库 私服 本地文件等,后续章节会详细介绍(可选)
repositories {
    jcenter()
}
```

#### 2.2.5 定义(申明)依赖

通常而言，依赖管理包括两部分，对依赖的管理以及发布物的管理；依赖是指构建项目所需的构件(jar包等)。例如，对于一个应用了spring普通的java web项目而言，spring相关jar包即项目所需的依赖。发布物，则是指项目产出的需要上传的项目产物。

```groovy
dependencies {
    implementation 'io.quarkus:quarkus-config-yaml'
}
```

> 采用变量统一控制版本号（上面的代码） 或者 自动获取最新版本依赖（下面的代码）

```groovy
dependencies {
    implementation "org.springframework.boot:spring-boot-starter-web:+"
}
```

如果你想某个库每次构建时都检查是否有新版本，那么可以采用`+`来让Gradle在每次构建时都检查并应用最新版本的依赖。当然也可以采用1.x,2.x的方式来获取某个大版本下的最新版本。

#### 2.2.6 自定义任务

```groovy
//自定义任务(可选)
// 使用task 后带任务名称 加上执行闭包{}
task t1 {
  println 't1'
}
// 任务名称后加上圆括号
task t2() {
  println 't2'
}
// IDEA 刷新 Task 点击 build 或 左下 Terminal 输入 gradle build 即可看到结果
t1
t2
```

### 2.3 依赖管理

> 依赖管理包括两部分，对依赖的管理以及发布物的管理；依赖是指构建项目所需的构件(jar包等)。例如，对于一个应用了spring普通的java web项目而言，spring相关jar包即项目所需的依赖。发布物，则是指项目产出的需要上传的项目产物。

#### 2.3.1 采用变量统一控制版本号

方式一：

```groovy
dependencies {
    def bootVersion = "1.3.5.RELEASE"
    compile     "org.springframework.boot:spring-boot-starter-web:${bootVersion}",  
                "org.springframework.boot:spring-boot-starter-data-jpa:${bootVersion}",
                "org.springframework.boot:spring-boot-starter-tomcat:${bootVersion}"
}
```

方式二：

```groovy
ext{

    springMVCVersion        = '5.1.14.RELEASE'
    springContextVersion    = '5.1.14.RELEASE'
    log4jVersion            = '2.13.0'
    junitVersion			= '4.12'
}

dependencies {

    compile group: 'org.springframework', name: 'spring-webmvc', version: springMVCVersion
    compile group: 'org.springframework', name: 'spring-context', version: springContextVersion

    compile group: 'org.apache.logging.log4j', name: 'log4j-api', version: log4jVersion
    compile group: 'org.apache.logging.log4j', name: 'log4j-core', version: log4jVersion

    testCompile group: 'junit', name: 'junit', version: junitVersion
}
```

<!--拓展-->：

可以建立一个单独的文件去设定相关依赖

> 在同级的build.gradle里新建一个xxx.gradle,在build.gradle里引入xxx.gradle

```groovy
// 版本
def versions = [:]
versions.support = "27.0.1"
versions.constraint_layout = "1.0.2"
// 依赖包
def deps = [:]
// Android支持包依赖
def support = [:]
support.annotations = "com.android.support:support-annotations:$versions.support"
deps.support = support
ext.deps = deps
// 编译版本
def build_versions = [:]
build_versions.min_sdk = 14
build_versions.target_sdk = 27
build_versions.build_tools = "27.0.3"
ext.build_versions = build_versions;
```

build.gradle

```groovy
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}
apply from :"config.gradle"


android {

//    compileSdk 32
    compileSdk build_versions.target_sdk
    defaultConfig {
     /*   applicationId "com.example.test_groovy"
        minSdk 21
        targetSdk 32
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"*/

    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {

    implementation 'androidx.core:core-ktx:1.3.2'
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.3.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.4'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}
```

目录结构

```groovy
E:.
│              
├─app
│  │  .gitignore
│  │  build.gradle
│  │  config.gradle
│  │  proguard-rules.pro
│  │  
│  ├─libs
│  └─src                        
└─gradle
    └─wrapper
            gradle-wrapper.jar
            gradle-wrapper.properties
```

#### 2.3.2 依赖的坐标

> 仓库中构件（jar包）的坐标是由configurationName "group:name:version:classifier@extension"组成的字符串构成，如同Maven中的GAV坐标，Gradle可借由此来定位你想搜寻的jar包。

申明方式

```groovy
// 采用map描述依赖
testCompile group: 'junit', name: 'junit', version: '4.0'
// 采用字符串方式描述依赖
testCompile 'junit:junit:4.0'
// 自定义写法
def ver = "4.0"
testCompile "junit:junit:${ver}"
```

| 名称              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| configurationName | 依赖的作用范围                                               |
| group             | 通常用来描述组织、公司、团队或者其它有象征代表意义的名字，比如阿里就是com.alibaba，一个group下一般会有多个artifact |
| name              | 依赖的名称，或者更直接来讲叫包名、模块、构件名、发布物名以及随便你怎么称呼。druid就是com.alibaba下的一个连接池库的名称 |
| version           | 见名知意，无它，版本号。                                     |
| classifier        | 类库版本，在前三项相同的情况下，如果目标依赖还存在对应不同JDK版本的版本，可以通过此属性指明 |
| extension         | 依赖的归档类型，如aar、jar等，默认不指定的话是jar            |

#### 2.3.3 依赖的范围

| 名称            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| compileOnly     | gradle2.12之后版本新添加的,2.12版本时期曾短暂的叫provided,后续版本已经改成了compileOnly,由java插件提供,适用于编译期需要而不需要打包的情况 |
| providedCompile | war插件提供的范围类型:与compile作用类似,但不会被添加到最终的war包中这是由于编译、测试阶段代码需要依赖此类jar包，而运行阶段容器已经提供了相应的支持，所以无需将这些文件打入到war包中了;例如Servlet API就是一个很明显的例子. |
| api             | 3.4以后由java-library提供 当其他模块依赖于此模块时，此模块使用api声明的依赖包是可以被其他模块使用 |
| implementation  | 3.4以后由java-library提供 当其他模块依赖此模块时，此模块使用implementation声明的依赖包只限于模块内部使用，不允许其他模块使用。 |
| compile         | 编译范围依赖在所有的classpath中可用，同时它们也会被打包。    |
| providedRuntime | 同proiveCompile类似。                                        |
| runtime         | runtime依赖在运行和测试系统的时候需要，但在编译的时候不需要。比如，你可能在编译的时候只需要JDBC API JAR，而只有在运行的时候才需要JDBC驱动实现。 |
| testCompile     | 测试期编译需要的附加依赖                                     |
| testRuntime     | 测试运行期需要                                               |
| archives        | -                                                            |
| default         | 配置默认依赖范围                                             |

<font color="red">**1.provided范围内的传递依赖也不会被打包**</font>

<font color="red">**2.api关键字是用来替代compile关键字的，因为compile关键字将来会被弃用。在高版本的gradle，使用compile关键字会报错并提示使用api关键字代替**</font>

<font color="red">**3.在同一个module下，implementation和compile的使用效果相同,不同的module下implementation关键字引用的包对于其他module来说是不可见的**</font>

### 2.4 依赖的分类

| 类型       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| 外部依赖   | 依赖存放于外部仓库中,如`jcenter` ,`mavenCentral`等仓库提供的依赖 |
| 项目依赖   | 依赖于其它项目(模块)的依赖                                   |
| 文件依赖   | 依赖存放在本地文件系统中,基于本地文件系统获取依赖            |
| 内置依赖   | 跟随Gradle发行包或者基于Gradle API的一些依赖,通常在插件开发时使用 |
| 子模块依赖 | 还没搞清楚是什么鬼                                           |

#### 2.4.1 外部依赖
未完待续。。。

