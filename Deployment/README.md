- [Deployment创建](#deployment创建)
    - [命令创建](#命令创建)
    - [yaml创建](#yaml创建)
- [状态解析](#状态解析)
- [常见命令](#常见命令)
- [参数解析](#参数解析)

---

## Deployment创建

> 一个 Deployment 为 `Pod` 和 `ReplicaSet` 提供声明式的更新能力。通过ReplicaSet管理Pod的变更。

### 命令创建

```shell
kubectl create deploy deploy-command --image=nginx:latest --replicas=1 -n helloworld# 指定镜像地址和副本数
```

### yaml创建

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: deploy # deploy name
    namespace: helloworld
    labels:
        app: deploy
spec:
    replicas: 1
    template:
        metadata:
            name: deploy
            labels: # 用于与deployment中的selector来匹配
                app: deploy
        spec:
            containers:
                -   name: deploy
                    image: nginx:latest
                    imagePullPolicy: IfNotPresent
            restartPolicy: Always
    selector: # 定义了匹配pod的规则
        matchExpressions: # 只有一个value时等同于matchLabels
            -   key: app
                operator: In
                values:
                    - deploy
```

## 状态解析

![微信截图_20220705171031](https://image.leejay.top/img/微信截图_20220705171031.png)

> 1. NAME: 集群中deployment名称
> 2. READY: "就绪副本数/期望个数"
> 3. UP-TO-DATE: 为达到期望状态已经更新的副本数
> 4. AVAILABLE: deployment可供用户使用的副本数
> 5. AGE: deployment运行的时间

![微信截图_20220705171606](https://image.leejay.top/img/微信截图_20220705171606.png)

> 1. NAME: ReplicaSet名称
> 2. DESIRED: deployment期望的副本个数
> 3. CURRENT: deployment中运行状态的副本数
> 4. READY: deployment中为用户提供服务的个数
> 5. AGE: deployment运行的时间

## 常见命令

```shell
# 修改deployment中的镜像并记录
kubectl -n [namespace] set image deploy [deployment name] [container name]=nginx:1.16.1 --record=true
# 查看deployment变更状态
kubectl -n [namespace] rollout status deploy [deployment name]
# 查看deployment修改历史
kubectl -n [namespace] rollout history deploy [deployment name]
# 回滚deployment
kubectl -n [namespace] rollout undo deploy [deployment name]
# 回滚到指定版本
kubectl -n [namespace] rollout undo deploy [deployment name] --to-revision=[revision]
# 缩放deployment
kubctl -n [namespace] scale deploy [deployment name] --replicas=[缩放数量]
```

## 参数解析

| 参数                              | 含义                                                         | 默认值        |
| --------------------------------- | ------------------------------------------------------------ | ------------- |
| .spec.revisionHistoryLimit        | 指定保留此 Deployment 的多少个旧的ReplicaSet，设置为0则无法进行回滚 | 10            |
| .spec.template.spec.restartPolicy | 容器的重启策略（适用pod下的所有容器），在Deployment中只能为`Always` | Always        |
| .spec.strategy                    | 用新 Pods 替换旧 Pods 的策略：<br />1. Recreate（创建新的Pods前会杀死现有的pods）<br />2. RollingUpdate（滚动更新的方式更新Pods）<br />maxUnavailable：更新过程中不可用pod的个数上限，maxSurge：新旧pod总数量上限 | RollingUpdate |
| .spec.progressDeadlineSeconds     | Deployment失败后等待取得进展的秒数。                         | 无            |
| .spec.minReadySeconds             | 指定新创建的 Pod 在没有任意容器崩溃情况下的最小就绪（配合探针）时间 | 0             |
| .spec.paused                      | 用于暂停和恢复 Deployment 的字段（true/false）               | 无            |
