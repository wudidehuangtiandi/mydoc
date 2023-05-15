# ELK的部署及使用



## 一.什么是ELK

*通常，日志被分散的储存不同的设备上。如果你管理数十上百台服务器，你还在使用依次登录每台机器的传统方法查阅日志。这样是不是感觉很繁琐和效率低下。当务之急我们使用集中化的日志管理，例如：开源的syslog，将所有服务器上的日志收集汇总。*

*集中化管理日志后，日志的统计和检索又成为一件比较麻烦的事情，一般我们使用grep、awk和wc等Linux命令能实现检索和统计，但是对于要求更高的查询、排序和统计等要求和庞大的机器数量依然使用这样的方法难免有点力不从心。*

*开源实时日志分析ELK平台能够完美的解决我们上述的问题，ELK由`ElasticSearch`、`Logstash`和`Kiabana`三个开源工具组成。*



`Elasticsearch`是个开源分布式搜索引擎，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载

`Logstash`是一个完全开源的工具，他可以对你的日志进行收集、分析，并将其存储供以后使用（如，搜索）。

`kibana` 也是一个开源和免费的工具，`Kibana`可以为` Logstash` 和` ElasticSearch` 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志。

## 二.部署，配置及接入

> 这里只描述如何再centos里搭建，不接入docker，主要原因是之前部署的时候这些没有放到docker中，以后可以尝试在docker中部署

### 1.Elasticsearch

> `ElasticSearch`是Elastic Stack的核心，同时`Elasticsearch` 是一个分布式、`RESTful`风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。作为`Elastic Stack`的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

