---
layout: next
title: 使用kubeadm搭建kubernetes集群
date: 2024-12-02 22:23:52
categories: k8s
tags: k8s
---

# 环境要求
3台Linux虚拟机, 能联网, 我使用的发行版是Rocky Linux 9.4 
最低配置: 2CPU, 2G内存, 20G硬盘
3台虚拟机的IP和主机名如下
```
k8s-master 192.168.52.200 (k8s主节点)
k8s-node1 192.168.52.201 (k8s从节点1)
k8s-node2 192.168.52.202 (k8s从节点2)
```

# 0. 准备操作

## 关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```

## 关闭SELINUX
```
setenforce 0
sed --follow-symlinks -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```
<!-- more -->
## 修改主机名
k8s主节点执行
```
hostnamectl set-hostname k8s-master
```
从节点1执行
```
hostnamectl set-hostname k8s-node1
```
从节点2执行
```
hostnamectl set-hostname k8s-node2
```

## 修改hosts文件
修改所有机器的/etc/hosts文件, 添加三台虚拟机的IP和主机名
```
192.168.52.200 k8s-master 
192.168.52.201 k8s-node1 
192.168.52.202 k8s-node2 
```

## 禁用交换分区
kubelet默认行为是在节点上检测到交换内存时无法启动，所以这里先禁用交换分区。临时禁用交换分区方法:
```
swapoff -a
```
如需要永久禁用交换分区，修改`/etc/fstab`, 删除swap分区那一行的配置。

## 加载内核模块, 再设置内核参数
临时加载内核模块
```
modprobe ip_vs_rr
modprobe br_netfilter
```
每次启动自动加载
vim /etc/modules-load.d/k8s.conf
```
overlay
br_netfilter
```
再设置内核参数
```
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables 	= 1
net.ipv4.ip_forward 				= 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf
```

# 1. 安装containerd, kubeadm, kubelet, kubectl, calico

## 国内机器需要更换YUM国内源
```
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
    -i.bak \
    /etc/yum.repos.d/[Rr]ocky-*.repo
dnf makecache
```

## 所有机器安装containerd
```
dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo # 国内用阿里源
#dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y containerd
```
配置containerd
```
containerd config default > /etc/containerd/config.toml
```
再修改`/etc/containerd/config.toml`
* 把SystemdCGroup的值改成true
* 把sandbox_image改为`registry.aliyuncs.com/google_containers/pause:3.9` (国内用户需要做这一步，把镜像换成阿里的)

启动containerd
```
systemctl daemon-reload
systemctl enable --now containerd
```
查看containerd版本
```
# ctr version
Client:
  Version:  1.7.24
# runc -v
runc version 1.2.2
```

## 所有机器安装kubeadm, kubelet, kubectl
先添加k8s的repo
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
再通过yum安装kubelet, kubeadm, kubectl
```
yum install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
```
查看kubelet, kubeadm, kubectl版本
```
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"28", GitVersion:"v1.28.2", GitCommit:"89a4ea3e1e4ddd7f7572286090359983e0387b2f", GitTreeState:"clean", BuildDate:"2023-09-13T09:34:32Z", GoVersion:"go1.20.8", Compiler:"gc", Platform:"linux/amd64"}

[root@k8s-master yum.repos.d]# kubectl version
Client Version: v1.28.2
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
The connection to the server localhost:8080 was refused - did you specify the right host or port?

[root@k8s-master yum.repos.d]# kubelet --version
Kubernetes v1.28.2
```
说明: kubelet现在每隔几秒就会重启，它陷入了一个等待 kubeadm 指令的死循环, 这是符合预期的。 接下来需要在主节点执行kubeadm init，初始化k8s集群

# 2. 初始化k8s集群

## 在主节点执行kubeadm init
```
kubeadm init --apiserver-advertise-address 192.168.52.200 \
			 --image-repository registry.aliyuncs.com/google_containers \
			 --kubernetes-version v1.28.2 \
			 --pod-network-cidr=198.18.0.0/16
```
参数说明: 
–apiserver-advertise-address：监听地址，填主节点IP
–image-repository：国内用户需指定镜像地址为阿里云的，默认是海外镜像你无法访问。
–kubernetes-version：指定kubernetes的版本
–pod-network-cidr=198.18.0.0/16 (这个cidr表示Pod的IP地址范围，根据你的网络环境自定义，不能和其他IP发生冲突即可) 

执行时间较长，耐心等几分钟。 执行成功后，会打印如下内容，提示你下一步怎么做
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.52.200:6443 --token zk8fth.5psohqfk9lomq0tw \
        --discovery-token-ca-cert-hash sha256:f32851bb6a86cc7f0a394f1d77e1db5b217cde1b0f40909ee3916959519173f7
