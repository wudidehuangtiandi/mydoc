

# k8s的初步搭建及使用

>本次以单master，双node作为演示。仅用于初步的学习参考。版本信息如下

```shell
[root@master ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.4", GitCommit:"e6c093d87ea4cbb530a7b2ae91e54c0842d8308a", GitTreeState:"clean", BuildDate:"2022-02-16T12:38:05Z", GoVersion:"go1.17.7", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.4", GitCommit:"e6c093d87ea4cbb530a7b2ae91e54c0842d8308a", GitTreeState:"clean", BuildDate:"2022-02-16T12:32:02Z", GoVersion:"go1.17.7", Compiler:"gc", Platform:"linux/amd64"}

```

## 一.搭建

环境准备：centos7.6虚拟机三台，至少需要2G内存，否则初始化集群会爆内存不足

1.查看主机名，修改成master,node1,node2

```
hostnamectl status
hostnamectl set-hostname xxx   修改为master,mode1,mode2
```

2.修改三台机器的hosts并且同步时间，仅用于演示，正式部署建议使用DNS服务器及时间同步服务器

```
修改三台机器的 etc/hosts
添加
192.168.191.131 node1
192.168.191.132 node2
192.168.191.130 master
三台机采用网络时间同步
systemctl start chronyd
systemctl enable chronyd
```

3.所有节点关闭 SELinux（linux安全服务，开着会有奇葩问题），关闭iptables(防止规则混淆)

```
修改/etc/selinux/config ,重启后生效
SELINUX=disabled
下面这个有的系统可能没有
systemctl stop iptables
systemctl disable iptables
```

4.所有节点关闭防火墙，生产环境这个要慎重，

```
systemctl stop firewalld
systemctl disable firewalld
```

5.所有节点禁用虚拟分区

```
编辑/etc/fstab注释掉swap分区一行即 /dev/mapper/centos-swap swap  ,重启后生效
```

6.所有节点修改linux内核参数增加网桥过滤和转发功能

```
增加etc/sysctl.d/kubernetes.conf文件
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

重新加载
sysctl -p
加载网桥过滤模块
modprobe br_netfilter
查看是否成功
lsmod | grep br_netfilter
```

> 可选步骤（所有节点）

载入ipvs模块，service中有基于iptable和ipvs两种代理模型后者性能明显要高但是要手动载入模块（master node都要）

```
yum install ipset ipvsadmin -y
增加脚本
cat <<EOF > /etc/sysconfig/modules/ipvs.modules
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

为脚本加权限
chmod +x /etc/sysconfig/modules/ipvs.modules
执行脚本
/bin/bash /etc/sysconfig/modules/ipvs.modules
查看是否成功
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

7.所有节点添加K8S安装源及docker安装源

```
# 添加 k8s 安装源
cat <<EOF > kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
mv kubernetes.repo /etc/yum.repos.d/


# 添加 Docker 安装源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```

8.重启,所有节点安装所需组件(虽然主节点不需要kubelet，但是我们梭哈了)

```
reboot

yum install -y kubelet kubeadm kubectl docker-ce
```

9.所有节点启动 kubelet、docker，并设置开机启动（所有节点）

```
systemctl enable kubelet
systemctl start kubelet
systemctl enable docker
systemctl start docker
```

```

这边如果刚才选择了ipvs模式则要增加以下命令(ipvsadm看一下内核是否支持,不支持就sudo yum -y install ipvsadm)
KUBELET_CGROUP_ARGS="--CGROUP-DRIVER=systemed"
KUBE_PROXY_MODE="ipvs"
```

10.所有节点修改docker配置

```
cat <<EOF > daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://ud6340vz.mirror.aliyuncs.com"]
}
EOF
mv daemon.json /etc/docker/

然后重启服务生效
systemctl daemon-reload
systemctl restart docker
```

11.仅在主节点操作，初始化集群

```
# 初始化集群控制台 Control plane
# 失败了可以用 kubeadm reset 重置(如果部署过了要rm -rf $HOME/.kube/config ) 
kubeadm init --image-repository=registry.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.191.130 --service-cidr=10.96.0.0/12  --pod-network-cidr=10.244.0.0/16

参数说明
--apiserver-advertise-address=xx.xx.xx.xx   这个参数就是master主机的IP地址，例如我的Master主机的IP是：192.168.181.131
--image-repository=registry.aliyuncs.com/google_containers  这个是镜像地址，由于国外地址无法访问，故使用的阿里云仓库地址：registry.aliyuncs.com/google_containers
--kubernetes-version=v1.17.4   这个参数是下载的k8s软件版本号
--service-cidr=10.96.0.0/12    这个参数后的IP地址直接就套用10.96.0.0/12 ,以后安装时也套用即可，不要更改
--pod-network-cidr=10.244.0.0/16   k8s内部的pod节点之间网络可以使用的IP段，不能和service-cidr写一样，如果不知道怎么配，就先用这个10.244.0.0/16

最后弹出结果
kubeadm join 192.168.191.130:6443 --token 7g0jed.eldb436kh34xeuj0 \
	--discovery-token-ca-cert-hash sha256:435f869461ee6290fbd350a48d93cfeeaafef5fa44e7fd8c09874a21731462a3
表示成功，将该条命令保存下来
# 忘记了重新获取：kubeadm token create --print-join-command


```

12.仅主节点设置在本机的环境变量

```
# 复制授权文件，以便 kubectl 可以有权限访问集群
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 在其他机器上创建 ~/.kube/config 文件也能通过 kubectl 访问到集群
```

13.在两个Node节点上运行刚刚保存的命令

```
注意要去除换行符
kubeadm join 192.168.191.130:6443 --token 7g0jed.eldb436kh34xeuj0 --discovery-token-ca-cert-hash sha256:435f869461ee6290fbd350a48d93cfeeaafef5fa44e7fd8c09874a21731462a3
```

14.最后主节点安装网络插件，否则都是Notready

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
这个的话国内被墙了修改Host
185.199.109.133 raw.githubusercontent.com

http://ip.tool.chinaz.com/raw.githubusercontent.com 这个网站可以去查下raw.githubusercontent.com 跳转的IP ，能ping通的话改下Host即可
```

> 至此我们一个单master双node的节点就安装完毕了

我们使用`kubectl get nodes`可以看到如下

```
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   16m   v1.23.4
node1    Ready    <none>                 14m   v1.23.4
node2    Ready    <none>                 14m   v1.23.4
```

查看pods信息`kubectl get pods -A`

```
kube-system   coredns-6d8c4cb4d-p6q7j          1/1     Running   0          73m
kube-system   coredns-6d8c4cb4d-xxsm9          1/1     Running   0          73m
kube-system   etcd-master                      1/1     Running   5          74m
kube-system   kube-apiserver-master            1/1     Running   5          74m
kube-system   kube-controller-manager-master   1/1     Running   0          74m
kube-system   kube-flannel-ds-69jv9            1/1     Running   0          20m
kube-system   kube-flannel-ds-drrw6            1/1     Running   0          20m
kube-system   kube-flannel-ds-dwd9x            1/1     Running   0          20m
kube-system   kube-proxy-lljmk                 1/1     Running   0          71m
kube-system   kube-proxy-sr569                 1/1     Running   0          71m
kube-system   kube-proxy-xqgcb                 1/1     Running   0          73m
kube-system   kube-scheduler-master            1/1     Running   5          74m
```

> kubectl logs 【pod name】 可以看下pod报错信息（不行加上--namespace kube-system）

!>这边如果发现一直处于 ContainerCreating状态可以用下面这个命令看下情况 

```
kubectl describe pod xxx(name字段) 

kubectl describe pods/xxx --namespace kube-system
```

如果长时间没有READY，三台机`reboot`下就行了。pods都ready即可



下面贴以下卸载k8s及日志查看的方式

```
节点重置：
kubeadm reset

kubeadm卸载:
卸载服务，所有节点
kubeadm reset -f
清空iptables规则，所有节点
iptables -F 
iptables -X
清空ipvs规则，所有节点
ipvsadm -C
清空CNI规则，所有节点
rm -rf /etc/cni/net.d
清空CNI规则，所有节点
rm -rf $HOME/.kube/config 

日志查看
journalctl -u kubelet | tail -n 300
```

## 二.nginx的简单部署

> 测试运行一个nginx容器，注意以后的操作只需要通过master节点即可

部署nginx

```
kubectl create deployment nginx --image=nginx:1.14-alpine
```

暴露端口

```
kubectl expose deployment nginx --port=80 --type=NodePort
```

查看service看下Nginx暴露的端口号

```
kubectl get service nginx
```

我们这边发现端口映射是30194

![avatar](https://picture.zhanghong110.top/docsify/16454115748721.png)

可以看到成功访问

## 三.资源管理

> 在kubernetes中，所有的内容都抽象为资源，用户需要通过操作资源来管理kubernetes。

kubernetes的本质上就是一个集群系统，用户可以在集群中部署各种服务，所谓的部署服务，其实就是在kubernetes集群中运行一个个的容器，并将指定的程序跑在容器中。

kubernetes的最小管理单元是pod而不是容器，所以只能将容器放在`Pod`中，而kubernetes一般也不会直接管理Pod，而是通过`Pod控制器`来管理Pod的。

Pod可以提供服务之后，就要考虑如何访问Pod中服务，kubernetes提供了`Service`资源实现这个功能。

当然，如果Pod中程序的数据需要持久化，kubernetes还提供了各种`存储`系统。

![avatar](https://picture.zhanghong110.top/docsify/20200406225334627.png)

资源管理的方式:

- 命令式对象管理：直接使用命令去操作kubernetes资源


  ``` powershell
  kubectl run nginx-pod --image=nginx:1.17.1 --port=80
  ```

- 命令式对象配置：通过命令配置和配置文件去操作kubernetes资源

  ```powershell
  kubectl create/patch -f nginx-pod.yaml
  ```

- 声明式对象配置：通过apply命令和配置文件去操作kubernetes资源

  ```powershell
  kubectl apply -f nginx-pod.yaml
  ```

  

### 3.1 命令式对象管理

**kubectl命令**

kubectl是kubernetes集群的命令行工具，通过它能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署。kubectl命令的语法如下：

```
kubectl [command] [type] [name] [flags]
```

**comand**：指定要对资源执行的操作，例如create、get、delete

**type**：指定资源类型，比如deployment、pod、service

**name**：指定资源的名称，名称大小写敏感

**flags**：指定额外的可选参数

```shell
# 查看所有pod
kubectl get pod 

# 查看某个pod
kubectl get pod pod_name

# 查看某个pod,以yaml格式展示结果
kubectl get pod pod_name -o yaml
```

**资源类型**

kubernetes中所有的内容都抽象为资源，可以通过下面的命令进行查看:

```
kubectl api-resources
```

经常使用的资源有下面这些：

| 资源分类      | 资源名称                 | 缩写    | 资源作用        |
| :------------ | :----------------------- | :------ | :-------------- |
| 集群级别资源  | nodes                    | no      | 集群组成部分    |
| namespaces    | ns                       | 隔离Pod |                 |
| pod资源       | pods                     | po      | 装载容器        |
| pod资源控制器 | replicationcontrollers   | rc      | 控制pod资源     |
|               | replicasets              | rs      | 控制pod资源     |
|               | deployments              | deploy  | 控制pod资源     |
|               | daemonsets               | ds      | 控制pod资源     |
|               | jobs                     |         | 控制pod资源     |
|               | cronjobs                 | cj      | 控制pod资源     |
|               | horizontalpodautoscalers | hpa     | 控制pod资源     |
|               | statefulsets             | sts     | 控制pod资源     |
| 服务发现资源  | services                 | svc     | 统一pod对外接口 |
|               | ingress                  | ing     | 统一pod对外接口 |
| 存储资源      | volumeattachments        |         | 存储            |
|               | persistentvolumes        | pv      | 存储            |
|               | persistentvolumeclaims   | pvc     | 存储            |
| 配置资源      | configmaps               | cm      | 配置            |
|               | secrets                  |         | 配置            |

**操作**

kubernetes允许对资源进行多种操作，可以通过--help查看详细的操作命令

```
kubectl --help
```

经常使用的操作有下面这些：

| 命令分类   | 命令         | 翻译                        | 命令作用                     |
| :--------- | :----------- | :-------------------------- | :--------------------------- |
| 基本命令   | create       | 创建                        | 创建一个资源                 |
|            | edit         | 编辑                        | 编辑一个资源                 |
|            | get          | 获取                        | 获取一个资源                 |
|            | patch        | 更新                        | 更新一个资源                 |
|            | delete       | 删除                        | 删除一个资源                 |
|            | explain      | 解释                        | 展示资源文档                 |
| 运行和调试 | run          | 运行                        | 在集群中运行一个指定的镜像   |
|            | expose       | 暴露                        | 暴露资源为Service            |
|            | describe     | 描述                        | 显示资源内部信息             |
|            | logs         | 日志输出容器在 pod 中的日志 | 输出容器在 pod 中的日志      |
|            | attach       | 缠绕进入运行中的容器        | 进入运行中的容器             |
|            | exec         | 执行容器中的一个命令        | 执行容器中的一个命令         |
|            | cp           | 复制                        | 在Pod内外复制文件            |
|            | rollout      | 首次展示                    | 管理资源的发布               |
|            | scale        | 规模                        | 扩(缩)容Pod的数量            |
|            | autoscale    | 自动调整                    | 自动调整Pod的数量            |
| 高级命令   | apply        | rc                          | 通过文件对资源进行配置       |
|            | label        | 标签                        | 更新资源上的标签             |
| 其他命令   | cluster-info | 集群信息                    | 显示集群信息                 |
|            | version      | 版本                        | 显示当前Server和Client的版本 |

下面以一个namespace / pod的创建和删除简单演示下命令的使用：

```shell
# 创建一个namespace
[root@master ~]# kubectl create namespace dev
namespace/dev created

# 获取namespace
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   21h
dev               Active   21s
kube-node-lease   Active   21h
kube-public       Active   21h
kube-system       Active   21h

# 在此namespace下创建并运行一个nginx的Pod
[root@master ~]# kubectl run pod --image=nginx:latest -n dev
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/pod created

# 查看新创建的pod
[root@master ~]# kubectl get pod -n dev
NAME  READY   STATUS    RESTARTS   AGE
pod   1/1     Running   0          21s

# 删除指定的pod
[root@master ~]# kubectl delete pod pod -n dev 
pod "pod" deleted

# 删除指定的namespace
[root@master ~]# kubectl delete ns dev
namespace "dev" deleted
```

### 3.2 命令式对象配置

命令式对象配置就是使用命令配合配置文件一起来操作kubernetes资源。

1） 创建一个nginxpod.yaml，内容如下：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev

---

apiVersion: v1
kind: Pod
metadata:
  name: nginxpod
  namespace: dev
spec:
  containers:
  - name: nginx-containers
    image: nginx:latest
```

2）执行create命令，创建资源：

```powershell
[root@master ~]# kubectl create -f nginxpod.yaml
namespace/dev created
pod/nginxpod created
```

此时发现创建了两个资源对象，分别是namespace和pod

3）执行get命令，查看资源：

```shell
[root@master ~]#  kubectl get -f nginxpod.yaml
NAME            STATUS   AGE
namespace/dev   Active   18s

NAME            READY   STATUS    RESTARTS   AGE
pod/nginxpod    1/1     Running   0          17s
```

这样就显示了两个资源对象的信息

4）执行delete命令，删除资源：

```shell
[root@master ~]# kubectl delete -f nginxpod.yaml
namespace "dev" deleted
pod "nginxpod" deleted
```

此时发现两个资源对象被删除了

```
总结:
    命令式对象配置的方式操作资源，可以简单的认为：命令  +  yaml配置文件（里面是命令需要的各种参数）
```

### 3.3 声明式对象配置

声明式对象配置跟命令式对象配置很相似，但是它只有一个命令apply。

