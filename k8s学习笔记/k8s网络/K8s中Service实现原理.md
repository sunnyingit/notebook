# K8s中Service的实现原理

Service是k8s里重要的服务对象，解决两个问题：
1. Pod的访问，Pod的IP不是固定的，我们不能通过IP去访问Pod，所以抽象了Service对象。
2. Pod的访问需要实现负载均衡。

理解Service需要的知识储备是：
1. 理解Pod的概念，请参考[]()
2. 理解iptables的网络转发
3. 理解内核IPVS技术


## Service对象

在K8s中，Service是一种抽象，并没有一个实体叫做Service。
Service通过标签选择器来选择属于特定标签的Pod，并提供了一个抽象的方式来访问这些Pod。

具体来说，Service通过以下方式与Pod相关联：

1. 选择器：
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

现在的问题是，如何访问到Service？

可以想到的方法是给Service一个IP地址，这样通过IP就可以访问到Service。


## Iptables

Iptables是Service IP实现的核心原理。

我们学习需要Iptables，需要了解Linux操作系统的防火墙。防火墙不是真实的一堵墙，本质上只是一个钩子(hook)，用于对网络数据进行过滤，修改，重定向，以及网络地址转换（nat）。

防火墙实现的工具是Linux内核软件netfilter，netfilter读取配置文件，也就是Iptables文件进行对网络数据的处理。

如图所示：

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
| 网卡    +------>+ Pre_Routing+-->+ Router      +---> Forward     +---->Router    +------> PostRouting      +-----> 网卡       |
+--------+    |  +------------+   +-------------+   +-------------+    +----------+      +-----------------+  |    +-----------+
              |                                                                                               |
              |                             操作系统内核                                                        |
              +-----------------------------------------------------------------------------------------------+

```

当主机网卡接收到网络数据后，把网络数据发送到操作系统内核，操作系统通过网络协议可以获取到请求的相关信息，比如目标网络IP，目标端口等。
Netfilter的执行过程：
1. Pre_Routing阶段：获取到目标网络IP之后，在这个节段可以对网络数据进行修改，比如修改目标网络IP，目标网络端口等。
2. Input阶段：内核拿到数据后，把数据给到应用程序之前，对数据进行过滤，修改等操作发生在这个阶段。
3. FORWARD阶段：经过路由选择之后要转发的数据包经过此Hook，也就是数据发往默认网关之前的阶段。
4. OUTPUT阶段：用户程序产生的数据包在写入到内核前需要经过这个节段。
5. PostRouting阶段：刚刚通过 FORWARD 和 OUTPUT 关卡的数据包要通过一次路由选择由哪个接口送往网络中，经过路由之后的数据包要经过这个阶段。

上面提到这些阶段，在iptables 的术语里叫做链（chain），内置的链包括上面提到的5个：

1. PreRouting
2. Forward
3. Input
4. Output
5. PostRouting

一般的场景里面，数据包的流向基本是：

1. 到本主机某进程的报文：PreRouting -> Input -> Process -> Output -> PostRouting。
2. 由本主机路由转发的报文：PreRouting -> Forward -> PostRouting。

第二种情况只是转发数据，并没有应用程序处理数据，当数据通过默认网关转发的时候，就会经过Forward阶段。

iptables 默认有五条链chain，分别对应上面提到的五个阶段：Pre_routing/Input/Forward/Output/Post_routing，每个阶段都可以创建规则。

每一条链上的一串规则里面有些功能是相似的，比如，A类规则都是对IP或者端口进行过滤，B 类规则都是修改报文。

我们考虑能否将这些功能相似的规则放到一起，这样管理 iptables 规则会更方便。

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


我们试想一下，在宿主中iptables中在Pre_Routing阶段创建一个Nat规则：将目标地址是{service-ip}:{port}修改为{pod-ip}:{port}就可以了。

原理就是这样简单！

实际上，Service是由kube-proxy和Netfilter来共同实现的，kube-proxy负责更新iptables， Netfilter负责应用iptables。


假如Service的IP是192.168.0.1:80, 要转发的后端Pod的IP是192.168.0.2:8080，192.168.0.3:8080，192.168.0.4:8080。

当Service创建的时候，kube-proxy就会在*所有的宿主机*上创建如下规则：

1. 创建转发规则：

```
-A KUBE-SERVICES -d 192.168.0.1/32 -p tcp -m comment --comment "default/my-service: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

解释如下：
1. -d: 匹配目标地址
2. -p: 匹配协议,如tcp,udp,icmp
3. --dport: 匹配目标端口
4. -j：执行的动作或者链名

