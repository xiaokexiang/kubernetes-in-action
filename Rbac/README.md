## Rbac
### 概念
Kubernetes中的RBAC(Role-Based Access Control)是基于角色的访问控制机制，允许用户更根据其角色授予对kubernetes资源的访问限制．
它由以下几个概念组成:
- ServiceAccount: kubernetes中用于访问资源的身份凭证
- Role: 角色定义了用户(ServiceAccount)可以对kubernetes资源执行的操作
- ClusterRole: 类似Role,但是他是在集群访问内生效，而不是特定命名空间
- RoleBinding: 将角色绑定到用户，从而授予用户对特定资源的访问权限
- ClusterRoleBinding: 类似RoleBinding,但是他是将ClusterRole绑定到用户，从而授予用户对特定资源的访问权限

### ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kube-system
  name: ${serviceAccountName}
secrets:
  - name: admin-reader-token-mwvnp
```
> sa一般通过`kubectl create sa admin-reader -n kube-system`命令创建，会自动创建同ns下的secret，secret中的ca.crt是集群的CA证书，token是访问凭证

### Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ${RoleName}
  namespace: ${namespace}
rules:
  - apiGroups: [ "","extensions","apps" ] # 规定coreApi组
    resources: [ "configmaps","pods", "services", "endpoints", "secrets" ] # 规定能访问的资源类型
    verbs: [ "get","list","watch" ] # 规定能进行的操作
```

### RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ${roleBindingName}
  namespace: ${namespace}
roleRef: # 绑定角色名称
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ${RoleName}
subjects:
  - kind: ServiceAccount # 绑定sa账户名称和命名空间
    name: ${serviceAccountName}
    namespace: ${serviceAccountNamespace}
```

### ClusterRole
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader # 不需要定义namespace，不受限
rules:
  - apiGroups: [""] # 规定coreApi组
    resources: ["secrets"] # 规定能访问的资源类型
    verbs: ["get", "watch", "list"] # 规定能进行的操作
```

### ClusterRoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ${clusterRoleBindingName}
subjects: # 绑定sa的账户
  - apiGroup: rbac.authorization.k8s.io
    kind: ServiceAccount
    name: ${serviceAccountName}
    namespace: ${serviceAccountNamespace}
roleRef: # 绑定集群角色，不需要定义namespace
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ${clusterRoleName}
```

### 生成token

```shell
#!/bin/bash
kubectl create sa management-admin -n kube-system 2>/dev/null
kubectl create clusterrolebinding management-admin --clusterrole=cluster-admin --serviceaccount=kube-system:management-admin 2>/dev/null
key=$(kubectl get sa management-admin -o=custom-columns=:.secrets[0].name -n kube-system | grep 'management')
token=$(kubectl -n kube-system get secret ${key} -o yaml | grep token: | awk '{print $2}' | base64 -d)
echo $token
```

### 基于token访问集群

```bash
curl -XGET -k https://{cluster_ip}:6443/api -H 'Authorization: Bearer ${TOKEN}'
```

```json
{
    "kind": "APIVersions",
    "versions": [
        "v1"
    ],
    "serverAddressByClientCIDRs": [
        {
            "clientCIDR": "0.0.0.0/0",
            "serverAddress": "10.50.18.27:6443"
        }
    ]
}
```

### <a href="https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/workload-resources/pod-v1/">api参考</a>

