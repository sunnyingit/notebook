# K8s中Service的实现原理

Service是k8s里重要的服务对象，解决两个问题：
1. Pod的访问，Pod 的IP不是固定的，我们不能通过固定ip去访问Pod，所以抽象了Service对象。
2. Pod的负载均衡

理解Service需要的知识储备是：
1. 理解Pod的概念
2. 理解iptables的网络转发
3. 理解内核IPVS技术


## Service对象

在Kubernetes中，Service是一种抽象。

Service通过标签选择器来选择属于特定标签的Pod，并提供了一个抽象的方式来访问这些Pod。
具体来说，Service通过以下方式与Pod相关联：

1. 选择器

例如，以下Service定义选择标签为“app=my-app”的Pod：

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
```

2. 端口
Service还定义了一个或多个端口，用于监听来自客户端的流量，当客户端向Service发出请求时，Service会将请求转发到关联的Pod。
例如，上述Service定义将监听80端口，并将请求转发到Pod的8080端口。

现在的问题是，如何访问到Service，可以想到的方法是给Service一个IP地址，这样通过IP就可以访问到Service。

什么叫做给Service一个IP呢，是不是在宿主创建一个虚拟网卡然后绑定一个IP，这样宿主机都需要创建虚拟网卡并绑定同一个IP，会发生IP地址冲突，从而导致网络连接问题。

创建虚拟网卡的路走不通，我们可以用另外一种技术实现，那就是iptables。


## iptables
iptables是Service实现的核心原理，我们必须要掌握这块知识才能充分理解Service的实现。

首先我们需要了解防火墙。Linux操作系统中防火墙不是真实的一堵墙，本质上只是一个钩子(hook)，用于对网络数据进行过滤，修改，重定向，以及网络地址转换（nat）。

防火墙实现的工具值Linux内核软件netfilter，netfilter读取配置文件，也就是iptables文件进行对网络数据的处理。

防火墙主要作用于网络层。

```
                                  +--------------------------------------+
                                  |                                      |
                                  |           用户空间                    |
                                  |                                      |
                                  +-----^------------------+-------------+
                                        |                  |
              +-----------------------------------------------------------------------------------------------+
              |                         |                  |                                                  |
              |                   +-----+------+     +-----v----+                                             |
              |                   |Input       |     |OutPut    +----------+                                  |
              |                   +-----^------+     +----------+          |                                  |
              |                         |                                  |                                  |
+--------+    |  +------------+   +-----+-------+   +-------------+    +---v------+      +-----------------+  |    +-----------+
| 网卡    +------>+ PRE_ROUTING+-->+ Router      +---> Forward     +---->Router    +------> PostRouting      +-----> 网卡       |
+--------+    |  +------------+   +-------------+   +-------------+    +----------+      +-----------------+  |    +-----------+
              |                                                                                               |
              |                             操作系统内核                                                        |
              +-----------------------------------------------------------------------------------------------+

