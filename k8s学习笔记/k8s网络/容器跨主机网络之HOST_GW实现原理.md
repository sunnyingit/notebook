# 容器跨主机网络之HOST_GW实现原理

容器跨宿主机访问是指容器访问另一个宿主机上的容器。很显然，最基本的前提是*宿主机之间能相互访问*，也就是说宿主机是在同一个网段内。

今天要分享的Flannel 的 host-gw模式实现了容器跨主机网络的访问。

理解host-gw模式需要的知识储备是：
1. [Docker容器网络实现原理](https://github.com/sunnyingit/notebook/blob/master/k8s%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/k8s%E7%BD%91%E7%BB%9C/Docker%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)。
2. 路由表转发规则。


## 路由表
我们先再次复习一下路由表。

"ip route" 命令的输出通常显示操作系统的 IP 路由表，其中列出了每个网络目的地和它们应该使用的下一跳。

下一跳地址就是把数据转发到下一个IP地址。

以下是一个可能的输出示例：

```
default via 192.168.1.1 dev eth0 proto static metric 100
10.0.0.0/8 via 192.168.2.1 dev eth1 proto static metric 200
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10 metric 100
192.168.2.0/24 dev eth1 proto kernel scope link src 192.168.2.10 metric 200
```

这个输出中包含了四个路由表项，每个表项由不同的字段组成，解释如下：
1. 目标网络地址：表示数据包要发送到的网络的IP地址或子网。
2. 下一跳：表示数据包要通过哪个网络接口或路由器发送，可以理解为网关。
3. 设备：表示与下一跳相关联的网络接口。
4. 协议：表示路由表项是由何种协议生成的（例如，静态路由或动态路由）。
5. 度量：表示路由表项的优先级或距离，通常越小越优先。
6. 源IP地址：表示与网络接口相关联的本地IP地址，通常由内核自动添加。

该路由表项的含义是：当匹配不到目标网络时，使用defaut规则，通过eth0设备，把数据转发给192.168.1.1网关。

```
default via 192.168.1.1 dev eth0 proto static metric 100
```
解释如下：
1. default：表示这是默认路由表项，也就是当没有其他路由表项与目标地址匹配时，应该使用该路由表项发送数据包。
2. via 192.168.1.1：表示下一跳网关的 IP 地址，也就是该数据包要通过哪个网关发送。
3. dev eth0：表示使用哪个网络接口连接到下一跳路由器，这里是使用 eth0 网卡。
4. proto static：表示该路由表项是静态路由，即手动配置的路由，而不是通过动态路由协议自动学习的路由。
5. metric 100：表示该路由表项的度量值，通常用于区分优先级。

```
92.168.2.0/24 dev eth1 proto kernel scope link src 192.168.2.10 metric 200
```
解释如下：

1. proto kernel：表示该路由表项是内核自动生成的路由，也就是由 Linux 内核根据系统配置和网络拓扑自动生成的路由。
2. scope link：表示这个路由表项只适用于本地网络接口，也就是本地主机上的网络设备。
3. src 192.168.2.10：表示这个路由表项的源 IP 地址，用于指定将要使用这个路由的数据包的源地址。

因此，该路由表项的含义是：当需要将数据包发送到 192.168.2.0/24 网络范围内的目标地址时，使用 eth1 网络接口，并将源IP地址设置为 192.168.2.10。

由于这是内核自动生成的本地网络路由，因此度量值比静态路由高，为200。

这里有个问题：为什么我们需要修改源IP地址？ 这个问题先放一放，我们接着往下看。

## 准备工作

我们先构建一个跨主机的网络，如图所示：

```


                   +------------------------------------------------------+
                   |                 Host1和Host2可以相互访问               |
                   |                                                      |
                   |                                                      |
                   |                                                      |
                   |                                                      |
+----------------------------------------+             +-----------------------------------------+
|  Host1     +------------------+        |             |  Host2    +-------------------+         |
|            |   Eth0           |        |             |           |  Eth0             |         |
|            |                  |        |             |           |                   |         |
|            +------------------+        |             |           +-------------------+         |
|             ip:10.168.0.1/24           |             |            ip:10.168.0.2/24             |
|                                        |             |                                         |
|                                        |             |           +--------------------+        |
|            +------------------+        |             |           |                    |        |
|            |                  |        |             |   +------->  Cni0              <------+ |
|       +---->   Cni0           <------+ |             |   |       |                    |      | |
|       |    |                  |      | |             |   |       +--------------------+      | |
|       |    +------------------+      | |             |   |         ip:192.168.1.1/24         | |
|       |      ip:192.168.0.1/24       | |             |   |                                   | |
|       |                              | |             |   |                                   | |
|       |                              | |             |   | +--------------+  +-------------+ | |
|       |                              | |             |   | |              |  |             | | |
| +-----+-----+         +------------+ | |             |   | |  Container3  |  | Container4  | | |
| |           |         |            | | |             |   +-+              |  |             +-+ |
| | Container1|         | Container2 | | |             |     |              |  |             |   |
| |           |         |            +-+ |             |     +--------------+  +-------------+   |
| |           |         |            |   |             |     ip:192.168.1.2/24  ip:192.168.1.3/24|
| +-----------+         +------------+   |             |                                         |
| ip:192.168.0.2/24     ip:192.168.0.3/24|             |                                         |
+----------------------------------------+             +-----------------------------------------+
```

这是一个简单的网络拓扑，其中有两个主机Host1和Host2，以及四个容器Container1、Container2、Container3和Container4。

1. Host1和Host2使用10.168.0.0/24子网进行通信。
2. 容器Container1和Container2分别连接到主机Host1的Cni0虚拟交换机上，网段号是192.168.0.0/24。
3. 容器Container3和Container4分别连接到主机Host2的Cni0虚拟交换机上，网段号是192.168.1.0/24。


如果看不懂这个网路部署，请自行复习[Docker容器网络实现原理](https://github.com/sunnyingit/notebook/blob/master/k8s%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/k8s%E7%BD%91%E7%BB%9C/Docker%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)。

Cni0虚拟网卡是K8s初始化的时候，在每个宿主机器上创建的虚拟网卡，类似于Docker创建的Docker0虚拟交换机。

这个网络和Docker唯一的区别是虚拟网卡由Docker0变成了Cni0。

这个这个网络是通过K8s而不是Docker部署的，所以Docker0改成了Cni0，其他的原理都和一样。


## 创建路由规则
我们要做的事情是让Container1跨子网访问Container3。

在[Docker容器网络实现原理](https://github.com/sunnyingit/notebook/blob/master/k8s%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/k8s%E7%BD%91%E7%BB%9C/Docker%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)文章中，我们已经分享了Container1访问Host1，Host2访问Container3的原理。

> 如果不明白，请务必学习，否则你可能看不懂下文。

还有一个前提，Host1又可以访问Host2，因为他们在同一个子网内。

所以，我们只需要实现：当Container1访问Host1时，Host1把数据转发给Host2就行啦。

怎么转发呢，通过路由表呗。

我们给Host1主机添加路由规则：
```
规则1：192.168.1.0/24 via 10.168.0.2 dev Eth0
规则2：192.168.1.0/24 dev cni proto kernel scope link src 192.168.0.1 metric 200
```
解释一下：
1. 规则1表示访问子网192.168.1.0/24的请求通过Eth0转发到网关10.168.0.2上，也就是Host2的ip。
2. 规则2表示访问子网192.168.1.0/24的请求，修改源ip为192.168.0.1，也就是Host1中cni的ip

那Container1(192.168.0.2)跨子网访问Container3(192.168.1.2)的过程如下：
1. Container1发现目标地址(192.168.1.2)和自己不是一个网段，于是把数据转发给Cni0网卡。
2. Cni0网卡在Host1上，于是Host1(10.168.0.1)拿到了转发的数据，发现目标IP(192.168.0.2)和自己不是一个网段，于是查找路由表。
3. Host1匹配到了规则1，于是把数据发送到了Host2上。
4. Host2再把数据发送给了Container3。

这样，我们就实现了容器跨主机网络。

有一个问题我们还没有解释，在host1发送数据时，为什么需要把源ip地址从192.168.0.2改成Cni0的ip地址192.168.0.1？
因为如果不修改源IP地址为Scope link地址，数据包可能会被路由到其他物理链路上，导致通信失败。

## Flannel的Host-Gw的实现

Flannel项目是CoreOS公司主推的容器网络方案。

事实上，Flannel项目本身只是一个框架，真正为我们提供容器网络功能的，是Flannel的后端实现。

而Flannel的Host-Gw网络模式就是上文我们讲的原理。

简单来讲，k8s选择使用Flannel的Host-GW部署网络时，必要的步骤就是：

1. 安装Flannel：使用kubectl命令行工具安装Flannel：
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

2. 使用Host-GW模式：
通过创建kube-flannel-cfg.yml的配置文件指定host-gw模式
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel",
      "delegate": {
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "host-gw"
      }
    }
```

3. 将Flannel部署到Kubernetes集群中：
```
kubectl apply -f kube-flannel.yml
kubectl apply -f kube-flannel-cfg.yml
```

其中kube-flannel.yml配置文件是创建Flannel对象需要的配置文件，其格式为：
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        app: flannel
    spec:
      hostNetwork: true
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.14.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=${NODE_NAME}-eth0
        resources:
          requests:
            cpu: "100m"
        securityContext:
          privileged: true
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
          readOnly: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      volumes:
      - name: run
        hostPath:
          path: /run
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      tolerations:
      - operator: Exists
        effect: NoSchedule
```

4. 测试Flannel：
```
kubectl run --image busybox:1.28 pingtest --restart=Never --rm -it -- ping <ip-address>
```
将<ip-address>替换为要测试的IP地址。这将在Kubernetes集群中创建一个临时的busybox容器，并在其中运行ping命令以测试网络连接性。如果一切正常，您应该能够ping通指定的IP地址。

有个问题：Flannel需要配置容器，Cni0的IP，IP哪里来的，谁来管理这些IP，谁负责配置宿主机的路由规则？

当Flannel启动时，Flannel将为集群中的每个节点分配一个唯一的子网，并为每个Pod分配一个唯一的IP地址。

Flannel通过Etcd和宿主机上的flanneld来维护路由信息和配置IP信息，具体的实现我们不需要了解。


## 总结
理解了容器的网桥模式和路由表后，host-gw的原理就很清楚了，从本文中，我们可以学习到：
1. k8s如何安装Flannel网络插件。
1. Flannel host-gw的核心是基于规则：`<目的容器IP地址段> via <网关的IP地址> dev eth0`来转发网络数据。
2. k8s的集群中所有的节点能互联的话，Flannel必须为节点都创建到其他节点的路由规则。
3. Flannel使用Etcd来维护和管理集群IP。
