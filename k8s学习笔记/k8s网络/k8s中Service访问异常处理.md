## Service常用命令
k8s常用用于操作 Service IP的命令如下：
1. 获取 Service IP：
```
kubectl get services <service-name>
```

2. 更新 Service IP
```
kubectl patch service <service-name> -p '{"spec": {"clusterIP": "<new-ip>"}}'
```

3. 删除 Service IP
```
kubectl delete service <service-name>
```
4. 创建 Service IP
```
kubectl create service clusterip <service-name> --tcp=<port>:<target-port> --dry-run=client -o yaml | kubectl apply -f -
```

5. 查看 Service IP 配置
```
kubectl describe services <service-name>
```
6. 要查看Service IP绑定的Pod消息
```
kubectl get endpoints <service-name>
```
这将返回类似以下输出的列表：
```
NAME         ENDPOINTS                  AGE
my-service   10.0.0.1:8080,10.0.0.2:8080 1d
```

## Service访问异常处理思路

### 无法通过Service Name访问
1. 查看CoreDNS Pod状态是否正常
```
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

这个命令将列出在 kube-system 命名空间中标签为 k8s-app=kube-dns 的所有 Pod，这个标签是用来标识 CoreDNS Pod 的。
```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxxx-xxxxx   1/1     Running   0          xxh
```
其中 READY 列表示 CoreDNS Pod 中容器的就绪状态，STATUS 列表示容器的当前状态，RESTARTS 表示容器的重启次数，AGE 表示 Pod 运行的时间。

如果 CoreDNS Pod 处于运行状态，状态列中将显示 Running，READY 列的第一个数字为 1 表示容器已经启动并就绪。
如果 CoreDNS Pod 处于错误状态，您将看到类似以下的输出：

```
Copy code
NAME                       READY   STATUS             RESTARTS   AGE
coredns-xxxxxxxxxx-xxxxx   0/1     CrashLoopBackOff   7          xxh
```

其中 STATUS 列显示 CrashLoopBackOff，这意味着容器一直崩溃并尝试重新启动。

这时可以使用`kubectl describe pod <pod-name> -n kube-system` 命令查看 Pod 的详细信息，以便更好地了解问题所在。

2. 查看某个Service Name是否可以正常解析
执行以下命令启动一个 BusyBox 容器：
```
kubectl run -it --rm --image=busybox:1.28.4 -- sh
```

这将在 Kubernetes 中启动一个名为 sh 的 BusyBox 容器，并进入容器的命令行模式。
在容器中执行以下命令来查看 DNS 服务是否正常：
```
nslookup <service-name>
```

输出的内容格式为：
```
Server:    <Kubernetes DNS 服务的 IP 地址>
Address 1: <Kubernetes DNS 服务的 IP 地址>

Name:      <service-name>.<namespace>.svc.cluster.local
Address 1: <Service 对应的 IP 地址>
```
其中，<Kubernetes DNS 服务的 IP 地址> 是 Kubernetes 集群中 DNS 服务的 IP 地址。


### 无法通过 Service IP访问

1. 查看Service的端口，EndPoints信息，确保配置无误

`kubectl describe service <service-name>`：

输出结果如下：
```
Name:              <service-name>
Namespace:         <namespace>
Labels:            <label-key>=<label-value>
Annotations:       <annotation-key>=<annotation-value>
Selector:          <selector-key>=<selector-value>
Type:              <service-type>
IP:                <service-ip-address>
Port:              <port>
TargetPort:        <target-port>
Endpoints:         <endpoint-ip-address-1>:<endpoint-port-1>,<endpoint-ip-address-2>:<endpoint-port-2>,...
Session Affinity:  <session-affinity-mode>
Events:            <event-1>,<event-2>,...

```

其中Endpoints是该 Service 关联的后端Pod的IP地址和端口号列表。我们需要保证Service配置了正确的端口，以及拥有符合预期的EndPoints。

2. 查看Pod是否正常工作

使用 kubectl 命令执行一个测试请求
```
kubectl exec <pod-name> -- curl <container-ip>:<port>
```

使用 kubectl 命令检查 Pod 的状态
```
kubectl get pods
```
如果 Pod 正在运行并且状态为 Running，则表示 Pod 已经启动并且可以正常访问。

使用 kubectl 命令查看 Pod 的日志：
```
kubectl logs <pod-name>
```

3. 查看kube-proxy组件是否正常工作
kube-proxy负责更新iptables或者是创建ipvs模式，所以必须保障kube-proxy服务正常。

使用以下命令检查 kube-proxy 是否正在运行：
```
kubectl get pods -n kube-system | grep kube-proxy
```

如果输出结果中 kube-proxy 的状态为 Running，则 kube-proxy 正在运行：
```
kube-proxy-xxxxxxxxx-xxxxx   1/1     Running   0          xxh ago
```

使用以下命令检查 kube-proxy 是否能够正常连接到 Kubernetes API 服务器：
```
kubectl logs -n kube-system kube-proxy-xxxxxxxxx-xxxxx
```


检查输出结果是否有类似于以下内容的日志记录：
```
I0326 10:00:00.000000       1 server_others.go:143] Using iptables Proxier.
I0326 10:00:00.000000       1 server.go:648] Version: v1.21.xxxxx
I0326 10:00:00.000000       1 conntrack.go:100] Set sysctl 'net/netfilter/nf_conntrack_max' to 131072
```

如果日志记录显示 kube-proxy 正常连接到 Kubernetes API 服务器，则 kube-proxy 运行正常。
检查 kube-proxy 是否已经为服务创建了必要的 iptables 规则：
```
sudo iptables-save | grep KUBE-SERVICES
```
