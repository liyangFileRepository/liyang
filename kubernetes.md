# KUBERNETES
> kubernetes 是一种管理容器的工具
![Kubernetes 组件](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)
## 组件

1. APISERVER：所有服务访问的唯一入口
2. CrontrollerManager：维持期望的副本数量
3. Scheduler：负责介绍任务，选择合适的节点进行分配任务
4. ETCD：键值对数据库，储存k8s集群所有重要信息（持久化）
5. kubelet：直接跟容器引擎交互，实现容器的生命周期管理
6. kube-proxy：负责写入规则至 IPTABLES、IPVS 实现微服务映射访问的

> CoreDNS、Ingress Controller、Prometheus、Dashboard、Federation

- CoreDNS：可以为集群中的SVC创建一个域名IP的对应关系解析
- IngressController：官方只能实现四层代理，它可以实现七层代理
- prometheus：提供一个k8s集群的监控能力
- Dashboard：给k8s集群提供一个B/S结构访问体系
- Federation：提供一个可以跨集群中心多k8s统一管理功能
- Elk：提供k8s集群日志的统一分析介入平台

## pod

pod按照定义可以分为两种，一种是自主式pod，一种是控制器管理的pod
### 自主式pod

### 控制器管理的pod

- ReplicationController & ReplicaSet & Deployment
  - HPA（HorizatalPodAutoScale）
- StatefulSet
- DaemonSet
- Job，CronJob
---
> 常用的控制器主要是三种，分别是 deployment、statefulset 和 daemonset。

1. ReplicationController 用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的 pod 来代替；而如果异常多出来的容器也会自动回收。在新版本的 kubernetes 中建议使用 ReplicaSet 来取代 ReplicationController 。
ReplicaSet 跟 ReplicationController 没有本质的不同，只是名字不一样，并且ReplicaSet 支持集合式的 selector 。
虽然 RelicaSet 可以独立使用，但一般开始建议使用 <a href='https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/'>Deployment</a> 来自动管理 ReplicaSet ，这样就无需担心跟其他机制的不兼容问题（比如 ReplicaSet 不支持 Rolling-update 但 Deployment 支持）
HorizatalPodAutoScale 仅适用于 Deployment 和 ReplicaSet ，在 V1 版本中仅支持根据 pod 的CPU 利用率扩缩容，在 v1alpha 版本中，支持根据内存和用户自定义的 metric 扩缩容。

2. StatefulSet 是为了解决有状态服务的问题（对应 Deployment 和 ReplicaSets 是为无状态服务而设计），其应用场景包括：
- 稳定的持久化存储，即 pod 重新调度后还是能访问到想通的持久化数据，基于 PVC 来实现
- 稳定的网络标志，即 pod 重新调度后其 PodName 和 HostName 不变，基于 Headless Service（没有 ClusterIP 的 Service）来实现
- 有序部署，有序扩展，即 pod 是有顺序的，在部署还活着扩展的时候要根据定义的顺序依次依次进行（即 0 ~ N-1，在下一个 pod 实现之前所有之前的 pod 必须都是 Running 和 Ready 状态），基于 init containers 来实现
- 有序收缩，有序删除（即从 N-1 ~ 0 ）

3. DaemonSet 确保全部（或者一些）Node 上运行一个 pod 的副本。当有 Node 加入集群时,也会为他们新增一个 pod 。当有 Node 从集群中移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 pod

 

 使用 DaemonSet 的一些典型案例：

 - 运行集群存储 daemon ，例如在每个 Node 上运行 glusterd、ceph
 - 在每个 Node 上运行日志收集 daemon ，例如 fluentd、logstash
 - 在每个 Node 上运行监控 daemon，例如 Prometheus Node Exporter

 Job 负责批处理任务，即仅执行一次的任务，他保证批处理任务的一个或者多个pod成功结束

 

 CronJob 管理基于时间的 Job，即：

 - 在给定时间点只运行一次
 - 周期性地在给定时间点运行

## kubernetes的网络通讯模式

kubernetes的网络模型假定了所有 pod 都在一个可以直接连通的扁平的网络空间中，这在 GCE（Google Computer Engine）里面是现成的网络模型，kubernetes 假定这个网络已经存在。

而在私有云里搭建 kubernetes 集群，就不能假定这个网络已经存在了。我们需要自己实现这个网络假设，将不同节点上的 Docker 容器之间的互相访问先大同，然后运行 kubernetes。（OVS + Tun）

