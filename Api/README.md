## 基于REST API 访问集群

### 创建token

```shell
#!/bin/bash
kubectl create sa management-admin -n kube-system 2>/dev/null
kubectl create clusterrolebinding management-admin --clusterrole=cluster-admin --serviceaccount=kube-system:management-admin 2>/dev/null
key=$(kubectl get sa management-admin -o=custom-columns=:.secrets[0].name -n kube-system | grep 'management')
token=$(kubectl -n kube-system get secret ${key} -o yaml | grep token: | awk '{print $2}' | base64 -d)
echo $token

# 等待令牌控制器使用令牌填充 secret:
while ! kubectl describe secret default-token | grep -E '^token' >/dev/null; do
  echo "waiting for token..." >&2
  sleep 1
done

# 获取令牌
echo $(kubectl get secret default-token -o jsonpath='{.data.token}' | base64 --decode)
```

### 访问集群

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