```
我使用的是root用户，参照上面的提示，只需export环境变量KUBECONFIG，操作如下：
编辑/etc/profile，结尾添加一行
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
使环境变量立即生效
```
source /etc/profile
```

此时，kubectl已经可以查到如下pod, 但coredns pod运行不成功。下一步需要在主节点上安装网络插件
```
kubectl get pods -A
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-66f779496c-8ctsc             0/1     Pending   0          3m11s
kube-system   coredns-66f779496c-hx76v             0/1     Pending   0          3m11s
kube-system   etcd-k8s-master                      1/1     Running   0          3m24s
kube-system   kube-apiserver-k8s-master            1/1     Running   0          3m24s
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          3m24s
kube-system   kube-proxy-89c9k                     1/1     Running   0          3m11s
kube-system   kube-scheduler-k8s-master            1/1     Running   0          3m24s
```

# 3. 在主节点安装calico网络插件
安装网络插件，可以选择calico或者flannel，我这里选calico，安装最新版本
calico安装，我参考了官方文档: https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

1、Install the Tigera Calico operator and custom resource definitions.
```
yum install -y wget
wget https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
```
2、Install Calico by creating the necessary custom resource. 
```
wget https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
修改custom-resources.yaml, 把cidr改成198.18.0.0/16
kubectl create -f custom-resources.yaml
```
3、Confirm that all of the pods are running with the following command.
```
watch kubectl get pods -n calico-system
```
Wait until each pod has the STATUS of Running.

4、Remove the taints on the control plane so that you can schedule pods on it.
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
node/k8s-master untainted
```
5、 Confirm that you now have a node in your cluster with the following command.
```
kubectl get nodes -o wide
NAME         STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                      KERNEL-VERSION                 CONTAINER-RUNTIME
k8s-master   Ready    control-plane   5h16m   v1.28.2   10.206.216.96   <none>        Rocky Linux 9.4 (Blue Onyx)   5.14.0-427.13.1.el9_4.x86_64   containerd://1.7.24
```

如果是国内用户，最常见问题是Pod启动失败，无法拉取镜像。 你可以用kubectl describe查看启动日志，获取拉取失败的镜像信息，然后从国内源拉取，再改一下tag即可
calico安装需要的镜像如下:
```
[root@k8s-master ~]# ctr -n k8s.io image list | awk '{print $1}'
REF
docker.io/calico/apiserver:v3.29.1
docker.io/calico/cni:v3.29.1
docker.io/calico/csi:v3.29.1
docker.io/calico/kube-controllers:v3.29.1
docker.io/calico/node-driver-registrar:v3.29.1
docker.io/calico/node:v3.29.1
docker.io/calico/pod2daemon-flexvol:v3.29.1
docker.io/calico/typha:v3.29.1
quay.io/tigera/operator:v1.36.2
```
国内镜像站: https://docker.aityp.com

举例: 手动拉取docker.io/calico/kube-controllers:v3.29.1镜像，再打tag
```
ctr -n k8s.io images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.29.1
ctr -n k8s.io images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.29.1 docker.io/calico/kube-controllers:v3.29.1
```

## 安装calicoctl
calicoctl安装，参考了官方文档: https://docs.tigera.io/calico/latest/operations/calicoctl/install
```
curl -L https://github.com/projectcalico/calico/releases/download/v3.29.1/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
cp calicoctl /usr/bin/
```

查看calico Pod是否创建成功
```
kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-658d97c59c-gftzg   1/1     Running   0          12m
kube-system   calico-node-x7dhl                          1/1     Running   0          12m
kube-system   coredns-66f779496c-8ctsc                   1/1     Running   0          62m
kube-system   coredns-66f779496c-hx76v                   1/1     Running   0          62m
kube-system   etcd-k8s-master                            1/1     Running   0          62m
kube-system   kube-apiserver-k8s-master                  1/1     Running   0          62m
kube-system   kube-controller-manager-k8s-master         1/1     Running   0          62m
kube-system   kube-proxy-89c9k                           1/1     Running   0          62m
kube-system   kube-scheduler-k8s-master                  1/1     Running   0          62m
```
此时，通过calicoctl可以查到主节点k8s-master
```
# calicoctl node status
Calico process is running.

IPv4 BGP status
No IPv4 peers found.

IPv6 BGP status
No IPv6 peers found.

# calicoctl get nodes
NAME
k8s-master
```

# 4. 把其他两个从节点加入集群

## 先在Master节点上获取token
```
kubeadm token list | awk '{print $1}'
TOKEN
zk8fth.5psohqfk9lomq0tw
```
默认token 24小时内过期，如果过期了，可以在主节点上重新创建新token
```
kubeadm token create
```

