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