```shell
# 首先执行一次kubectl apply -f yaml文件，发现创建了资源
[root@master ~]#  kubectl apply -f nginxpod.yaml
namespace/dev created
pod/nginxpod created

# 再次执行一次kubectl apply -f yaml文件，发现说资源没有变动
[root@master ~]#  kubectl apply -f nginxpod.yaml
namespace/dev unchanged
pod/nginxpod unchanged
```

```powershell
总结:
    其实声明式对象配置就是使用apply描述一个资源最终的状态（在yaml中定义状态）
    使用apply操作资源：
        如果资源不存在，就创建，相当于 kubectl create
        如果资源已存在，就更新，相当于 kubectl patch
```

> 扩展：kubectl可以在node节点上运行吗 ?

kubectl的运行是需要进行配置的，它的配置文件是$HOME/.kube，如果想要在node节点运行此命令，需要将master上的.kube文件复制到node节点上，即在master节点上执行下面操作：

后面为目标机器账户和IP；

```shell
scp -r  $HOME/.kube  root@192.168.191.131:/root
```

> 使用推荐: 三种方式应该怎么用 ?

创建/更新资源 使用声明式对象配置 kubectl apply -f XXX.yaml

删除资源 使用命令式对象配置 kubectl delete -f XXX.yaml

查询资源 使用命令式对象管理 kubectl get(describe) 资源名称

## 四.基本概念与nginx的集群搭建

> 我们现在开始尝试搭建一个nginx集群,并能够对其进行访问,在此期间我们需要了解很多概念，掌握一些使用方法。

### 4.1 namespace

> Namespace是kubernetes系统中的一种非常重要资源，它的主要作用是用来实现**多套环境的资源隔离**或者**多租户的资源隔离**。

默认情况下，kubernetes集群中的所有的Pod都是可以相互访问的。但是在实际中，可能不想让两个Pod之间进行互相的访问，那此时就可以将两个Pod划分到不同的namespace下。kubernetes通过将集群内部的资源分配到不同的Namespace中，可以形成逻辑上的"组"，以方便不同的组的资源进行隔离使用和管理。

可以通过kubernetes的授权机制，将不同的namespace交给不同租户进行管理，这样就实现了多租户的资源隔离。此时还能结合kubernetes的资源配额机制，限定不同租户能占用的资源，例如CPU使用量、内存使用量等等，来实现租户可用资源的管理。

> kubernetes在集群启动之后，会默认创建几个namespace

