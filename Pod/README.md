- [Pod的创建](#pod的创建)
  - [命令创建](#命令创建)
  - [yaml创建](#yaml创建)
- [Pod与容器的生命周期](#pod与容器的生命周期)
- [Pod的参数](#pod的参数)
  - [imagePullPolicy](#imagepullpolicy)
  - [restartPolicy](#restartpolicy)
  - [探针](#探针)
  - [Pod拓扑分布约束](#pod拓扑分布约束)
- [Pod的节点调度](#pod的节点调度)
  - [nodeSelector](#nodeselector)
  - [亲和与反亲和](#亲和与反亲和)
  - [nodeName](#nodename)
  - [污点与容忍度](#污点与容忍度)
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

容器的重启策略，适用于pod中的所有容器，默认值是Always

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

---

## Pod的节点调度

### nodeSelector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: pod2
spec:
  nodeSelector:
    hello: world # 会被调度到含有此标签的节点上
```

> kubectl get nodes --show-labels 中包含`hello=world`标签的节点，pod才会被调度上去。

### 亲和与反亲和

与nodeSelector类似，可以根据节点上的标签来约束 Pod 可以调度到哪些节点上。

>  亲和：将pod放在符合条件的node上运行，反亲和: 不要将多个Pod放在同一个节点上。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: pod2
spec:
  affinity: 
    nodeAffinity: # node亲和性， podAntiAffinity反亲和
      requiredDuringSchedulingIgnoredDuringExecution: # 调度器只有在规则被满足的时候才能执行调度
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                  - linux
      preferredDuringSchedulingIgnoredDuringExecution:  # 调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。
        - weight: 1
          preference:
            matchExpressions:
              - key: another-node-label-key
                operator: In
                values:
                  - another-node-label-value
```

> 1. `requiredDuringSchedulingIgnoredDuringExecution`： 调度器只有在规则被满足的时候才能执行调度。
> 2. `preferredDuringSchedulingIgnoredDuringExecution`： 调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该Pod。
>
> 3. 如果在kubernetes调度pod的过程中发生了变更，Pod仍将继续运行。
>
> 4. 如果同时指定了`nodeSelector`和`nodeAffinity`那么两者都需要满足才能调度Pod。
>
> 5. 指定了多个与 `nodeAffinity` 类型关联的 `nodeSelectorTerms`， 只要其中一个 `nodeSelectorTerms` 满足的话，Pod 就可以被调度到节点上。
>
> 6. 指定了多个与同一 `nodeSelectorTerms` 关联的 `matchExpressions`， 则只有当所有 `matchExpressions` 都满足时 Pod 才可以被调度到节点上。

### nodeName

优先级高于nodeSelector，如果nodeName的值不存在，那么Pod无法运行，如果指定了存在的node，那么Pod只会被调度到此node上。

### 污点与容忍度

`节点亲和性`是Pod的一种属性，它使Pod能吸引到一类特定的节点，`污点（Taint）`则相反，它使节点能够排斥一类特定的 Pod。

`容忍度（Toleration）`是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的节点。 容忍度允许调度但并不保证调度（作为其功能的一部分，调度器也会[评估其他参数](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/pod-priority-preemption/)）。

`污点和容忍度`相互配合，可以用来避免 Pod 被分配到不合适的节点上。 每个节点上都可以应用一个或多个污点，对于那些不包含`对应节点污点属性（容忍度）`的Pod，是不会被该节点接受的。

```shell
# 给节点添加一个污点 给节点node1增加一个污点，它的键名是 key1，键值是 value1，效果是 NoSchedule。 
# 这表示只有拥有和这个污点相匹配的容忍度的 Pod 才能够被分配到 node1 这个节点。
kubectl taint nodes node1 key1=value1:NoSchedule
# 移除节点的污点
kubectl taint nodes node1 key1=value1:NoSchedule-
# 查看节点的污点属性
kubectl describe node <nodeName> | grep Taints -A 10
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    app: pod2
spec:
  nodeSelector:
    hello: world # 会被调度到含有此标签的节点上
  tolerations: # 容忍度，pod不会被调度到不包含key1污点的节点
    - key: "key1"
      operator: "Equal" # 默认为Equal,Equal需要指定value
      effect: "NoSchedule" # NoExecute/NoSchedule
      value: value1
    - key: "key2"
      operator: "Exists" # 如果是Exists，则不需要指定value
      effect: "NoExecute" # 当pod容忍度与node污点不匹配的时候会驱逐pod
      tolerationSeconds: 3600 # Pod继续运行3600s后才会被驱逐（前提是污点还在）
  
```

> 1. NoSchedule：新的不能容忍的pod一定不会被调度，原有的pod不受影响。NoExecute：新的不能容忍的pod不仅不会被调度，还会驱逐老的Pod。
> 2. operator为Equal时需要指定value，若为Exists则无需指定value。
> 3. 若node上有多个污点，那么pod需要都能容忍这些污点才能调度到node上。
> 4. NoExecute与tolerationSeconds搭配使用，tolerationSeconds秒后才会驱逐pod。

---

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