[到官网下载](https://www.elastic.co/cn/downloads/elasticsearch)

选择`linux-x86_64.tar.gz`

```
解压到相应目录
tar -zxvf elasticsearch-7.10.2-linux-x86_64.tar.gz -C /usr/local
修改配置
cd /usr/local/elasticsearch-7.10.2/config/
vim elasticsearch.yml
node.name: node-1
path.data: /usr/local/elasticsearch-7.10.2/data
path.logs: /usr/local/elasticsearch-7.10.2/logs
network.host: 127.0.0.1
http.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]
创建es用户 因为ElasticSearch不支持Root用户直接操作，因此我们需要创建一个es用户
useradd es
chown -R es:es /usr/local/elasticsearch-7.10.2
#启动
切换用户成es用户进行操作
su - es
/usr/local/elasticsearch-7.10.2/bin/elasticsearch
后台启动
/usr/local/elasticsearch-7.10.2/bin/elasticsearch -d 
在浏览器打开9200端口地址
```

### 2.Logstash

> `Logstash`是一个开源的服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到最喜欢的存储库中（我们的存储库当然是`ElasticSearch`）

[到官网下载](https://www.elastic.co/cn/downloads/logstash)

同样选择`linux-x86_64.tar.gz`

```
tar -zxvf logstash-7.10.2.tar.gz -C /usr/local
新增配置文件
cd /usr/local/logstash-7.10.2/bin
vim logstash-elasticsearch.conf
input {
	stdin {}
}
output {
	elasticsearch {
		hosts => 'xxx.xxx.xxx.xxx:9200'
	}
	stdout {
		codec => rubydebug
	}
}
#启动
./logstash -f logstash-elasticsearch.conf
```

### 3.Kibana

> Kibana 是一款开源的数据分析和可视化平台，它是 Elastic Stack 成员之一，设计用于和 Elasticsearch 协作。您可以使用 Kibana 对 Elasticsearch 索引中的数据进行搜索、查看、交互操作。您可以很方便的利用图表、表格及地图对数据进行多元化的分析和呈现。

[到官网下载](https://www.elastic.co/cn/downloads/kibana)

同样选择`linux-x86_64.tar.gz`

```
解压到相应目录
tar -zxvf kibana-7.10.2-linux-x86_64.tar.gz -C /usr/local
mv /usr/local/kibana-7.10.2-linux-x86_64 /usr/local/kibana-7.10.2
修改配置
cd /usr/local/kibana-7.10.2/config
vim kibana.yml
server.port: 5601 
server.host: "0.0.0.0" 
elasticsearch.hosts: ["http://xxx.xxx.xxx.xxx:9200"] 
kibana.index: ".kibana"
授权es用户
chown -R es:es /usr/local/kibana-7.10.2/
#启动
切换用户成es用户进行操作
su - es
/usr/local/kibana-7.10.2/bin/kibana 
后台启动
/usr/local/kibana-7.10.2/bin/kibana &

#切换中文
在config/kibana.yml添加
i18n.locale: "zh-CN"

对应服务器安装logstash，配置规则，例如logstash-apache.conf
input {

  tcp {
    host => "127.0.0.1"
    port => "4900"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4901"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4902"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4903"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4904"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4905"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4906"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4907"
    mode => "server"
    codec => json_lines
  }
   tcp {
    host => "127.0.0.1"
    port => "4908"
    mode => "server"
    codec => json_lines
  }

 beats {
   host => "127.0.0.1"
   port => "5044"
   add_field => {"appname" => "self-help"} 
 }
}

filter {
  #Only matched data are send to output. 
}

output {

  # For detail config for elasticsearch as output,
  # See: https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html

  # stdout {
  #  codec => rubydebug
  # }

   elasticsearch {
      action => "index"          #The operation on ES
      hosts  => "http://127.0.0.1:9200"   #ElasticSearch host, can be array.
      index => "log_%{[appname]}-%{+YYYY.MM.dd}"
   }
  
}

启动logstash
./logstash -f logstash-apache.conf
通过kibana可视化检索各个服务日志数据

java程序需要添加如下依赖
       <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>5.2</version>
        </dependency>
对应的log4j配置文件中需要添加appender
 <appender name="stash"
              class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>xxx.xxx.xxx.xxx:4907</destination>
        <!-- encoder必须配置,有多种可选 -->
        <encoder charset="UTF-8"
                 class="net.logstash.logback.encoder.LogstashEncoder" >
            <!-- "appname":"ye_test" 的作用是指定创建索引的名字时用，并且在生成的文档中会多了这个字段  -->
            <customFields>{"appname":"xxxx"}</customFields>
        </encoder>
    </appender>
       <root level="info">
        <appender-ref ref="stash"/>
    </root>

对应的docker配置参见docker文档
```



!>本文档可能有一定的局限性,仅用于本公司的医废项目中，以后等待docker运用部署后再进行更新

> 时间问题，在设置，高级设置中有dateFormat:tz 选项默认时浏览器时间，并在日志时间上加上这个时区，比如东8区，将在时间上加8小时再展示。而elasticsearch中的日志时间已经经过了时区处理，因此这里的默认处理方式，对以UTC时间记录的日志是正确的，而以本地时区时间记录的日志处理是错误的。将**dateFormat:tz设置改为UTC时区即可解决问题**



下面贴一下索引如何创建



![avatar](https://picture.zhanghong110.top/docsify/162934116025871.png)

![avatar](https://picture.zhanghong110.top/docsify/162934116025872.png)

![avatar](https://picture.zhanghong110.top/docsify/162934116025873.png)

![avatar](https://picture.zhanghong110.top/docsify/162934116025874.png)

通过以上方式就可以实现日志索引的发布，就可以在外部查询到日志信息。

## 三.docker 部署elk,暴露外部端口及整合elastalert实现企业微信机器人推送

本次的部署建立在没有固定IP及域名的情况下,采用docker compose 搭建,我们在`home/elk`目录下建立如下文件

1.docker-compose.yml

```yaml
version: "3"
services:
  es:
    image: elasticsearch:7.6.1
    labels:
      co.elastic.logs/enabled: "false"
    hostname: docker-es
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    volumes:
      - ./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./data/elasticsearch:/usr/share/elasticsearch/data

  kibana:
    image: kibana:7.6.1
    labels:
      co.elastic.logs/enabled: "false"
    hostname: docker-kibana
    ports:
      - "5601:5601"
    volumes:
      - ./kibana.yml:/usr/share/kibana/config/kibana.yml
    depends_on:
      - es

  logstash:
    image: logstash:7.6.1
    hostname: docker-logstash
    ports:
      - "5044:5044"
      - "5045:5045"
      - "9600:9600"
    volumes:
      - ./logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./data/logs:/logs
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "LS_OPTS=--config.reload.automatic"
    depends_on:
      - es

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.6.1
    labels:
      co.elastic.logs/enabled: "false"
    user: root
    hostname: docker-filebeat
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - "/var/lib/docker/containers:/var/lib/docker/containers:ro"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    depends_on:
      - es
```

2.elastalert.yaml (单独用，不做用于dockercompose)

```yaml
rules_folder: /opt/elastalert/rules

run_every:
  seconds: 10

buffer_time:
  minutes: 15

es_host: 192.168.200.13
es_port: 9200

es_username: elastic
es_password: 密码

writeback_index: elastalert_status

alert_time_limit:
  days: 2
```

3.elasticsearch.yml

```yaml
network.host: 0.0.0.0
http.host: 0.0.0.0
http.port: 9200
transport.port: 9300
http.cors.enabled: true
http.cors.allow-origin: "*"
xpack.security.enabled: true
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
```

4.filebeat.yml

```yaml
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

processors:
  - add_cloud_metadata: ~

output.elasticsearch:
  hosts: '192.168.200.13:9200'
  username: 'beats_system'
  password: '密码'
```

5.kibana.yml

```yaml 
server:
  host: "0.0.0.0"
  port: 5601

# ES
elasticsearch:
  # 这里替换成自己服务器的IP，不能用127.0.0.1和0.0.0.0
  hosts: ["http://192.168.200.13:9200"]
  username: "kibana"
  password: "密码"

# 汉化
i18n.locale: "zh-CN"
```

6.logstash.conf

```yaml
input {
        http {
      host => "0.0.0.0"
      port => 5045
      additional_codecs => {"application/json"=>"json"}
      codec => "plain"
      threads => 4
      ssl => false
	}
}

output {
    elasticsearch {
        hosts => [ "http://192.168.200.13:9200" ]
        user => "elastic"
        password => "密码"
        index => "log_%{[appname]}-%{+YYYY.MM.dd}"
    }
    stdout {
        codec => rubydebug                      # 输出到命令窗口
    }
}
```

8.logstash.yml

```yaml
http.host: "0.0.0.0"
path.config: /usr/share/logstash/pipeline/*.conf
xpack.monitoring.enabled: false
```

9.建立在home/elk下建立data目录,建立elasticsearch文件夹，logs文件夹,password.txt

10.建立rules文件夹，建立wechat.yaml（与dockercompose无关）

```yaml
name: lo   #告警模板名
realert:              #2分钟内不重复告警
  minutes: 1
type: frequency
index: lo*    #要查询的索引的名称, ES中存在的索引
num_events: 1         #此参数特定于frequency类型，而且是触发警报时的阈值，周期内出现5次
timeframe:            #监控周期为1分钟
  minutes: 1
filter:               #配置监控条件
- query:
      query_string:
        query:  'error'

alert: post2
http_post2_url: "微信机器人的链接，需要替换"
http_post2_payload:
  {
    "msgtype": "markdown",
    "markdown": {
       "content":"
         报错应用:    <font color=\"comment\">{{appname}}</font> \n
         报错详情:    <font color=\"comment\">{{message}}</font> 
"
    }
  }
  
http_post2_headers:
  Content-Type: application/json
```

> elk所需的结构如下

- elk
  - docker-compose.yml
  - elasticsearch.yml
  - kibana.yml
  - logstash.yml
  - logstash.conf
  - filebeat.yml
  - data/
    - elasticsearch/
    - logs/
    - password.txt

data/elasticsearch/ 文件夹是用来保存elasticsearch数据用的
data/password.txt 文件是用来保存密码的，如果记得住密码，请忽略
data/logs 需要采集的日志的目录，也就是你应用写日志的位置，正常情况不会这么干，这里只是用来测试效果

!> 注意,上面的es地址同kibana.yml，这里替换成自己服务器的IP，不能用127.0.0.1和0.0.0.0

需要的基本上都搞好了，可以开始运行了，启动命令：

到home/elk目录下运行

```shell
docker-compose up -d
```

如果容器启动失败，有可能是这几个yml的文件的权限不够，把这几权限赋予755，切记这里的logstash日志输入输出配置文件,一定要授权755权限，不能777或其他，如不授权ELK读取不了此文件，注意，data目录下的文件夹也要给与777权限，否则可能会导致elasticsearch启动失败。

```shell
chmod -R 777 xxx
```

> 我们到这边会发现，密码什么的明明都没有，我们运行起来后需要使用入地下命令生成下密码

```shell
docker-compose exec -T es elasticsearch-setup-passwords auto --batch
```

完事之后会给出密码

```shell
Changed password for user apm_system
PASSWORD apm_system = xxxx

Changed password for user kibana
PASSWORD kibana = xxxx

Changed password for user logstash_system
PASSWORD logstash_system = xxxx

Changed password for user beats_system
PASSWORD beats_system = xxxx

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = xxxx

Changed password for user elastic
PASSWORD elastic = xxxx
```

我们记录下保存到password.txt中备忘

然后重新修改配置文件， **`kibana.yml`**, **`logstash.conf`**, **`filebeat.yml`**等等的配置文件，凡是需要使用账号密码的都要改，注意，elastic为超管账号，如果发现有的账号权限有问题，就用这个号。完事之后重启

```shell 
docker-compose down
docker-compose up -d
```

此时我们访问对应的5601端口，出现欢界面就成功了。

下面，应为我们没有固定IP（可以看到我们Logstash使用的是http而非tcp)，又需要整合外部服务的日志，所以我们需要代理出logstash的5045端口，如上面所示，我简单贴下我的nginx代理信息

```nginx

server{
    listen 9001;
    server_name xxxx.xxx.xxx;
    location / {
               proxy_set_header Host $http_host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header REMOTE-HOST $remote_addr;
	                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                                proxy_set_header Upgrade $http_upgrade;
                               proxy_set_header Connection "Upgrade";
                proxy_pass http://192.168.200.13:5045;
    }
}
```

然后我们对应的服务需要使用http来暴露，原先的java依赖logstash-logback-encoder只支持TCP所以不行，我们换成如下依赖

```xml
       <dependency>
            <groupId>org.logback-extensions</groupId>
            <artifactId>logback-ext-loggly</artifactId>
            <version>0.1.5</version>
        </dependency>
        <!--log to json-->
        <dependency>
            <groupId>ch.qos.logback.contrib</groupId>
            <artifactId>logback-jackson</artifactId>
            <version>0.1.5</version>
        </dependency>
        <!--log to json-->
        <dependency>
            <groupId>ch.qos.logback.contrib</groupId>
            <artifactId>logback-json-classic</artifactId>
            <version>0.1.5</version>
        </dependency>
```

完事之后在logbackx.xml中添加如下appender即可,这边建议使用LogglyBatchAppender,如果使用LogglyAppender会导致项目响应缓慢

```xml
    <appender name="stash" class="ch.qos.logback.ext.loggly.LogglyBatchAppender">
        <!--请求的地址-->
        <endpointUrl>http://xxx.xxx.xxx:9001</endpointUrl>
        <!--定义输出格式JSON-->
        <layout class="com.cgsm.framework.log.MyJsonLayOut">
            <name>yfsq</name>
            <jsonFormatter
                    class="ch.qos.logback.contrib.jackson.JacksonJsonFormatter">
                <prettyPrint>true</prettyPrint>
            </jsonFormatter>
        </layout>
    </appender>
        <root level="info">
        <appender-ref ref="stash"/>
    </root>
```

这边我们需要自定义`MyJsonLayOut`用来增加服务`appname`。原生的是不支的，代码如下

```Java
package com.cgsm.framework.log;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.contrib.json.classic.JsonLayout;
import org.springframework.stereotype.Component;

import java.util.LinkedHashMap;
import java.util.Map;

public class MyJsonLayOut extends JsonLayout {

    public static final String APP_NAME = "appname";

    private String name;

    public MyJsonLayOut(){
        super();
    }

    public MyJsonLayOut(String name){
        super();
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    protected Map toJsonMap(ILoggingEvent event) {
        Map<String, Object> map = new LinkedHashMap<String, Object>();
        addTimestamp(TIMESTAMP_ATTR_NAME, this.includeTimestamp, event.getTimeStamp(), map);
        add(APP_NAME,true,name,map);
        add(LEVEL_ATTR_NAME, this.includeLevel, String.valueOf(event.getLevel()), map);
        add(THREAD_ATTR_NAME, this.includeThreadName, event.getThreadName(), map);
        addMap(MDC_ATTR_NAME, this.includeMDC, event.getMDCPropertyMap(), map);
        add(LOGGER_ATTR_NAME, this.includeLoggerName, event.getLoggerName(), map);
        add(FORMATTED_MESSAGE_ATTR_NAME, this.includeFormattedMessage, event.getFormattedMessage(), map);
        add(MESSAGE_ATTR_NAME, this.includeMessage, event.getMessage(), map);
        add(CONTEXT_ATTR_NAME, this.includeContextName, event.getLoggerContextVO().getName(), map);
        addThrowableInfo(EXCEPTION_ATTR_NAME, this.includeException, event, map);
        addCustomDataToJsonMap(map, event);
        return map;
    }
}
```

这样定义就完事了。

最后我们需要启动`elastalert`,我们这里使用的是elastalert2，镜像为`jertel/elastalert2`,1代没法支持企业微信机器人（如果没有固定IP及域名，采用企业微信应用推送是不可行的，智能用机器人推送），它依赖一个yaml及rules中的规则，两个配置都在上面贴出了，我们在企业微信群里增加个机器人，将其链接配置到我们的yaml文件中。

```yaml
docker run -d \
    -v /home/elk/elastalert.yaml:/opt/elastalert/config.yaml \
    -v /home/elk/rules:/opt/elastalert/rules \
    --name elastalert jertel/elastalert2:latest
```

启动之后即可。我们尝试增加一个error。可以发现企业微信机器人推送成功了。推送内容如下

```markdown
 报错应用:    yfsq 
 报错详情:    Validation failed for argument [0] in public com.cgsm.common.core.domain.AjaxResult com.cgsm.web.controller.system.SysLoginController.login(com.cgsm.common.core.domain.dto.wx.WxAppLoginBody): [Field error in object 'wxAppLoginBody' on field 'username': rejected value [null]; codes [NotBlank.wxAppLoginBody.username,NotBlank.username,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [wxAppLoginBody.username,username]; arguments []; default message [username]]; default message [用户名]]  
```

