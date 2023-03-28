# Pod的网络模型

本文主要分析Pod的网络模型。

我们分析过容器的bridge网络模型和K8s的host_gw网络模型，但我们还没有分析过Pod的网络模型。

我们先了解一下关于Pod的基本结论：
1. Pod是容器集合的逻辑描述概念。
1. 每个 Pod 都有一个唯一的 IP 地址。
2. Pod 中的所有容器都共享这个 IP 地址，并且它们可以通过该地址进行通信。

在分析Pod的网络模型之前，我说一下之前关于Pod IP的一个困惑：Pod只是容器集合一个逻辑描述概念，为什么会有ip？

比如Service的IP模式下，Service的ip是iptables的一个规则描述信息，在IPVS模式下，Service的ip是一个虚拟网卡绑定的ip，那Pod的ip是什么呢？访问Pod的ip为什么会把请求转发到容器呢？

所有容器共享一个Pod的IP地址，共享是如何实现的呢，容器IP和Pod IP有什么关联？

如果你也有类似的疑问，请继续往下看。

理解本文需要掌握的知识：
1. 理解虚拟网桥，Veth网络设备。
2. 理解容器的Bridge网络模型
3. 理解K8s跨主机Host_Gw网络模型


对以上知识点不清楚的可以复习[](), [](), []()。

## Pod和容器

先回顾一下容器的概念，我们知道，容器的本质是*进程*。

这个进程和普通进程的区别在于容器进程有隔离的资源空间，包括：
1. 网络：每个容器都有自己的网络栈，包括独立的IP地址、端口和网络接口。
2. 内存：每个容器都有自己的内存空间，容器中的应用程序只能访问容器中分配给它的内存，而不能直接访问宿主机器或其他容器的内存。
3. CPU：每个容器都可以限制或分配自己的CPU资源，以确保容器中的应用程序获得足够的CPU时间。
4. 文件系统：每个容器都有自己的文件系统，容器中的文件系统与宿主机器的文件系统是隔离的，容器中的文件对于宿主机器来说是不可见的。
5. IPC Namespace：容器还可以拥有自己的 IPC（Inter-Process Communication）Namespace，使得容器内的进程与宿主机器上的进程隔离开来，容器内的进程只能与同一 Namespace 内的其他进程进行 IPC 通信。
6. 用户 Namespace：每个容器也可以拥有自己的用户 Namespace，使得容器内的进程与宿主机器上的进程隔离开来，容器内的用户在宿主机器上是不可见的。
7. UTS Namespace：容器还可以拥有自己的 UTS（Unix Timesharing System）Namespace，使得容器内的进程与宿主机器上的进程隔离开来，容器内的进程可以拥有自己的主机名和域名，与宿主机器上的进程的主机名和域名不冲突。

Pod的本质是什么呢？

Pod的本质只是一个逻辑概念，在一个Pod中，所有容器都可以共享以下操作系统资源：
1. 网络命名空间：Pod 中的所有容器共享同一个网络命名空间，它们可以访问同一个网络接口和同一个 IP 地址。
2. IPC 命名空间：Pod 中的所有容器共享同一个 IPC 命名空间，它们可以通过共享内存等方式进行进程间通信。
3. UTS 命名空间：Pod 中的所有容器共享同一个 UTS 命名空间，它们可以共享同一个主机名和域名。
4. PID 命名空间：Pod 中的所有容器共享同一个 PID 命名空间，它们可以查看同一个进程列表。

换句话说，共享这些资源的容器构成了Pod。

Pod和容器的概念，就好比进程和进程组，进程是真实存在的操作系统中的，但是进程组只是对多个进程的一种逻辑描述。
拥有同一个父进程ID的进程集合，我们叫做进程组，那Pod中的容器是怎么组合在一起的呢，如图所示：
```

+-------------------+            +------------------+
|                   |            |                  |
|  Container-1      |            |  Container-2     |
|                   |            |                  |
+-----+-------------+            +-------------+----+
      |                                        |
      |                                        |
      |       +----------------------+         |
      |       |                      |         |
      +------->  Infra_Container     <---------+
              |                      |
              +----------------------+
```

