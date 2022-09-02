- [kubernetes网络](#kubernetes网络)
- [Service](#service)
  - [Service创建](#service创建)
  - [Service代理模式](#service代理模式)
    - [userspace 代理模式](#userspace-代理模式)
    - [iptables 代理模式](#iptables-代理模式)
    - [IPVS 代理模式](#ipvs-代理模式)
  - [服务发现](#服务发现)
    - [环境变量](#环境变量)
    - [DNS](#dns)
  - [无头服务（headless）](#无头服务headless)

---
## kubernetes网络

> - 一个 Pod 中的容器之间[通过本地回路（loopback）通信](https://kubernetes.io/zh-cn/docs/concepts/services-networking/dns-pod-service/)。
> - 集群网络在不同 pod 之间提供通信。
> - [Service 资源](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)允许你 [向外暴露 Pods 中运行的应用](https://kubernetes.io/zh-cn/docs/concepts/services-networking/connect-applications-service/)， 以支持来自于集群外部的访问。
> - 可以使用 Services 来[发布仅供集群内部使用的服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service-traffic-policy/)。

## Service

> Kubernetes 为 Pods 提供自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名， 并且可以在它们之间进行负载均衡。

### Service创建

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: my-app
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80 # pod的容器端口
          name: http-web-svc # 端口定义,可以被targetPort引用
      resources:
        limits:
          memory: 256Mi
          cpu: "1"
        requests:
          memory: 256Mi
          cpu: "0.2"

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: my-app # 会代理符合此标签的pod,service不设置选择标签,那么需要手动创建ep
  ports:
    - name: name-of-service-port
      protocol: TCP
      port: 8081 # svc端口
      targetPort: http-web-svc # pod暴露的端口

```
> 1. service(8081) -> endpoints(80) ——> pod(80)。
> 2. 如果service不指定选择标签，那么不会创建endpoints对象，则需要手动创建。
> 2. 若endpoints包含的端点数量超过1000，会将endpoints对象数量截断到1000。

### Service代理模式

> 在 Kubernetes 集群中，每个 Node 运行一个 `kube-proxy` 进程。`kube-proxy`负责为 Service 实现了一种 VIP（虚拟 IP）的形式。

#### userspace 代理模式

1. kube-proxy监视控制平面对`Service`对象和`Endpoints`对象的添加和移除操作。
2. 每个Service的创建都会在Node上打开一个随机端口，任何请求都会被kube-proxy代理到Serivce的后端Pods中的某个Pod上（由SessionAffinity决定，默认为Round-Robin：轮替模式）。
3. 配置iptables规则，用于捕获达到该Service的clusterIP和Port的请求。并重定向到代理端口，再通过代理端口请求到对应的后端Pod。

![userspace](https://image.leejay.top/img/userspace.png)

> 这种模式需要在内核态（iptables）和用户态（kube-proxy）之间来回切换，所以存在较大的性能损耗。

#### iptables 代理模式

1. kube-proxy监视控制平面对`Service`对象和`Endpoints`对象的添加和移除操作。
2. 每个 Service，它会配置iptables规则，从而捕获到达该Service的`clusterIP`和Port的请求，并将请求重定向到Serivce的后端Pods中的某个Pod上

（默认随机选择一个后端）

3. 此模式下如果所选的第一个pod没有相应，则连接失败，这与userspace模式下的自动切换为其他pod并重试不同。iptables模式下建议使用`探针`保证kube-proxy看到的都是正常的后端，避免流量被kube-proxy发送到已知已失败的Pod。

![iptables](https://image.leejay.top/img/iptables.png)

#### IPVS 代理模式

1. kube-proxy监视控制平面对`Service`对象和`Endpoints`对象的添加和移除操作。
2. 调用 `netlink` 接口相应地创建 IPVS 规则， 并定期将 IPVS 规则与 Kubernetes 服务和端点同步。
3. 确保 IPVS 状态与所需状态匹配。访问服务时，IPVS 将流量定向到后端 Pod 之一。

4. 与iptables类似，ipvs基于netfilter 的 hook 功能，但使用哈希表作为底层数据结构并在内核空间中工作。这意味着ipvs可以更快地重定向流量，并且在同步代理规则时具有更好的性能。（与iptables的区别在于`请求流量的调度功能由ipvs实现`，其他仍由iptables实现）
5. ipvs为负载均衡算法提供了更多选项，如，rr轮询，lc最小连接数，dh目标哈希，sh源哈希，sed最短期望延迟，nq不排队调度。

![ipvs](https://image.leejay.top/img/ipvs.png)

### 服务发现

#### 环境变量

当Service创建后，kubelet会自动给管理的Pod添加如下环境变量（Service创建需先于Pod）

```shell
{SERVICE_NAME}_SERVICE_HOST=${SERVICE_IP}
{SERVICE_PORT}_SERVICE_PORT=${SERVICE_PORT}
```

#### DNS

集群中的CoreDNS监控kubernetes API中的新服务，并为每个服务创建一组DNS记录，如果在整个集群中都启用了 DNS，则所有 Pod 都应该能够通过其 DNS 名称自动解析服务。

若在`helloworld`namespace下存在一个名为`hello-service`的nginx服务，它通过8080端口可以访问到Pod上的80端口（简单的nginx服务），那么进入Pod容器内部，通过`curl hello-service.helloworld:8080`则可以访问到hello-serive服务。

### 无头服务（headless）

有时不需要或不想要负载均衡，以及单独的 Service IP。 遇到这种情况，可以通过指定 Cluster IP（`spec.clusterIP`）的值为 `"None"` 来创建 `Headless` Service。

对于无头 `Services` 并不会分配 Cluster IP，`kube-proxy`不会处理它们， 而且平台也不会为它们进行负载均衡和路由。 DNS 如何实现自动配置，依赖于 Service 是否定义了选择算符。

> 无头服务与普通服务的区别: 无头服务（不分配clusterIP，不会被kube-proxy处理，也不会进行负载均衡和路由）可以通过解析service的dns，得到所有Pod的地址和DNS。普通服务只能解析service的dns得到service的clusterIP。

![deploy-sts](https://image.leejay.top/img/deploy-sts.png)

> statefulset下的pod中，进行DNS查询会返回所有pod的地址和DNS，Pod都会有域名，可以互相访问。而deployment下的pod中则会返回service的clusterIP地址，具体访问哪个pod由iptables或者ipvs决定。



