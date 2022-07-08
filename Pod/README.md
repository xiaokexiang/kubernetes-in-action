- [Pod的创建](#pod的创建)
  - [命令创建](#命令创建)
  - [yaml创建](#yaml创建)
- [Pod与容器的生命周期](#pod与容器的生命周期)
- [Pod的参数](#pod的参数)
  - [imagePullPolicy](#imagepullpolicy)
  - [restartPolicy](#restartpolicy)
  - [探针](#探针)
  - [Pod拓扑分布约束](#pod拓扑分布约束)
- [常用命令](#常用命令)

---

## Pod的创建

> Pod是一组（一个或多个） 容器； 这些容器共享存储、网络、以及怎样运行这些容器的声明。 Pod 中的内容总是并置的并且一同调度，在共享的上下文中运行。

![](https://image.leejay.top/FozNgUDN5hlTyAnD_nittm3RosUs)

> 1. 同一个Pod内的多个容器共享相同的Linux命名空间，拥有相同的IP、主机名、网络接口等。
> 2. 同一个Pod内的容器不能出现相同的端口号，可以通过`localhost`地址进行互相访问。
> 3. 默认情况下容器间的目录互相隔离，但是可以通过`Volume`实现文件目录共享。

### 命令创建

```shell
kubectl -n <namespace> run <podName> --image=nginx:latest
```

### yaml创建

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: pod2
spec:
  topologySpreadConstraints:
    -   maxSkew: 2 # 描述 Pod 分布不均的程度
        topologyKey: hello
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            hello: world
  initContainers: # 设置init容器等待myservice就绪后才会创建容器
    -   name: init-pod
        image: busybox:latest
        command: [ 'sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done" ]
  containers:
    -   name: pod2
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        readinessProbe: # 容器是否准备好提供服务
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 10
        livenessProbe: # 容器是否存活
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 10
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  nodeSelector:
    hello: world # 会被调度到含有此标签的节点上
```

---

## Pod与容器的生命周期

| 值        | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| Pending   | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。 |
| Running   | Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。 |
| Succeeded | Pod 中的所有容器都已成功终止，并且不会再重启。               |
| Failed    | Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。 |
| Unknows   | 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。 |

---

## Pod的参数

### imagePullPolicy

镜像的拉取策略

- Never：只会在本地镜像中查找
- IfNotPresent：本地镜像没有才会去远端拉取
- Always：一直从远端拉取

### [restartPolicy](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)

容器的重启策略，适用域pod中的所有容器，默认值是Always

- Always：只要容器不在运行状态，就会自动重启容器
- OnFailure：只有在容器异常时才会自动重启容器（多容器需要都进入异常后，pod才会转为Failed）
- Never：从来不重启容器

### [探针](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/)

- livenessProbe：

  指示容器是否正在运行，如果失败，kubelet会杀死容器并根据`重启策略`决定。

- readinessProbe：

  指示容器是否准备好为请求提供服务。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。

- startupProbe

  指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，`kubelet` 将杀死容器，而容器依其[重启策略](#restartPolicy)进行重启。

### [Pod拓扑分布约束](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-topology-spread-constraints/)

- maxSkew

  描述Pod分布不均的程度（必须大于0），取决于`whenUnsatisfiable`的取值：

  当`DoNotSchedule`时，此值用于限制目标拓扑域中匹配的pod数与全局最小值的差值。

  当`ScheduleAnyway`，调度器会更为偏向能够降低偏差值的拓扑域。

- topologyKey

  节点标签的键，如果`两个节点使用相同的键并具有相同的标签值`，那么这两个节点被视为同一个拓扑域。

- whenUnsatisfiable

  - DoNotSchedule：（默认）告诉调度器不要调度。
  - 告诉调度器仍然继续调度，只是根据如何能将偏差最小化来对 节点进行排序。

## 常用命令

```shell
# 查看pod的spec中字段含义
kubectl explains pod.spec
# 通过yaml或json格式参看pod的配置信息
kubectl -n <namespace> get pod <podName> -o <yaml|json>
# 查看pod日志
kubectl -n <namespace> logs -f <podName>
kubectl -n <namespace> delete pod <podName>
# 强制删除pod，不再等待kubelet的确认消息
kubectl -n <namespace> delete pod <podName> --grace-period=0 --force
# 不通过service实现本地端口访问pod端口
kubectl -n <namespace> port-forward <podName> <本地端口>:<pod端口>
```