1. 同一个 pod 内的多个容器之间： lo （pause的网络栈）
2. 各 pod 之间的通讯： Overlay Network（全骨干网络）
3. Pod 与 Service 之间的通讯： 各节点的 IPtables 规则

### Flannel

flannel 是 Core OS 团队针对 kubernetes 设计的一个网络规划服务，简单来说，他的功能是让集群中的不同节点主机创建的 Docker 容器都具有群集群唯一的虚拟IP地址。而且他还能在这些 IP 地址之间建立一个覆盖网络（OverlayNetwork），通过这个覆盖网络，将数据包原封不动地传递到目标容器内。

ETCD 之 Flannel 提供说明：

- 存储管理 Flannel 可分配的 IP 地址段资源
- 监控 ETCD 中每个 pod 的实际地址，并在内存中建立维护 pod 节点路由表

---

pod 到外网： pod 向外网发送请求，查找路由表，转发数据包到宿主机的网卡，宿主网卡完成路由选择后，iptables 执行 masquerade ，把源 IP 更改为宿主网卡的 IP ，然后向外网服务器发送请求。

外网访问 pod： Service（Nodeport）

## 集群安装

### 平台规划

#### 1. 单master集群

|   节点名称   | 服务器配置 |   版本   |     网络规划      |
| :----------: | :--------: | :------: | :---------------: |
| k8s-master01 |  2c4g 20G  | CentOS 7 | 192.168.118.40/24 |
|  k8s-node01  |  4c4g 40G  | CentOS 7 | 192.168.118.50/24 |
|  k8s-node02  |  4c4g 40G  | CentOS 7 | 192.168.118.51/24 |

#### 2. 多master集群

|   节点名称   | 服务器配置 |
| :----------: | :--------: |
| k8s-master01 |  2c4g 20G  |
| k8s-master02 |  2c4g 20G  |
|  k8s-node01  |  4c4g 40G  |
|  k8s-node02  |  4c4g 40G  |

### 安装方式

- kubeadm

kueadm 是一个 k8s 部署工具，提供 kubeadm init 和 kubeadm join，用于快速部署 kubernetes 集群

- 二进制包

从 github 下载发行版的二进制包，手动部署每个组件，组成 kubernetes 集群

- kubeasz

https://www.jianshu.com/p/6c982870f7c5

### kubeadm 安装

#### 1. Init

```powershell
## 在三台机器上执行
# 配置IP
[root@k8s-master01 ~]# nmcli con mod ens33 ipv4.address 192.168.118.40/24 ipv4.gateway 192.168.118.2 ipv4.dns 223.5.5.5 ipv4.method manual connection.autoconnect yes
[root@k8s-master01 ~]# nmcli con up ens33
[root@k8s-master01 ~]# nmcli con reload
# 下载yum阿里云镜像源
[root@k8s-master01 ~]# curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
# 修改镜像源中的参数
[root@k8s-master01 ~]# sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
# 安装常用工具
[root@k8s-master01 ~]# yum install -y vim wget bash-completion lrzsz dos2unix net-tools
# 关闭防火墙并开机不自启
[root@k8s-master01 ~]# systemctl disable firewalld --now
# 关闭selinux
[root@k8s-master01 ~]# sed -i '/^SELINUX=/ cSELINUX=permissive' /etc/selinux/config
# 禁用swap分区
[root@k8s-master01 ~]# swapoff -a									# 临时
[root@k8s-master01 ~]# sed -ri 's/.*swap.*/#&/' /etc/fstab			# 永久
# 配置hosts
[root@k8s-master01 ~]# cat >> /etc/hosts << EOF
192.168.118.40 master
192.168.118.50 node1
192.168.118.51 node2
EOF
# 配置桥接的IPv4流量传递到IPtables的链
[root@k8s-master01 ~]# cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
[root@k8s-master01 ~]# sysctl --system
# 时间同步
[root@k8s-master01 ~]# yum install chrony
```

#### 2. Install

##### 2.1 Docker

> https://developer.aliyun.com/mirror/docker-ce?spm=a2c6h.13651102.0.0.3e221b11dfgl3r

```powershell
# 在三台机器上执行
[root@k8s-master01 ~]# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
[root@k8s-master01 ~]# yum list docker-ce.x86_64 --showduplicates | sort -r
[root@k8s-master01 ~]# yum install -y docker-ce
[root@k8s-master01 ~]# cat >> /etc/docker/daemon.json << EOF
{
"registry-mirrors":["https://82m9ar63.mirror.aliyuncs.com"]
}
EOF
[root@k8s-master01 ~]# systemctl restart docker
```



