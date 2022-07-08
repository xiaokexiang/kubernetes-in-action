- [前置知识](#前置知识)
    - [容器与虚拟机的区别](#容器与虚拟机的区别)
    - [容器的隔离机制实现](#容器的隔离机制实现)
        - [Linux命名空间](#linux命名空间)
        - [Linux控制组](#linux控制组)
    - [Docker](#docker)
        - [Command](#command)
        - [Dockerfile](#dockerfile)
        - [Docker镜像推送](#docker镜像推送)
- [Kubernetes](#kubernetes)
    - [Kubernetes部署](#kubernetes部署)
        - [基于minikube部署](#基于minikube部署)
        - [基于kubeadm部署](#基于kubeadm部署)
    - [Kubernetes概念](#kubernetes概念)
        - [Master&Node](#masternode)
        - [Kubernetes 架构图](#kubernetes-架构图)
        - [控制平面组件](#控制平面组件)
        - [Node组件](#node组件)
    - [<a href="./Pod/README.md">Pod</a>](#pod)
    - [<a href="./Deployment/README.md">Deployment</a>](#deployment)

---

## 前置知识

### 容器与虚拟机的区别

虚拟机（VM）是虚拟化底层计算机，每个VM不仅需要运行`操作系统的完整副本`，还需要运行操作系统需要运行的所有`硬件的虚拟副本`。这就意味着需要大量的硬件资源。

相比VM，容器只需要虚拟化`操作系统`。每个容器共享主机操作系统内核。相比VM功能类似，但是开销少很多。但是VM提供了完全隔离的环境。

容器内的进程是运行在宿主机的操作系统上的，而虚拟机内的进程是运行在不同的操作系统上的，但容器内的进程是与其他进程隔离的。、

> 1. VM内的指令执行流程：`VM程序指令 -> VM操作系统内核 -> 宿主机管理程序 -> 宿主机内核。 `
> 2. 容器会完全指定运行在宿主机上的同一个内核的系统调用，容器间是共享操作系统内核。

### 容器的隔离机制实现

#### Linux命名空间

每个进程只能看到自己的系统视图（文件、进程、网络接口、主机名等）。进程不单单只属于一个命名空间，而是属于`每个类型`的一个命名空间。类型包括`Mount(mnt)`、`Process ID(pid)`、`NetWork(net)`、`Inter-process communication(ipd)`
、`UTS`、`User ID(user)`。

#### Linux控制组

基于`cgroups`实现，它是Linux内核功能，限制一个进程或一组进程的资源使用不超过被分配的量。

---

### Docker

#### Command

```shell
# 运行容器并输出hello world
docker run busybox echo hello world
```

#### Dockerfile

```js
const http  = require('http'); 
const os  = require('os');

console .log ("Kub i a server starting ... "); 
var handler = function(request, response){
    console.log ("Rece i ved request from ” + request connection. remoteAddress"); 
    response.writeHead(200); 
    response.end("You've hit " + os.hostname() + " \n "); 
}
var www = http.createServer(handler) ; 
www.listen(8080);
// 构建nodejs项目用于容器部署
```

- 编写Dockerfile

```dockerfile
FROM node:7 # 基于什么镜像制作
ADD app.js /app.js # 将当前目录下的app.js移到容器根目录
ENTRYPOINT ["node", "app.js"] # 启动容器时的执行命令
```

- 镜像打包

```dockerfile
docker build -t ${name} .
```

- 进入容器内部

```shell
docker exec -it ${docker name} bash
```

> 1. `-i`表明标准输入流保持开放
> 2. `-t`用于分配一个伪终端

#### Docker镜像推送

```shell
docker ${local_image} ${your account name}/${remote_image}
docker busybox xiaokexiang/busy # 将本地镜像加自己账户名前缀的tag
docker login # docker 账户登录
docker push xiaokexiang/busybox # 推送到远端
```

---

## Kubernetes

### Kubernetes部署

#### 基于minikube部署

- kubectl

```shell
export REGISTRY_MIRROR=https://registry.cn-hangzhou.aliyuncs.com
curl -sSL https://kuboard.cn/install-script/v1.19.x/install_kubelet.sh | sh -s 1.19.2
```

- MiniKube

```shell
# 拉取二进制包（基于阿里云镜像）
# https://developer.aliyun.com/article/221687
curl -Lo minikube https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.17.1/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

# 创建用户（不能使用root启动）
adduser k8s
passwd k8s
sudo groupadd docker
# 添加到用户组
sudo usermod -aG docker k8s
// 激活用户组
newgrp docker

# 启动
minikube start
```

#### 基于kubeadm部署

<a href="./kubeadm部署.md">kubeadm部署</a>

```shell
# 查看node状态
kubectl get nodes
# 赋予node 角色信息
kubectl label nodes k8s-node1 node-role.kubernetes.io/node=
# 清除node 角色信息
kubectl label nodes k8s-node1 node-role.kubernetes.io/node-
```

### Kubernetes概念

#### Master&Node

![](https://image.leejay.top/Fj-qU9AfR_V8wil_7ax3NgelK7dN)

#### Kubernetes 架构图

![微信截图_20220530174436](https://image.leejay.top/img/微信截图_20220530174436.png)

#### 控制平面组件

![image-20220530180740420](https://image.leejay.top/img/image-20220530180740420.png)

#### Node组件

![image-20220530180752400](https://image.leejay.top/img/image-20220530180752400.png)

### <a href="./Pod/README.md">Pod</a>

### <a href="./Deployment/README.md">Deployment</a>

### <a href="./StatefuleSet/README.md">StatefulSet</a>