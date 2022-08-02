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

那么`demo-nacos-v1-0.demo-nacos.qwer123.svc.cluster.local`这玩意怎么得到的呢，我们可以随便进个副本,ping下dns，dns在kuberSphere上可以看到容器集群的DNS，这样进去一Ping除了名称后缀，其它的都相同，所以这边就是`demo-nacos-v1-0.demo-nacos.qwer123.svc.cluster.local`，`demo-nacos-v1-1.demo-nacos.qwer123.svc.cluster.local` ， `demo-nacos-v1-2.demo-nacos.qwer123.svc.cluster.local`（实际配置时都要加上:8848）



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

### 2.2.2 流水线

#### 2.2.2.1 后端微服务

> 下面我们学一下如何使用流水线，做到gitlab合并即发布。

这个其实就是一个图形化的过程来编写`jenkinsFile`

在我们开始前我们先要理解一个概念

> 无论后面多少个容器，指定多少东西，最后操作完文件都会返回给同一台K8S所在的机器。就是说实际上是在同一台机器上虚拟化容器去执行脚本，完了以后都会返回给这台机器，然后再去用这些东西，在下一个容器中操作。



先新建一个

![avatar](https://picture.zhanghong110.top/docsify/16566588807178.png)

然后我们点进去之后点创建，这里也是无脑下一步即可

![avatar](https://picture.zhanghong110.top/docsify/165665934330.png)

我们选中编辑流水线，点中间这个

![avatar](https://picture.zhanghong110.top/docsify/16566594836439.png)

首先我们可以指定一个大环境，包括maven nodejs等等，但是其实不影响

![avatar](https://picture.zhanghong110.top/docsify/16566601144263.png)

> 下面我们进行一个模板流水线的编辑

首先是项目拉取

点击第一步骤，点击添加步骤

![avatar](https://picture.zhanghong110.top/docsify/16566606542158.png)

选择指定容器

![avatar](https://picture.zhanghong110.top/docsify/16566607183323.png)

!>这里虽然是输入但是不能随便写有以下[选项](https://v3-2.docs.kubesphere.io/zh/docs/devops-user-guide/how-to-use/choose-jenkins-agent/),包含了不通的内置工具，我们第一步拉取代码，选maven即可

点击添加嵌套步骤

![avatar](https://picture.zhanghong110.top/docsify/16566628799108.png)



然后选择GIT，我们输入我们的项目地址，分支，这边凭证没有需要新建

![avatar](https://picture.zhanghong110.top/docsify/1656661181209.png)

我们输入账号密码，ID随便选一个，以后会保存，退出去设置分支即可

![avatar](https://picture.zhanghong110.top/docsify/16566613475782.png)

完了我们添加一个嵌套步骤，选择shell,加入`ls -al`命令打一下结果

![avatar](https://picture.zhanghong110.top/docsify/16566624078698.png)

> 测试运行下发现成功

下面重复的图就不展示了，只用文字描述。

我们开始第二步，项目打包

这里的话有一个需要注意的地方，默认的话Maven会从中央仓库下载东西，如果需要从阿里云或者私服下载，需要用admin账户去集群管理种修改下配置文件

![avatar](https://picture.zhanghong110.top/docsify/16566645394378.png)

像这个样子改下即可。



然后我们编辑下打包的流水线，只需要加一个shell命令即可，如下配置 具体过程还是最外层选择node,maven 指定容器(名称写maven)-》添加嵌套步骤-》添加shell脚本，容器选择maven环境即可。

> 打包并跳过测试 mvn clean package -Dmaven.test.skip=true

![avatar](https://picture.zhanghong110.top/docsify/16566715845290.png)

打包的话需要根据自己配置的dockerfile来构建不同的镜像，以xueyi-cloud为例子

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
ENTRYPOINT ["/bin/sh","-c","java -Djava.security.edg=file:/test/./urandom -jar app.jar ${PARAMS}"]
```



至此就编译完成了，我们下面应该做流水线的第三步，构建镜像，我们先还是指定最外层为node,maven,然后指定容器，命名为maven。然后添加shell脚本，添加嵌套步骤,脚本如下

```shell
# -t为镜像tag -f为dockerfile路径 最后一项为工作目录
docker build -t aoyang-order:latest -f aoyang-modules/aoyang-order/Dockerfile aoyang-modules/aoyang-order/ 
```

好了以后添加并行阶段，把所有服务都按照这样一样打包即可。

![avatar](https://picture.zhanghong110.top/docsify/16593342309711.png)

 

!>这边通过研究同事的文件发现，最外层也可直接用none 无所谓



 然后我们进入下一步，推送镜像，即把打包的镜像推送到镜像仓库，我们用私有的harbor作为演示，这边的脚本需要做一些动态取值，包括以下两个脚本

```shell
#给tag重命名,规则为仓库地址（域名或者IP）+仓库内命名空间+镜像名（重命名加上第几次构建）
docker tag aoyang-order:latest $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-order:SNAPSHOT-$SNAPSHOT_NUM
#推送刚才那个镜像
docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-order:SNAPSHOT-$SNAPSHOT_NUM
```

然后我们去kubesphere操作下，方式还是一样的，指定容器，添加嵌套步骤，添加shell脚本（里面放上面这两个命令），再添加嵌套步骤，增加一个凭证。凭证要注意下，我们分别将账号密码字段声明成以下变量，这样凭证密码变更后推送不会被影响到。

```shell
DOCKER_PWD_VAR
DOCKER_USER_VAR
```

![avatar](https://picture.zhanghong110.top/docsify/16593384973523.png)

!>注意到这里我们会发现凭证是嵌套在shell脚本中的，此时其实我们需要这些步骤都在凭证下执行，因此我们可以改下jenkinsfile的步骤。将shell脚本放到凭证下。

然后这边还少一个步骤，即docker login步骤。我们需要在shell脚本中加入以下命令，啥意思呢，就是将密码输出到登录命令后的输入的地方。

```shell
echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin
```

!>要以非交互方式运行docker login命令,可以将--*password- stdin*标志设置为通过STDIN提供密码。使用STDIN可以防止密码出现在shell的历史记录或日志文件中。

最后如下

![avatar](https://picture.zhanghong110.top/docsify/16593403596622.png)

完事了添加并行步骤,把所有模块都搞进去。

下面部署到测试环境。

> 首先我们要为每个服务准备一个deploy.yaml

我们还是空的流程步骤，然后不选指定容器了，添加步骤中选择`kubernetesDeploy`，然后创建凭证（这玩意用来授权给别的Node k8s的权限，等同于手动部署中的复制token到其它服务器）

![avatar](https://picture.zhanghong110.top/docsify/16593418898791.png)

每个模块都需要一个`deploy.yaml`，这边给出一个例子，具体参考`K8S`文档

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: aoyang-order
  name: aoyang-order
  namespace: lrzx-dev   #一定要写名称空间
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  selector:
    matchLabels:
      app: aoyang-order
  strategy:
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: aoyang-order
    spec:
      imagePullSecrets:
        - name: aoyang-harbor  #这个就是密钥的名字，需要提前在项目下配置账号密码（用以拉镜像）
      containers:
        - image: $REGISTRY/$ALIYUNHUB_NAMESPACE/aoyang-order:SNAPSHOT-$SNAPSHOT_NUM
 # 去掉就绪探针      
 #         readinessProbe:
 #             httpGet:
 #               path: /actuator/health
 #               port: 8080
 #             timeoutSeconds: 10
 #             failureThreshold: 30
 #             periodSeconds: 5
 #
          imagePullPolicy: Always
          name: app
          ports:
            - containerPort: 8080
              protocol: TCP
 # 注意这步可能会导致服务响应速度问题             
          resources:
            limits:
              cpu: 300m
              memory: 400Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: aoyang-order
  name: aoyang-order
  namespace: lrzx-dev
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: aoyang-order
  sessionAffinity: None
  type: ClusterIP
```

配置文件路径填`aoyang-modules/aoyang-order/deploy/**`即可,不同的包路径自行匹配。列子中的`deploy/deploy.yaml`位于各个模块`src`同级目录下

完事了添加并行即可

到这里就结束了，我们来看下最后的样子

![avatar](https://picture.zhanghong110.top/docsify/16593445745832.png)



最后给出一个生成的`jenkins.file`,可做参考。

```
pipeline {
  agent {
    node {
      label 'maven'
    }

  }
  stages {
    stage('拉取代码') {
      agent none
      steps {
        container('maven') {
          git(url: 'http://172.23.112.16/gechao/lrzx-cloud.git', credentialsId: 'gitlab-yph', branch: 'online', changelog: true, poll: false)
          sh 'ls -al'
        }

      }
    }

    stage('项目编译') {
      agent none
      steps {
        container('maven') {
          sh 'mvn clean package -Dmaven.test.skip=true'
        }

      }
    }

    stage('default-2') {
      parallel {
        stage('构建aoyang-basedata') {
          agent none
          steps {
            container('maven') {
              sh ''
              sh 'docker build -t aoyang-basedata:latest -f aoyang-modules/aoyang-basedata/Dockerfile aoyang-modules/aoyang-basedata/'
            }

          }
        }

        stage('构建aoyang-wx-miniapp') {
          agent none
          steps {
            container('maven') {
              sh ''
              sh 'docker build -t aoyang-wx-miniapp:latest -f aoyang-modules/aoyang-wx-miniapp/Dockerfile aoyang-modules/aoyang-wx-miniapp/'
            }

          }
        }

        stage('构建aoyang-order') {
          agent none
          steps {
            container('maven') {
              sh ''
              sh 'docker build -t aoyang-order:latest -f aoyang-modules/aoyang-order/Dockerfile aoyang-modules/aoyang-order/'
            }

          }
        }

        stage('构建aoyang-itsm') {
          agent none
          steps {
            container('maven') {
              sh ''
              sh 'docker build -t aoyang-itsm:latest -f aoyang-modules/aoyang-itsm/Dockerfile aoyang-modules/aoyang-itsm/'
            }

          }
        }

      }
    }

    stage('default-3') {
      parallel {
        stage('推送aoyang-basedata镜像') {
          agent none
          steps {
            container('maven') {
              withCredentials([usernamePassword(credentialsId : 'harbor-yph' ,passwordVariable : 'DOCKER_PWD_VAR' ,usernameVariable : 'DOCKER_USER_VAR' ,)]) {
                sh 'echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin'
                sh 'docker tag aoyang-basedata:latest $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-basedata:SNAPSHOT-$SNAPSHOT_NUM'
                sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-basedata:SNAPSHOT-$SNAPSHOT_NUM'
                sh 'docker '
              }

            }

          }
        }

        stage('推送aoyang-wx-miniapp镜像') {
          agent none
          steps {
            container('maven') {
              withCredentials([usernamePassword(credentialsId : 'harbor-yph' ,passwordVariable : 'DOCKER_PWD_VAR' ,usernameVariable : 'DOCKER_USER_VAR' ,)]) {
                sh 'echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin'
                sh 'docker tag aoyang-wx-miniapp:latest $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-wx-miniapp:SNAPSHOT-$SNAPSHOT_NUM'
                sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-wx-miniapp:SNAPSHOT-$SNAPSHOT_NUM'
              }

            }

          }
        }

        stage('推送aoyang-order镜像') {
          agent none
          steps {
            container('maven') {
              withCredentials([usernamePassword(credentialsId : 'harbor-yph' ,passwordVariable : 'DOCKER_PWD_VAR' ,usernameVariable : 'DOCKER_USER_VAR' ,)]) {
                sh 'echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin'
                sh 'docker tag aoyang-order:latest $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-order:SNAPSHOT-$SNAPSHOT_NUM'
                sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-order:SNAPSHOT-$SNAPSHOT_NUM'
              }
            }
          }
        }

        stage('推送aoyang-itsm镜像') {
          agent none
          steps {
            container('maven') {
              withCredentials([usernamePassword(credentialsId : 'harbor-yph' ,passwordVariable : 'DOCKER_PWD_VAR' ,usernameVariable : 'DOCKER_USER_VAR' ,)]) {
                sh 'echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin'
                sh 'docker tag aoyang-itsm:latest $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-itsm:SNAPSHOT-$SNAPSHOT_NUM'
                sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/aoyang-itsm:SNAPSHOT-$SNAPSHOT_NUM'
              }
            }
          }
        }
      }
    }

    stage('default-4') {
      parallel {
        stage('aoyang-basedata部署到测试环境') {
          agent none
          steps {
            kubernetesDeploy(configs: 'aoyang-modules/aoyang-basedata/deploy/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
          }
        }

        stage('aoyang-wx-miniapp部署到测试环境') {
          agent none
          steps {
            kubernetesDeploy(configs: 'aoyang-modules/aoyang-wx-miniapp/deploy/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
          }
        }

        stage('aoyang-order部署到测试环境') {
          agent none
          steps {
            kubernetesDeploy(configs: 'aoyang-modules/aoyang-order/deploy/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
          }
        }

        stage('aoyang-itsm部署到测试环境') {
          agent none
          steps {
            kubernetesDeploy(configs: 'aoyang-modules/aoyang-itsm/deploy/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
          }
        }

      }
    }

    stage('部署到生产环境') {
      agent none
      steps {
        input(id: 'deploy-to-production', message: 'deploy to production?')
        kubernetesDeploy(configs: 'deploy/prod-ol/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
      }
    }

  }
  environment {
    DOCKER_CREDENTIAL_ID = 'dockerhub-id'
    GITHUB_CREDENTIAL_ID = 'github-id'
    KUBECONFIG_CREDENTIAL_ID = 'harbor-config'
    REGISTRY = '192.168.169.113:8001'
    DOCKERHUB_NAMESPACE = 'lrzx-cloud'
    GITHUB_ACCOUNT = 'kubesphere'
    APP_NAME = 'devops-java-sample'
    ALIYUNHUB_NAMESPACE = 'lrzx-cloud'
    SNAPSHOT_NUM = '2'
  }
  parameters {
    string(name: 'TAG_NAME', defaultValue: '', description: '')
  }
}
```

#### 2.2.2.2 前端

前端的部署略有不同,下面一些重复的步骤我们用截图加解释的方式

第一步，拉取代码，注意容器使用nodejs

![avatar](https://picture.zhanghong110.top/docsify/16593521056720.png)



第二部项目编译，注意要指定淘宝镜像及打生产环境的包

![avatar](https://picture.zhanghong110.top/docsify/16593523272184.png)



第三步构建镜像

![avatar](https://picture.zhanghong110.top/docsify/16593524253395.png)

这步需要`dockerfile`,给个实例,这个实例中可以看到还需要指定nginx的配置文件

```dockerfile
# 基础镜像
FROM nginx
# author
MAINTAINER yphyyy

# 挂载目录
#VOLUME /home/lrzx/projects/lrzx-ui
# 创建目录
#RUN mkdir -p /home/lrzx/projects/lrzx-ui
# 指定路径
# WORKDIR /home/lrzx/projects/lrzx-ui
# 复制conf文件到路径
COPY nginx/conf/nginx.conf /etc/nginx/nginx.conf
# 复制html文件到路径

COPY dist /usr/share/nginx/html/

EXPOSE 80
```

根据dockerfile,我们在dist通缉目录下建`nginx/conf/nginx.conf`,内容如下

```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  _;
		    charset utf-8;

        location / {
            root   /usr/share/nginx/html;
            try_files $uri $uri/ /index.html;
            index  index.html index.htm;
        }

        location /prod-api/ {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header REMOTE-HOST $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://aoyang-gateway.lrzx-dev:8080/;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```



下一步，推送镜像，这个和后端一样，还是添加私服凭证，登录镜像仓库，然后给镜像打标签，然后推送给私服

![avatar](https://picture.zhanghong110.top/docsify/16593527059521.png)

我们贴一下具体的脚本

```shell
echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin

docker tag lrzx-ui:latest $REGISTRY/$DOCKERHUB_NAMESPACE/lrzx-ui:SNAPSHOT-$BUILD_NUM

docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/lrzx-ui:SNAPSHOT-$BUILD_NUM
```

最后一步，部署到测试环境

![avatar](https://picture.zhanghong110.top/docsify/16593528975833.png)

这里又需要`deploy.yaml`我们还是在`dist`同级增加`deploy/deploy.yaml`,我们同样给出一个例子

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: lrzx-ui
  name: lrzx-ui
  namespace: lrzx-dev   #一定要写名称空间
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  selector:
    matchLabels:
      app: lrzx-ui
  strategy:
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: lrzx-ui
    spec:
      imagePullSecrets:
        - name: aoyang-harbor  #提前在项目下配置访问阿里云的账号密码
      containers:
        - image: $REGISTRY/$ALIYUNHUB_NAMESPACE/lrzx-ui:SNAPSHOT-$BUILD_NUM
 #         readinessProbe:
 #             httpGet:
 #               path: /actuator/health
 #               port: 8080
 #             timeoutSeconds: 10
 #             failureThreshold: 30
 #             periodSeconds: 5
 #
          imagePullPolicy: Always
          name: app
          ports:
            - containerPort: 80
              protocol: TCP
          resources:
            limits:
              cpu: 300m
              memory: 400Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: lrzx-ui
  name: lrzx-ui
  namespace: lrzx-dev
spec:
  ports:
    - name: http
      port: 30446
      nodePort: 30446
      protocol: TCP
      targetPort: 80
  selector:
    app: lrzx-ui
#  sessionAffinity: None
  type: NodePort
```

至此前端也可以了，下面我们给出一个生成好的`jenkinsfile`

```
pipeline {
  agent {
    node {
      label 'nodejs'
    }

  }
  stages {
    stage('拉取代码测试') {
      agent none
      steps {
        container('nodejs') {
          git(url: 'http://172.23.112.16/gechao/lrzx-ui.git', credentialsId: 'gitlab-yph', branch: 'online', changelog: true, poll: false)
          sh 'ls -al'
        }

      }
    }

    stage('项目编译') {
      agent none
      steps {
        container('nodejs') {
          sh 'npm install --registry=https://registry.npm.taobao.org'
          sh 'npm run build:prod'
          sh 'ls'
        }

      }
    }

    stage('构建镜像') {
      agent none
      steps {
        container('nodejs') {
          sh 'ls dist/'
          sh 'docker build -t lrzx-ui:latest -f Dockerfile .'
        }

      }
    }

    stage('推送镜像') {
      agent none
      steps {
        container('nodejs') {
          withCredentials([usernamePassword(credentialsId : 'harbor-yph' ,usernameVariable : 'DOCKER_USER_VAR' ,passwordVariable : 'DOCKER_PWD_VAR' ,)]) {
            sh 'echo "$DOCKER_PWD_VAR" | docker login $REGISTRY -u "$DOCKER_USER_VAR" --password-stdin'
            sh 'docker tag lrzx-ui:latest $REGISTRY/$DOCKERHUB_NAMESPACE/lrzx-ui:SNAPSHOT-$BUILD_NUM'
            sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/lrzx-ui:SNAPSHOT-$BUILD_NUM'
          }

        }

      }
    }

    stage('lrzx-ui部署到测试环境') {
      agent none
      steps {
        kubernetesDeploy(configs: 'deploy/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
      }
    }

    stage('部署到生产环境') {
      agent none
      steps {
        input(id: 'deploy-to-production', message: 'deploy to production?')
        kubernetesDeploy(configs: 'deploy/prod-ol/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
      }
    }

  }
  environment {
    DOCKER_CREDENTIAL_ID = 'dockerhub-id'
    GITHUB_CREDENTIAL_ID = 'github-id'
    KUBECONFIG_CREDENTIAL_ID = 'harbor-config'
    REGISTRY = '192.168.169.113:8001'
    DOCKERHUB_NAMESPACE = 'lrzx-ui'
    GITHUB_ACCOUNT = 'kubesphere'
    APP_NAME = 'devops-java-sample'
    ALIYUNHUB_NAMESPACE = 'lrzx-ui'
    BUILD_NUM = '2'
  }
  parameters {
    string(name: 'TAG_NAME', defaultValue: '', description: '')
  }
}
```