![avatar](https://picture.zhanghong110.top/docsify/16456835921364.png)

我们来看一下关于`namespace`的常用命令

```
查看所有的namespace
kubectl get ns/namespace

查看指定的ns   
kubectl get ns default

指定输出格式 
kubectl get ns default -o yaml
kubernetes支持的格式有很多，比较常见的是wide、json、yaml

查看ns详情
kubectl describe ns default
这个打印的详情有两行讲一下
# ResourceQuota 针对namespace做的资源限制
# LimitRange针对namespace中的每个组件做的资源限制
No resource quota.
No LimitRange resource.

创建namespace
kubectl create ns dev

删除namespace
kubectl delete ns dev
```

配置的方式创建或者删除

首先准备一个配置文件ns-dev.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

然后就可以执行对应的创建和删除命令了：

创建：kubectl create -f ns-dev.yaml

删除：kubectl delete -f ns-dev.yaml

### 4.2 pod

> Pod是kubernetes集群进行管理的最小单元，程序要运行必须部署在容器中，而容器必须存在于Pod中。Pod可以认为是容器的封装，一个Pod中可以存在一个或者多个容器。

kubernetes在集群启动之后，集群中的各个组件也都是以Pod方式运行的。可以通过下面命令查看：

```
kubectl get pod -n kube-system
```

![avatar](https://picture.zhanghong110.top/docsify/16456849253205.png)

我们来看一下关于`pod`的相关命令

```
pod创建
kubernetes没有提供单独运行Pod的命令，都是通过Pod控制器来实现的
命令格式： kubectl run (pod控制器名称) [参数] 
# --image  指定Pod的镜像
# --port   指定端口
# --namespace  指定namespace
kubectl run nginx --image=nginx:latest --port=80 --namespace dev 

查看Pod信息
kubectl get pods -n dev

查看Pod的详细信息
kubectl describe pod nginx -n dev

获取podIP
kubectl get pods -n dev -o wide

访问pod
curl http://10.244.2.5:80

删除指定pod
kubectl delete pod nginx-7cbb8cd5d8-mf6mt -n default
注意：此时，显示删除Pod成功，但是再查询，发现又新产生了一个 
这是因为当前Pod是由Pod控制器创建的，控制器会监控Pod状况，一旦发现Pod死亡，会立即重建

此时要想删除Pod，必须删除Pod控制器
kubectl get deploy -n  default
kubectl delete deploy nginx -n default
稍等片刻，再查询Pod，发现Pod被删除了
```

配置方式创建或者删除

创建一个pod-nginx.yaml，内容如下：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
  - image: nginx:latest
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

创建：kubectl create -f pod-nginx.yaml

删除：kubectl delete -f pod-nginx.yam

### 4.3 label

> Label是kubernetes系统中的一个重要概念。它的作用就是在资源上添加标识，用来对它们进行区分和选择。

Label的特点：

- 一个Label会以key/value键值对的形式附加到各种对象上，如Node、Pod、Service等等
- 一个资源对象可以定义任意数量的Label ，同一个Label也可以被添加到任意数量的资源对象上去
- Label通常在资源对象定义时确定，当然也可以在对象创建后动态添加或者删除

可以通过Label实现资源的多维度分组，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

> 一些常用的Label 示例如下：
>
> - 版本标签："version":"release", "version":"stable"......
> - 环境标签："environment":"dev"，"environment":"test"，"environment":"pro"
> - 架构标签："tier":"frontend"，"tier":"backend"

标签定义完毕之后，还要考虑到标签的选择，这就要使用到Label Selector，即：

Label用于给某个资源对象定义标识

Label Selector用于查询和筛选拥有某些标签的资源对象

当前有两种Label Selector：

- 基于等式的Label Selector

  name = slave: 选择所有包含Label中key="name"且value="slave"的对象

  env != production: 选择所有包括Label中的key="env"且value不等于"production"的对象

- 基于集合的Label Selector

  name in (master, slave): 选择所有包含Label中的key="name"且value="master"或"slave"的对象

  name not in (frontend): 选择所有包含Label中的key="name"且value不等于"frontend"的对象

标签的选择条件可以使用多个，此时将多个Label Selector进行组合，使用逗号","进行分隔即可。例如：

name=slave，env!=production

name not in (frontend)，env!=production

下面我们看下label的一些基本命令

```
为pod资源打标签
kubectl label pod nginx-pod version=1.0 -n dev

为pod资源更新标签
kubectl label pod nginx-pod version=2.0 -n dev --overwrite

查看标签
kubectl get pod nginx-pod  -n dev --show-labels

筛选标签
kubectl get pod -n dev -l version=2.0  --show-labels
kubectl get pod -n dev -l version!=2.0 --show-labels

删除标签
kubectl label pod nginx-pod version- -n dev
```

配置方式

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
  labels:
    version: "3.0" 
    env: "test"
spec:
  containers:
  - image: nginx:latest
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的更新命令了：kubectl apply -f pod-nginx.yaml

### 4.4 deployment

>在kubernetes中，Pod是最小的控制单元，但是kubernetes很少直接控制Pod，一般都是通过Pod控制器来完成的。Pod控制器用于pod的管理，确保pod资源符合预期的状态，当pod的资源出现故障时，会尝试进行重启或重建pod。在kubernetes中Pod控制器的种类有很多，这里只介绍一种：Deployment。

命令操作

```
kubectl create deployment 名称  [参数] 
# --image  指定pod的镜像
# --port   指定端口
# --replicas  指定创建pod数量
# --namespace  指定namespace

kubectl create deployment nginx --port=80 --replicas=3 --image=nginx:1.14-alpine -n default

查看创建的Pod
kubectl get pods -n default

查看deployment的信息
kubectl get deploy -n default
打印的头作如下解释
# UP-TO-DATE：成功升级的副本数量
# AVAILABLE：可用副本的数量
展开
kubectl get deploy -n default -o wide

查看deployment的详细信息
kubectl describe deploy nginx -n default

删除
kubectl delete deploy nginx -n default
```

配置操作

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

创建：kubectl create -f deploy-nginx.yaml

删除：kubectl delete -f deploy-nginx.yaml

### 4.5 service

虽然每个Pod都会分配一个单独的Pod IP，然而却存在如下两问题：

- Pod IP 会随着Pod的重建产生变化
- Pod IP 仅仅是集群内可见的虚拟IP，外部无法访问

> 这样对于访问这个服务带来了难度。因此，kubernetes设计了Service来解决这个问题。Service可以看作是一组同类Pod**对外的访问接口**。借助Service，应用可以方便地实现服务发现和负载均衡。

![avatar](https://picture.zhanghong110.top/docsify/20200408194716912.png)

> 下面我们来看下如何创建集群内部可访问的service和集群外部可访问的service

首先看下我们拥有的三个pod

```
default       nginx-6cf5589999-h8v9b           1/1     Running   0             8m5s
default       nginx-6cf5589999-p67mj           1/1     Running   0             8m5s
default       nginx-6cf5589999-xbznc           1/1     Running   0             8m5s
```

集群内部可访问的service:

```
暴露service
kubectl expose deploy nginx --name=nginx-6cf5589999-h8v9b  --type=ClusterIP --port=80 --target-port=80 -n default

查看service
kubectl get svc nginx-6cf5589999-h8v9b -n default -o wide
贴一下值
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
nginx-6cf5589999-h8v9b   ClusterIP   10.106.43.66   <none>        80/TCP    95s   app=nginx
# 这里产生了一个CLUSTER-IP，这就是service的IP，在Service的生命周期中，这个地址是不会变动的
# 可以通过这个IP访问当前service对应的POD

我们试下，可以发现成功连上了
curl 10.106.43.66:80
```

集群外部也可访问的service

```
上面创建的Service的type类型为ClusterIP，这个ip地址只用集群内部可访问
如果需要创建外部也可以访问的Service，需要修改type为NodePort
kubectl expose deploy nginx --name=nginx-6cf5589999-p67mj   --type=NodePort --port=80 --target-port=80 -n default

此时查看
kubectl get svc nginx-6cf5589999-p67mj -n default -o wide
贴一下值
NAME                     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
nginx-6cf5589999-p67mj   NodePort   10.106.69.229   <none>        80:30680/TCP   7s    app=nginx


# 接下来就可以通过集群外的主机访问 节点IP:30566访问服务了
# 例如在的电脑主机上通过浏览器访问下面的地址(注意这边映射到主机了，service内部IP将不在有效，采用宿主机的IP+映射端口访问)
http://192.168.191.130:30680/
```

配置方式

```
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: dev
spec:
  clusterIP: 10.109.179.231 #固定svc的内网ip
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  type: ClusterIP
```

然后就可以执行对应的创建和删除命令了：

创建：kubectl create -f svc-nginx.yaml

删除：kubectl delete -f svc-nginx.yaml



> 至此，已经掌握了Namespace、Pod、Deployment、Service资源的基本操作，有了这些操作，就可以在kubernetes集群中实现一个服务的简单部署和访问了。



## 五.进阶

> 如果想要更好的使用kubernetes，就需要深入学习这几种资源的细节和原理。

### 5.1 pod详解



![avatar](https://picture.zhanghong110.top/docsify/20200407121501907.png)



每个Pod中都可以包含一个或者多个容器，这些容器可以分为两类：

- 用户程序所在的容器，数量可多可少

- Pause容器，这是每个Pod都会有的一个**根容器**，它的作用有两个：

  - 可以以它为依据，评估整个Pod的健康状态

  - 可以在根容器上设置Ip地址，其它容器都此Ip（Pod IP），以实现Pod内部的网路通信

    ```
    这里是Pod内部的通讯，Pod的之间的通讯采用虚拟二层网络技术来实现，我们当前环境用的是Flannel
    ```

#### 5.1.1 pod配置

下面我们看下pod的资源清单

```yaml
apiVersion: v1     #必选，版本号，例如v1
kind: Pod       　 #必选，资源类型，例如 Pod
metadata:       　 #必选，元数据
  name: string     #必选，Pod名称
  namespace: string  #Pod所属的命名空间,默认为"default"
  labels:       　　  #自定义标签列表
    - name: string      　          
spec:  #必选，Pod中容器的详细定义
  containers:  #必选，Pod中容器列表
  - name: string   #必选，容器名称
    image: string  #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
    command: [string]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      #容器的启动命令参数列表
    workingDir: string  #容器的工作目录
    volumeMounts:       #挂载到容器内部的存储卷配置
    - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean #是否为只读模式
    ports: #需要暴露的端口库号列表
    - name: string        #端口的名称
      containerPort: int  #容器需要监听的端口号
      hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string    #端口协议，支持TCP和UDP，默认TCP
    env:   #容器运行前需设置的环境变量列表
    - name: string  #环境变量名称
      value: string #环境变量的值
    resources: #资源限制和请求的设置
      limits:  #资源限制的设置
        cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests: #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string #内存请求,容器启动的初始可用数量
    lifecycle: #生命周期钩子
        postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
        preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
    livenessProbe:  #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
      exec:       　 #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
  restartPolicy: [Always | Never | OnFailure]  #Pod的重启策略
  nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
  nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
  - name: string
  hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes:   #在该pod上定义共享存储卷列表
  - name: string    #共享存储卷名称 （volumes类型有很多种）
    emptyDir: {}       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
    hostPath: string   #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
      path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
    secret:       　　　#类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
      scretname: string  
      items:     
      - key: string
        path: string
    configMap:         #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
      name: string
      items:
      - key: string
        path: string
```

```
小提示：
在这里，可通过一个命令来查看每种资源的可配置项
kubectl explain 资源类型         查看某种资源可以配置的一级属性
kubectl explain 资源类型.属性     查看属性的子属性
比如
kubectl explain pod
kubectl explain pod.metadata
```

在kubernetes中基本所有资源的一级属性都是一样的，主要包含5部分：

- apiVersion <string> 版本，由kubernetes内部定义，版本号必须可以用 kubectl api-versions 查询到
- kind <string> 类型，由kubernetes内部定义，版本号必须可以用 kubectl api-resources 查询到
- metadata <Object> 元数据，主要是资源标识和说明，常用的有name、namespace、labels等
- spec <Object> 描述，这是配置中最重要的一部分，里面是对各种资源配置的详细描述
- status <Object> 状态信息，里面的内容不需要定义，由kubernetes自动生成

在上面的属性中，spec是接下来研究的重点，继续看下它的常见子属性:

- containers <[]Object> 容器列表，用于定义容器的详细信息
- nodeName <String> 根据nodeName的值将pod调度到指定的Node节点上
- nodeSelector <map[]> 根据NodeSelector中定义的信息选择将该Pod调度到包含这些label的Node 上
- hostNetwork <boolean> 是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
- volumes <[]Object> 存储卷，用于定义Pod上面挂在的存储信息
- restartPolicy <string> 重启策略，表示Pod在遇到故障的时候的处理策略


> 我们可以明显看出containers的配置最长，我们重点研究下

```shell
#命令看下：
kubectl explain pod.spec.containers

#打印的精简及解释：
KIND:     Pod
VERSION:  v1
RESOURCE: containers <[]Object>   # 数组，代表可以有多个容器
FIELDS:
   name  <string>     # 容器名称
   image <string>     # 容器需要的镜像地址
   imagePullPolicy  <string> # 镜像拉取策略 
   command  <[]string> # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
   args     <[]string> # 容器的启动命令需要的参数列表
   env      <[]Object> # 容器环境变量的配置
   ports    <[]Object>     # 容器需要暴露的端口号列表
   resources <Object>      # 资源限制和资源请求的设置
```

我们尝试一下，建立如下配置文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: default
  labels:
    user: gc
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
```

上面定义了一个比较简单Pod的配置，里面有两个容器：

- nginx：用1.17.1版本的nginx镜像创建，（nginx是一个轻量级web容器）
- busybox：用1.30版本的busybox镜像创建，（busybox是一个小巧的linux命令集合）

在自己选定的路径创建以上配置文件命名为`pod-base.yaml`，我这边放在`/home/pod`下，之后我们都使用此目录

进入该目录后后执行命令

```shell
#启动
kubectl apply -f pod-base.yaml

# 查看Pod状况
# READY 1/2 : 表示当前Pod中有2个容器，其中1个准备就绪，1个未就绪
# RESTARTS  : 重启次数，因为有1个容器故障了，Pod一直在重启试图恢复它
 kubectl get pod -n default
 NAME       READY   STATUS             RESTARTS       AGE
pod-base   1/2     CrashLoopBackOff   5 (110s ago)   5m34s

# 可以通过describe查看内部的详情
# 此时已经运行起来了一个基本的Pod，虽然它暂时有问题(先不管它)
[root@k8s-master01 pod]# kubectl describe pod pod-base -n default
```



> 我们先来看下镜像的拉取

创建以下配置文件命名为`pod-imagepullpolicy.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-imagepullpolicy
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    imagePullPolicy: Never # 用于设置镜像拉取策略
  - name: busybox
    image: busybox:1.30
```



imagePullPolicy，用于设置镜像拉取策略，kubernetes支持配置三种拉取策略：

- Always：总是从远程仓库拉取镜像（一直远程下载）
- IfNotPresent：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像（本地有就本地 本地没远程下载）
- Never：只使用本地镜像，从不去远程仓库拉取，本地没有就报错 （一直使用本地）

> 默认值说明：
>
> 如果镜像tag为具体版本号， 默认策略是：IfNotPresent
>
> 如果镜像tag为：latest（最终版本） ，默认策略是always

创建pod

```shell
kubectl create -f pod-imagepullpolicy.yaml

# 查看Pod详情
# 应为我们配置了never所以有了如下日志
kubectl describe pod pod-imagepullpolicy -n default

Events:
  Type     Reason             Age               From               Message
  ----     ------             ----              ----               -------
  Normal   Scheduled          28s               default-scheduler  Successfully assigned default/pod-imagepullpolicy to node1
  Normal   Pulling            28s               kubelet            Pulling image "busybox:1.30"
  Normal   Pulled             9s                kubelet            Container image "busybox:1.30" already present on machine
  Normal   Pulled             9s                kubelet            Successfully pulled image "busybox:1.30" in 18.390761241s
  Normal   Created            9s (x2 over 9s)   kubelet            Created container busybox
  Normal   Started            8s (x2 over 9s)   kubelet            Started container busybox
  Warning  Failed             7s (x4 over 28s)  kubelet            Error: ErrImageNeverPull
  Warning  ErrImageNeverPull  7s (x4 over 28s)  kubelet            Container image "nginx:1.17.1" is not present with pull policy of Never
  Warning  BackOff            7s (x2 over 8s)   kubelet            Back-off restarting failed container
```



> 在前面的案例中，一直有一个问题没有解决，就是的busybox容器一直没有成功运行，那么到底是什么原因导致这个容器的故障呢？

busybox并不是一个程序，而是类似于一个工具类的集合，kubernetes集群启动管理后，它会自动关闭。解决方法就是让其一直在运行，这就用到了command配置。

我们采用如下的配置文件命名为`pod-command.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done;"]
```

command，用于在pod中的容器初始化完毕之后运行一个命令。

>稍微解释下上面命令的意思：
>
>"/bin/sh","-c", 使用sh执行命令
>
>touch /tmp/hello.txt; 创建一个/tmp/hello.txt 文件
>
>while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done; 每隔3秒向文件中写入当前时间

```shell
kubectl create  -f pod-command.yaml

# 查看Pod状态
# 此时发现两个pod都正常运行了
kubectl get pods pod-command -n default

# 我们补充一个命令
# kubectl exec  pod名称 -n 命名空间 -it -c 容器名称 /bin/sh  在容器内部执行命令
# 使用这个命令就可以进入某个容器的内部，然后进行相关操作了
# 比如，可以查看txt文件的内容

kubectl exec pod-command -n default -it -c busybox /bin/sh
tail -f /tmp/hello.txt
# 可以看到日志打印
01:36:59
01:37:02
01:37:05
01:37:08
01:37:11

#exit退出
```

![avatar](https://picture.zhanghong110.top/docsify/16457528419874.png)

```
这里有一个地方要说明：
    通过上面发现command已经可以完成启动命令和传递参数的功能，为什么这里还要提供一个args选项，用于传递参数呢?这其实跟docker有点关系，kubernetes中的command、args两项其实是实现覆盖Dockerfile中ENTRYPOINT的功能。
 1 如果command和args均没有写，那么用Dockerfile的配置。
 2 如果command写了，但args没有写，那么Dockerfile默认的配置会被忽略，执行输入的command
 3 如果command没写，但args写了，那么Dockerfile中配置的ENTRYPOINT的命令会被执行，使用当前args的参数
 4 如果command和args都写了，那么Dockerfile的配置被忽略，执行command并追加上args参数
```



> 接下来我们看下环境变量的配置

创建pod-env.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do /bin/echo $(date +%T);sleep 60; done;"]
    env: # 设置环境变量列表
    - name: "username"
      value: "admin"
    - name: "password"
      value: "123456"
```

```shell
kubectl create -f pod-env.yaml

#进入容器后输出
kubectl exec pod-env -n default -c busybox -it /bin/sh

/ # echo $username
admin
/ # echo $password
123456
```

!>这种方式不是很推荐，推荐将这些配置单独存储在配置文件中，这种方式将在后面介绍。

> 接下来我们来看如何配置端口,也就是containers的ports选项。

```shell
#先看下有啥子选项
kubectl explain pod.spec.containers.ports
KIND:     Pod
VERSION:  v1
RESOURCE: ports <[]Object>
FIELDS:
   name         <string>  # 端口名称，如果指定，必须保证name在pod中是唯一的		
   containerPort<integer> # 容器要监听的端口(0<x<65536)
   hostPort     <integer> # 容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本(一般省略) 
   hostIP       <string>  # 要将外部端口绑定到的主机IP(一般省略)
   protocol     <string>  # 端口协议。必须是UDP、TCP或SCTP。默认为“TCP”。
```

编写一个测试案例，创建`pod-ports.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: # 设置容器暴露的端口列表
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

```shell
kubectl create -f pod-ports.yaml

# 查看pod
# 在下面可以明显看到配置信息
kubectl get pod pod-ports -n default -o yaml
......
spec:
  containers:
  - image: nginx:1.17.1
    imagePullPolicy: IfNotPresent
    name: nginx
    ports:
    - containerPort: 80
      name: nginx-port
      protocol: TCP
......
```

访问容器中的程序需要使用的是`Podip:containerPort`

> 最后是资源配额

容器中的程序要运行，肯定是要占用一定资源的，比如cpu和内存等，如果不对某个容器的资源做限制，那么它就可能吃掉大量资源，导致其它容器无法运行。针对这种情况，kubernetes提供了对内存和cpu的资源进行配额的机制，这种机制主要通过resources选项实现，他有两个子选项：

- limits：用于限制运行时容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启
- requests ：用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动

可以通过上面两个选项设置资源的上下限。

这个嘛我们编辑下配置文件`pod-resources.yaml`

```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    resources: # 资源配额
      limits:  # 限制资源（上限）
        cpu: "2" # CPU限制，单位是core数
        memory: "200Mi" # 内存限制
      requests: # 请求资源（下限）
        cpu: "1"  # CPU限制，单位是core数
        memory: "100Mi"  # 内存限制
```

在这对cpu和memory的单位做一个说明：

- cpu：core数，可以为整数或小数
- memory： 内存大小，可以使用Gi、Mi、G、M等形式

实际上，1GB=1000MB=1000000KB=1000000000B

它不同于，1GiB=1024MiB=1048576KiB=1073741824B(这里不是iB)。

```shell
# 运行Pod
[root@master pod]# kubectl create  -f pod-resources.yaml
pod/pod-resources created

# 查看发现pod运行正常
[root@master pod]# kubectl get pod pod-resources -n default
NAME            READY   STATUS    RESTARTS   AGE
pod-resources   1/1     Running   0          54s


# 接下来，停止Pod(这个命令必须在指定目录下执行)
[root@master pod]# kubectl delete  -f pod-resources.yaml
pod "pod-resources" deleted

# 编辑pod，
    #  limits:  # 限制资源（上限）
    #    cpu: "2" # CPU限制，单位是core数
    #    memory: "10Gi" # 内存限制
    #  requests: # 请求资源（下限）
    #    cpu: "1"  # CPU限制，单位是core数
    #    memory: "5Gi"  # 内存限制

# 再次启动pod
[root@master pod]# kubectl create  -f pod-resources.yaml
pod/pod-resources created

# 查看Pod状态，发现Pod启动失败
[root@master pod]# kubectl get pod pod-resources -n default -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
pod-resources   0/1     Pending   0          3s    <none>   <none>   <none>           <none>
   
# 查看pod详情会发现，如下提示
[root@master pod]# kubectl describe pod pod-resources -n default
......
  Warning  FailedScheduling  29s   default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 Insufficient memory.
```

#### 5.1.2 pod的生命周期

我们一般将pod对象从创建至终的这段时间范围称为pod的生命周期，它主要包含下面的过程：

- pod创建过程
- 运行初始化容器（init container）过程
- 运行主容器（main container）
  - 容器启动后钩子（post start）、容器终止前钩子（pre stop）
  - 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）
- pod终止过程

![avatar](https://picture.zhanghong110.top/docsify/20200412111402706.png)

在整个生命周期中，Pod会出现5种**状态**（**相位**），分别如下：

- 挂起（Pending）：apiserver已经创建了pod资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中
- 运行中（Running）：pod已经被调度至某节点，并且所有容器都已经被kubelet创建完成
- 成功（Succeeded）：pod中的所有容器都已经成功终止并且不会被重启
- 失败（Failed）：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态
- 未知（Unknown）：apiserver无法正常获取到pod对象的状态信息，通常由网络通信失败所导致

> 我们根据以上的生命周期详细看下各个周期内的流程

pod创建过程

1. 用户通过kubectl或其他api客户端提交需要创建的pod信息给apiServer

2. apiServer开始生成pod对象的信息，并将信息存入etcd，然后返回确认信息至客户端

3. apiServer开始反映etcd中的pod对象的变化，其它组件使用watch机制来跟踪检查apiServer上的变动

4. scheduler发现有新的pod对象要创建，开始为Pod分配主机并将结果信息更新至apiServer

5. node节点上的kubelet发现有pod调度过来，尝试调用docker启动容器，并将结果回送至apiServer

6. apiServer将接收到的pod状态信息存入etcd中

![avatar](https://picture.zhanghong110.top/docsify/20200406184656917.png)

pod终止过程

1. 用户向apiServer发送删除pod对象的命令
2. apiServcer中的pod对象信息会随着时间的推移而更新，在宽限期内（默认30s），pod被视为dead
3. 将pod标记为terminating状态
4. kubelet在监控到pod对象转为terminating状态的同时启动pod关闭过程
5. 端点控制器监控到pod对象的关闭行为时将其从所有匹配到此端点的service资源的端点列表中移除
6. 如果当前pod对象定义了preStop钩子处理器，则在其标记为terminating后即会以同步的方式启动执行
7. pod对象中的容器进程收到停止信号
8. 宽限期结束后，若pod中还存在仍在运行的进程，那么pod对象会收到立即终止的信号
9. kubelet请求apiServer将此pod资源的宽限期设置为0从而完成删除操作，此时pod对于用户已不可见



> 容器的初始化

初始化容器是在pod的主容器启动之前要运行的容器，主要是做一些主容器的前置工作，它具有两大特征：

1. 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么kubernetes需要重启它直到成功完成
2. 初始化容器必须按照定义的顺序执行，当且仅当前一个成功之后，后面的一个才能运行

初始化容器有很多的应用场景，下面列出的是最常见的几个：

- 提供主容器镜像中不具备的工具程序或自定义代码
- 初始化容器要先于应用容器串行启动并运行完成，因此可用于延后应用容器的启动直至其依赖的条件得到满足

这个也表现在command属性中  比如command: ['sh', '-c', 'until ping 192.168.90.15 -c 1 ; do echo waiting for reids...; sleep 2; done;'] 这样必须ping通了这个IP才会启动，不然所有的容器都会卡着



> 钩子函数

钩子函数能够感知自身生命周期中的事件，并在相应的时刻到来时运行用户指定的程序代码。

kubernetes在主容器的启动之后和停止之前提供了两个钩子函数：

- post start：容器创建之后执行，如果失败了会重启容器
- pre stop ：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作

钩子处理器支持使用下面三种方式定义动作：

- Exec命令：在容器内执行一次命令

  ```yaml
  ……
    lifecycle:
      postStart: 
        exec:
          command:
          - cat
          - /tmp/healthy
  ……
  ```

- TCPSocket：在当前容器尝试访问指定的socket

  ```yaml
  ……      
    lifecycle:
      postStart:
        tcpSocket:
          port: 8080
  ……
  ```

- HTTPGet：在当前容器中向某url发起http请求

  ```yaml
  ……
    lifecycle:
      postStart:
        httpGet:
          path: / #URI地址
          port: 80 #端口号
          host: 192.168.5.3 #主机地址
          scheme: HTTP #支持的协议，http或者https
  ……
  ```

接下来，以exec方式为例，演示下钩子函数的使用，创建`pod-hook-exec.yaml`文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hook-exec
  namespace: default
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    lifecycle:
      postStart: 
        exec: # 在容器启动的时候执行一个命令，修改掉nginx的默认首页内容
          command: ["/bin/sh", "-c", "echo postStart... > /usr/share/nginx/html/index.html"]
      preStop:
        exec: # 在容器停止之前停止nginx服务
          command: ["/usr/sbin/nginx","-s","quit"]
```

我们试一下

```shell
# 创建pod
[root@master pod]# kubectl create -f pod-hook-exec.yaml
pod/pod-hook-exec created

# 查看pod
[root@master ~]# kubectl get pods  pod-hook-exec -n default -o wide
NAME            READY   STATUS    RESTARTS        AGE   IP            NODE    NOMINATED NODE   READINESS GATES
pod-hook-exec   1/1     Running   7 (6m59s ago)   73m   10.244.1.59   node1   <none>           <none>
  
# 访问pod(注意这个IP只有在node1里才能访问到，应为并没有暴露，所以得去node1里执行这个命令才可以)
[root@master ]# curl 10.244.1.59
postStart...
```



> 容器探测

容器探测用于检测容器中的应用实例是否正常工作，是保障业务可用性的一种传统机制。如果经过探测，实例的状态不符合预期，那么kubernetes就会把该问题实例" 摘除 "，不承担业务流量。kubernetes提供了两种探针来实现容器探测，分别是：

- liveness probes：存活性探针，用于检测应用实例当前是否处于正常运行状态，如果不是，k8s会重启容器
- readiness probes：就绪性探针，用于检测应用实例当前是否可以接收请求，如果不能，k8s不会转发流量

livenessProbe 决定是否重启容器，readinessProbe 决定是否将请求转发给容器。

上面两种探针目前均支持三种探测方式：

- Exec命令：在容器内执行一次命令，如果命令执行的退出码为0，则认为程序正常，否则不正常

  ```yaml
  ……
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
  ……
  ```

- TCPSocket：将会尝试访问一个用户容器的端口，如果能够建立这条连接，则认为程序正常，否则不正常

  ```yaml
  ……      
    livenessProbe:
      tcpSocket:
        port: 8080
  ……
  ```

- HTTPGet：调用容器内Web应用的URL，如果返回的状态码在200和399之间，则认为程序正常，否则不正常

  ```yaml
  ……
    livenessProbe:
      httpGet:
        path: / #URI地址
        port: 80 #端口号
        host: 127.0.0.1 #主机地址
        scheme: HTTP #支持的协议，http或者https
  ……
  ```

我们做个演示

创建`pod-liveness-exec.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      exec:
        command: ["/bin/cat","/tmp/hello.txt"] # 执行一个查看文件的命令
```

创建pod，观察效果

```shell
kubectl create -f pod-liveness-exec.yaml

#查看详情
kubectl describe pods pod-liveness-exec -n default

Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  32s               default-scheduler  Successfully assigned default/pod-liveness-exec to node2
  Normal   Pulled     2s (x2 over 31s)  kubelet            Container image "nginx:1.17.1" already present on machine
  Normal   Created    2s (x2 over 31s)  kubelet            Created container nginx
  Normal   Started    2s (x2 over 31s)  kubelet            Started container nginx
  Warning  Unhealthy  2s (x3 over 22s)  kubelet            Liveness probe failed: /bin/cat: /tmp/hello.txt: No such file or directory
  Normal   Killing    2s                kubelet            Container nginx failed liveness probe, will be restarted

#我们可以看到，没有这个文件
# 观察上面的信息就会发现nginx容器启动之后就进行了健康检查
# 检查失败之后，容器被kill掉，然后尝试进行重启（这是重启策略的作用，后面讲解）
# 稍等一会之后，再观察pod信息，就可以看到RESTARTS不再是0，而是一直增长

kubectl get pods pod-liveness-exec -n default

NAME                READY   STATUS    RESTARTS      AGE
pod-liveness-exec   1/1     Running   3 (20s ago)   110s

# 当然接下来，可以修改成一个存在的文件，比如/tmp/hello.txt，再试，结果就正常了......
```

接下来的两种我们就不一一尝试了，我们贴一下配置

```yaml
#httpget
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:  # 其实就是访问http://127.0.0.1:80/hello  
        scheme: HTTP #支持的协议，http或者https
        port: 80 #端口号
        path: /hello #URI地址
        
# 当然接下来，可以修改成一个可以访问的路径path，比如/，再试，结果就正常了......
```

```yaml
#tcpsocket
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-tcpsocket
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 8080 # 尝试访问8080端口
        
# 当然接下来，可以修改成一个可以访问的端口，比如80，再试，结果就正常了......
```

至此，已经使用liveness Probe演示了三种探测方式，但是查看livenessProbe的子属性，会发现除了这三种方式，还有一些其他的配置，在这里一并解释下：

```shell
[root@master pod]# kubectl explain pod.spec.containers.livenessProbe
FIELDS:
   exec <Object>  
   tcpSocket    <Object>
   httpGet      <Object>
   initialDelaySeconds  <integer>  # 容器启动后等待多少秒执行第一次探测
   timeoutSeconds       <integer>  # 探测超时时间。默认1秒，最小1秒
   periodSeconds        <integer>  # 执行探测的频率。默认是10秒，最小1秒
   failureThreshold     <integer>  # 连续探测失败多少次才被认定为失败。默认是3。最小值是1
   successThreshold     <integer>  # 连续探测成功多少次才被认定为成功。默认是1
```

举个栗子

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80 
        path: /
      initialDelaySeconds: 30 # 容器启动后30s开始探测
      timeoutSeconds: 5 # 探测超时时间为5s
```



> 重启策略

一旦容器探测出现了问题，kubernetes就会对容器所在的Pod进行重启，其实这是由pod的重启策略决定的，pod的重启策略有 3 种，分别如下：

- Always ：容器失效时，自动重启该容器，这也是默认值。
- OnFailure ： 容器终止运行且退出码不为0时重启
- Never ： 不论状态为何，都不重启该容器

重启策略适用于pod对象中的所有容器，首次需要重启的容器，将在其需要时立即进行重启，随后再次需要重启的操作将由kubelet延迟一段时间后进行，且反复的重启操作的延迟时长以此为10s、20s、40s、80s、160s和300s，300s是最大延迟时长。

我们做个演示

创建`pod-restartpolicy.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80
        path: /hello
  restartPolicy: Never # 设置重启策略为Never
```

```shell
kubectl create -f pod-restartpolicy.yaml

#查看Pod详情，发现nginx容器失败
kubectl  describe pods pod-restartpolicy  -n default

Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  34m                default-scheduler  Successfully assigned default/pod-restartpolicy to node1
  Normal   Pulled     34m                kubelet            Container image "nginx:1.17.1" already present on machine
  Normal   Created    34m                kubelet            Created container nginx
  Normal   Started    34m                kubelet            Started container nginx
  Warning  Unhealthy  33m (x3 over 33m)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    33m                kubelet            Stopping container nginx

#过了一活我们查看下详情
[root@master pod]# kubectl  get pods pod-restartpolicy -n default
NAME                READY   STATUS      RESTARTS   AGE
pod-restartpolicy   0/1     Completed   0          35m

#发现并没有重启
```

#### 5.1.3 pod的调度

在默认情况下，一个Pod在哪个Node节点上运行，是由Scheduler组件采用相应的算法计算出来的，这个过程是不受人工控制的。但是在实际使用中，这并不满足的需求，因为很多情况下，我们想控制某些Pod到达某些节点上，那么应该怎么做呢？这就要求了解kubernetes对Pod的调度规则，kubernetes提供了四大类调度方式：

- 自动调度：运行在哪个节点上完全由Scheduler经过一系列的算法计算得出
- 定向调度：NodeName、NodeSelector
- 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity
- 污点（容忍）调度：Taints、Toleration

> 定向调度

定向调度，指的是利用在pod上声明nodeName或者nodeSelector，以此将Pod调度到期望的node节点上。注意，这里的调度是强制的，这就意味着即使要调度的目标Node不存在，也会向上面进行调度，只不过pod运行失败而已。

**首先是NodeName**

我们做下演示

创建`pod-nodename.yaml`如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: node2 # 指定调度到node2节点上
```

```shell
kubectl create -f pod-nodename.yaml

#可以看到真的去了node2
[root@master pod]# kubectl get pods pod-nodename -n default -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
pod-nodename   1/1     Running   0          42s   10.244.2.20   node2   <none>           <none>

# 接下来，删除pod，修改nodeName的值为node3（并没有node3节点）
[root@master pod]# kubectl delete -f pod-nodename.yaml
pod "pod-nodename" deleted
[root@master pod]# vim pod-nodename.yaml
[root@master pod]# kubectl create -f pod-nodename.yaml
pod/pod-nodename created

#再次查看，发现已经向Node3节点调度，但是由于不存在node3节点，所以pod无法正常运行
[root@master pod]# kubectl get pods pod-nodename -n default -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP       NODE    ......
pod-nodename   0/1     Pending   0          6s    <none>   node3   ......  
```

**下面是nodeselector**

NodeSelector用于将pod调度到添加了指定标签的node节点上。它是通过kubernetes的label-selector机制实现的，也就是说，在pod创建之前，会由scheduler使用MatchNodeSelector调度策略进行label匹配，找出目标node，然后将pod调度到目标节点，该匹配规则是强制约束。

1 首先分别为node节点添加标签

```shell
[root@master pod]# kubectl label nodes node1 nodeenv=pro
node/node2 labeled
[root@master pod]# kubectl label nodes node2 nodeenv=test
node/node2 labeled
```

2 创建一个`pod-nodeselector.yaml`文件，并使用它创建Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeSelector: 
    nodeenv: pro # 指定调度到具有nodeenv=pro标签的节点上
```

```shell
kubectl create -f pod-nodeselector.yaml

#我们可以看到真的在node1上
[root@master pod]# kubectl get pods pod-nodeselector -n default -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
pod-nodeselector   1/1     Running   0          14s   10.244.1.63   node1   <none>           <none>

# 接下来，删除pod，修改nodeSelector的值为nodeenv: xxxx（不存在打有此标签的节点）
[root@master pod]# kubectl delete -f pod-nodeselector.yaml
pod "pod-nodeselector" deleted
[root@master pod]# vim pod-nodeselector.yaml
[root@master pod]# kubectl create -f pod-nodeselector.yaml
pod/pod-nodeselector created

#再次查看，发现pod无法正常运行,Node的值为none
[root@master pod]# kubectl get pods  pod-nodeselector -n default -o wide
NAME               READY   STATUS    RESTARTS   AGE    IP       NODE     NOMINATED NODE   READINESS GATES
pod-nodeselector   0/1     Pending   0          110s   <none>   <none>   <none>           <none>


# 查看详情,发现node selector匹配失败的提示
[root@master pod]# kubectl describe pods pod-nodeselector -n default
.......
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  38s (x3 over 2m43s)  default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't match Pod's node affinity/selector.
#可见这个node没有匹配到这里和上面有所区别
```



> 亲和性调度

上面，我们介绍了两种定向调度的方式，使用起来非常方便，但是也有一定的问题，那就是如果没有满足条件的Node，那么Pod将不会被运行，即使在集群中还有可用Node列表也不行，这就限制了它的使用场景。

基于上面的问题，kubernetes还提供了一种亲和性调度（Affinity）。它在NodeSelector的基础之上的进行了扩展，可以通过配置的形式，实现优先选择满足条件的Node进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活。

Affinity主要分为三类：

- nodeAffinity(node亲和性）: 以node为目标，解决pod可以调度到哪些node的问题
- podAffinity(pod亲和性) : 以pod为目标，解决pod可以和哪些已存在的pod部署在同一个拓扑域中的问题
- podAntiAffinity(pod反亲和性) : 以pod为目标，解决pod不能和哪些已存在pod部署在同一个拓扑域中的问题

关于亲和性(反亲和性)使用场景的说明：

**亲和性**：如果两个应用频繁交互，那就有必要利用亲和性让两个应用的尽可能的靠近，这样可以减少因网络通信而带来的性能损耗。

**反亲和性**：当应用的采用多副本部署时，有必要采用反亲和性让各个应用实例打散分布在各个node上，这样可以提高服务的高可用性。



**nodeAffinity**

首先来看一下`NodeAffinity`,这里又分为两种

```markdown
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  节点选择列表
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operat or 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution 优先调度到满足指定的规则的Node，相当于软限制 (倾向)
    preference   一个节点选择器项，与相应的权重相关联
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt
	weight 倾向权重，在范围1-100。
```

```shell
关系符的使用说明:

- matchExpressions:
  - key: nodeenv              # 匹配存在标签的key为nodeenv的节点
    operator: Exists
  - key: nodeenv              # 匹配标签的key为nodeenv,且value是"xxx"或"yyy"的节点
    operator: In
    values: ["xxx","yyy"]
  - key: nodeenv              # 匹配标签的key为nodeenv,且value大于"xxx"的节点
    operator: Gt
    values: "xxx"
```

如果没看懂不要经我们先用`requiredDuringSchedulingIgnoredDuringExecution`写个示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        nodeSelectorTerms:
        - matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

这个我们就不作演示了，效果就是匹配标签`nodeenv`有xxx或者yyy的，没有的话就会报错，一直处于pending状态，提示发现调度失败，提示node选择失败

下面看下`preferredDuringSchedulingIgnoredDuringExecution`的示例

```shell
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-preferred
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      preferredDuringSchedulingIgnoredDuringExecution: # 软限制
      - weight: 1
        preference:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签(当前环境没有)
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

可以发现虽然没有xxx,yyy但是还是可以运行成功，这就是软限制和硬限制的区别

```
NodeAffinity规则设置的注意事项：
    1 如果同时定义了nodeSelector和nodeAffinity，那么必须两个条件都得到满足，Pod才能运行在指定的Node上
    2 如果nodeAffinity指定了多个nodeSelectorTerms，那么只需要其中一个能够匹配成功即可
    3 如果一个nodeSelectorTerms中有多个matchExpressions ，则一个节点必须满足所有的才能匹配成功
    4 如果一个pod所在的Node在Pod运行期间其标签发生了改变，不再符合该Pod的节点亲和性需求，则系统将忽略此变化
```



**podAffinity**

PodAffinity主要实现以运行的Pod为参照，实现让新创建的Pod跟参照pod在一个区域的功能。

首先看一下配置项

```markdown
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  硬限制
    namespaces       指定参照pod的namespace
    topologyKey      指定调度作用域
    labelSelector    标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels    指多个matchExpressions映射的内容
  preferredDuringSchedulingIgnoredDuringExecution 软限制
    podAffinityTerm  选项
      namespaces      
      topologyKey
      labelSelector
        matchExpressions  
          key    键
          values 值
          operator
        matchLabels 
    weight 倾向权重，在范围1-100
```

```markdown
topologyKey用于指定调度时作用域,例如:
    如果指定为kubernetes.io/hostname，那就是以Node节点为区分范围
	如果指定为beta.kubernetes.io/os,则以Node节点的操作系统类型来区分
```

可以看到也是分两种

首先看下`requiredDuringSchedulingIgnoredDuringExecution`的示例

配置表达的意思是：新Pod必须要与拥有标签nodeenv=xxx或者nodeenv=yyy的pod在同一Node上，显然现在没有这样pod。

```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-required
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    podAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: podenv
            operator: In
            values: ["xxx","yyy"]
        topologyKey: kubernetes.io/hostname
```

当前如果没有这样的pod（拥有标签nodeenv=xxx或者nodeenv=yyy）则会一直处于pending状态

如果有则会运行

`preferredDuringSchedulingIgnoredDuringExecution`的话一个意思，只是不再强制要求即使没有这样的pod当前pod也能运行起来，这里就不做示范了



**PodAntiAffinity**

PodAntiAffinity主要实现以运行的Pod为参照，让新创建的Pod跟参照pod不在一个区域中的功能。

它的配置方式和选项跟PodAffinty是一样的，这里不再做详细解释，直接示范。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podantiaffinity-required
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    podAntiAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配podenv的值在["pro"]中的标签
          - key: podenv
            operator: In
            values: ["pro"]
        topologyKey: kubernetes.io/hostname
```

上面配置表达的意思是：新Pod必须要与拥有标签nodeenv=pro的pod不在同一Node上。当然也是分为硬限制和软限制。软限制我们也不做示范了和上面一个意思。



> 污点和容忍调度

**污点调度**

Node被设置上污点之后就和Pod之间存在了一种相斥的关系，进而拒绝Pod调度进来，甚至可以将已经存在的Pod驱逐出去。

污点的格式为：`key=value:effect`, key和value是污点的标签，effect描述污点的作用，支持如下三个选项：

- PreferNoSchedule：kubernetes将尽量避免把Pod调度到具有该污点的Node上，除非没有其他节点可调度
- NoSchedule：kubernetes将不会把Pod调度到具有该污点的Node上，但不会影响当前Node上已存在的Pod
- NoExecute：kubernetes将不会把Pod调度到具有该污点的Node上，同时也会将Node上已存在的Pod驱离



使用kubectl设置和去除污点的命令示例如下：

![avatar](https://picture.zhanghong110.top/docsify/20200605021831545.png)

```
# 设置污点
kubectl taint nodes node1 key=value:effect

# 去除污点
kubectl taint nodes node1 key:effect-

# 去除所有污点
kubectl taint nodes node1 key-
```



> 使用kubeadm搭建的集群，默认就会给master节点添加一个污点标记,所以pod就不会调度到master节点上。



**容忍调度**

上面介绍了污点的作用，我们可以在node上添加污点用于拒绝pod调度上来，但是如果就是想将一个pod调度到一个有污点的node上去，这时候应该怎么做呢？这就要使用到**容忍**。

> 污点就是拒绝，容忍就是忽略，Node通过污点拒绝pod调度上去，Pod通过容忍忽略拒绝

我们看下容忍的命令

```shell
[root@master pod]# kubectl explain pod.spec.tolerations
......
FIELDS:
   key       # 对应着要容忍的污点的键，空意味着匹配所有的键
   value     # 对应着要容忍的污点的值
   operator  # key-value的运算符，支持Equal和Exists（默认）
   effect    # 对应污点的effect，空意味着匹配所有影响
   tolerationSeconds   # 容忍时间, 当effect为NoExecute时生效，表示pod在Node上的停留时间
```

我们示例一个yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  tolerations:      # 添加容忍
  - key: "tag"        # 要容忍的污点的key
    operator: "Equal" # 操作符
    value: "example"    # 容忍的污点的value
    effect: "NoExecute"   # 添加容忍的规则，这里必须和标记的污点规则相同
```





### 5.2 pod控制器

Pod是kubernetes的最小管理单元，在kubernetes中，按照pod的创建方式可以将其分为两类：

- 自主式pod：kubernetes直接创建出来的Pod，这种pod删除后就没有了，也不会重建
- 控制器创建的pod：kubernetes通过控制器创建的pod，这种pod删除了之后还会自动重建

Pod控制器是管理pod的中间层，使用Pod控制器之后，只需要告诉Pod控制器，想要多少个什么样的Pod就可以了，它会创建出满足条件的Pod并确保每一个Pod资源处于用户期望的目标状态。如果Pod资源在运行中出现故障，它会基于指定策略重新编排Pod。



在kubernetes中，有很多类型的pod控制器，每种都有自己的适合的场景，常见的有下面这些：

- ReplicationController：比较原始的pod控制器，已经被废弃，由ReplicaSet替代
- ReplicaSet：保证副本数量一直维持在期望值，并支持pod数量扩缩容，镜像版本升级
- Deployment：通过控制ReplicaSet来控制Pod，并支持滚动升级、回退版本
- Horizontal Pod Autoscaler：可以根据集群负载自动水平调整Pod的数量，实现削峰填谷
- DaemonSet：在集群中的指定Node上运行且仅运行一个副本，一般用于守护进程类的任务
- Job：它创建出来的pod只要完成任务就立即退出，不需要重启或重建，用于执行一次性任务
- Cronjob：它创建的Pod负责周期性任务控制，不需要持续后台运行
- StatefulSet：管理有状态应用



#### 5.2.1 ReplicaSet

ReplicaSet的主要作用是**保证一定数量的pod正常运行**，它会持续监听这些Pod的运行状态，一旦Pod发生故障，就会重启或重建。同时它还支持对pod数量的扩缩容和镜像版本的升降级。

ReplicaSet的资源清单文件：

```yaml
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: rs
spec: # 详情描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

在这里面，需要新了解的配置项就是`spec`下面几个选项：

- replicas：指定副本数量，其实就是当前rs创建出来的pod的数量，默认为1

- selector：选择器，它的作用是建立pod控制器和pod之间的关联关系，采用的Label Selector机制

  在pod模板上定义label，在控制器上定义选择器，就可以表明当前控制器能管理哪些pod了

- template：模板，就是当前控制器创建pod所使用的模板板，里面其实就是前一章学过的pod的定义

我们来创建一个实列看下

创建`pc-replicaset.yaml`文件，内容如下：

```yaml
apiVersion: apps/v1
kind: ReplicaSet   
metadata:
  name: pc-replicaset
  namespace: default
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

```shell
kubectl create -f pc-replicaset.yaml

# 查看rs
# DESIRED:期望副本数量  
# CURRENT:当前副本数量  
# READY:已经准备好提供服务的副本数量
[root@master pod]# kubectl get rs pc-replicaset -n default -o wide
NAME            DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         SELECTOR
pc-replicaset   3         3         3       2m15s   nginx        nginx:1.17.1   app=nginx-pod

# 查看当前控制器创建出来的pod
# 这里发现控制器创建出来的pod的名称是在控制器名称后面拼接了-xxxxx随机码
[root@master pod]# kubectl get pod -A
NAMESPACE     NAME                             READY   STATUS             RESTARTS          AGE
default       pc-replicaset-d8bg8              1/1     Running            0                 2m24s
default       pc-replicaset-hqf7f              1/1     Running            0                 2m24s
default       pc-replicaset-zg4vj              1/1     Running            0                 2m24s
```

```shell
# 扩容缩容
# 编辑rs的副本数量，修改spec:replicas: 4即可
kubectl edit rs pc-replicaset -n default

#可以看到变成了4
[root@master pod]# kubectl get rs -n default -o wide
NAME            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
pc-replicaset   4         4         4       16m   nginx        nginx:1.17.1   app=nginx-pod

# 当然也可以直接使用命令实现
# 使用scale命令实现扩缩容， 后面--replicas=n直接指定目标数量即可
[root@master pod]# kubectl scale rs pc-replicaset --replicas=2 -n default
replicaset.apps/pc-replicaset scaled

#看一下
[root@master pod]# kubectl get rs -n default -o wide
NAME            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
pc-replicaset   2         2         2       22m   nginx        nginx:1.17.1   app=nginx-pod
```

```shell
# 镜像的版本变更
# 编辑rs的容器镜像 - image: nginx:1.17.2
[root@k8s-master01 ~]# kubectl edit rs pc-replicaset -n default

# 同样的道理，也可以使用命令完成这个工作
# kubectl set image rs rs名称 容器=镜像版本 -n namespace
kubectl set image rs pc-replicaset nginx=nginx:1.17.2  -n default

#可以看到两种方式都已经是换了版本了，达到了一样的效果
[root@master pod]# kubectl get rs -n default -o wide
NAME            DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES         SELECTOR
pc-replicaset   4         4         4       18m   nginx        nginx:1.17.2   app=nginx-pod
```

```shell
# 删除ReplicaSet
# 使用kubectl delete命令会删除此RS以及它管理的Pod
# 在kubernetes删除RS前，会将RS的replicasclear调整为0，等待所有的Pod被删除后，在执行RS对象的删除
[root@master pod]# kubectl delete rs pc-replicaset -n default
replicaset.apps "pc-replicaset" deleted

# 如果希望仅仅删除RS对象（保留Pod），可以使用kubectl delete命令时添加--cascade=false选项（不推荐）。
[root@master pod]# kubectl delete rs pc-replicaset -n default --cascade=false
replicaset.apps "pc-replicaset" deleted
[root@master pod]# kubectl get pods -n default

# 也可以使用yaml直接删除(推荐)
[root@master pod]# kubectl delete -f pc-replicaset.yaml
replicaset.apps "pc-replicaset" deleted
```



#### 5.2.2 Deployment(Deploy)

为了更好的解决服务编排的问题，kubernetes在V1.2版本开始，引入了Deployment控制器。值得一提的是，这种控制器并不直接管理pod，而是通过管理ReplicaSet来简介管理Pod，即：Deployment管理ReplicaSet，ReplicaSet管理Pod。所以Deployment比ReplicaSet功能更加强大。我们可以看下下面这张图。

![avatar](https://picture.zhanghong110.top/docsify/20200612005524778.png)

Deployment主要功能有下面几个：

- 支持ReplicaSet的所有功能
- 支持发布的停止、继续
- 支持滚动升级和回滚版本

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留历史版本
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

下面我们试一试，创建`pc-deployment.yaml`，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: default
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

```shell
kubectl create -f pc-deployment.yaml --record=true


# 查看deployment
# UP-TO-DATE 最新版本的pod的数量
# AVAILABLE  当前可用的pod的数量
[root@master pod]# kubectl get deploy pc-deployment -n default
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   3/3     3            3           24s

# 查看rs
# 发现rs的名称是在原来deployment的名字后面添加了一个10位数的随机串
[root@master pod]# kubectl get rs -n default
NAME                       DESIRED   CURRENT   READY   AGE
pc-deployment-6f7f65b46d   3         3         3       2m26s

# 查看pod
[root@master pod]#  kubectl get pods -n default
NAME                             READY   STATUS             RESTARTS        AGE
pc-deployment-6f7f65b46d-kj2sk   1/1     Running            0               2m57s
pc-deployment-6f7f65b46d-ltt66   1/1     Running            0               2m57s
pc-deployment-6f7f65b46d-pmlhj   1/1     Running            0               2m57s
```

```shell
#扩容缩容
#扩容
kubectl scale deploy pc-deployment --replicas=5  -n default

#查看deployment
[root@master pod]# kubectl get deploy pc-deployment -n default
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
pc-deployment   5/5     5            5           4m32s

#或者编辑副本数量都可以(注意是这个，直接改yml无效，上面也一样)
kubectl edit deploy pc-deployment -n default
```

```markdown
镜像更新
strategy：指定新的Pod替换旧的Pod的策略， 支持两个属性：
  type：指定策略类型，支持两种策略
    Recreate：在创建出新的Pod之前会先杀掉所有已存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本Pod
  rollingUpdate：当type为RollingUpdate时生效，用于为RollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

这个我们不演示了，写两个配置文件看下

```yaml
#滚动更新
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate:
      maxSurge: 25% 
      maxUnavailable: 25%
```

```yaml
#重建更新
spec:
  strategy: # 策略
    type: Recreate # 重建更新
```

```shell
#更新
kubectl set image deployment pc-deployment nginx=nginx:1.17.2 -n default
```



 版本回退

deployment支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能，下面具体来看.

kubectl rollout： 版本升级相关功能，支持下面的选项：

- status	显示当前升级状态
- history   显示 升级历史记录
- pause    暂停版本升级过程
- resume   继续已经暂停的版本升级过程
- restart    重启版本升级过程
- undo 回滚到上一级版本（可以使用--to-revision回滚到指定版本）

```shell
#显示当前升级版本的状态
kubectl rollout status deploy pc-deployment -n default

#查看升级的历史记录
kubectl rollout history deploy pc-deployment -n default

REVISION  CHANGE-CAUSE
1         kubectl create --filename=pc-deployment.yaml --record=true
2         kubectl create --filename=pc-deployment.yaml --record=true
#可以看到有一次升级记录

#看下版本
[root@master pod]# kubectl get deploy -n default -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
pc-deployment   2/2     2            2           4m44s   nginx        nginx:1.17.2   app=nginx-pod

# 版本回滚
# 这里直接使用--to-revision=1回滚到了1版本， 如果省略这个选项，就是回退到上个版本
kubectl rollout undo deployment pc-deployment --to-revision=1 -n default

# 再看下版本发现回退了
[root@master pod]# kubectl get deploy -n default -o wide
NAME            READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
pc-deployment   2/2     2            2           5m42s   nginx        nginx:1.17.1   app=nginx-pod
```



金丝雀发布

Deployment控制器支持控制更新过程中的控制，如“暂停(pause)”或“继续(resume)”更新操作。

比如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

```shell
#更新版本，并且暂停
kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n default && kubectl rollout pause deployment pc-deployment  -n default
#我们看下更新情况，可以看到暂停了
kubectl rollout status deploy pc-deployment -n default
#继续更新
kubectl rollout resume deploy pc-deployment -n default
#发现更新成功
[root@master ~]# kubectl get rs -n default -o wide
NAME                       DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES         SELECTOR
pc-deployment-6f7f65b46d   0         0         0       3h1m   nginx        nginx:1.17.1   app=nginx-pod,pod-template-hash=6f7f65b46d
pc-deployment-86f4996797   0         0         0       178m   nginx        nginx:1.17.2   app=nginx-pod,pod-template-hash=86f4996797
pc-deployment-cf7c57879    2         2         2       12m    nginx        nginx:1.17.4   app=nginx-pod,pod-template-hash=cf7c57879
```



删除操作

```shell
# 删除deployment，其下的rs和pod也将被删除
[root@master pod]# kubectl delete -f pc-deployment.yaml
deployment.apps "pc-deployment" deleted

#或者
kubectl delete deploy [name] -n [namespace]
```



#### 5.2.3 Horizontal Pod Autoscaler(HPA)

在以上的两个控制器中，我们已经可以实现通过手工执行`kubectl scale`命令实现Pod扩容或缩容，但是这显然不符合Kubernetes的定位目标--自动化、智能化。 Kubernetes期望可以实现通过监测Pod的使用情况，实现pod数量的自动调整，于是就产生了Horizontal Pod Autoscaler（HPA）这种控制器。

HPA可以获取每个Pod利用率，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现Pod的数量的调整。其实HPA与之前的Deployment一样，也属于一种Kubernetes资源对象，它通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数，这是HPA的实现原理。

![avatar](https://picture.zhanghong110.top/docsify/20200608155858271.png)

这个我们来演示一下



> 注意这是必须组件,根据我们的版本,搞一个对应版本的,我们这应该是0.6.X版本的

```shell
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.1/components.yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      hostNetwork: true
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-preferred-address-types=InternalIP
        - --kubelet-insecure-tls
        image: k8s.gcr.io/metrics-server/metrics-server:v0.6.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 4443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```

这个修改了三处

```markdown
args添加了
  - --kubelet-preferred-address-types=InternalIP
  - --kubelet-insecure-tls
增加hostNetwork: true
```

!>这边不修改HOST可能导致失败，镜像拉取问题

```shell
#进入/root运行
kubectl apply -f components.yaml
#成功运行
[root@master ~]# kubectl get pods -A
kube-system   metrics-server-755b5d5c47-z82l4   1/1     Running   0              42s
#使用kubectl top node 查看资源使用情况
[root@master pod]# kubectl top node
NAME     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
master   208m         5%     947Mi           46%       
node1    48m          1%     360Mi           16%       
node2    38m          0%     336Mi           15% 
[root@master pod]# kubectl top pod -n kube-system
NAME                              CPU(cores)   MEMORY(bytes)   
coredns-6d8c4cb4d-2zkjt           2m           16Mi            
coredns-6d8c4cb4d-5pfmx           2m           16Mi            
etcd-master                       24m          58Mi            
kube-apiserver-master             59m          245Mi           
kube-controller-manager-master    29m          49Mi            
kube-flannel-ds-5wf72             2m           31Mi            
kube-flannel-ds-b9d4n             4m           37Mi            
kube-flannel-ds-pk86x             5m           21Mi            
kube-proxy-2mttn                  3m           21Mi            
kube-proxy-khblr                  1m           18Mi            
kube-proxy-n2vcs                  1m           19Mi            
kube-scheduler-master             6m           21Mi            
metrics-server-755b5d5c47-z82l4   5m           21Mi
```

创建一个`pc-hpa-pod.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        resources: # 资源配额
          limits:  # 限制资源（上限）
            cpu: "1" # CPU限制，单位是core数
          requests: # 请求资源（下限）
            cpu: "2"  # CPU限制，单位是core数
```

```shell
#创建deployment(删除换成delete就可以)
kubectl create -f pc-hpa-pod.yaml
#创建service
kubectl expose deployment nginx --type=NodePort --port=80 -n default
#查看
kubectl get deployment,pod,svc -n default

[root@master pod]# kubectl get deployment,pod,svc -n default
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1      1            1          57m

NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-8b55d946b-jbzql   1/1     Running   0          4m5s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        26h
service/nginx        NodePort    10.108.172.77   <none>        80:31666/TCP   57m
```

部署HPA，创建`pc-hpa.yaml`

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: pc-hpa
  namespace: default
spec:
  minReplicas: 1  #最小pod数量
  maxReplicas: 10 #最大pod数量
  targetCPUUtilizationPercentage: 2 # CPU使用率指标，注意这个百分之2是为了比较容易压测到
  scaleTargetRef:   # 指定要控制的nginx信息
    apiVersion: apps/v1  #这个卡了我好久
    kind: Deployment
    name: nginx
```

```shell
#启动
kubectl create -f pc-hpa.yaml
#查看HPA
[root@master pod]# kubectl get hpa -n default
NAME     REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
pc-hpa   Deployment/nginx   0%/2%     1         10        1          9m19s
```

压力测试,为了简单我们采用postman的runner而不是jmeter,简单循环一万次，目的是占用CPU超过百分之2

```shell 
#我们看下刚才哪个nginx到了哪个node上，可以看到是node1（这边的话service暴露了其实三个节点的IP都可以访问）
[root@master pod]# kubectl get pod  nginx-f87cbb8b5-2hp5n  -n default -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
nginx-f87cbb8b5-2hp5n   1/1     Running   0          20m   10.244.1.100   node1   <none>           <none>

#我们对http://192.168.191.131:32723进行压力测试
#看下结果，感动哭了（一波N折），发现它随着压力动态扩容了
[root@master pod]# kubectl get hpa -n default
NAME     REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
pc-hpa   Deployment/nginx   2%/2%     1         10        1          2m39s
[root@master pod]# kubectl get pods -n default
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8b55d946b-l95jj   1/1     Running   0          53m
[root@master pod]# kubectl get hpa -n default
NAME     REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
pc-hpa   Deployment/nginx   3%/2%     1         10        1          2m42s
[root@master pod]# kubectl get pods -n default
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8b55d946b-jbzql   1/1     Running   0          5s
nginx-8b55d946b-l95jj   1/1     Running   0          53m
#过了一段时间，没请求发现它缩回去了
kubectl get pods -n default
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8b55d946b-l95jj   1/1     Running   0          58m
```

#### 5.2.4 DaemonSet

DaemonSet类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本。一般适用于日志收集、节点监控等场景。也就是说，如果一个Pod提供的功能是节点级别的（每个节点都需要且只需要一个），那么这类Pod就适合使用DaemonSet类型的控制器创建。



![avatar](https://picture.zhanghong110.top/docsify/20200612010223537.png)

DaemonSet控制器的特点：

- 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
- 当节点从集群中移除时，Pod 也就被垃圾回收了

首先看下资源文件清单

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

创建`pc-daemonset.yaml`做个演示

```yaml
apiVersion: apps/v1
kind: DaemonSet      
metadata:
  name: pc-daemonset
  namespace: default
spec: 
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

运行

```shell
kubectl create -f  pc-daemonset.yaml

#查看下
[root@master pod]# kubectl get ds -n default -o wide
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS   IMAGES         SELECTOR
pc-daemonset   2         2         2       2            2           <none>          40s   nginx        nginx:1.17.1   app=nginx-pod

#发现每个节点都有一个
[root@master pod]# kubectl get pods -n default -o wide
NAME                 READY   STATUS    RESTARTS   AGE    IP           NODE    NOMINATED NODE   READINESS GATES
pc-daemonset-p88q8   1/1     Running   0          117s   10.244.1.2   node2   <none>           <none>
pc-daemonset-tdffb   1/1     Running   0          117s   10.244.2.2   node1   <none>           <none>

# 删除daemonset
[root@k8s-master01 ~]# kubectl delete -f pc-daemonset.yaml
daemonset.apps "pc-daemonset" deleted
```

#### 5.2.5 Job

Job，主要用于负责**批量处理(一次要处理指定数量任务)**短暂的**一次性(每个任务仅运行一次就结束)**任务。Job特点如下：

- 当Job创建的pod执行成功结束时，Job将记录成功结束的pod数量
- 当成功结束的pod达到指定的数量时，Job将完成执行

资源清单

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定job需要成功运行Pods的次数。默认值: 1
  parallelism: 1 # 指定job在任一时刻应该并发运行Pods的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定job可运行的时间期限，超过时间还未结束，系统将会尝试进行终止。
  backoffLimit: 6 # 指定job失败后进行重试的次数。默认是6
  manualSelector: true # 是否可以使用selector选择器选择pod，默认是false
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或者OnFailure
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```

```markdown
关于重启策略设置的说明：
    如果指定为OnFailure，则job会在pod出现故障时重启容器，而不是创建pod，failed次数不变
    如果指定为Never，则job会在pod出现故障时创建新的pod，并且故障pod不会消失，也不会重启，failed次数加1
    如果指定为Always的话，就意味着一直重启，意味着job任务会重复去执行了，当然不对，所以不能设置为Always
```

我们演示下

创建`pc-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job      
metadata:
  name: pc-job
  namespace: default
spec:
  manualSelector: true
  selector:
    matchLabels:
      app: counter-pod
  template:
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

```shell
kubectl create -f pc-job.yaml
#查看下
[root@master pod]# kubectl get job -n default -o wide  -w
NAME     COMPLETIONS   DURATION   AGE   CONTAINERS   IMAGES         SELECTOR
pc-job   1/1           29s        36s   counter      busybox:1.30   app=counter-pod

#观察pod,可见任务结束后会变为Completed
[root@master pod]# kubectl get pods -n default -w
NAME           READY   STATUS      RESTARTS   AGE
pc-job-xnlm5   0/1     Completed   0          90s

# 接下来，调整下pod运行的总数量和并行数量 即：在spec下设置下面两个选项
#  completions: 6 # 指定job需要成功运行Pods的次数为6
#  parallelism: 3 # 指定job并发运行Pods的数量为3
#  然后重新运行job，观察效果，此时会发现，job会每次运行3个pod，总共执行了6个pod
[root@master pod]# kubectl get pods -n default -w
NAME           READY   STATUS    RESTARTS   AGE
pc-job-b2f26   1/1     Running   0          19s
pc-job-f8nfr   1/1     Running   0          19s
pc-job-htmwq   1/1     Running   0          19s
pc-job-b2f26   0/1     Completed   0          29s
pc-job-c4jbq   0/1     Pending     0          0s
pc-job-c4jbq   0/1     Pending     0          0s
pc-job-b2f26   0/1     Completed   0          29s
pc-job-c4jbq   0/1     ContainerCreating   0          0s
pc-job-c4jbq   1/1     Running             0          1s
pc-job-f8nfr   0/1     Completed           0          30s
pc-job-cwsxz   0/1     Pending             0          0s
pc-job-cwsxz   0/1     Pending             0          0s
pc-job-htmwq   0/1     Completed           0          30s
pc-job-f8nfr   0/1     Completed           0          30s
pc-job-cwsxz   0/1     ContainerCreating   0          0s
pc-job-gl6ms   0/1     Pending             0          0s
pc-job-gl6ms   0/1     Pending             0          0s
pc-job-gl6ms   0/1     ContainerCreating   0          0s
pc-job-htmwq   0/1     Completed           0          30s
pc-job-gl6ms   1/1     Running             0          2s
pc-job-cwsxz   1/1     Running             0          2s
pc-job-c4jbq   0/1     Completed           0          28s
pc-job-c4jbq   0/1     Completed           0          28s
pc-job-cwsxz   0/1     Completed           0          28s
pc-job-cwsxz   0/1     Completed           0          28s
pc-job-gl6ms   0/1     Completed           0          28s
pc-job-gl6ms   0/1     Completed           0          28s

#删除
kubectl delete -f pc-job.yaml
```

#### 5.2.6 CronJob(CJ)

CronJob控制器以 Job控制器资源为其管控对象，并借助它管理pod资源对象，Job控制器定义的作业任务在其控制器资源创建之后便会立即执行，但CronJob可以以类似于Linux操作系统的周期性任务作业计划的方式控制其运行**时间点**及**重复运行**的方式。也就是说，**CronJob可以在特定的时间点(反复的)去运行job任务**。

![avatar](https://picture.zhanghong110.top/docsify/20200618213149531.png)

模板清单

```yaml
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运行时间点,用于控制任务在什么时间执行
  concurrencyPolicy: # 并发执行策略，用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业
  failedJobHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
  startingDeadlineSeconds: # 启动作业错误的超时时长
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象;下面其实就是job的定义
    metadata:
    spec:
      completions: 1
      parallelism: 1
      activeDeadlineSeconds: 30
      backoffLimit: 6
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
        matchExpressions: 规则
          - {key: app, operator: In, values: [counter-pod]}
      template:
        metadata:
          labels:
            app: counter-pod
        spec:
          restartPolicy: Never 
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```

```markdown
需要重点解释的几个选项：
schedule: cron表达式，用于指定任务的执行时间
    */1    *      *    *     *
    <分钟> <小时> <日> <月份> <星期>

    分钟 值从 0 到 59.
    小时 值从 0 到 23.
    日 值从 1 到 31.
    月 值从 1 到 12.
    星期 值从 0 到 6, 0 代表星期日
    多个时间可以用逗号隔开； 范围可以用连字符给出；*可以作为通配符； /表示每...
concurrencyPolicy:
    Allow:   允许Jobs并发运行(默认)
    Forbid:  禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行
    Replace: 替换，取消当前正在运行的作业并用新作业替换它
```

我们演示下

创建`pc-cronjob.yaml`

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pc-cronjob
  namespace: default
  labels:
    controller: cronjob
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

```shell
kubectl create -f pc-cronjob.yaml

#cronjob
[root@master pod]# kubectl get cronjobs -n default
NAME         SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
pc-cronjob   */1 * * * *   False     0        <none>          30s

# 查看job
[root@master pod]# kubectl get jobs -n default
NAME                  COMPLETIONS   DURATION   AGE
pc-cronjob-27432751   1/1           28s        38s

#查看pod
[root@master pod]# kubectl get pods -n default
NAME                        READY   STATUS      RESTARTS   AGE
pc-cronjob-27432751-bwdhw   0/1     Completed   0          74s
pc-cronjob-27432752-x8g4g   1/1     Running     0          14s


# 删除cronjob
kubectl  delete -f pc-cronjob.yaml
```



### 5.3 service详解

在kubernetes中，pod是应用程序的载体，我们可以通过pod的ip来访问应用程序，但是pod的ip地址不是固定的，这也就意味着不方便直接采用pod的ip对服务进行访问。

为了解决这个问题，kubernetes提供了Service资源，Service会对提供同一个服务的多个pod进行聚合，并且提供一个统一的入口地址。通过访问Service的入口地址就能访问到后面的pod服务。

![avatar](https://picture.zhanghong110.top/docsify/1626783758946.png)

Service在很多情况下只是一个概念，真正起作用的其实是kube-proxy服务进程，每个Node节点上都运行着一个kube-proxy服务进程。当创建Service的时候会通过api-server向etcd写入创建的service的信息，而kube-proxy会基于监听的机制发现这种Service的变动，然后**它会将最新的Service信息转换成对应的访问规则**。

![avatar](https://picture.zhanghong110.top/docsify/20200509121254425.png)

```shell
# 10.97.97.97:80 是service提供的访问入口
# 当访问这个入口的时候，可以发现后面有三个pod的服务在等待调用，
# kube-proxy会基于rr（轮询）的策略，将请求分发到其中一个pod上去
# 这个规则会同时在集群内的所有节点上都生成，所以在任何一个节点上访问都可以。
[root@node1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0
```

kube-proxy目前支持三种工作模式

#### 5.3.1 userspace 模式

userspace模式下，kube-proxy会为每一个Service创建一个监听端口，发向Cluster IP的请求被Iptables规则重定向到kube-proxy监听的端口上，kube-proxy根据LB算法选择一个提供服务的Pod并和其建立链接，以将请求转发到Pod上。  该模式下，kube-proxy充当了一个四层负责均衡器的角色。由于kube-proxy运行在userspace中，在进行转发处理时会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率比较低。

![avatar](https://picture.zhanghong110.top/docsify/20200509151424280.png)



#### 5.3.2 iptables 模式

iptables模式下，kube-proxy为service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod IP。  该模式下kube-proxy不承担四层负责均衡器的角色，只负责创建iptables规则。该模式的优点是较userspace模式效率更高，但不能提供灵活的LB策略，当后端Pod不可用时也无法进行重试。

![avatar](https://picture.zhanghong110.top/docsify/20200509152947714.png)



#### 5.3.3 ipvs 模式

ipvs模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs规则。ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法。

![avatar](https://picture.zhanghong110.top/docsify/20200509153731363.png)



#### 5.3.4 service类型

```yaml
kind: Service  # 资源类型
apiVersion: v1  # 资源版本
metadata: # 元数据
  name: service # 资源名称
  namespace: dev # 命名空间
spec: # 描述
  selector: # 标签选择器，用于确定当前service代理哪些pod
    app: nginx
  type: # Service类型，指定service的访问方式
  clusterIP:  # 虚拟服务的ip地址
  sessionAffinity: # session亲和性，支持ClientIP、None两个选项
  ports: # 端口信息
    - protocol: TCP 
      port: 3017  # service端口
      targetPort: 5003 # pod端口
      nodePort: 31122 # 主机端口
```

- ClusterIP：默认值，它是Kubernetes系统自动分配的虚拟IP，只能在集群内部访问
- NodePort：将Service通过指定的Node上的端口暴露给外部，通过此方法，就可以在集群外部访问服务
- LoadBalancer：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境支持
- ExternalName： 把集群外部的服务引入集群内部，直接使用



#### 5.3.5 service使用

环境准备：

创建`deployment.yaml`内容如下

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: default
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

```shell
 kubectl create -f deployment.yaml
 
 #查看下情况
[root@master pod]#  kubectl get pods -n default -o wide --show-labels
NAME                             READY   STATUS    RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES   LABELS
pc-deployment-6756f95949-b9zkt   1/1     Running   0          3m33s   10.244.1.10   node2   <none>           <none>            app=nginx-pod,pod-template-hash=6756f95949
pc-deployment-6756f95949-qgmjb   1/1     Running   0          3m33s   10.244.1.11   node2   <none>           <none>            app=nginx-pod,pod-template-hash=6756f95949
pc-deployment-6756f95949-vdmqh   1/1     Running   0          3m33s   10.244.2.10   node1   <none>           <none>            app=nginx-pod,pod-template-hash=6756f95949


# 为了方便后面的测试，修改下三台nginx的index.html页面（三台修改的IP地址不一致）
# kubectl exec -it pc-deployment-6756f95949-vdmqh -n default /bin/sh
# echo "10.244.2.10" > /usr/share/nginx/html/index.html（意思就是将输出改为IP）
# exit

#修改完毕之后，访问测试
[root@master pod]# curl 10.244.1.10
10.244.1.10
[root@master pod]# curl 10.244.1.11
10.244.1.11
[root@master pod]# curl 10.244.2.10
10.244.2.10
```

> 下面我们开始演示

**ClusterIP类型的Service**

创建`service-clusterip.yaml`文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
  namespace: default
spec:
  selector:
    app: nginx-pod
  clusterIP: 10.97.97.97 # service的ip地址，如果不写，默认会生成一个
  type: ClusterIP
  ports:
  - port: 80  # Service端口       
    targetPort: 80 # pod端口
```

```shell
#创建service
kubectl create -f service-clusterip.yaml

[root@master pod]# kubectl get svc -n default -o wide
NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes          ClusterIP   10.96.0.1     <none>        443/TCP   43h   <none>
service-clusterip   ClusterIP   10.97.97.97   <none>        80/TCP    26s   app=nginx-pod

# 查看service的详细信息
# 在这里有一个Endpoints列表，里面就是当前service可以负载到的服务入口
[root@master pod]# kubectl get svc -n default -o wide
NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes          ClusterIP   10.96.0.1     <none>        443/TCP   43h   <none>
service-clusterip   ClusterIP   10.97.97.97   <none>        80/TCP    26s   app=nginx-pod
[root@master pod]# kubectl describe svc service-clusterip -n default
Name:              service-clusterip
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx-pod
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.97.97.97
IPs:               10.97.97.97
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.10:80,10.244.1.11:80,10.244.2.10:80
Session Affinity:  None
Events:            <none>

#访问下
 curl 10.97.97.97:80
```

**endpoint**

Endpoint是kubernetes中的一个资源对象，存储在etcd中，用来记录一个service对应的所有pod的访问地址，它是根据service配置文件中selector描述产生的。

一个Service由一组Pod组成，这些Pod通过Endpoints暴露出来，**Endpoints是实现实际服务的端点集合**。换句话说，service和pod之间的联系是通过endpoints实现的。

```shell
[root@master pod]# kubectl get endpoints -n default -o wide
NAME                ENDPOINTS                                      AGE
kubernetes          192.168.191.130:6443                           43h
service-clusterip   10.244.1.10:80,10.244.1.11:80,10.244.2.10:80   24m
```

负载分发策略

对Service的访问被分发到了后端的Pod上去，目前kubernetes提供了两种负载分发策略：

- 如果不定义，默认使用kube-proxy的策略，比如随机、轮询

- 基于客户端地址的会话保持模式，即来自同一个客户端发起的所有请求都会转发到固定的一个Pod上

  此模式可以使在spec中添加`sessionAffinity:ClientIP`选项

```shell
#循环访问测试
[root@master pod]# while true;do curl 10.97.97.97:80; sleep 5; done;
10.244.2.10
10.244.2.10
10.244.1.10
10.244.2.10
10.244.1.10
10.244.1.11
10.244.1.10
10.244.2.10

#我们给service加上选项
sessionAffinity: ClientIP

#再试一下，发现固定了
[root@master pod]# while true;do curl 10.97.97.97:80; sleep 5; done;
10.244.1.10
10.244.1.10
10.244.1.10
10.244.1.10
10.244.1.10
10.244.1.10

#删除
[root@master pod]# kubectl delete -f service-clusterip.yaml
service "service-clusterip" deleted
```

**HeadLiness类型的Service**

当不想要service提供负载均衡，我们可以采用无头服务，只能用域名访问，我们来做个实列

创建`service-headliness.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: default
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
  - port: 80    
    targetPort: 80
```

```shell
kubectl create -f service-headliness.yaml

#获取service看下情况,可见没有IP
[root@master pod]# kubectl get svc service-headliness -n default -o wide
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service-headliness   ClusterIP   None         <none>        80/TCP    42s   app=nginx-pod

[root@master pod]# kubectl get pods -A
NAMESPACE     NAME                              READY   STATUS    RESTARTS        AGE
default       pc-deployment-6756f95949-b9zkt    1/1     Running   1 (133m ago)    4h41m
default       pc-deployment-6756f95949-qgmjb    1/1     Running   1 (133m ago)    4h41m
default       pc-deployment-6756f95949-vdmqh    1/1     Running   1 (133m ago)    4h41m


#我们随便进入一台看一下默认的域名解析
kubectl exec -it  pc-deployment-6756f95949-b9zkt -n default /bin/sh

#查看下默认的域名解析服务器
cat /etc/resolv.conf
#得到如下信息
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

#然后我们看下这个域名解析到了哪里（注意域名的组成为service名+命名空间+svc.cluster.local（这个可以配置不配的话默认就是这个））
dig @10.96.0.10 service-headliness.default.svc.cluster.local
#可以看到结果这个域名指向了这三台机器，这时候就可以通过ip访问到了。
;; ANSWER SECTION:
service-headliness.default.svc.cluster.local. 30 IN A 10.244.1.13
service-headliness.default.svc.cluster.local. 30 IN A 10.244.1.12
service-headliness.default.svc.cluster.local. 30 IN A 10.244.2.11
#这时候，发现成功访问
curl 10.244.1.13 

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



 **NodePort类型的Service**

在之前的样例中，创建的Service的ip地址只有集群内部才可以访问，如果希望将Service暴露给集群外部使用，那么就要使用到另外一种类型的Service，称为NodePort类型。NodePort的工作原理其实就是**将service的端口映射到Node的一个端口上**，然后就可以通过`NodeIp:NodePort`来访问service了。

我们直接演示

创建`service-nodeport.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
  namespace: default
spec:
  selector:
    app: nginx-pod
  type: NodePort # service类型
  ports:
  - port: 80
    nodePort: 30002 # 指定绑定的node的端口(默认的取值范围是：30000-32767), 如果不指定，会默认分配
    targetPort: 80
```

```shell 
kubectl create -f service-nodeport.yaml
#查看下状态
kubectl get svc -n default -o wide
service-nodeport     NodePort    10.103.141.167   <none>        80:30002/TCP   48s   app=nginx-pod

# 接下来可以通过电脑主机的浏览器去访问集群中任意一个nodeip的30002端口，即可访问到pod
# 这个我们演示过N次了
```

![avatar](https://picture.zhanghong110.top/docsify/1646133614351.png)



**LoadBalancer类型的Service**

LoadBalancer和NodePort很相似，目的都是向外部暴露一个端口，区别在于LoadBalancer会在集群的外部再来做一个负载均衡设备，而这个设备需要外部环境支持的，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

![avatar](https://picture.zhanghong110.top/docsify/20200510103945494.png)

****



**ExternalName类型的Service**

ExternalName类型的Service用于引入集群外部的服务，它通过`externalName`属性指定外部一个服务的地址，然后在集群内部访问此service就可以访问到外部的服务了。

![avatar](https://picture.zhanghong110.top/docsify/20200510113311209.png)

我们创建`service-externalname.yaml`

```shell
apiVersion: v1
kind: Service
metadata:
  name: service-externalname
  namespace: default
spec:
  type: ExternalName # service类型
  externalName: www.baidu.com  #改成ip地址也可以
```

```shell
kubectl  create -f service-externalname.yaml

#看下
dig @10.96.0.10 service-externalname.default.svc.cluster.local
#结果如下
;; ANSWER SECTION:
;; ANSWER SECTION:
service-externalname.default.svc.cluster.local.	30 IN CNAME www.baidu.com.
www.baidu.com.		30	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	30	IN	A	220.181.38.149
www.a.shifen.com.	30	IN	A	220.181.38.150

#输入以下地址可以看到百度内容
curl 220.181.38.150 
```

#### **5.3.6 Ingress介绍**

在前面课程中已经提到，Service对集群之外暴露服务的主要方式有两种：NotePort和LoadBalancer，但是这两种方式，都有一定的缺点：

- NodePort方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB方式的缺点是每个service需要一个LB，浪费、麻烦，并且需要kubernetes之外设备的支持

基于这种现状，kubernetes提供了Ingress资源对象，Ingress只需要一个NodePort或者一个LB就可以满足暴露多个Service的需求。工作机制大致如下图表示：

![avatar](https://picture.zhanghong110.top/docsify/20200623092808049.png)

> 这张图暴露了，没错，就是参考了黑马的教程嘿嘿。

实际上，Ingress相当于一个7层的负载均衡器，是kubernetes对反向代理的一个抽象，它的工作原理类似于Nginx，可以理解成在**Ingress里建立诸多映射规则，Ingress Controller通过监听这些配置规则并转化成Nginx的反向代理配置 , 然后对外部提供服务**。在这里有两个核心概念：

- ingress：kubernetes中的一个对象，作用是定义请求如何转发到service的规则
- ingress controller：具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发，实现方式有很多，比如Nginx, Contour, Haproxy等等

Ingress（以Nginx为例）的工作原理如下：

1. 用户编写Ingress规则，说明哪个域名对应kubernetes集群中的哪个Service
2. Ingress控制器动态感知Ingress服务规则的变化，然后生成一段对应的Nginx反向代理配置
3. Ingress控制器会将生成的Nginx配置写入到一个运行着的Nginx服务中，并动态更新
4. 到此为止，其实真正在工作的就是一个Nginx了，内部配置了用户定义的请求转发规则

![avatar](https://picture.zhanghong110.top/docsify/20200516112704764.png)



下面我们来演示下，首先搭建`ingress-nginx`

```shell
#环境搭建
#在home下建立/home/pod/ingress-controller 

#这里可以去看下匹配版本，新版的也有不同,具体可以看下这个
https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal

#我们这边去它项目如下路径拉取对应版本的配置文件我们这边如下所示，安装Bare metal clusters版本
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/baremetal/deploy.yaml
```

```shell
#去到文件路径下跑起来
kubectl apply -f ./
#查看下
[root@master ingress-controller]# kubectl get pod -n ingress-nginx
NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-nwmg2       0/1     Completed   0          22s
ingress-nginx-admission-patch-bmrhh        0/1     Completed   1          22s
ingress-nginx-controller-f9d6fc8d8-2cjcb   1/1     Running     0          22s
[root@master ingress-controller]# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.104.5.39     <none>        80:32117/TCP,443:32128/TCP   30s
ingress-nginx-controller-admission   ClusterIP   10.106.178.42   <none>        443/TCP                      30s
```

接下来搭建如下图所示服务

![avatar](https://picture.zhanghong110.top/docsify/20200516102419998.png)

`tomcat-nginx.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: default
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```shell
[root@master pod]# kubectl create -f tomcat-nginx.yaml
deployment.apps/nginx-deployment created
deployment.apps/tomcat-deployment created
service/nginx-service created
service/tomcat-service created
[root@master pod]# kubectl get svc -n default
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
kubernetes             ClusterIP      10.96.0.1        <none>          443/TCP        3d2h
nginx-service          ClusterIP      None             <none>          80/TCP         12s
tomcat-service         ClusterIP      None             <none>          8080/TCP       12s
```

**http代理**

主要目的是将某域名转发到指定的`service`上去

创建`ingress-http.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http
  namespace: default
spec:
  ingressClassName: nginx 
  rules:
  - host: nginx.itheima.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: tomcat.itheima.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

```shell
[root@master pod]# kubectl create -f ingress-http.yaml
ingress.networking.k8s.io/ingress-http created
#查看下
[root@master pod]# kubectl get ing ingress-http -n default
NAME           CLASS   HOSTS                                  ADDRESS           PORTS   AGE
ingress-http   nginx   nginx.itheima.com,tomcat.itheima.com   192.168.191.132   80      96s
#详情
[root@master pod]# kubectl describe ing ingress-http  -n default
Name:             ingress-http
Labels:           <none>
Namespace:        default
Address:          192.168.191.132
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                Path  Backends
  ----                ----  --------
  nginx.itheima.com   
                      /   nginx-service:80 (10.244.1.14:80,10.244.1.15:80,10.244.1.30:80 + 3 more...)
  tomcat.itheima.com  
                      /   tomcat-service:8080 (10.244.1.29:8080,10.244.1.31:8080,10.244.2.23:8080)
Annotations:          <none>
Events:
  Type    Reason  Age                 From                      Message
  ----    ------  ----                ----                      -------
  Normal  Sync    82s (x2 over 2m3s)  nginx-ingress-controller  Scheduled for sync

# 接下来,在本地电脑上配置host文件(C:\Windows\System32\drivers\etc\hosts),解析上面的两个域名到192.168.109.100(master)上
#192.168.191.130 nginx.itheima.com
#192.168.191.130 tomcat.itheima.com
# 然后,就可以分别访问tomcat.itheima.com:32117  和  nginx.itheima.com:32117 查看效果了,亲测可以，就不贴图了
```

**https代理**

相比上面那个多一步生成证书

```shell
# 生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=itheima.com"

# 创建密钥
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

创建`ingress-https.yaml`,这个密钥在哪个路径下生成的就会在哪里，这里和配置文件同级

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-https
  namespace: default
spec:
  tls:
  - hosts:
      - nginx.itheima.com
      - tomcat.itheima.com
    secretName: tls-secret # 指定秘钥
  ingressClassName: nginx 
  rules:
  - host: nginx.itheima.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: tomcat.itheima.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

```shell
#不删了会冲突
kubectl delete -f ingress-http.yaml
#运行
kubectl create -f ingress-https.yaml
#查看
root@master pod]# kubectl get ing ingress-https -n default
NAME            CLASS   HOSTS                                  ADDRESS           PORTS     AGE
ingress-https   nginx   nginx.itheima.com,tomcat.itheima.com   192.168.191.132   80, 443   2m51s
#描述
[root@master pod]# kubectl describe ing ingress-https -n default
Name:             ingress-https
Labels:           <none>
Namespace:        default
Address:          192.168.191.132
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  tls-secret terminates nginx.itheima.com,tomcat.itheima.com
Rules:
  Host                Path  Backends
  ----                ----  --------
  nginx.itheima.com   
                      /   nginx-service:80 (10.244.1.14:80,10.244.1.15:80,10.244.1.30:80 + 3 more...)
  tomcat.itheima.com  
                      /   tomcat-service:8080 (10.244.1.29:8080,10.244.1.31:8080,10.244.2.23:8080)
Annotations:          <none>
Events:
  Type    Reason  Age                   From                      Message
  ----    ------  ----                  ----                      -------
  Normal  Sync    2m3s (x2 over 2m57s)  nginx-ingress-controller  Scheduled for sync
#看下ingress服务443定义在哪个端口
[root@master pod]# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.104.5.39     <none>        80:32117/TCP,443:32128/TCP   58m
ingress-nginx-controller-admission   ClusterIP   10.106.178.42   <none>        443/TCP                      58m

#利用上面一个示例改过的HOST访问，换成HTTPS我们可以看到非常成功
```

> 做个总结的话就是依赖于ingress（nginx实现），将域名匹配到service从而分发到Pod上。

### 5.4 数据存储

在前面已经提到，容器的生命周期可能很短，会被频繁地创建和销毁。那么容器在销毁时，保存在容器中的数据也会被清除。这种结果对用户来说，在某些情况下是不乐意看到的。为了持久化保存容器的数据，kubernetes引入了Volume(存储卷)的概念。

Volume是Pod中能够被多个容器访问的共享目录，它被定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下，kubernetes通过Volume实现同一个Pod中不同容器之间的数据共享以及数据的持久化存储。Volume的生命容器不与Pod中单个容器的生命周期相关，当容器终止或者重启时，Volume中的数据也不会丢失。

kubernetes的Volume支持多种类型，比较常见的有下面几个：

- 简单存储：EmptyDir、HostPath、NFS
- 高级存储：PV、PVC
- 配置存储：ConfigMap、Secret

#### 5.4.1 基本存储

**EmptyDir**

EmptyDir是最基础的Volume类型，一个EmptyDir就是Host上的一个空目录。

EmptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为kubernetes会自动分配一个目录，当Pod销毁时， EmptyDir中的数据也会被永久删除。 EmptyDir用途如下：

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）

下面我们演示下

在`一个Pod中`准备两个容器`nginx`和`busybox`，然后声明一个Volume分别挂在到两个容器的目录中，然后nginx容器负责向Volume中写日志，busybox中通过命令将日志内容读到控制台。

创建一个`volume-emptydir.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:  # 将logs-volume挂在到nginx容器中，对应的目录为 /var/log/nginx
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件中内容
    volumeMounts:  # 将logs-volume 挂在到busybox容器中，对应的目录为 /logs
    - name: logs-volume
      mountPath: /logs
  volumes: # 声明volume， name为logs-volume，类型为emptyDir
  - name: logs-volume
    emptyDir: {}
```

```shell
kubectl create -f volume-emptydir.yaml

#查看
[root@master pod]# kubectl get pods volume-emptydir -n default -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
volume-emptydir   2/2     Running   0          5s    10.244.1.38   node2   <none>           <none>

# 通过podIp访问nginx(curl就会使用busybox，这样它的日志就会打印到控制台)
curl 10.244.1.38

# 通过kubectl logs命令查看指定容器的标准输出
[root@master pod]# kubectl logs -f volume-emptydir -n default -c busybox
10.244.0.0 - - [06/Mar/2022:02:06:09 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

#这个例子想表达的感觉不是很明确，总体意思就是nginx和busybox的日志都打印到了这个emptyDir中，我们只是去看了下busybox的成功日志而已。
```

**HostPath**

EmptyDir中数据不会被持久化，它会随着Pod的结束而销毁，如果想简单的将数据持久化到主机中，可以选择HostPath。

HostPath就是将Node主机中一个实际目录挂在到Pod中，以供容器使用，这样的设计就可以保证Pod销毁了，但是数据依据可以存在于Node主机上。

我们演示下

创建一个`volume-hostpath.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    hostPath: 
      path: /root/logs
      type: DirectoryOrCreate  # 目录存在就使用，不存在就先创建后使用
```

```markdown
关于type的值的一点说明：
    DirectoryOrCreate 目录存在就使用，不存在就先创建后使用
    Directory   目录必须存在
    FileOrCreate  文件存在就使用，不存在就先创建后使用
    File 文件必须存在 
    Socket  unix套接字必须存在
    CharDevice  字符设备必须存在
    BlockDevice 块设备必须存在
```

```shell
kubectl create -f volume-hostpath.yaml
#查看
[root@master pod]# kubectl get pods volume-hostpath -n default -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
volume-hostpath   2/2     Running   0          21s   10.244.2.28   node1   <none>           <none>
#访问
curl 10.244.2.28

[root@master pod]# kubectl logs -f volume-hostpath -n default -c busybox
10.244.0.0 - - [06/Mar/2022:02:32:19 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

# 接下来就可以去host的/root/logs目录下查看存储的文件了
###  注意: 下面的操作需要到Pod所在的节点运行（我们是node1）
[root@node1 ~]# ls /root/logs/
access.log  error.log

# 同样的道理，如果在此目录下创建一个文件，到容器中也是可以看到的
```

**NFS**

HostPath可以解决数据持久化的问题，但是一旦Node节点故障了，Pod如果转移到了别的节点，又会出现问题了，此时需要准备单独的网络存储系统，比较常用的用NFS、CIFS。

NFS是一个网络文件存储系统，可以搭建一台NFS服务器，然后将Pod中的存储直接连接到NFS系统上，这样的话，无论Pod在节点上怎么转移，只要Node跟NFS的对接没问题，数据就可以成功访问。

首先要准备nfs的服务器，这里为了简单，直接是master节点做nfs服务器，我们在home下创建nfs

```shell
# 安装nfs服务
yum install nfs-utils -y

# 准备一个共享目录
mkdir /home/nfs/data -pv

# 将共享目录以读写权限暴露给192.168.191.0/24网段中的所有主机,修改/etc/exports
# 查看下
[root@master nfs]# more /etc/exports
/home/nfs/data  192.168.191.0/24(rw,no_root_squash)

# 启动nfs服务
systemctl restart nfs
```

```shell
接下来，要在的每个node节点上都安装下nfs，这样的目的是为了node节点可以驱动nfs设备
# 在node上安装nfs服务，注意不需要启动
# yum install nfs-utils -y
```

接下来，就可以编写pod的配置文件了，创建`volume-nfs.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-nfs
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] 
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    nfs:
      server: 192.168.191.130  #nfs服务器地址
      path: /home/nfs/data #共享文件路径
```

```shell
kubectl create -f volume-nfs.yaml

[root@master pod]# kubectl get pods volume-nfs -n default
NAME         READY   STATUS    RESTARTS   AGE
volume-nfs   2/2     Running   0          17s
#查看下这个目录发现已经有文件了
[root@master pod]# ls /home/nfs/data
access.log  error.log
```

#### 5.4.2 高级存储

前面已经学习了使用NFS提供存储，此时就要求用户会搭建NFS系统，并且会在yaml配置nfs。由于kubernetes支持的存储系统有很多，要求客户全都掌握，显然不现实。为了能够屏蔽底层存储实现的细节，方便用户使用， kubernetes引入PV和PVC两种资源对象。

- PV（Persistent Volume）是持久化卷的意思，是对底层的共享存储的一种抽象。一般情况下PV由kubernetes管理员进行创建和配置，它与底层具体的共享存储技术有关，并通过插件完成与共享存储的对接。
- PVC（Persistent Volume Claim）是持久卷声明的意思，是用户对于存储需求的一种声明。换句话说，PVC其实就是用户向kubernetes系统发出的一种资源需求申请。

![avatar](https://picture.zhanghong110.top/docsify/1647605690.png)

使用了PV和PVC之后，工作可以得到进一步的细分：

- 存储：存储工程师维护
- PV： kubernetes管理员维护
- PVC：kubernetes用户维护

**PV**

PV是存储资源的抽象，下面是资源清单文件:

```YAML
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，与底层真正存储对应
  capacity:  # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes:  # 访问模式
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
```

PV 的关键配置参数说明：

- **存储类型**

  底层实际存储的类型，kubernetes支持多种存储类型，每种存储类型的配置都有所差异

- **存储能力（capacity）**

目前只支持存储空间的设置( storage=1Gi )，不过未来可能会加入IOPS、吞吐量等指标的配置

- **访问模式（accessModes）**

  用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：

  - ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
  - ReadOnlyMany（ROX）： 只读权限，可以被多个节点挂载
  - ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

  `需要注意的是，底层不同的存储类型可能支持的访问模式不同`

- **回收策略（persistentVolumeReclaimPolicy）**

  当PV不再被使用了之后，对其的处理方式。目前支持三种策略：

  - Retain （保留） 保留数据，需要管理员手工清理数据
  - Recycle（回收） 清除 PV 中的数据，效果相当于执行 rm -rf /thevolume/*
  - Delete （删除） 与 PV 相连的后端存储完成 volume 的删除操作，当然这常见于云服务商的存储服务

  `需要注意的是，底层不同的存储类型可能支持的回收策略不同`

- **存储类别**

  PV可以通过storageClassName参数指定一个存储类别

  - 具有特定类别的PV只能与请求了该类别的PVC进行绑定
  - 未设定类别的PV则只能与不请求任何类别的PVC进行绑定

- **状态（status）**

  一个 PV 的生命周期中，可能会处于4中不同的阶段：

  - Available（可用）： 表示可用状态，还未被任何 PVC 绑定
  - Bound（已绑定）： 表示 PV 已经被 PVC 绑定
  - Released（已释放）： 表示 PVC 被删除，但是资源还未被集群重新声明
  - Failed（失败）： 表示该 PV 的自动回收失败

使用NFS作为存储，来演示PV的使用，创建3个PV，对应NFS中的3个暴露的路径。

我们接着之前基本存储的master NFS进入`/home/nfs`，创建pv1,pv2,pv3三个文件夹

```shell
#之后我们进入/etc/exports ,增加如下三个配置，注意不能空行
/home/nfs/pv1 192.168.191.0/24(rw,no_root_squash)
/home/nfs/pv2 192.168.191.0/24(rw,no_root_squash)
/home/nfs/pv3 192.168.191.0/24(rw,no_root_squash)

# 重启服务
[root@nfs ~]#  systemctl restart nfs
```

回到`/home/pod`创建`pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv1
spec:
  capacity: 
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /home/nfs/pv1
    server: 192.168.191.130

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv2
spec:
  capacity: 
    storage: 2Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /home/nfs/pv2
    server: 192.168.191.130
    
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv3
spec:
  capacity: 
    storage: 3Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv3
    server: 192.168.191.130
```

```shell
[root@master pod]# kubectl create -f pv.yaml
persistentvolume/pv1 created
persistentvolume/pv2 created
persistentvolume/pv3 created

#查看
[root@master pod]# kubectl get pv -o wide
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE   VOLUMEMODE
pv1    1Gi        RWX            Retain           Available                                   27s   Filesystem
pv2    2Gi        RWX            Retain           Available                                   27s   Filesystem
pv3    3Gi        RWX            Retain           Available                                   27s   Filesystem
```

> 至此我们准备好了三个PV

**PVC**

PVC是资源的申请，用来声明对存储空间、访问模式、存储类别需求信息。下面是资源清单文件:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  namespace: default
spec:
  accessModes: # 访问模式
  selector: # 采用标签对PV选择
  storageClassName: # 存储类别
  resources: # 请求空间
    requests:
      storage: 5Gi
```

PVC 的关键配置参数说明：

- **访问模式（accessModes）**

用于描述用户应用对存储资源的访问权限

- **选择条件（selector）**

  通过Label Selector的设置，可使PVC对于系统中己存在的PV进行筛选

- **存储类别（storageClassName）**

  PVC在定义时可以设定需要的后端存储的类别，只有设置了该class的pv才能被系统选出

- **资源请求（Resources ）**

  描述对存储资源的请求

创建`pvc.yaml`，申请pv

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: default
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
  namespace: default
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc3
  namespace: default
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```shell
[root@master pod]# kubectl create -f pvc.yaml
persistentvolumeclaim/pvc1 created
persistentvolumeclaim/pvc2 created
persistentvolumeclaim/pvc3 created

#查看pvc
[root@master pod]#  kubectl get pvc  -n default -o wide
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE   VOLUMEMODE
pvc1   Bound    pv1      1Gi        RWX                           49s   Filesystem
pvc2   Bound    pv2      2Gi        RWX                           49s   Filesystem
pvc3   Bound    pv3      3Gi        RWX                           49s   Filesystem

#查看PV,PVC会根据需求空间大小匹配PV
[root@master pod]# kubectl get pv -o wide
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS   REASON   AGE   VOLUMEMODE
pv1    1Gi        RWX            Retain           Bound    default/pvc1                           20m   Filesystem
pv2    2Gi        RWX            Retain           Bound    default/pvc2                           20m   Filesystem
pv3    3Gi        RWX            Retain           Bound    default/pvc3                           20m   Filesystem
```

> 下面我们来测试下

创建`pods.yaml`使用PV,这个文件的意思就是说将/root挂在到pvcx 然后通过pvx 实际挂在到nfs的路径下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc1
        readOnly: false
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod2 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc2
        readOnly: false
```

```shell
kubectl create -f pods.yaml

#看下结果
kubectl get pods -n default -o wide
pod1                                 1/1     Running   0             58s   10.244.2.39   node1   <none>           <none>
pod2                                 1/1     Running   0             58s   10.244.1.57   node2   <none>           <none>

#查看PVC
[root@master pod]# kubectl get pvc -n default -o wide
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE   VOLUMEMODE
pvc1   Bound    pv1      1Gi        RWX                           16m   Filesystem
pvc2   Bound    pv2      2Gi        RWX                           16m   Filesystem
pvc3   Bound    pv3      3Gi        RWX                           16m   Filesystem

#查看PV
[root@master pod]# kubectl get pv -n default -o wide
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS   REASON   AGE   VOLUMEMODE
pv1    1Gi        RWX            Retain           Bound    default/pvc1                           17m   Filesystem
pv2    2Gi        RWX            Retain           Bound    default/pvc2                           17m   Filesystem
pv3    3Gi        RWX            Retain           Bound    default/pvc3                           17m   Filesystem

#看下存储文件
[root@master pod]# more /home/nfs/pv1/out.txt
pod1
pod1
pod1
pod1

[root@master pod]# more /home/nfs/pv2/out.txt
pod2
pod2
pod2
pod2
pod2
```

**5.4.3 生命周期**

PVC和PV是一一对应的，PV和PVC之间的相互作用遵循以下生命周期：

- **资源供应**：管理员手动创建底层存储和PV

- **资源绑定**：用户创建PVC，kubernetes负责根据PVC的声明去寻找PV，并绑定

  在用户定义好PVC之后，系统将根据PVC对存储资源的请求在已存在的PV中选择一个满足条件的

  - 一旦找到，就将该PV与用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了
  - 如果找不到，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV

  PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了

- **资源使用**：用户可在pod中像volume一样使用pvc

  Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。

- **资源释放**：用户删除pvc来释放pv

  当存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用。

- **资源回收**：kubernetes根据pv设置的回收策略进行资源的回收

  对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用

  ![avatar](https://picture.zhanghong110.top/docsify/1647611166.png)


#### 5.4.3 配置存储

**ConfigMap**

ConfigMap是一种比较特殊的存储卷，它的主要作用是用来存储配置信息的。

创建`configmap.yaml`，内容如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
  namespace: default
data:
  info: |
    username:admin
    password:123456
```

```shell
[root@master pod]# kubectl create -f configmap.yaml
configmap/configmap created

#查看下
[root@master pod]# kubectl describe cm configmap -n default
Name:         configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
info:
----
username:admin
password:123456


BinaryData
====

Events:  <none>
```

接下来创建一个`pod-configmap.yaml`，将上面创建的`configmap`挂载进去

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将configmap挂载到目录
    - name: config
      mountPath: /configmap/config
  volumes: # 引用configmap
  - name: config
    configMap:
      name: configmap
```

```shell
kubectl create -f pod-configmap.yaml

[root@master pod]# kubectl get pod pod-configmap -n default
NAME            READY   STATUS    RESTARTS   AGE
pod-configmap   1/1     Running   0          13s

#进入容器
[root@master pod]# kubectl exec -it pod-configmap -n default /bin/sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# cd /configmap/config/
# ls
info
# more info
username:admin
password:123456

# 可以看到映射已经成功，每个configmap都映射成了一个目录
# key--->文件     value---->文件中的内容
# 此时如果更新configmap的内容, 容器中的值也会动态更新
```

 **Secret**

在kubernetes中，还存在一种和ConfigMap非常类似的对象，称为Secret对象。它主要用于存储敏感信息，例如密码、秘钥、证书等等。

首先使用base64对数据进行编码

```shell
echo -n 'admin' | base64 #准备username
YWRtaW4=
echo -n '123456' | base64 #准备password
MTIzNDU2
```

接下来编写`secret.yaml`，并创建Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: default
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

```shell
[root@master pod]# kubectl create -f secret.yaml
secret/secret created

#查看secret
[root@master pod]# kubectl describe secret secret -n default
Name:         secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
username:  5 bytes
```

 创建`pod-secret.yaml`，将上面创建的secret挂载进去：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将secret挂载到目录
    - name: config
      mountPath: /secret/config
  volumes:
  - name: config
    secret:
      secretName: secret
```

```shell
kubectl create -f pod-secret.yaml

[root@master pod]# kubectl get pod pod-secret -n default
NAME         READY   STATUS    RESTARTS   AGE
pod-secret   1/1     Running   0          12s

#进入容器可以看到已经解码了
[root@master pod]# kubectl exec -it pod-secret /bin/sh -n default
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# ls /secret/config/
password  username
# more /secret/config/username
admin
# more /secret/config/password
123456
```

> 至此，已经实现了利用secret实现了信息的编码。

### 5.5 安全认证

!>这一章节没有手动实现，属于了解章节，有兴趣的可以去以下提供的链接看下，不过它这个版本偏老需要注意下

[原文档地址](https://picture.zhanghong110.top/docsify/k8s详细教程.zip)

### 5.6 DASHBOARD

之前在kubernetes中完成的所有操作都是通过命令行工具kubectl完成的。其实，为了提供更丰富的用户体验，kubernetes还开发了一个基于web的用户界面（Dashboard）。用户可以使用Dashboard部署容器化的应用，还可以监控应用的状态，执行故障排查以及管理kubernetes中各种资源。



```shell
# 下载yaml
# wget  https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

# 修改kubernetes-dashboard的Service类型，注意别改错地方
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort  # 新增
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30009  # 新增
  selector:
    k8s-app: kubernetes-dashboard

#启动POD
kubectl create -f recommended.yaml

#看下状态
[root@master pod]# kubectl get pod,svc -n kubernetes-dashboard
NAME                                            READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-79459f84f-7cg7d   1/1     Running   0          89s
pod/kubernetes-dashboard-76dc96b85f-kztzl       1/1     Running   0          89s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.97.239.160   <none>        8000/TCP        89s
service/kubernetes-dashboard        NodePort    10.108.57.210   <none>        443:30009/TCP   89s
```

创建访问账户，获取token

```shell
# 创建账号
[root@master pod]#  kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
serviceaccount/dashboard-admin created

#授权
[root@master pod]# kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin-rb created

# 获取账号token
[root@master pod]# kubectl get secrets -n kubernetes-dashboard | grep dashboard-admin
dashboard-admin-token-c2xlf        kubernetes.io/service-account-token   3      52s

#看下TOKEN信息，每个TOKEN不一样不要照抄
[root@master pod]#  kubectl describe secrets dashboard-admin-token-c2xlf  -n kubernetes-dashboard
Name:         dashboard-admin-token-c2xlf
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: b64610c1-3582-4702-86f8-fbf51dafb4be

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ik9WQ3RLZnp4ZnFhY1JQZ2JyQTFmdkdSWVpCMnZIdW1Ld3YyQmlodWZLM2sifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tYzJ4bGYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYjY0NjEwYzEtMzU4Mi00NzAyLTg2ZjgtZmJmNTFkYWZiNGJlIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.HeAC-l-s_Duoa-4pczInZTjUuDZ-yXpFt7dTc8KEMXNzDAKWWg79pxeDg0aBPadtJCbFJ2MwSXuujuqDq_S743AunUVY7UmRgMAPdf7YfasJjDt3DitjvBU1aMWVy4aEZ3lzFSbw-WlhPN7G4PH_HHTKNpnOMMPD5Wb8PkW0ygs3_eIXuNjYNl28i891bgFl7Y7fvnCxRRM8TxNWatLcBvg7qjSPtxh0Jt6F4HHsXDU7LbBX36a3qkrxZB1fBrctrCod3FTRG6q02Vz1a7aCspQ2E_zCjfC8ZMxZsissLPprB9rr486v2T9smQhMJ8DStj6ligdDv7Vh1qRnt6B5Kg
```

通过浏览器访问Dashboard的UI

在登录页面上输入上面的token

![avatar](https://picture.zhanghong110.top/docsify/1647745927.jpg)

![avatar](https://picture.zhanghong110.top/docsify/1647746122.jpg)

> 至此我们可以通过控制面板操作我们原来绝大多数用命令行完成的事情了

[本次文档所有的POD实列](https://picture.zhanghong110.top/docsify/pod.zip)