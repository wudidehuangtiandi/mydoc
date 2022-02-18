# k8s的初步搭建及使用

>本次以单master，双node作为演示。仅用于初步的学习参考。

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
kubeadm init --image-repository=registry.aliyuncs.com/google_containers

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

199.232.68.133 raw.githubusercontent.com
199.232.68.133 user-images.githubusercontent.com
199.232.68.133 avatars2.githubusercontent.com
199.232.68.133 avatars1.githubusercontent.com
```

> 至此我们一个单master双node的节点就安装完毕了

我们使用`kubectl get nodes`可以看到如下

```
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   16m   v1.23.4
node1    Ready    <none>                 14m   v1.23.4
node2    Ready    <none>                 14m   v1.23.4
```

如果长时间没有READY，三台机`reboot`下就行了。

下面贴以下卸载k8s及日志查看的方式

```
卸载清理k8s

kubeadm reset -f
modprobe -r ipip
lsmod
rm -rf ~/.kube/
rm -rf /etc/kubernetes/
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /etc/systemd/system/kubelet.service
rm -rf /usr/bin/kube*
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf /var/lib/etcd
rm -rf /var/etcd
yum remove -y kubelet kubeadm kubectl docker-ce

日志查看
journalctl -u kubelet | tail -n 300
```

待续