## 再从主节点上获取--discovery-token-ca-cert-hash的值
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
f32851bb6a86cc7f0a394f1d77e1db5b217cde1b0f40909ee3916959519173f7
```

## 最后,在两个从节点执行kubeadm join命令，将从节点加入集群:
```
kubeadm join --token zk8fth.5psohqfk9lomq0tw \
		192.168.52.200:6443 \
		--discovery-token-ca-cert-hash sha256:f32851bb6a86cc7f0a394f1d77e1db5b217cde1b0f40909ee3916959519173f7

[preflight] Running pre-flight checks
        [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
将kubelet设为开机自启动
```
systemctl enable kubelet.service
```

## 回到主节点，查看从节点是否加入成功
需要等几分钟，直到所有Pod创建完成，如下:
```
watch kubectl get pods -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-7f67554766-dfzc9          1/1     Running   0          22m
calico-apiserver   calico-apiserver-7f67554766-xsdlk          1/1     Running   0          22m
calico-system      calico-kube-controllers-6b7df74554-qxqgf   1/1     Running   0          22m
calico-system      calico-node-7lhhg                          1/1     Running   0          10m
calico-system      calico-node-ksrfp                          1/1     Running   0          22m
calico-system      calico-node-lnt8q                          1/1     Running   0          10m
calico-system      calico-typha-6855cf6f56-dzfd4              1/1     Running   0          22m
calico-system      calico-typha-6855cf6f56-z77jt              1/1     Running   0          10m
calico-system      csi-node-driver-cn6h5                      2/2     Running   0          10m
calico-system      csi-node-driver-pqrw8                      2/2     Running   0          10m
calico-system      csi-node-driver-rhd5m                      2/2     Running   0          22m
kube-system        coredns-5dd5756b68-84wqr                   1/1     Running   0          5h32m
kube-system        coredns-5dd5756b68-hc9xm                   1/1     Running   0          5h32m
kube-system        etcd-k8s-master                            1/1     Running   0          5h32m
kube-system        kube-apiserver-k8s-master                  1/1     Running   0          5h32m
kube-system        kube-controller-manager-k8s-master         1/1     Running   0          5h32m
kube-system        kube-proxy-2hmd5                           1/1     Running   0          10m
kube-system        kube-proxy-4h2cz                           1/1     Running   0          10m
kube-system        kube-proxy-6qptd                           1/1     Running   0          5h32m
kube-system        kube-scheduler-k8s-master                  1/1     Running   0          5h32m
tigera-operator    tigera-operator-c7ccbd65-rddsw             1/1     Running   0          5h25m
```
可以查到两个从节点已经Ready
```
# kubectl get nodes
NAME         STATUS   ROLES           AGE     VERSION
k8s-master   Ready    control-plane   5h33m   v1.28.2
k8s-node1    Ready    <none>          11m     v1.28.2
k8s-node2    Ready    <none>          11m     v1.28.2

# calicoctl node status
calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 10.206.216.98 | node-to-node mesh | up    | 08:48:32 | Established |
| 10.206.216.99 | node-to-node mesh | up    | 08:49:43 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

# calicoctl get nodes
NAME
k8s-master
k8s-node1
k8s-node2
```

# 5. 测试集群，在集群上部署Nginx

## 创建Nginx deployment, 设置副本数为3
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3 # 副本数设置为3，假设集群中有三个节点，每个节点上会尝试运行一个Nginx实例（但k8s会根据资源情况和调度策略来实际分配）
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```
使用`kubectl apply -f nginx-deployment.yaml`命令部署Nginx
```
# kubectl get pods  -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-2bk2q   1/1     Running   0          22m   198.18.169.130   k8s-node2    <none>           <none>
nginx-deployment-7c79c4bf97-5pdr5   1/1     Running   0          22m   198.18.235.199   k8s-master   <none>           <none>
nginx-deployment-7c79c4bf97-w4s8h   1/1     Running   0          22m   198.18.36.66     k8s-node1    <none>           <none>
```
可以看到，每个节点都成功部署了一个Nginx

## 删除所有节点的Nginx Pod
```
[root@k8s-master ~]# kubectl delete deploy nginx-deployment
deployment.apps "nginx-deployment" deleted
```

# 6. 从节点退出集群

在主节点下删除节点
```
kubectl delete node k8s-node1
kubectl delete node k8s-node2
```
从节点退出后，如果想重新加入，在从节点上执行如下命令
```
systemctl stop kubelet
rm -rf /etc/kubernetes/*
kubeadm join ... # 从节点上执行，重新加入集群
```

# 参考
【1】 [https://medium.com/@redswitches/install-kubernetes-on-rocky-linux-9-b01909d6ba72](https://medium.com/@redswitches/install-kubernetes-on-rocky-linux-9-b01909d6ba72)
【2】 [https://juejin.cn/post/7286669548740591673](https://juejin.cn/post/7286669548740591673)
【3】 [https://cloud.tencent.com/developer/article/2255721](https://cloud.tencent.com/developer/article/2255721)