上面k8s的命令的解释为：凡是目的地址是192.168.0.1:80的数据，都应该跳转到另外一条名叫`KUBE-SVC-NWV5X2332I4OT4T3`的链来处理。

2. 创建动作链

```
iptables -N KUBE-SVC-NWV5X2332I4OT4T3
iptables -A KUBE-SVC-NWV5X2332I4OT4T3 -m statistic --mode random --probability 0.33 -j DNAT --to-destination 192.168.0.2:8080
iptables -A KUBE-SVC-NWV5X2332I4OT4T3 -m statistic --mode random --probability 0.33 -j DNAT --to-destination 192.168.0.3:8080
iptables -A KUBE-SVC-NWV5X2332I4OT4T3 -j DNAT --to-destination 192.168.0.4:8080
```

这样当访问`192.168.0.1:80`的时候，数据会被转发到到`192.168.0.2，192.168.0.3， 192.168.0.4`上的任意一个。

访问的权重是是使用`--probability 0.33332999982`这个参数来控制的。

最终，为了方便管理和保存规则iptable规则，可以将自定义的规则保存到一个文件中，并使用iptables-restore命令加载和应用规则文件。

例如，假设将规则保存到一个名为iptables.rules的文件中，则可以使用以下命令加载和应用规则：

```
iptables-restore < iptables.rules
```

## IPVS模式

其实，通过上面的讲解，你可以看到，kube-proxy 通过 iptables 处理 Service 的过程，其实需要在宿主机上设置相当多的 iptables 规则。
kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的。

不难想到，当你的宿主机上有大量 Pod 的时候，成百上千条 iptables 规则不断地被刷新，会大量占用该宿主机的 CPU 资源，甚至会让宿主机“卡”在这个过程中。
所以说，一直以来，基于 iptables 的 Service 实现，都是制约 k8s 项目承载更多量级的 Pod 的主要障碍。

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

建议你为kube-proxy设置`–proxy-mode=ipvs`来开启这个功能。它为 k8s 集群规模带来的提升，还是非常巨大的。

## NodePort模式

Service配置虚拟ip时，K8s集群会自动为每个Service配一个*唯一的IP地址*，并在需要时动态更新它。
Service ip只能在集群中访问，无法被外部访问，这个很好理解，因为Service ip并不是一个真实的ip，它本质上是一个iptables的规则，或者是一个虚拟网卡的ip。

那如果需要从外部访问Service，如何实现呢？

答案是NodePort模式。

创建Service时指定NodePort模式：

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```

在这个 Service 的定义里，我们声明它的类型是，type=NodePort。然后，在ports字段里声明了Service的8080端口代理Pod的80端口，Service的443端口代理Pod的443端口。

访问的方式是：
```
<任何一台宿主机的IP地址>:8080
```

为什么访问宿主机的ip和端口，如何转发到对应的Pod呢？

原理是添加一条iptables规则：当访问到本机的8080端口是，把请求转发到Pod的ip和端口上。
```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

`KUBE-SVC-67RL4FN6JRUPOJYM`就是上文的动作链。


当我们的Service使用NodePort创建支持公网访问的ip的时候，访问的链路可能是这样：
```
                        Pod2把数据返回给Client
+---------------------------------------------------------------------+
|                                                                     |
|                                                                     |
|    +------------+         +-------------+      +--------------+     |
|    |            |         |             |      |              |     |
+---->  Client    +--------->   Pod1      +------>  Pod2        +-----+
     |            |         |             |      |              |
     +------------+         +-------------+      +--------------+
     ip:192.168.0.1         ip:10.168.0.1        ip:10.168.10.2
```

1. 当Client访问Pod1(10.168.0.1) 的时候，有可能Pod1由于负载均衡通过iptables把请求转发给Pod2 10.168.10.2。
2. Pod2接受请求后，处理请求完成后，需要把数据返回给客户端，他从请求报文中发现源ip地址是192.168.0.1，也就是Client的ip。
3. Pod2会询问Client的MAC地址，后续会经过一系列交换机和路由器，最终拿到Client的MAC地址，然后把数据发回给Client。
3. 在Client应用程序看来，它请求的是(10.168.0.1)，是无法处理(10.168.10.2)的响应数据的，因为Client创建网络是连接时(Client-Ip + Client+Port + Pod1-IP + Pod1-Port + 协议)。