##### 2.2 kubernetes

> https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.3e221b11dfgl3r

```powershell
# 在三台机器上执行
[root@k8s-master01 ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

[root@k8s-master01 ~]# yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0

[root@k8s-master01 ~]# systemctl enable kubelet --now
```



##### 2.3 kubeadm

~~~powershell
# 在master机器上执行
[root@k8s-master01 ~]# kubeadm init \
--apiserver-advertise-address=192.168.118.40 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.18.0 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16

# 在执行kubeadm时可以查看下载的组件
[root@k8s-master01 ~]# docker images
REPOSITORY                                                        TAG       IMAGE ID       CREATED         SIZE
registry.aliyuncs.com/google_containers/kube-proxy                v1.18.0   43940c34f24f   17 months ago   117MB
registry.aliyuncs.com/google_containers/kube-apiserver            v1.18.0   74060cea7f70   17 months ago   173MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.18.0   d3e55153f52f   17 months ago   162MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.18.0   a31f78c7c8ce   17 months ago   95.3MB
registry.aliyuncs.com/google_containers/pause                     3.2       80d28bedfe5d   18 months ago   683kB
registry.aliyuncs.com/google_containers/coredns                   1.6.7     67da37a9a360   19 months ago   43.8MB
registry.aliyuncs.com/google_containers/etcd                      3.4.3-0   303ce5db0e90   22 months ago   288MB

# 在kubeadm安装完成之后，终端会有这么一段命令
[root@k8s-master01 ~]# mkdir -p $HOME/.kube
[root@k8s-master01 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master01 ~]# chown $(id -u):$(id -g) $HOME/.kube/config
# 给集群中加入节点
[root@k8s-master01 ~]# kubeadm join 192.168.118.40:6443 --token 2dbdvj.xnob4lzedaqn29gq \
    --discovery-token-ca-cert-hash sha256:f58169b17e3989ec00f53b7385d52949b6fb7b2cbaec08023ed8560a49bfb6fd 
# 默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：
[root@k8s-master01 ~]# kubeadm create --print-join-command
~~~



##### 2.4 CNI网络插件

~~~powershell
[root@k8s-master01 ~]# wget https://github.com/flannel-io/flannel/blob/24941444a861e7503d3e68722da9aa77c12db104/Documentation/kube-flannel.yml
[root@k8s-master01 ~]# wget https://raw.githubuser.com/coreos/flannel/master/Documentation/kube-flannel.yml

[root@k8s-master01 ~]# kubectl apply -f https://raw.githubuser.com/coreos/flannel/master/Documentation/kube-flannel.yml

[root@k8s-master01 ~]# kubectl get pods -n kube-system
NAME                                   READY   STATUS              RESTARTS   AGE
coredns-7ff77c879f-9qwh7               0/1     ContainerCreating   0          28m
coredns-7ff77c879f-jdxqb               0/1     ContainerCreating   0          28m
etcd-k8s-master01                      1/1     Running             0          28m
kube-apiserver-k8s-master01            1/1     Running             0          28m
kube-controller-manager-k8s-master01   1/1     Running             0          28m
kube-flannel-ds-6wdsw                  1/1     Running             0          60s
kube-flannel-ds-w5hfx                  1/1     Running             0          60s
kube-flannel-ds-xbhpm                  1/1     Running             0          60s
kube-proxy-6pv7b                       1/1     Running             0          19m
kube-proxy-6s48g                       1/1     Running             0          28m
kube-proxy-zrpfn                       1/1     Running             0          19m
kube-scheduler-k8s-master01            1/1     Running             0          28m

~~~

##### 2.5 创建pod

```powershell
[root@k8s-master01 ~]# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
[root@k8s-master01 ~]# kubectl get pods
NAME                    READY   STATUS              RESTARTS   AGE
nginx-f89759699-d5hmx   0/1     ContainerCreating   0          42s
[root@k8s-master01 ~]# kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-d5hmx   1/1     Running   0          54s

[root@k8s-master01 ~]# kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed
[root@k8s-master01 ~]# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
nginx-f89759699-d5hmx   1/1     Running   0          3m    10.244.1.3   k8s-node01   <none>           <none>

[root@k8s-master01 ~]# kubectl get pod,svc
NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-f89759699-d5hmx   1/1     Running   0          3m42s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        35m
service/nginx        NodePort    10.99.118.181   <none>        80:31889/TCP   2m14s
```

http://192.168.118.51:31889/

### 二进制包安装

