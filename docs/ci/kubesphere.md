

# KubeSphere 

## 一.集群的配置及安装

>Kubernetes 已经成为了容器编排的实施标准，而以 Kubernetes 为核心的云原生技术以及生态正在快速和蓬勃地发展。然而，仅 Kubernetes 本身就有复杂的架构和很高的学习成本，包括集群的安装运维、存储、网络、可观测性DevOps、应用管理、多租户等等。而为了解决这一系列难题，KubeSphere 应运而生。
>
>KubeSphere 愿景是打造一个以 Kubernetes 为内核的云原生分布式操作系统，它的架构可以非常方便地使第三方应用与云原生生态组件进行即插即用（plug-and-play）的集成，支持云原生应用在多云与多集群的统一分发和运维管理。

kubersphere有几种安装方式，在我们拥有一个集群的情况下可以使用Kubernetes,如果是纯净的linux系统则可以使用后者。[官网链接](https://kubesphere.com.cn/)

![avatar](https://picture.zhanghong110.top/docsify/16533716019918.png)

> 我们这边采用后者，前者的话可以参考本文档的K8S集群部署部分以及官网的使用Kubernetes安装部分。
>
> 后者的话使用一些简单命令（ KubeKey）即可实现在 Linux 上以 All-in-One 模式安装 KubeSphere对于刚接触 KubeSphere 并想快速上手该[容器平台](https://kubesphere.io/)的用户，All-in-One 安装模式是最佳的选择，它能够帮助您零配置快速部署 KubeSphere 和 Kubernetes。后期可以通过图形化界面来增加插件。
>
> 本次我们介绍里面的[集群安装方式](https://kubesphere.com.cn/docs/installing-on-linux/introduction/multioverview/)，单点的也可以参照官网的学习视屏。



**目标：使用KubeKey安装一个K8S集群**

**准备工作：一台master（4c8g）,两台worker(8c16g) ,centos7 **

```shell
#第一步修改三台主机的hostname,分别为master node1 node2,修改完后重新连接
hostnamectl set-hostname master

#第二部使用KubeKey创建集群（以下命令未作说明均在master）
#如果是国内连不上github则需要
export KKZONE=cn
#然后找个目录，我们使用/home,执行以下命令下载KubeKey
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.0.0 sh -
#为kk添加可执行权限
chmod +x kk

#第三步创建配置文件./kk create config [--with-kubernetes version] [--with-kubesphere version] [(-f | --file) path],版本么自己去看下对应的，我这边采用这个版本，然后将会获得一个配置文件。
./kk create config --with-kubernetes v1.21.5 --with-kubesphere v3.2.1
```

> 下面我们修改下这个配置文件，主要改下hosts和执行下控制节点和工作节点，其它都可以保持默认。

```yaml
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  #主要改下这里
  - {name: master, address: 172.16.0.1, internalAddress: 172.16.0.1, user: ubuntu, password: "Qcloud@123"}
  - {name: node1, address: 172.16.0.2, internalAddress: 172.16.0.2, user: ubuntu, password: "Qcloud@123"}
  - {name: node2, address: 172.16.0.3, internalAddress: 172.16.0.3, user: ubuntu, password: "Qcloud@123"}
  roleGroups:
  #第二个要改的地方
    etcd:
    - master
    control-plane: 
    - master
    worker:
    - node1
    - node2
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    # internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.21.5
    clusterName: cluster.local
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    plainHTTP: false
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []



---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.2.1
spec:
  persistence:
    storageClass: ""
  authentication:
    jwtSecret: ""
  local_registry: ""
  namespace_override: ""
  # dev_tag: ""
  etcd:
    monitoring: false
    endpointIps: localhost
    port: 2379
    tlsEnable: true
  common:
    core:
      console:
        enableMultiLogin: true
        port: 30880
        type: NodePort
    # apiserver:
    #  resources: {}
    # controllerManager:
    #  resources: {}
    redis:
      enabled: false
      volumeSize: 2Gi
    openldap:
      enabled: false
      volumeSize: 2Gi
    minio:
      volumeSize: 20Gi
    monitoring:
      # type: external
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
      GPUMonitoring:
        enabled: false
    gpu:
      kinds:         
      - resourceName: "nvidia.com/gpu"
        resourceType: "GPU"
        default: true
    es:
      # master:
      #   volumeSize: 4Gi
      #   replicas: 1
      #   resources: {}
      # data:
      #   volumeSize: 20Gi
      #   replicas: 1
      #   resources: {}
      logMaxAge: 7
      elkPrefix: logstash
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchHost: ""
      externalElasticsearchPort: ""
  alerting:
    enabled: false
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:
    enabled: false
    # operator:
    #   resources: {}
    # webhook:
    #   resources: {}
  devops:
    enabled: false
    jenkinsMemoryLim: 2Gi
    jenkinsMemoryReq: 1500Mi
    jenkinsVolumeSize: 8Gi
    jenkinsJavaOpts_Xms: 512m
    jenkinsJavaOpts_Xmx: 512m
    jenkinsJavaOpts_MaxRAM: 2g
  events:
    enabled: false
    # operator:
    #   resources: {}
    # exporter:
    #   resources: {}
    # ruler:
    #   enabled: true
    #   replicas: 2
    #   resources: {}
  logging:
    enabled: false
    containerruntime: docker
    logsidecar:
      enabled: true
      replicas: 2
      # resources: {}
  metrics_server:
    enabled: false
  monitoring:
    storageClass: ""
    # kube_rbac_proxy:
    #   resources: {}
    # kube_state_metrics:
    #   resources: {}
    # prometheus:
    #   replicas: 1
    #   volumeSize: 20Gi
    #   resources: {}
    #   operator:
    #     resources: {}
    #   adapter:
    #     resources: {}
    # node_exporter:
    #   resources: {}
    # alertmanager:
    #   replicas: 1
    #   resources: {}
    # notification_manager:
    #   resources: {}
    #   operator:
    #     resources: {}
    #   proxy:
    #     resources: {}
    gpu:
      nvidia_dcgm_exporter:
        enabled: false
        # resources: {}
  multicluster:
    clusterRole: none 
  network:
    networkpolicy:
      enabled: false
    ippool:
      type: none
    topology:
      type: none
  openpitrix:
    store:
      enabled: false
  servicemesh:
    enabled: false
  kubeedge:
    enabled: false   
    cloudCore:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      cloudhubPort: "10000"
      cloudhubQuicPort: "10001"
      cloudhubHttpsPort: "10002"
      cloudstreamPort: "10003"
      tunnelPort: "10004"
      cloudHub:
        advertiseAddress:
          - ""
        nodeLimit: "100"
      service:
        cloudhubNodePort: "30000"
        cloudhubQuicNodePort: "30001"
        cloudhubHttpsNodePort: "30002"
        cloudstreamNodePort: "30003"
        tunnelNodePort: "30004"
    edgeWatcher:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      edgeWatcherAgent:
        nodeSelector: {"node-role.kubernetes.io/worker": ""}
        tolerations: []
```

> 下面开始集群的创建

```shell
#开始创建
./kk create cluster -f config-sample.yaml

#执行完上一个命令后先会去检查三个节点可能会有必要组件缺失，我们装一下即可，这边报 conntrack is required.
#我们三个节点都执行下
yum install -y conntrack

#之后再次执行./kk create cluster -f config-sample.yaml 输入yes即可。
#中间流程可能会报某某应用安装失败，让你export KKZONE=cn后重试，实际输入后重试即可解决问题
#成功后可以看到以下界面
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://172.22.137.80:30880
Account: admin
Password: P@88w0rd

NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     "Cluster Management". If any service is not
     ready, please wait patiently until all components 
     are up and running.
  2. Please change the default password after login.

#####################################################
https://kubesphere.io             2022-05-24 15:57:00
#####################################################

#我们查看下
[root@master home]# kubectl get pods -A
NAMESPACE                      NAME                                               READY   STATUS    RESTARTS   AGE
kube-system                    calico-kube-controllers-75ddb95444-d9c9n           1/1     Running   0          5m45s
kube-system                    calico-node-58qcn                                  1/1     Running   0          5m45s
kube-system                    calico-node-cskx4                                  1/1     Running   0          5m45s
kube-system                    calico-node-rgnwg                                  1/1     Running   0          5m45s
kube-system                    coredns-5495dd7c88-c2dcj                           1/1     Running   0          5m56s
kube-system                    coredns-5495dd7c88-vc4zp                           1/1     Running   0          5m56s
kube-system                    kube-apiserver-master                              1/1     Running   0          6m4s
kube-system                    kube-controller-manager-master                     1/1     Running   0          6m4s
kube-system                    kube-proxy-cl47n                                   1/1     Running   0          5m47s
kube-system                    kube-proxy-hs77k                                   1/1     Running   0          5m56s
kube-system                    kube-proxy-sng59                                   1/1     Running   0          5m47s
kube-system                    kube-scheduler-master                              1/1     Running   0          6m4s
kube-system                    nodelocaldns-99qsc                                 1/1     Running   0          5m47s
kube-system                    nodelocaldns-lv5cw                                 1/1     Running   0          5m47s
kube-system                    nodelocaldns-m2h2w                                 1/1     Running   0          5m56s
kube-system                    openebs-localpv-provisioner-6c9dcb5c54-2c62g       1/1     Running   0          5m44s
kube-system                    snapshot-controller-0                              1/1     Running   0          4m54s
kubesphere-controls-system     default-http-backend-5bf68ff9b8-k72t9              1/1     Running   0          4m11s
kubesphere-controls-system     kubectl-admin-6667774bb-6vzbj                      1/1     Running   0          2m10s
kubesphere-monitoring-system   alertmanager-main-0                                2/2     Running   0          3m9s
kubesphere-monitoring-system   alertmanager-main-1                                2/2     Running   0          3m8s
kubesphere-monitoring-system   alertmanager-main-2                                2/2     Running   0          3m7s
kubesphere-monitoring-system   kube-state-metrics-5547ddd4cc-ztq5q                3/3     Running   0          3m19s
kubesphere-monitoring-system   node-exporter-4mpp5                                2/2     Running   0          3m20s
kubesphere-monitoring-system   node-exporter-622m9                                2/2     Running   0          3m20s
kubesphere-monitoring-system   node-exporter-nqszz                                2/2     Running   0          3m21s
kubesphere-monitoring-system   notification-manager-deployment-78664576cb-59qcm   2/2     Running   0          89s
kubesphere-monitoring-system   notification-manager-deployment-78664576cb-stshx   2/2     Running   0          89s
kubesphere-monitoring-system   notification-manager-operator-7d44854f54-62xgz     2/2     Running   0          3m3s
kubesphere-monitoring-system   prometheus-k8s-0                                   2/2     Running   1          3m9s
kubesphere-monitoring-system   prometheus-k8s-1                                   2/2     Running   1          3m8s
kubesphere-monitoring-system   prometheus-operator-5c5db79546-7dfsd               2/2     Running   0          3m22s
kubesphere-system              ks-apiserver-75848d78cf-qlfv5                      1/1     Running   0          2m46s
kubesphere-system              ks-console-65f4d44d88-zwc4j                        1/1     Running   0          4m11s
kubesphere-system              ks-controller-manager-9c7bc9b84-nsxmz              1/1     Running   0          2m45s
kubesphere-system              ks-installer-69df988b79-wk2wk                      1/1     Running   0          5m43s
```

> 下面我们去云服务器开下端口,开放30000到32767即可，至此一个最低限度的kubesphere集群就安装完成了

## 二.集群的使用

> 下面我们来研究下如何使用它

## 1.系统权限

进入kubesphere,我们可以点击左上角的平台管理，选择访问控制，来分配角色和权限

![avatar](https://picture.zhanghong110.top/docsify/16536123982461.png)

它这边是支持多租户的，所以会有企业空间这么一个选项

![avatar](https://picture.zhanghong110.top/docsify/16536125537980.png)

![avatar](https://picture.zhanghong110.top/docsify/16536125732204.png)

可以看到默认有如下四个角色，第一个用来创建及分配企业空间，第二个用来添加用户，第三个是平台普通用户，第四个是管理员。这些权限都是平台级别的，当你用某个用户进入后，比如你平台角色是`[workspaces-manager]`进入后菜单就会和下图类似（版本不同，懒得截图）

![avatar](https://picture.zhanghong110.top/docsify/165361284717.png)

就是你自己的企业，可以自己分配角色及邀请平台用户了。在分配项目后又有一个角色部分。搞得还是比较复杂的，可以自行研究。

## 2.项目部署

我们刚进系统，默认使用的是admin用户，此时我们没有学习以上这套权限，可能比较懵逼，我们演示环境就只使用`admin`账户展示，首先刚进去的页面如下。

![avatar](https://picture.zhanghong110.top/docsify/16536143268108.png)

默认是处于工作台界面，可以看见左上角，平台管理，应用商店,平台资源中可以看到企业空间及用户管理，还有应用模板，我们先记下来

然后点击左上角平台管理，有如下界面

![avatar](https://picture.zhanghong110.top/docsify/16536146928421.png)

我们如果点开访问控制，就会回到下述页面，拥有的企业空间及用户就是我们刚才在平台资源中看到的企业空间及用户。

![avatar](https://picture.zhanghong110.top/docsify/16536125537980.png)

我们如果不点击访问控制，点击集群管理就是下面这个页面

![avatar](https://picture.zhanghong110.top/docsify/16536148834907.png)

这个页面的话，都是集群相关的信息，这里面的项目，主要功能是创建，分配企业空间及配额控制，总体来说都是集群层面的东西。如果我们真的要创建项目。不是从这里面进去。

我们应当从刚才访问控制的企业空间或者平台资源的企业空间进入。如下图

![avatar](https://picture.zhanghong110.top/docsify/16536150968568.png)

进入后创建项目然后再点击进入如下图

![avatar](https://picture.zhanghong110.top/docsify/16536151762605.png)

终于我们终于进入了项目空间

![avatar](https://picture.zhanghong110.top/docsify/1653615293240.png)

> 到这里我们初步进入了一个项目空间，可以开始部署了。下面的内容需要一定的K8S基础及代码基础及中间件基础及docker基础。

我们部署一个项目需要选一种

工作负载，对应K8S的pod控制器，部署，有状态副本集，守护进程，分别对应K8S的`deployment`,`statefulset`, `daemonset`（一般中间件选择第二个，自己的服务选择第一个）。

存储配置对应K8S的`pvc`(这边下面还有配置选项，可以映射配置文件)。

服务对应的是`service`

最后是应用路由，对应K8S的`service` ingress暴露访问方式。

容器组对应了所有Pod

![avatar](https://picture.zhanghong110.top/docsify/16536173727141.png)

!>上面这段讲完，可能之后操作后会有疑惑，这边来讲几点

1.工作负载创建了以后，会产生一个pod，一个类型为ClusterIP的service一个对应类型的pod控制器。

2.如果创建服务们也会在相应的工作负载里产生一个pod控制器。

3.工作负载和服务就互相对应，一个pod控制器，一个service

4.负载均衡可以通过单service里的endpoint实现

5.多个service的负载均衡才设计lb及ingress(这边的应用路由菜单)

### 2.2.1 示例部署

> 本次示例我们从简单的组件开始，然后均以集群为主

#### 2.2.1.1 nacos

nacos没有数据要挂载，只需要配置即可，且集群化比较简单。

我们进入项目空间后先增加配置如下图

![avatar](https://picture.zhanghong110.top/docsify/16538979653792.png)

名称如下，然后点击下一步

![avatar](https://picture.zhanghong110.top/docsify/16538981928963.png)

我们先看下nacos的的conf。

![avatar](https://picture.zhanghong110.top/docsify/1653898387939.png)

根据官网可知一个是nacos的配置文件，一个是集群用。[nacos官网地址](https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html)。我们将application.properties中的配置复制过去，创立键值对。

![avatar](https://picture.zhanghong110.top/docsify/16538988942434.png)

我们先都粘贴默认的配置文件,第二个配置文件cluster.conf
![avatar](https://picture.zhanghong110.top/docsify/16538997141059.png)

完了以后我们创建一个有状态服务，如下

![avatar](https://picture.zhanghong110.top/docsify/165389983624.png)

命名为demo-nacoe，点击下一步

![avatar](https://picture.zhanghong110.top/docsify/1653899919910.png)

创建三个副本，选择容器

![avatar](https://picture.zhanghong110.top/docsify/16539000226133.png)

我们去dockerhub选择`2.0.4`（和配置文件契合即可）副本，添加到项目中。注意只需要名称不需要`docker pull`

![avatar](https://picture.zhanghong110.top/docsify/16539002057276.png)

做如下配置

![avatar](https://picture.zhanghong110.top/docsify/16539005004341.png)

![avatar](https://picture.zhanghong110.top/docsify/16539005466703.png)

nacos没啥卷挂在，挂个配置文件即可，如下

![avatar](https://picture.zhanghong110.top/docsify/16539007216888.png)

选择字典，只读挂载，如下图。挂到`/home/nacos/conf`下个路径下。意思就是把nacos的默认配置文件覆盖掉，这个地址可以去nacos镜像或者官网上看下。

![avatar](https://picture.zhanghong110.top/docsify/16539008439504.png)

!>注意，这样操作结果覆盖掉了配置文件，就生了这两货，所以不对，应该如下操作，挂载配置文件子路径，就是我们设置的配置文件一个个覆盖的意思

如下操作指定子路径，选择特定键值，两个配置文件都要这么操作。

![avatar](https://picture.zhanghong110.top/docsify/16539064192855.png)

![avatar](https://picture.zhanghong110.top/docsify/16539066906026.png)

最后一个高级设置不用管，点击创建即可。

!>注意，这边的话配置文件忘记讲了，有两个注意点，第一，nacos的application.properties需要改到能连上的库,还有就是那个集群的端口，为了能够在服务挂了的情况下代替的服务依然能够访问，需要使用域名，这边是 demo-nacos-v1-0.demo-nacos.qwer123.svc.cluster.local（cluster.conf要改成前面这个+`:8848`）

域名这个是由`statefulset`POD控制器决定的，有状态服务的特点是（这个我们K8S中漏了）

Pod一致性：包含次序（启动、停止次序）、网络一致性。此一致性与Pod相关，与被调度到哪个node节点无关；
稳定的次序：对于N个副本的StatefulSet，每个Pod都在[0，N)的范围内分配一个数字序号，且是唯一的；
稳定的网络：Pod的hostname模式为( s t a t e f u l s e t 名 称 ) − (statefulset名称)-(statefulset名称)−(序号)；
稳定的存储：通过VolumeClaimTemplate为每个Pod创建一个PV。删除、减少副本，不会删除相关的卷。

那么`demo-nacos-v1-0.demo-nacos.qwer123.svc.cluster.local`这玩意怎么得到的呢，我们可以祟拜你进个副本,ping下dns，dns在kuberSphere上可以看到容器集群的DNS，这样进去一Ping除了名称后缀，其它的都相同，所以这边就是`demo-nacos-v1-0.demo-nacos.qwer123.svc.cluster.local`，`demo-nacos-v1-1.demo-nacos.qwer123.svc.cluster.local` ， `demo-nacos-v1-2.demo-nacos.qwer123.svc.cluster.local`（实际配置时都要加上:8848）



然后可以进去看下容器情况，查看下容器日志

![avatar](https://picture.zhanghong110.top/docsify/16539085815023.png)

日志如下所示

```

          ,--.

        ,--.'|

    ,--,:  : |                                           Nacos 2.0.4

 ,`--.'`|  ' :                       ,---.               Running in cluster mode, All function modules

 |   :  :  | |                      '   ,'\   .--.--.    Port: 8849

 :   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 1

 |   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://10.233.96.111:8849/nacos/index.html

 '   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_

 |   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io

 '   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \

 |   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /

 '   : |     ;  :   .'   \   :    : `----'  '--'.     /

 ;   |.'     |  ,     .-./\   \  /            `--'---'

 '---'        `--`---'     `----'

 

 2022-05-30 19:01:17,797 INFO The server IP list of Nacos is []

 

 2022-05-30 19:01:18,798 INFO Nacos is starting...

 

 2022-05-30 19:01:19,799 INFO Nacos is starting...

 

 2022-05-30 19:01:20,803 INFO Nacos is starting...

 

 2022-05-30 19:01:21,810 INFO Nacos is starting...

 

 2022-05-30 19:01:22,813 INFO Nacos is starting...

 

 2022-05-30 19:01:23,815 INFO Nacos is starting...

 

 2022-05-30 19:01:24,816 INFO Nacos is starting...

 

 2022-05-30 19:01:25,816 INFO Nacos is starting...

 

 2022-05-30 19:01:26,816 INFO Nacos is starting...

 

 2022-05-30 19:01:27,823 INFO Nacos is starting...

 

 2022-05-30 19:01:28,824 INFO Nacos is starting...

 

 2022-05-30 19:01:29,137 INFO Nacos started successfully in cluster mode. use external storage

 
```

成功了，我们创建一个工作负载，做一个端口暴露，还是在服务中，如下点击

![avatar](https://picture.zhanghong110.top/docsify/16539090545943.png)

随便起个名字，然后下面这边选个有状态副本集。端口全都选择8848

![avatar](https://picture.zhanghong110.top/docsify/16539093195643.png)

然后下一步，点击NodePort即可

![avatar](https://picture.zhanghong110.top/docsify/16539094753409.png)

![avatar](https://picture.zhanghong110.top/docsify/16539095571370.png)

我们这就搞定了，三个节点随便找个端口访问。比如（这边给忘了8849也要映射下）

http://39.107.48.128:31904/nacos/#/login

连接成功，测试链接就可以了。

!>我们会发现，虽然控制台没有问题，但是服务连不上。

Nacos2.0版本相比1.X新增了gRPC的通信方式，因此需要增加2个端口。新增端口是在配置的主端口(server.port)基础上，进行一定偏移量自动生成。

| 端口 | 与主端口的偏移量 | 描述                                                       |
| ---- | ---------------- | ---------------------------------------------------------- |
| 9848 | 1000             | 客户端gRPC请求服务端端口，用于客户端向服务端发起连接和请求 |
| 9849 | 1001             | 服务端gRPC请求服务端端口，用于服务间同步等                 |

> 可知我们之前的暴露还存在问题，8848和9848暴露是不行的，还要9849，对应的映射也需要+1000和+1001

我们按照下图做一遍

![avatar](https://picture.zhanghong110.top/docsify/1653958748493.png)

![avatar](https://picture.zhanghong110.top/docsify/16539602437811.png)

这里进去多暴露一组，注意这边的话需要服务全部重启，可以把容器整到0再重新加到3即可。

然后

![avatar](https://picture.zhanghong110.top/docsify/1653960495355.png)

![avatar](https://picture.zhanghong110.top/docsify/16539605718871.png)

编辑服务，加一组8849，然后编辑yaml指定到之前8848暴露的端口+1001即可。

> 再尝试链接，问题解决。

#### 2.2.1.2  ruoyi gateway

> 例如mysql及redis这些集群上云，通过上面nacos的配置我们可以看出集群的配置需要服务本身也支持（包括数据同步啊之类的都要考虑），我们之后在研究，下面演示个微服务服务上云，采用手动模式。

在这之前我们先回顾下不用K8S的docker部署。

我们除了需要知道gateway的端口（方便nginx映射）意外还需要暴露前端的访问地址。

上k8s以后，gateway的端口就不用知道了，我们使用k8s service 提供的DNS域名来访问，这样无论启动多少个网关pod都不需要Ip,其它服务的访问也可以统一暴露一样的端口(pod隔离)。

准备工作: 需要一个harbor私服或者阿里云的镜像私服（个人版免费的）。

**镜像的制作与推送**

ruoyi自带了dockerfile如下,aoyang为公司名，这边以前全局替换的，影响不大。

```dockerfile
# 基础镜像
FROM  openjdk:8-jre
# author
MAINTAINER aoyang
# 挂载目录
VOLUME /home/aoyang
# 创建目录
RUN mkdir -p /home/aoyang
# 指定路径
WORKDIR /home/aoyang
# 复制jar文件到路径
COPY ./jar/aoyang-gateway.jar /home/aoyang/aoyang-gateway.jar
# 启动网关服务
ENTRYPOINT ["java","-jar","aoyang-gateway.jar"]
```

我们使用它还需要做点改造

```dockerfile
# 基础镜像,改成jdk,方便以后进容器输出一些jdk才带的命令
FROM  openjdk:8-jdk

LABEL maintainer=gc

#环境,使用生产环境（nacos要有）,我们使用k8s内部集群的话这边应该还要加上--spring.cloud.nacos.discovery.server-addr=service的DNS地址+端口 及--spring.cloud.nacos.config.server-add=service的DNS地址+端口。
ENV PARAMS="--server.port=8080 --spring.profiles.active=prod"

#修改镜像时区
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone

#文件复制,实际根目录只有一个jar包
COPY target/*.jar /app.jar

#所有服务统一暴露8080,pod ip不同
EXPOSE 8080

#启动脚本，要包含环境变量params,-Djava.security.edg=file:/dev/./urandom这个是固定值用于防止启动时阻塞。
ENTRYPOINT ["/bin/sh","-c","java -Djava.security.edg=file:/dev/./urandom -jar app.jar ${PARAMS}"]
```

我们新建一个文件夹随便叫个名字，里面新建target文件夹，然后新建文件Dockerfile,将这里面的内容贴进去。

![avatar](https://picture.zhanghong110.top/docsify/16541394543973.png)

我们将它传到master节点。

```shell
#进入目录
cd ry-gateway

#构建
docker build -t ry-gateway:v1.0 -f Dockerfile .

#构建完成
Successfully built bac184287646
Successfully tagged ay-gateway:v1.0
[root@k8s-master ry-gateway]# docker images
REPOSITORY                           TAG       IMAGE ID       CREATED          SIZE
ay-gateway                           v1.0      bac184287646   42 seconds ago   627MB
```

下面就是docker登录仓库，然后推送到仓库。

我们这采用harbor私有库,发现报如下错，这是因为我们[docker](https://so.csdn.net/so/search?q=docker&spm=1001.2101.3001.7020) client使用的是https，而我们搭建的Harbor私库用的是http的，所有会有这样的报错，导致访问不了。

```shell
[root@k8s-master ry-gateway]# docker login http://xxx.xxx.xxx.xxx:xx/
Username: admin
Password: 
Error response from daemon: Get "https://xxx.xxx.xxx.xxx:xx/v2/": http: server gave HTTP response to HTTPS client
```

[解决方案](https://blog.csdn.net/qing040513/article/details/115750334)可以看下，我这边换成我私服布置的docker就可以直连了。

```shell
[root@centerm ry-gateway]# docker login 127.0.0.1:xx
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

推送镜像

```shell
#镜像不是都能推的，需要先改名成符合规范的名称,我们在harbor中创建名为demo的项目,然后比如我们镜像现在叫这个
[root@centerm ry-gateway]# docker images
REPOSITORY                      TAG        IMAGE ID       CREATED         SIZE
ry-gateway                      v1.0       5de2acb5dcc4   9 minutes ago   627MB

#改名,改成harbor地址+项目名+镜像名：版本 , 或者docker tag 5de2acb5dcc4 xxx.zhanghong110.top/demo/ry-gateway:v1.0,这边需要额外配置否则会报X509，建议还是ip的方式，拉取的时候用域名不影响
docker tag 5de2acb5dcc4 127.0.0.1:89/demo/ry-gateway:v1.0

#推送,前面是重命名的镜像+个版本号即可，或者docker push xxx.zhanghong110.top/demo/ry-gateway:v1.0,这边需要额外配置否则会报X509，建议还是ip的方式，拉取的时候用域名不影响
docker push 127.0.0.1:89/demo/ry-gateway:v1.0
```

推送后看到如下结果即表示成功了。

![avatar](https://picture.zhanghong110.top/docsify/16541516694060.png)

下面我们登录kuberSphere尝试部署。

!>注意，这里发现K8s安装的docker要链接私服还是得用https，显然我们不太可能一台机器一台机器去改，所以呢就需要harbor以https的方式安装

完了之后，我们去配置中心密钥里面增加一个类型为仓库镜像的密钥。

![avatar](https://picture.zhanghong110.top/docsify/16541786325806.png)

之后我们会去创建工作负载，使用无状态的

![avatar](https://picture.zhanghong110.top/docsify/16541789569102.png)

然后这样选择使用默认端口，后面也不用挂存储卷一直下一步即可。

![avatar](https://picture.zhanghong110.top/docsify/16541837671838.png)

![avatar](https://picture.zhanghong110.top/docsify/16541834977546.png)

在检查下日志，发现运行良好。

> 至此我们完成了手动部署一个微服务到KuberSphere上