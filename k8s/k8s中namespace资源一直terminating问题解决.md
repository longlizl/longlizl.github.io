

查看需要删除得ns资源名称

```shell
[root@k8s-kubernetes-master ~]# kubectl get ns | grep Terminating
kubesphere-monitoring-federated   Terminating   24h
kubesphere-monitoring-system      Terminating   24h
```

### 1.开启apiserver本地代理

```shell
[root@k8s-kubernetes-master ~]# kubectl proxy
Starting to serve on 127.0.0.1:8001
```

### 2. 导出名称空间json文件

```shell
[root@k8s-kubernetes-master ~]# kubectl get ns kubesphere-monitoring-federated  -o json > kubesphere-monitoring-federated.json
[root@k8s-kubernetes-master ~]# kubectl get ns kubesphere-monitoring-system -o json  > kubesphere-monitoring-system.json
```

### 3. 删除json文件中metadata.finalizers字段及其内容删除

```shell
# 把文件里这一段配置去掉
        "finalizers": [
            "finalizers.kubesphere.io/namespaces"
        ],
```

### 4. 执行以下命令删除

```shell
curl -k -H "Content-Type: application/json" -X PUT --data-binary @kubesphere-monitoring-system.json http://127.0.0.1:8001/api/v1/namespaces/kubesphere-monitoring-system/finalize
curl -k -H "Content-Type: application/json" -X PUT --data-binary @kubesphere-monitoring-federated.json http://127.0.0.1:8001/api/v1/namespaces/kubesphere-monitoring-federated/finalize
```

### 5. 查看名称空间

```shell
[root@k8s-kubernetes-master ~]#kubectl get ns
NAME              STATUS   AGE
default           Active   7d
kube-node-lease   Active   7d
kube-public       Active   7d
kube-system       Active   7d
pro-dev           Active   6d23h
test-dev          Active   3d
```

