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

对应的docker配置参见docker文档
```



!>本文档可能有一定的局限性,仅用于本公司的医废项目中，以后等待docker运用部署后再进行更新

> 时间问题，在设置，高级设置中有dateFormat:tz 选项默认时浏览器时间，并在日志时间上加上这个时区，比如东8区，将在时间上加8小时再展示。而elasticsearch中的日志时间已经经过了时区处理，因此这里的默认处理方式，对以UTC时间记录的日志是正确的，而以本地时区时间记录的日志处理是错误的。将**dateFormat:tz设置改为UTC时区即可解决问题**



下面贴一下索引如何创建



![avatar](https://picture.zhanghong110.top/docsify/162934116025871.png)

![avatar](https://picture.zhanghong110.top/docsify/162934116025872.png)

![avatar](https://picture.zhanghong110.top/docsify/162934116025873.png)

![avatar](https://picture.zhanghong110.top/docsify/162934116025874.png)

通过以上几个方式就可以实现日志索引的发布，就可以在外部查询到日志信息。