特别注意使用iptables的是转发网络数据时*网络层层转发*，这种情况和Nginx这样的转发不一样，Nginx是*应用层转发*。
应用层的转发，说白了就是重新构建一个连接，这个过程中源ip会改成Nginx所在的ip，而不是Client的ip。

当然源client的ip也可以通过其他方式传递下去。而最终数据会先响应给Nignx，然后Nignx在响应给Client。

如果避免这种这种问题呢？

原理很简单，就是在Pod1转发请求是，将源ip由(client192.168.0.1) 换成自己的ip(10.168.0.1)。这样Pod2应答的时候，就会把数据发给Pod1，而不是Client。

这条规则设置在 POSTROUTING 检查点，也就是说，它给即将离开这台主机的 IP 包，进行了一次 SNAT 操作，将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的IP地址：
```
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

当然，这个SNAT操作只需要对 Service 转发出来的 IP 包进行，否则普通的 IP 包就被影响了。而iptables做这个判断的依据，就是查看该 IP 包是否有--mark 0x4000/0x4000 的标志。

我们在深入思考一下，为什么IP模式下，也使用了iptable就行转发，是否也需要做SNAT操作？

答案是，不需要。

因为请求的时候，Pod1发起的请求构成时：
1. 源ip是Pod1的ip。
2. 目标ip是Service ip但是在Pre_Routing之前被改成了Pod3的ip。
3. 所以创建套接字的时候，socket组合是(Pod1的ip + 端口 + Pod3的ip + 端口 + 协议)。
4. Pod3数据返回给Pod1理所应当。


## ExternalTrafficPolicy配置
在使用NodePort模式下，会对Client ip进行改写。对于Pod需要明确知道所有请求来源的场景来说，这是不可以的。

所以这时候，你就可以将 Service 的 spec.externalTrafficPolicy 字段设置为 local，这就保证了所有 Pod 通过 Service 收到请求之后，一定可以看到真正的、外部 client 的源地址。

原理很简单：设置宿主iptables的时候，只会转发到本机Pod。

这样请求就不会从Pod1转发到Pod2了，这里也带了一个问题，如果本机上没有对于的Pod，那本次请求就没有数据返回，所以Service访问的方式应该是：
```
拥有对应Pod的宿主IP地:8080
```

## ExternalName类型

创建ExternalName类型类型如下：

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.service.example.com
```

不需要指定selector，Pod的IP和端口信息是通过域名my.service.example.com解析而来的。

k8s 的 Service 还允许你为Service分配公有 IP 地址，比如下面这个例子：
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```

## LoadBalancer类型
这种类型适用于公有云上的k8s服务。

具体的做法如下：
1. 在 Kubernetes 集群中创建一个 Service，并将其类型设置为 LoadBalancer。
2. Kubernetes 控制器会自动创建一个云负载均衡实例，并将 Service 的 IP 地址和端口号映射到该实例上。
3. 当有外部流量请求访问该 Service 时，请求会先到达阿里云负载均衡实例，实例会根据预设的负载均衡算法将请求转发到后端的Pod上。
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/alicloud-load-balancer-bandwidth: "10"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
  externalTrafficPolicy: Cluster

```

创建了一个名为 "my-service" 的 Service，并将其类型设置为 LoadBalancer。
该 Service 会将所有标签选择器 "app=my-app" 的 Pod 与其关联起来。此外，该 Service 还暴露了两个端口：80 和 443。这些端口与后端 Pod 的端口 8080 和 8443 相对应。

查看Service配置的IP：
```
1. kubectl get svc my-service
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                     AGE
my-service   LoadBalancer   10.43.201.184  123.456.789.10  80:31826/TCP,443:30491/TCP   1h

2. kubectl describe services example-service
```

## DNS服务器

在 K8s中，CoreDNS 是一个开源的、轻量级的 DNS 服务器，它被用作 k8s 集群中的默认 DNS 服务。
CoreDNS 帮助 k8s 集群中的各个组件进行服务发现和网络通信，并能够解析 k8s 集群内部的 DNS 名称。

Service和Pod都会被分配对应的 DNS A 记录，A记录就是从域名解析IP的记录。

对于 ClusterIP 模式的 Service 来说，当你访问这条 A 记录的时候，它解析到的就是该 Service 的 VIP 地址。

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
通过本文，我们可以学习到：
1. 学习了Netfilters的原理和配置规则。
2. 学了Service的虚拟IP实现原理。
3. 学习了Service的IPVS的实现原理。
4. Service支持集群外访问的实现：NodePort，LoadBalancer，ExternalName
4. 学了Service, Pod的DNS A记录。
