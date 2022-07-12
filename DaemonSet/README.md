- [daemonSet的创建](#daemonset的创建)

> DaemonSet确保全部（或某些）节点上运行一个Pod的副本，当有节点加入集群时，也会为它新增一个Pod，当有节点从集群移除时，这些 Pod 也会被回收。删除DaemonSet将会删除它创建的所有Pod。
>
> 常用于每个节点上运行集群守护进程、运行日志收集守护进程、运行监控守护进程。

## daemonSet的创建

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
  namespace: helloworld
spec:
  selector:
    matchLabels:
      app: ssd-monitor # 与.spec.template.metadata.labels 匹配
  template:
    metadata:
      labels:
        app: ssd-monitor # 与.spec.selector.matchLabels匹配
    spec:
      containers:
        - name: main
          image: luksa/ssd-monitor
      nodeSelector:
        hello: world # 调度到包含此标签的node上
```

daemonSet会默认添加如下容忍度：

| `node.kubernetes.io/not-ready`           | NoExecute  | 1.13+ | 当出现类似网络断开的情况导致节点问题时，DaemonSet Pod 不会被逐出。 |
| ---------------------------------------- | ---------- | ----- | ------------------------------------------------------------ |
| `node.kubernetes.io/unreachable`         | NoExecute  | 1.13+ | 当出现类似于网络断开的情况导致节点问题时，DaemonSet Pod 不会被逐出。 |
| `node.kubernetes.io/disk-pressure`       | NoSchedule | 1.8+  | DaemonSet Pod 被默认调度器调度时能够容忍磁盘压力属性。       |
| `node.kubernetes.io/memory-pressure`     | NoSchedule | 1.8+  | DaemonSet Pod 被默认调度器调度时能够容忍内存压力属性。       |
| `node.kubernetes.io/unschedulable`       | NoSchedule | 1.12+ | DaemonSet Pod 能够容忍默认调度器所设置的 `unschedulable` 属性. |
| `node.kubernetes.io/network-unavailable` | NoSchedule | 1.12+ | DaemonSet 在使用宿主网络时，能够容忍默认调度器所设置的 `network-unavailable` 属性。 |

> 换个角度：也就是节点出现如上污点时，daemonSet能够被调度或不被驱逐。