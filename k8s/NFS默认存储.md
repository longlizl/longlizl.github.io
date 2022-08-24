### 1.NFS服务安装

```shell
# 安装nfs服务
yum -y install nfs
vim /etc/exports
/opt/nginx/nginxhtml *(rw,no_root_squash,sync)
systemctl start nfs
systemctl enable nfs
```

### 2. NFS-rbac授权

```shell
# nfs rbac授权
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

kubectl apply -f nfs-client-sa.yaml
```

### 3. 创建nfs-client授权

```shell
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
kubectl apply -f nfs-client.yaml
```

### 4. 创建默认存储类

```shell
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
kubectl apply -f storageclass-nfs.yaml
# 查看默认存储信息
[root@k8s-kubernetes-master ~]# kubectl get sc
NAME                    PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storage-nfs (default)   storage.pri/nfs   Delete          Immediate           false                  19h
```

### 5.创建一个pvc自动绑定绑定默认存储

```shell
# 创建一个测试名称空间test-storage
kubectl creat ns test-storage
# 创建pvc绑定默认存储
cat > storageclass-nfs.yaml < 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: storage.pri/nfs
reclaimPolicy: Delete
parameters:
  server: 192.168.205.150
  path: /opt/kubesphere/data
  readOnly: "false"
EOF
kubectl apply -f storageclass-nfs.yaml
# 查看pvc信息
[root@k8s-kubernetes-master ~]# kubectl get pvc -n test-storage
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pv-claim   Bound    pvc-77936243-ceeb-40a1-a040-f0e6d13f3ad6   1Gi        RWO            storage-nfs    63m
```

