### 1. 创建一个默认的StorageClass存储类（kubesphere默认会找默认存储如果不存在会报错）

nfs服务提前准备

```yaml
cat > storageclass-nfs.yaml  << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: storage.pri/nfs
reclaimPolicy: Delete
EOF

[root@k8s-kubernetes-master ~]# kubectl apply -f storageclass-nfs.yaml 
storageclass.storage.k8s.io/storage-nfs created
[root@k8s-kubernetes-master ~]# kubectl get sc
NAME                    PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storage-nfs (default)   storage.pri/nfs   Delete          Immediate           false                  3s

# nfs-client-sa.yaml授权配置
cat > nfs-client-sa.yaml 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
 
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
 
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
EOF
# Deployment配置
cat > nfs-client.yaml << 'EOF'
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: vbouchaud/nfs-client-provisioner
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME #供应者的名字
              value: storage.pri/nfs #名字虽然可以随便起，以后引用要一致
            - name: NFS_SERVER
              value: 192.168.205.150
            - name: NFS_PATH
              value: /opt/kubesphere/data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.205.150
            path: /opt/kubesphere/data
EOF 
kubectl apply -f nfs-client-sa.yaml
kubectl apply -f nfs-client.yaml
```

### 2. 官网下载所需要yaml资源

```shell
wget https://github.com/kubesphere/ks-installer/releases/download/v3.3.0/kubesphere-installer.yaml
wget https://github.com/kubesphere/ks-installer/releases/download/v3.3.0/cluster-configuration.yaml
kubectl apply -f kubesphere-installer.yaml 
kubectl apply -f cluster-configuration.yaml 
```

```shell
# 查看安装成功后日志信息如下图
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f

```

![image-20220818180057694](https://longlizl.github.io/k8s/images/6.png)

如果中途安装失败报如下错误信息（此错误会在重新安装服务时候会遇到）

```shell
failed: [localhost] (item={'ns': 'kubesphere-system', 'kind': 'users.iam.kubesphere.io', 'resource': 'admin', 'release': 'ks-core'}) => {"ansible_loop_var": "item", "changed": true, "cmd": "/usr/local/bin/kubectl -n kubesphere-system annotate --overwrite users.iam.kubesphere.io admin meta.helm.sh/release-name=ks-core && /usr/local/bin/kubectl -n kubesphere-system annotate --overwrite users.iam.kubesphere.io admin meta.helm.sh/release-namespace=kubesphere-system && /usr/local/bin/kubectl -n kubesphere-system label --overwrite users.iam.kubesphere.io admin app.kubernetes.io/managed-by=Helm\n", "delta": "0:00:00.297977", "end": "2021-12-13 20:55:02.145813", "failed_when_result": true, "item": {"kind": "users.iam.kubesphere.io", "ns": "kubesphere-system", "release": "ks-core", "resource": "admin"}, "msg": "non-zero return code", "rc": 1, "start": "2021-12-13 20:55:01.847836", "stderr": "Error from server (InternalError): Internal error occurred: failed calling webhook \"users.iam.kubesphere.io\": Post \"https://ks-controller-manager.kubesphere-system.svc:443/validate-email-iam-kubesphere-io-v1alpha2?timeout=30s\": service \"ks-controller-manager\" not found", "stderr_lines": ["Error from server (InternalError): Internal error occurred: failed calling webhook \"users.iam.kubesphere.io\": Post \"https://ks-controller-manager.kubesphere-system.svc:443/validate-email-iam-kubesphere-io-v1alpha2?timeout=30s\": service \"ks-controller-manager\" not found"], "stdout": "", "stdout_lines": []}

PLAY RECAP *********************************************************************

localhost : ok=16 changed=10 unreachable=0 failed=1 skipped=6 rescued=0 ignored=0
```

使用以下命令

```shell
# 1.将failurePolicy值改为Ignore
kubectl edit validatingwebhookconfigurations users.iam.kubesphere.io
# 2.重启ks-installer pod
# 3.重新创建配置
kubectl apply -f cluster-configuration.yaml

```

### 3. 查看存储挂载信息

```shell
[root@k8s-kubernetes-master ~]# kubectl get pvc -A
NAMESPACE                      NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
default                        test-pv-claim                        Bound    pvc-5edb887d-1746-412c-bf08-b7cd9be36325   1Gi        RWO            storage-nfs    16h
kubesphere-monitoring-system   prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-f3373b5c-0fe1-40a3-8d90-726d13ef8ba5   20Gi       RWO            storage-nfs    16h
kubesphere-monitoring-system   prometheus-k8s-db-prometheus-k8s-1   Bound    pvc-26640964-eaac-4c6e-8f57-2708fa71f838   20Gi       RWO            storage-nfs    16h
# 查看挂载目录信息
[root@k8s-kubernetes-master ~]# ll /opt/kubesphere/data/
总用量 0
drwxrwxrwx 2 root root  6 8月  23 17:26 default-test-pv-claim-pvc-5edb887d-1746-412c-bf08-b7cd9be36325
drwxrwxrwx 3 root root 27 8月  23 17:47 kubesphere-monitoring-system-prometheus-k8s-db-prometheus-k8s-0-pvc-f3373b5c-0fe1-40a3-8d90-726d13ef8ba5
drwxrwxrwx 3 root root 27 8月  23 17:47 kubesphere-monitoring-system-prometheus-k8s-db-prometheus-k8s-1-pvc-26640964-eaac-4c6e-8f57-2708fa71f838
```



### 4. 访问服务（登陆时需修改密码）

http://192.168.205.150:30880/

![image-20220819164826137](https://longlizl.github.io/k8s/images/7.png)