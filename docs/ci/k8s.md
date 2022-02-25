

# k8s的初步搭建及使用

>本次以单master，双node作为演示。仅用于初步的学习参考。

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
这边如果刚才选择了ipvs模式则要增加以下命令
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
# 失败了可以用 kubeadm reset 重置
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

## 四.nginx集群搭建

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

kubectl create deployment nginx --port=8 --replicas=3 --image=nginx:1.14-alpine -n default

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