```

当主机网卡接收到网络数据后，把网络数据发送到操作系统内核，操作系统通过网络协议可以获取到请求的相关信息，比如目标网络IP，目前端口等，Netfilter的执行过程：
1. PRE_ROUTING阶段：获取到目标网络IP之后，在这个节段可以对网络数据进行修改，比如修改目标网络IP，目标网络端口等。
2. Input阶段：内核拿到数据后，把数据给到应用程序之前，对数据进行过滤，修改等操作发生在这个阶段。
3. FORWARD阶段：经过路由选择之后要转发的数据包经过此Hook，也就是数据发往默认网关之前的阶段。
4. OUTPUT阶段：用户程序产生的数据包在写入到内核前需要经过这个节段。
5. PostRouting阶段：刚刚通过 FORWARD 和 OUTPUT 关卡的数据包要通过一次路由选择由哪个接口送往网络中，经过路由之后的数据包要经过这个阶段。

上面提到的这些处于网络协议栈的“关卡”，在 iptables 的术语里叫做“链（chain）”，内置的链包括上面提到的5个：

1. PreRouting
2. Forward
3. Input
4. Output
5. PostRouting

一般的场景里面，数据包的流向基本是：

1. 到本主机某进程的报文：PreRouting -> Input -> Process -> Output -> PostRouting。
2. 由本主机路由转发的报文：PreRouting -> Forward -> PostRouting。


iptables 默认有五条链chain，分别对应上面提到的五个阶段：Pre_routing/Input/Forward/Output/Post_routing，每个阶段都可以创建规则。

每一条“链”上的一串规则里面有些功能是相似的，比如，A 类规则都是对 IP 或者端口进行过滤，B 类规则都是修改报文，我们考虑能否将这些功能相似的规则放到一起，这样管理 iptables 规则会更方便。

iptables 把具有相同功能的规则集合叫做“表”，并且定一个四种表：
1. filter表：负责过滤功能，与之对应的内核模块是 iptables_filter。
2. nat(Network Address Translation) 表：网络地址转换功能，比如 SNAT、可以对源IP进行修改，DNAT可以对目标IP进行修改，与之对应的内核模块是 iptables_nat。
3. mangle表：解包报文、修改并封包，与之对应的内核模块是 iptables_mangle。
4. raw表：不使用Nat表，启用的连接追踪机制；与之对应的内核模块是 iptables_raw。

说明白就是三大类操作：
1. 修改报文，规则是mangle。
2. 过滤报文，规则是filter。
3. 修改源IP地址，目标IP地址，规则是nat。


## Service的实现

现在我们回到Service的实现上来。

假如Service的IP是192.168.0.1, 而后端Pod的IP是192.168.0.2， 我们试想一下，在宿主中iptables中在Pre_Routing阶段创建一个Nat规则：将目标地址是192.168.0.1修改为目标地址为192.168.0.2。

这样，当宿主机访问Service IP 192.168.0.1时，宿主机在Pre-Routing之前就会把目标地址改为192.168.0.2，这样就实现了Pod的访问啦。

原理就是这样简单！

实际上，Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。

例如 my-service 的 Service一旦它被提交给 K8s，那么 kube-proxy 就可以通过 Service的Informer感知到这个事件。

而作为对这个事件的响应，kube-proxy就会在*所有的宿主机*上创建这样一条iptables规则:

```
-A KUBE-SERVICES -d 192.168.0.1/32 -p tcp -m comment --comment "default/my-service: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

解释如下：
1. -d: 匹配目标地址
2. -p: 匹配协议,如tcp,udp,icmp
3. --dport: 匹配目标端口
4. -j：执行的动作。

执行的动作通常包括：
1. ACCEPT ：接收数据包。
2. DROP ：丢弃数据包。
3. REDIRECT ：重定向、映射、透明代理。
4. SNAT ：源地址转换。
5. DNAT ：目标地址转换。
6. MASQUERADE ：IP伪装（NAT），用于ADSL。
7. LOG ：日志记录。
8. SEMARK : 添加SEMARK标记以供网域内强制访问控制（MAC）

综上所述，上面k8s的命令的解释为：凡是目的地址是 192.168.0.1、目的端口是 80 的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3 的 iptables 链进行处理。

KUBE-SVC-NWV5X2332I4OT4T3对应的动作是:

```
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/my-service:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 192.168.0.1/32 -m comment --comment "default/my-service:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 192.168.0.1:8080
```

我们前面已经看到，192.168.0.1正是这个 Service 的 VIP， 192.168.0.1:8080是Pod的ip。

为了方便管理和保存规则，可以将自定义的规则保存到一个文件中，并使用iptables-restore命令加载和应用规则文件。例如，假设将规则保存到一个名为iptables.rules的文件中，则可以使用以下命令加载和应用规则：
```
iptables-restore < iptables.rules
```

## IPVS模式

其实，通过上面的讲解，你可以看到，kube-proxy 通过 iptables 处理 Service 的过程，其实需要在宿主机上设置相当多的 iptables 规则。
kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的。

不难想到，当你的宿主机上有大量 Pod 的时候，成百上千条 iptables 规则不断地被刷新，会大量占用该宿主机的 CPU 资源，甚至会让宿主机“卡”在这个过程中。
所以说，一直以来，基于 iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。

IPVS 模式的 Service，就是解决为了解决这个问题。


IPVS 在内核中的实现其实也是基于 Netfilter 的NAT模式实现的。


IPVS 的实现原理是在 Linux 内核中创建一个虚拟 IP 地址，并将这个地址映射到多个实际的服务器 IP 地址上。

如果虚拟的IP是Service IP, 实际的服务器IP是Pod IP，这样也能实现Service到Pod的转发。

下面是一个简单的 IPVS 创建过程的例子：

