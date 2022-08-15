### 1. 准备一台NFS文件存储服务器(以nfs为底层存储系统为例)

```shell
# 这里将主节点配置成一台nfs服务器
yum -y install nfs
vim /etc/exports
/opt/nginx/nginxhtml *(rw,no_root_squash,sync)
systemctl start nfs
systemctl enable nfs
# 其他各节点安装nfs服务
yum -y install nfs
systemctl start nfs
systemctl enable nfs
```

### 2.  配置pv与pvc文件

```shell
# 配置pv
cat > nginx-pv.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/nginx/nginxhtml
    server: 192.168.205.150
EOF

# 配置pvc
cat > nginx-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginxhtmlpvc
  namespace: pro-dev
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
 EOF
 # 查看pv与pvc信息
 [root@k8s-kubernetes-master ~]# kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
nginx-pv   5Gi        RWX            Retain           Bound    pro-dev/nginxhtmlpvc                           150m
[root@k8s-kubernetes-master ~]# kubectl get pvc -n pro-dev
NAME           STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nginxhtmlpvc   Bound    nginx-pv   5Gi        RWX                           140m

```

### 3. pvc绑定到pod控制器上

```shell
cat > nginx.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: pro-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.22
        ports:
        - containerPort: 80
        # 资源请求与限制
        resources:
          requests:
            memory: "0.5Gi"
            cpu: "100m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html/index.html
            subPath: index.html # 子路径修改内容后不能去动态更新到pod容器里
#          - name: config
#            mountPath: /etc/nginx/nginx.conf
#            subPath: nginx.conf
        # 就绪探针
        readinessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        # 存活性探针
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 15
      volumes:
        - name: html
          persistentVolumeClaim:
            claimName: nginxhtmlpvc
---
# 定义nodePort类型service控制器
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: pro-dev
spec:
  selector:
    app: nginx-pod
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31002
EOF

kubectl  create -f nginx.yaml --record=true

[root@k8s-kubernetes-master ~]# kubectl get pod -n pro-dev -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP               NODE                   NOMINATED NODE   READINESS GATES
nginx-deployment-88866c8f5-k6zk2   1/1     Running   0          30m   10.244.195.72    k8s-kubernetes-node1   <none>           <none>
nginx-deployment-88866c8f5-vvzt5   1/1     Running   0          30m   10.244.130.136   k8s-kubernetes-node2   <none>           <none>

```

### 4. 准备一个测试页面

```shell
vim /opt/nginx/nginxhtml/index.html
------------------------------------------welcome to website-----------------------------------------------------
# 这里需要注意一点的是我们之前绑定pod绑定PVC时使用了subPath子路径挂载，在后续更改其index.html文件内容时是不能动态更新的，访问时会报错。需要重新删掉pod在创建。如果只配置 mountPath: /usr/share/nginx/html 是可以动态更新
```

使用任意node节点+port端口访问

![image-20220815141626045](https://longlizl.github.io/k8s/images/13.png)

我们现在去掉nginx.yaml中子路径配置，使用 mountPath: /usr/share/nginx/html 改成如下

```
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html
```

变更当前配置

```shell
kubectl apply -f nginx.yaml
```

修改index.html页面

```shell
echo '---hello word---' > /opt/nginx/nginxhtml/index.html
```

再去访问页面可以看到页面已经更改了

![image-20220815143331580](https://longlizl.github.io/k8s/images/14.png)