Pod的实现需要使用一个中间容器，这个容器叫作Infra容器。
在这个Pod中，Infra容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与Infra容器关联在一起。

现在来回答第一个问题：Pod的Network Namespace和Pod IP是什么？

其实Pod的Network Namespace就是创建一个Network Namespace，在这个空间里面创建一个虚拟网卡，同时给给虚拟网卡分配一个IP，这个IP就是Pod的ip。

因为Pod的Network Namespace需要和宿主的Network Namespace 通信，所以创建的虚拟设备是Veth，如下：
```
sudo ip netns add <POD_NAMESPACE>
sudo ip link add <VETH_NAME> type veth peer name <CONTAINER_VETH_NAME>
sudo ip link set <VETH_NAME> netns <POD_NAMESPACE>
sudo ip netns exec <POD_NAMESPACE> ip link set <VETH_NAME> up
sudo ip netns exec <POD_NAMESPACE> ip addr add <POD_IP_ADDR>/<POD_NETMASK_CIDR> dev <VETH_NAME>
sudo ip link set <CONTAINER_VETH_NAME> up
```

这个操作是Kubernetes 使用一个名为 CNI的插件来实现网络的。


第二个问题：Pod中的所有容器都共享Pod IP地址是什么意思？
举个例子大家就明白了，主机的linux操作系统都有默认的网络空间，在linux系统上创建多个进程，此时多个进程之间可以相互通过localhost+port的方式相互访问，
同时，外部要想访问主机的进程可以通过主机ip+prot也可以访问对于的进程。

这里可以理解为，多个进程共享了默认网络空间。

容器共享Pod IP地址就是在Pod的网络空间中，所有的容器可以通过localhost+port相互访问，外部要想访问容器，可以使用Pod IP+Port就可以访问。


那容器怎么共享 Network Namespace呢，使用命令：
```
docker run --network=<network_name> <image_name>
```

最后一个问题，容器IP和Pod IP有什么联系？

首先，每个 Pod 都有一个唯一的 IP 地址，Pod 中的所有容器都共享这个 IP 地址，并且它们可以通过该地址进行通信。
每个容器也有自己的 IP 地址，称为容器 IP 地址。容器 IP 地址是由 Docker 分配的，因为 Kubernetes 使用 Docker 作为其默认的容器运行时。

容器IP是只有在容器内部可见的，外部的网络无法直接访问它，只能通过Pod ip访问容器。

因此，虽然 Pod IP 地址和容器 IP 地址都存在，但它们是不同的地址，并且在许多情况下，我们更关心的是 Pod IP 地址。

可以使用 kubectl describe pod 命令来查看 Pod 的 IP 地址和容器的 IP 地址。 例如：
```
$ kubectl describe pod my-pod
Name:         my-pod
Namespace:    default
...
IP:           10.244.1.2
...
Containers:
  my-container1:
    ...
    State:          Running
      ...
      IP:             10.244.1.3
  my-container2:
    ...
    State:          Running
      ...
      IP:             10.244.1.4
```

怎么通过DNS获取Pod ip呢？
```
$ kubectl exec -it my-pod -c my-container1 -- nslookup my-pod.default.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   my-pod.default.svc.cluster.local
Address: 10.244.1.2

```
## Pod和Infra容器

通常情况下，Infra 容器是由 Kubernetes 系统自动创建的，而且对于应用程序开发人员而言，它们是透明的，无需关心 Infra 容器的具体实现细节。

Infra容器具有以下特征：
1. Infra 容器运行在 Pod 的命名空间内。
2. Infra 容器可以分配 Pod 的IP地址和端口，并为其他容器提供网络代理服务，以便它们可以与外部通信。
3. Infra 容器通常会挂载一些卷（例如，Pod 的 /etc/hosts 文件和 Pod 的主机名等），以便其他容器可以访问它们。
4. Infra 容器通常具有特权，以便能够执行与其他容器不同的操作（例如，管理网络和文件系统）。