假设有三台服务器，它们的 IP 地址分别为 192.168.1.1、192.168.1.2、192.168.1.3，需要使用 IPVS 在它们之间进行负载均衡。

1. 安装 IPVS 内核模块：
```
modprobe ip_vs
modprobe ip_vs_rr
```

2. 创建虚拟网卡：
使用 ipvsadm 命令创建一个虚拟 IP 地址，并将它映射到实际的服务器 IP 地址上：
```
ipvsadm -A -t 192.168.1.100:80 -s rr
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.1:80 -m
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.2:80 -m
ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.3:80 -m
```

其中，-A 表示添加一个虚拟服务，-t 指定虚拟 IP 地址和端口号，-s 指定负载均衡算法，这里使用了 rr 表示轮询算法；-a 表示添加一台实际服务器，-r 指定实际服务器 IP 地址和端口号，-m 表示使用 masquerading 模式进行负载均衡。

3. 开启 IP 转发：
在 Linux 系统中，默认情况下 IP 转发是关闭的，需要使用以下命令开启 IP 转发：
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```
4. 查看列出当前系统中正在运行的 IPVS 服务：
```
ipvsadm -ln

IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP  192.168.1.100:80 rr
  -> 192.168.1.1:80       Masq    1      0          0
  -> 192.168.1.2:80       Masq    1      0          0
  -> 192.168.1.3:80       Masq    1      0          0
```
该输出表示当前系统中运行了一个 TCP 类型的 IPVS 服务，使用了轮询（rr）算法，虚拟服务的 IP 地址和端口号为 192.168.1.100:80，实际服务器的 IP 地址和端口号分别为 192.168.1.1:80、192.168.1.2:80、192.168.1.3:80，权重均为 1，当前没有连接。

k8s设置网络时，会为我们自动创建以上步骤，这里只是学习一下原理。

IPVS模式的优点是并不需要在宿主机上为每个Pod设置iptables规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。

建议你为kube-proxy设置`–proxy-mode=ipvs`来开启这个功能。它为 Kubernetes 集群规模带来的提升，还是非常巨大的。


## DNS服务器
Service 和 Pod 都会被分配对应的 DNS A 记录（从域名解析 IP 的记录）。

对于 ClusterIP 模式的 Service 来说（比如我们上面的例子），它的 A 记录的格式是：..svc.cluster.local。当你访问这条 A 记录的时候，它解析到的就是该 Service 的 VIP 地址。

该A记录的格式通常为：
```
<service-name>.<namespace>.svc.cluster.local
```

其中，<service-name>是Service对象的名称，<namespace>是Service对象所在的命名空间。svc.cluster.local是集群DNS的域名后缀。

例如，如果在名为default的命名空间中创建了名为my-service的Service对象，则该Service的虚拟IP地址为<ClusterIP>，A记录为my-service.default.svc.cluster.local。


而对于指定了 clusterIP=None 的 Headless Service 来说，它的 A 记录的格式也是：
```
<service-name>.<namespace>.svc.cluster.local。
```

但是，当你访问这条 A 记录的时候，它返回的是所有被代理的 Pod 的 IP 地址的集合。当然，如果你的客户端没办法解析这个集合的话，它可能会只会拿到第一个 Pod 的 IP 地址。

此外，对于 ClusterIP 模式的 Service 来说，它代理的 Pod 被自动分配的 A 记录的格式是：
```
<pod-ip-address>.<namespace-name>.pod.cluster.local
```

其中，<pod-ip-address> 是 Pod 的 IP 地址，<namespace-name> 是该 Pod 所在的命名空间名称。

例如，如果一个 Pod 的 IP 地址是 10.0.0.2，它所在的命名空间是 my-namespace，那么它的 DNS A 记录就是：
10.0.0.2.my-namespace.pod.cluster.local

这个 DNS A 记录可以用来直接访问该 Pod。

对于Headless Service绑定的pod的A记录格式是：
```
<podName>.<serviceName>.<namespace>.svc.cluster.local
```
其中<podName> 是 Pod 的名称，<serviceName> 是 Service 的名称，<namespace> 是 Service 所在的命名空间名称。


## 总结
1. 学习了Netfilters的原理和配置规则。
2. 学了Service的虚拟IP实现原理。
3. 学习了Service的IPVS的实现原理。
4. 学了Service, Pod的DNS A记录。
