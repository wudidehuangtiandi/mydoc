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

待续