Pod中的每个容器都可以访问相同的文件系统，包括 Pod 的 /etc/hosts 文件。因此，Infra 容器可以通过挂载共享的卷，将 Pod 的 /etc/hosts 文件提供给其他容器。
例如，以下是一个使用 ConfigMap 挂载共享卷的示例 YAML 文件：
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myapp
    image: myapp
  - name: infra
    image: infra
    volumeMounts:
    - name: hostfile
      mountPath: /etc/hosts
  volumes:
  - name: hostfile
    configMap:
      name: myconfigmap

```
在上面的示例中，Pod 中包含两个容器：myapp 容器和 infra 容器。infra 容器挂载了名为 myconfigmap 的 ConfigMap，将其作为共享的卷，然后将共享的卷挂载到 Pod 的 /etc/hosts 文件中。这样，myapp 容器就可以访问 Pod 的 /etc/hosts 文件，并获取所需的网络信息。

在 Kubernetes 中，每个 Pod 都有一个唯一的主机名，可以通过环境变量 $HOSTNAME 获取。因此，Infra 容器可以设置环境变量，并将 Pod 的主机名作为值传递给其他容器。
例如，以下是一个使用环境变量传递 Pod 主机名的示例 YAML 文件：
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myapp
    image: myapp
    env:
    - name: POD_HOSTNAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
  - name: infra
    image: infra
```
在上面的示例中，myapp 容器设置了一个环境变量 $POD_HOSTNAME，并通过 valueFrom 字段引用 Pod 的 metadata.name 属性。这样，myapp 容器就可以获取 Pod 的主机名，并将其用于网络通信和连接等操作。


# Pod网络通信

目前我知道，Container有自己的Network NameSpace，Pod也有Network NameSpace，宿主机也有Network NameSpace，这三者之间的关系大概如图：
```
             宿主机ip:10.168.0.1
+-----------------------------------+---------------+
|             |Etho                 |               |
|             |宿主机网卡             |               |
|             +---------------------+               |
|                                                   |
|      +--------------------------------------+     |
|      | Cni0                                 |     |
| +----> K8s宿主机虚拟网桥                      |     |
| |    +------^------------^----------^-------+     |
| |           |            |          |             |
| |    +--------------------------------------+     |
| |    |      |            |          |       |     |
| |    |   +--+---+    +---+---+   +--+---+   |     |
| |    |   |容器1 |    |容器2   |   | 容器3 |   |     |
| |    |   |      |    |       |   |      |   |     |
| |    |   +------+    +-------+   +---+--+   |     |
| +----+     |容器ip:192.168.0.2        |      |     |
|      |     |             |           |      |     |
|      |     |  +----------v-------+   |      |     |
|      |     |  | Pod内网桥         |   |      |     |
|      |     +--> 连接Pod内容器      <---+      |     |
|      |        +------------------+          |     |
|      |                                      |     |
|      |        +-------------------+         |     |
|      |        |Pod虚拟网卡         |         |     |
|      +----------------------------+---------+     |
|          Pod ip:192.168.0.1                       |
+---------------------------------------------------+
```
虽然容器，Pod和宿主机都处于不同的Network NameSpace，但是他们可以互联，原理是：
1. 宿主机可以通过路由表访问Pod是因为Pod和宿主机的虚拟网桥+Veth设备相连
2. 宿主机可以通过路由表访问容器是因为容器和宿主机通过虚拟网桥+Veth设备相连
3. Pod可以访问容器，是因为Pod和容器通过虚拟网桥+veth设备相连。

当容器通过Service IP跨宿主机访问服务时，过程是：
1. 容器的网络数据通过Cni0给到宿主机。
2. 宿主机通过Iptables转发Service IP到目标主机的Pod IP上。
3. 目标主机收到请求后，查找路由表对比目标Pod IP，把请求转发给Pod。
4. Pod收到请求后，根据端口信息转发到对应的Container。
