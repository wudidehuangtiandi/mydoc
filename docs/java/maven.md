# MAVEN常用操作及问题解决

## 1.MAVEN的下载及安装

[官网下载地址](https://maven.apache.org/download.cgi)

我们直接下zip版本的二进制压缩包，解压即可

![avatar](https://picture.zhanghong110.top/docsify/16526815177440.png)

解压后增加环境变量即可

![avatar](https://picture.zhanghong110.top/docsify/16526830153117.png)

接下去打开根目录下conf下配置文件`settings.xml`修改下本地仓库地址，源（例子采用阿里的，有公司镜像的可以改成公司的）及默认环境配置

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

<!--
 | This is the configuration file for Maven. It can be specified at two levels:
 |
 |  1. User Level. This settings.xml file provides configuration for a single user,
 |                 and is normally provided in ${user.home}/.m2/settings.xml.
 |
 |                 NOTE: This location can be overridden with the CLI option:
 |
 |                 -s /path/to/user/settings.xml
 |
 |  2. Global Level. This settings.xml file provides configuration for all Maven
 |                 users on a machine (assuming they're all using the same Maven
 |                 installation). It's normally provided in
 |                 ${maven.conf}/settings.xml.
 |
 |                 NOTE: This location can be overridden with the CLI option:
 |
 |                 -gs /path/to/global/settings.xml
 |
 | The sections in this sample file are intended to give you a running start at
 | getting the most out of your Maven installation. Where appropriate, the default
 | values (values used when the setting is not specified) are provided.
 |
 |-->
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 https://maven.apache.org/xsd/settings-1.2.0.xsd">
  <!-- localRepository
   | The path to the local repository maven will use to store artifacts.
   |
   | Default: ${user.home}/.m2/repository
  <localRepository>/path/to/local/repo</localRepository>
  -->
   <localRepository>E:\mavenrepository\maven_repository</localRepository>
  <!-- interactiveMode
   | This will determine whether maven prompts you when it needs input. If set to false,
   | maven will use a sensible default value, perhaps based on some other setting, for
   | the parameter in question.
   |
   | Default: true
  <interactiveMode>true</interactiveMode>
  -->

  <!-- offline
   | Determines whether maven should attempt to connect to the network when executing a build.
   | This will have an effect on artifact downloads, artifact deployment, and others.
   |
   | Default: false
  <offline>false</offline>
  -->

  <!-- pluginGroups
   | This is a list of additional group identifiers that will be searched when resolving plugins by their prefix, i.e.
   | when invoking a command line like "mvn prefix:goal". Maven will automatically add the group identifiers
   | "org.apache.maven.plugins" and "org.codehaus.mojo" if these are not already contained in the list.
   |-->
  <pluginGroups>
    <!-- pluginGroup
     | Specifies a further group identifier to use for plugin lookup.
    <pluginGroup>com.your.plugins</pluginGroup>
    -->
  </pluginGroups>

  <!-- proxies
   | This is a list of proxies which can be used on this machine to connect to the network.
   | Unless otherwise specified (by system property or command-line switch), the first proxy
   | specification in this list marked as active will be used.
   |-->
  <proxies>
    <!-- proxy
     | Specification for one proxy, to be used in connecting to the network.
     |
    <proxy>
      <id>optional</id>
      <active>true</active>
      <protocol>http</protocol>
      <username>proxyuser</username>
      <password>proxypass</password>
      <host>proxy.host.net</host>
      <port>80</port>
      <nonProxyHosts>local.net|some.host.com</nonProxyHosts>
    </proxy>
    -->
  </proxies>

  <!-- servers
   | This is a list of authentication profiles, keyed by the server-id used within the system.
   | Authentication profiles can be used whenever maven must make a connection to a remote server.
   |-->
  <servers>
    <!-- server
     | Specifies the authentication information to use when connecting to a particular server, identified by
     | a unique name within the system (referred to by the 'id' attribute below).
     |
     | NOTE: You should either specify username/password OR privateKey/passphrase, since these pairings are
     |       used together.
     |
    <server>
      <id>deploymentRepo</id>
      <username>repouser</username>
      <password>repopwd</password>
    </server>
    -->

    <!-- Another sample, using keys to authenticate.
    <server>
      <id>siteServer</id>
      <privateKey>/path/to/private/key</privateKey>
      <passphrase>optional; leave empty if not used.</passphrase>
    </server>
    -->
  </servers>

  <!-- mirrors
   | This is a list of mirrors to be used in downloading artifacts from remote repositories.
   |
   | It works like this: a POM may declare a repository to use in resolving certain artifacts.
   | However, this repository may have problems with heavy traffic at times, so people have mirrored
   | it to several places.
   |
   | That repository definition will have a unique id, so we can create a mirror reference for that
   | repository, to be used as an alternate download site. The mirror site will be the preferred
   | server for that repository.
   |-->
  <mirrors>
    <!-- mirror
     | Specifies a repository mirror site to use instead of a given repository. The repository that
     | this mirror serves has an ID that matches the mirrorOf element of this mirror. IDs are used
     | for inheritance and direct lookup purposes, and must be unique across the set of mirrors.
     |
    <mirror>
      <id>mirrorId</id>
      <mirrorOf>repositoryId</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://my.repository.com/repo/path</url>
    </mirror>
     -->
    <mirror>
      <id>maven-default-http-blocker</id>
      <mirrorOf>external:http:*</mirrorOf>
      <name>Pseudo repository to mirror external repositories initially using HTTP.</name>
      <url>http://0.0.0.0/</url>
      <blocked>true</blocked>
    </mirror>
	
	<mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>       
    </mirror>
  </mirrors>

  <!-- profiles
   | This is a list of profiles which can be activated in a variety of ways, and which can modify
   | the build process. Profiles provided in the settings.xml are intended to provide local machine-
   | specific paths and repository locations which allow the build to work in the local environment.
   |
   | For example, if you have an integration testing plugin - like cactus - that needs to know where
   | your Tomcat instance is installed, you can provide a variable here such that the variable is
   | dereferenced during the build process to configure the cactus plugin.
   |
   | As noted above, profiles can be activated in a variety of ways. One way - the activeProfiles
   | section of this document (settings.xml) - will be discussed later. Another way essentially
   | relies on the detection of a system property, either matching a particular value for the property,
   | or merely testing its existence. Profiles can also be activated by JDK version prefix, where a
   | value of '1.4' might activate a profile when the build is executed on a JDK version of '1.4.2_07'.
   | Finally, the list of active profiles can be specified directly from the command line.
   |
   | NOTE: For profiles defined in the settings.xml, you are restricted to specifying only artifact
   |       repositories, plugin repositories, and free-form properties to be used as configuration
   |       variables for plugins in the POM.
   |
   |-->
  <profiles>
        <profile>
            <id>jdk1.8</id>
            <activation>
                <activeByDefault>true</activeByDefault>
                <jdk>1.8</jdk>
            </activation>
            <properties>
                <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
                <maven.compiler.source>1.8</maven.compiler.source>
                <maven.compiler.target>1.8</maven.compiler.target>
                <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
            </properties>
        </profile>
    <!-- profile
     | Specifies a set of introductions to the build process, to be activated using one or more of the
     | mechanisms described above. For inheritance purposes, and to activate profiles via <activatedProfiles/>
     | or the command line, profiles have to have an ID that is unique.
     |
     | An encouraged best practice for profile identification is to use a consistent naming convention
     | for profiles, such as 'env-dev', 'env-test', 'env-production', 'user-jdcasey', 'user-brett', etc.
     | This will make it more intuitive to understand what the set of introduced profiles is attempting
     | to accomplish, particularly when you only have a list of profile id's for debug.
     |
     | This profile example uses the JDK version to trigger activation, and provides a JDK-specific repo.
    <profile>
      <id>jdk-1.4</id>

      <activation>
        <jdk>1.4</jdk>
      </activation>

      <repositories>
        <repository>
          <id>jdk14</id>
          <name>Repository for JDK 1.4 builds</name>
          <url>http://www.myhost.com/maven/jdk14</url>
          <layout>default</layout>
          <snapshotPolicy>always</snapshotPolicy>
        </repository>
      </repositories>
    </profile>
    -->

    <!--
     | Here is another profile, activated by the system property 'target-env' with a value of 'dev',
     | which provides a specific path to the Tomcat instance. To use this, your plugin configuration
     | might hypothetically look like:
     |
     | ...
     | <plugin>
     |   <groupId>org.myco.myplugins</groupId>
     |   <artifactId>myplugin</artifactId>
     |
     |   <configuration>
     |     <tomcatLocation>${tomcatPath}</tomcatLocation>
     |   </configuration>
     | </plugin>
     | ...
     |
     | NOTE: If you just wanted to inject this configuration whenever someone set 'target-env' to
     |       anything, you could just leave off the <value/> inside the activation-property.
     |
    <profile>
      <id>env-dev</id>

      <activation>
        <property>
          <name>target-env</name>
          <value>dev</value>
        </property>
      </activation>

      <properties>
        <tomcatPath>/path/to/tomcat/instance</tomcatPath>
      </properties>
    </profile>
    -->
  </profiles>

  <!-- activeProfiles
   | List of profiles that are active for all builds.
   |
  <activeProfiles>
    <activeProfile>alwaysActiveProfile</activeProfile>
    <activeProfile>anotherAlwaysActiveProfile</activeProfile>
  </activeProfiles>
  -->
</settings>
```

!> 注意maven 3.8.5对ideal 2021.3及以下版本不适配，新版的没试过。因为版本过高我们回退下到3.6.3

> ideal集成maven如下图

![avatar](https://picture.zhanghong110.top/docsify/16526881711904.png)

## 2.常见问题的处理及一些概念及经验

> 1.MAVEN一直处于loading无法成功引入依赖问题

我们经常会碰到配置了可以链接的源`ideal`却一直处于加载状态，我们通常会认为是网络问题，其实增加maven堆栈内存即可顺利下载，如下图所示。

![avatar](https://picture.zhanghong110.top/docsify/1652688551703.png)



> 2.自定义插件找不到

表现为`Cannot resolve plugin org.apache.maven.plugins:maven-compiler-plugin:<unknown> `此类异常。我们有时候需要指定打包JDK版本或者跳过测试内容时常常需要指定一些自定义的插件。症状如下图所示，我们的`pom`采用了引用`spring-boot-dependencies`约束版本而不是继承`spring-boot-dependencies`。

![avatar](https://picture.zhanghong110.top/docsify/1652690253279.png)

问题分析：我们通过查阅spring官方文档可知（[地址](https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/htmlsingle/)），我们有两种方式去管理spring依赖。继承spring-boot-starter-parent POM。也可能有自己的企业标准parent。如果你不想使用spring-boot-starter-parent，你依然可以通过使用spring-boot-dependencies的scope=import利用依赖管理的便利。根据实验这两种方式都可以引入插件。

当我们的项目是个空pom时可以发现插件依然存在默认版本如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
<!--    <parent>-->
<!--        <groupId>org.springframework.boot</groupId>-->
<!--        <artifactId>spring-boot-starter-parent</artifactId>-->
<!--        <version>2.6.7</version>-->
<!--        <relativePath/> &lt;!&ndash; lookup parent from repository &ndash;&gt;-->
<!--    </parent>-->
    <groupId>com.example</groupId>
    <artifactId>demo1</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo1</name>
    <description>demo1</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
<!--        <dependency>-->
<!--            <groupId>org.springframework.boot</groupId>-->
<!--            <artifactId>spring-boot-starter</artifactId>-->
<!--        </dependency>-->

<!--        <dependency>-->
<!--            <groupId>org.springframework.boot</groupId>-->
<!--            <artifactId>spring-boot-starter-test</artifactId>-->
<!--            <scope>test</scope>-->
<!--        </dependency>-->
    </dependencies>

    <build>
        <plugins>
<!--            <plugin>-->
<!--                <groupId>org.springframework.boot</groupId>-->
<!--                <artifactId>spring-boot-maven-plugin</artifactId>-->
<!--            </plugin>-->
        </plugins>
    </build>

</project>
```

![avatar](https://picture.zhanghong110.top/docsify/16529388956358.png)

当我们放开配置时

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.6.7</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>demo1</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo1</name>
	<description>demo1</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>

```

![avatar](https://picture.zhanghong110.top/docsify/16529397056179.png)

当我们改成直接引用`spring-boot-dependencies`如下


```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
<!--    <parent>-->
<!--        <groupId>org.springframework.boot</groupId>-->
<!--        <artifactId>spring-boot-starter-parent</artifactId>-->
<!--        <version>2.6.7</version>-->
<!--        <relativePath/> &lt;!&ndash; lookup parent from repository &ndash;&gt;-->
<!--    </parent>-->
    <groupId>com.example</groupId>
    <artifactId>demo1</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo1</name>
    <description>demo1</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <!-- Import dependency management from Spring Boot -->
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.6.7</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

![avatar](https://picture.zhanghong110.top/docsify/1652941517357.png)

通过对比可以发现，当直接引用`spring-boot-dependencies`时，版本并不会继承。通过查询spring官网发现有如下语句

If you do not want to use the `spring-boot-starter-parent`, you can still keep the benefit of the dependency management (but not the plugin management) by using an `import` scoped dependency

可见括号中排除了插件版本。由此我们可以猜测，当直接引用`spring-boot-dependencies`时需要增加版本号即可解决问题。

问题解决：增加<version>3.8.1</version>版本号即可

扩展：我们有此处发现，maven带有默认的插件配置，这个源自于哪里呢。

我们来到MAVEN根目录下的lib包中，其中有一个名为`maven-core-3.6.3.jar`的Jar包，我们用压缩工具打开后来到`META-INF/plexus/components.xml`这个xml下，如下（太长我们展示一部分）

```xml
<component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>pom</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>

    
    <component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>jar</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <process-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:resources
              </process-resources>
              <compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:compile
              </compile>
              <process-test-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:testResources
              </process-test-resources>
              <test-compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:testCompile
              </test-compile>
              <test>
                org.apache.maven.plugins:maven-surefire-plugin:2.12.4:test
              </test>
              <package>
                org.apache.maven.plugins:maven-jar-plugin:2.4:jar
              </package>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>

    
    <component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>ejb</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <process-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:resources
              </process-resources>
              <compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:compile
              </compile>
              <process-test-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:testResources
              </process-test-resources>
              <test-compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:testCompile
              </test-compile>
              <test>
                org.apache.maven.plugins:maven-surefire-plugin:2.12.4:test
              </test>
              <package>
                org.apache.maven.plugins:maven-ejb-plugin:2.3:ejb
              </package>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>

    
    <component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>maven-plugin</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <process-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:resources
              </process-resources>
              <compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:compile
              </compile>
              <process-classes>
                org.apache.maven.plugins:maven-plugin-plugin:3.2:descriptor
              </process-classes>
              <process-test-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:testResources
              </process-test-resources>
              <test-compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:testCompile
              </test-compile>
              <test>
                org.apache.maven.plugins:maven-surefire-plugin:2.12.4:test
              </test>
              <package>
                org.apache.maven.plugins:maven-jar-plugin:2.4:jar,
                org.apache.maven.plugins:maven-plugin-plugin:3.2:addPluginArtifactMetadata
              </package>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>

    
    <component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>war</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <process-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:resources
              </process-resources>
              <compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:compile
              </compile>
              <process-test-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:testResources
              </process-test-resources>
              <test-compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:testCompile
              </test-compile>
              <test>
                org.apache.maven.plugins:maven-surefire-plugin:2.12.4:test
              </test>
              <package>
                org.apache.maven.plugins:maven-war-plugin:2.2:war
              </package>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>

    
    <component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>ear</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <generate-resources>
                org.apache.maven.plugins:maven-ear-plugin:2.8:generate-application-xml
              </generate-resources>
              <process-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:resources
              </process-resources>
              <package>
                org.apache.maven.plugins:maven-ear-plugin:2.8:ear
              </package>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>

    
    <component>
      <role>org.apache.maven.lifecycle.mapping.LifecycleMapping</role>
      <role-hint>rar</role-hint>
      <implementation>org.apache.maven.lifecycle.mapping.DefaultLifecycleMapping</implementation>
      <configuration>
        <lifecycles>
          <lifecycle>
            <id>default</id>
            
            <phases>
              <process-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:resources
              </process-resources>
              <compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:compile
              </compile>
              <process-test-resources>
                org.apache.maven.plugins:maven-resources-plugin:2.6:testResources
              </process-test-resources>
              <test-compile>
                org.apache.maven.plugins:maven-compiler-plugin:3.1:testCompile
              </test-compile>
              <test>
                org.apache.maven.plugins:maven-surefire-plugin:2.12.4:test
              </test>
              <package>
                org.apache.maven.plugins:maven-rar-plugin:2.2:rar
              </package>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
            </phases>
            
          </lifecycle>
        </lifecycles>
      </configuration>
    </component>
```

这边可以发现其默认定义的版本。

思考：那么理论上讲，就算我们的项目这么定义应该也不至于报`Cannot resolve plugin org.apache.maven.plugins:maven-compiler-plugin:<unknown> `这种问题，应该会采用默认的工具才对。这里就要注意下我们打的包了，我们可以看见我们根节点是POM，所以应该是

```xml
 <phases>
              <install>
                org.apache.maven.plugins:maven-install-plugin:2.4:install
              </install>
              <deploy>
                org.apache.maven.plugins:maven-deploy-plugin:2.7:deploy
              </deploy>
</phases>
```

这两加一个`DefaultLifecycleMapping`,实际引入后我们发现

![avatar](https://picture.zhanghong110.top/docsify/16529466836950.png)

猜测`DefaultLifecycleMapping`包含了`clean`和`site`两大生命周期，实际，切换打包方式比如用`jar`包的方式也是这样，可以证实。

三大生命周期我们可以在`components.xml`下面看到。

```xml
<component>
      <role>org.apache.maven.lifecycle.Lifecycle</role>
      <implementation>org.apache.maven.lifecycle.Lifecycle</implementation>
      <role-hint>default</role-hint>
      <configuration>
        <id>default</id>
        
        <phases>
          <phase>validate</phase>
          <phase>initialize</phase>
          <phase>generate-sources</phase>
          <phase>process-sources</phase>
          <phase>generate-resources</phase>
          <phase>process-resources</phase>
          <phase>compile</phase>
          <phase>process-classes</phase>
          <phase>generate-test-sources</phase>
          <phase>process-test-sources</phase>
          <phase>generate-test-resources</phase>
          <phase>process-test-resources</phase>
          <phase>test-compile</phase>
          <phase>process-test-classes</phase>
          <phase>test</phase>
          <phase>prepare-package</phase>
          <phase>package</phase>
          <phase>pre-integration-test</phase>
          <phase>integration-test</phase>
          <phase>post-integration-test</phase>
          <phase>verify</phase>
          <phase>install</phase>
          <phase>deploy</phase>
        </phases>
        
      </configuration>
    </component><component>
      <role>org.apache.maven.lifecycle.Lifecycle</role>
      <implementation>org.apache.maven.lifecycle.Lifecycle</implementation>
      <role-hint>clean</role-hint>
      <configuration>
        <id>clean</id>
        
        <phases>
          <phase>pre-clean</phase>
          <phase>clean</phase>
          <phase>post-clean</phase>
        </phases>
        <default-phases>
          <clean>
            org.apache.maven.plugins:maven-clean-plugin:2.5:clean
          </clean>
        </default-phases>
        
      </configuration>
    </component><component>
      <role>org.apache.maven.lifecycle.Lifecycle</role>
      <implementation>org.apache.maven.lifecycle.Lifecycle</implementation>
      <role-hint>site</role-hint>
      <configuration>
        <id>site</id>
        
        <phases>
          <phase>pre-site</phase>
          <phase>site</phase>
          <phase>post-site</phase>
          <phase>site-deploy</phase>
        </phases>
        <default-phases>
          <site>
            org.apache.maven.plugins:maven-site-plugin:3.3:site
          </site>
          <site-deploy>
            org.apache.maven.plugins:maven-site-plugin:3.3:deploy
          </site-deploy>
        </default-phases>
        
      </configuration>
```

这样。就能解释了，为啥我们使用下面这样的配置报`Unresolved plugin: 'org.apache.maven.plugins:maven-compiler-plugin:<unknown>'`。首先是应为采用了`spring-boot-dependencies`没有`compiler`插件。另一个是`pom`的原因本来就是不带`compiler`插件，故而实际就没有了，指定后需要指定一个版本。

```xml
<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>

                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

