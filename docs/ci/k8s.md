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

```shell
scp  -r  $HOME/.kube   node1: $HOME/
```

> 使用推荐: 三种方式应该怎么用 ?

创建/更新资源 使用声明式对象配置 kubectl apply -f XXX.yaml

删除资源 使用命令式对象配置 kubectl delete -f XXX.yaml

查询资源 使用命令式对象管理 kubectl get(describe) 资源名称