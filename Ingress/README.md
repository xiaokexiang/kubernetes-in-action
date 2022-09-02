- [安装ingress-nginx-controller](#安装ingress-nginx-controller)
- [创建ingress服务](#创建ingress服务)
  - [TLS](#tls)

---
> [Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.24/#ingress-v1beta1-networking-k8s-io) 公开从集群外部到集群内[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。
>
> 为了让 Ingress 资源工作，集群必须有一个正在运行的 Ingress 控制器。通过目录下的ingress-nginx-controller.yaml安装[ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/)控制器。

![微信截图_202208091ingress1](https://image.leejay.top/img/微信截图_202208091ingress1.png)

### 安装ingress-nginx-controller

执行`kubectl apply -f ingress-nginx-controller.yaml`命令，等待ingress控制器安装，如下图所示即为成功。

![ingress-controller](https://image.leejay.top/img/ingress-controller.png)

### 创建ingress服务

在部署ingress服务之前，我们需要先部署service服务，设置type为`ClusterIP`，确保通过ServiceIP + Port能够正常访问Pod。

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
  name: nginx-example
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true" # 未指定ingressClassName字段的Ingress默认分配这个IngressClass.
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: helloworld
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2 # 用于替换(.*) http://www.helloworld.com:31166/nginx -> nginx-service:8088/ 用于重写请求
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, PUT, POST, DELETE, PATCH, OPTIONS" # 跨域相关
    nginx.ingress.kubernetes.io/default-backend: nginx-service # 指定默认后端
    nginx.ingress.kubernetes.io/limit-rps: "1" # 限制一个ip每秒一次请求，超出返回503
    nginx.ingress.kubernetes.io/server-snippet: | # 根据agent判断，若是手机端访问重定向到百度
        set $agentflag 0;

        if ($http_user_agent ~* "(Mobile)" ){
          set $agentflag 1;
        }

        if ( $agentflag = 1 ) {
          return 301 https://baidu.com;
        }

spec:
  ingressClassName: nginx-example # 对 IngressClass 资源的引用。 IngressClass 资源包含额外的配置，其中包括应当实现该类的控制器名称。
  rules:
  - host: www.helloworld.com # 域名
    http:
      paths:
      - path: "/nginx(/|$)(.*)" # 路由匹配
        pathType: Prefix
        backend:
          service:
            name: nginx-service # service名称
            port:
              number: 8088 # service port
```

> 点击查看[annotations]("https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations")文档

![ingress2](https://image.leejay.top/img/ingress2.png)

执行`kubectl -n helloworld describe ingress nginx-ingress`查看ingress具体信息

![ingress3](https://image.leejay.top/img/ingress3.png)

最后我们需要在本地电脑配置hosts: #{IP} www.helloworld.com，然后访问`www.helloworld.com:#{ingress-nginx-controller-svc-port}/nginx/`来测试ingress是否生效。

#### TLS

创建包含TL私钥喝证书的Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 编码的证书
  tls.key: base64 编码的私钥
type: kubernetes.io/tls
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls # 指定secret名称
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1 # service名称
            port:
              number: 80
```
