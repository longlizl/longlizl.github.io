### 1.HPA的工作原理  

​        Kubernetes中的某个Metrics Server持续采集所有Pod副本的指标数据。HPA控制器通过Metrics Server的API获取这些数据，基于用户定义的扩缩容规则进行计算，得到目标Pod的副本数量。当目标Pod副本数量 与 当 前 副 本 数 量 不 同 时 ， HPA 控 制 器 就 向 Pod 的 副 本 控 制 器（Deployment、RC或ReplicaSet）发起scale操作，调整Pod的副本数量，完成扩缩容操作。如下图展示了HPA体系中的关键组件和工作流程。接下来首先对HPA能够管理的指标类型、扩缩容算法、HPA对象的配置进行详细说明，然后通过一个完整的示例对如何搭建和使用基于自定义指标的HPA体系进行说明。  

![image-20220816142339574](https://longlizl.github.io/k8s/images/15.png)

### 2. 准备好Metrics Server  yaml文件

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: cnskylee/metrics-server:v0.5.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```

```shell
# 创建pod
kubectl apply -f metrics-server.yaml
[root@k8s-kubernetes-master metrics-server]# kubectl get pod -n kube-system | grep metrics-server
metrics-server-67b7f545f-l8xz9                  1/1     Running   0             34m
# 查看node资源使用
[root@k8s-kubernetes-master metrics-server]# kubectl top node
NAME                    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-kubernetes-master   419m         10%    1164Mi          67%       
k8s-kubernetes-node1    215m         5%     835Mi           48%       
k8s-kubernetes-node2    221m         5%     888Mi           51%  
```

### 3. 创建一个deployment测试

```yaml
cat > test_nginx.yaml  << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: test-dev
spec:
  replicas: 1
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
---
# 创建一个server
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: test-dev
spec:
  selector:
    app: nginx-pod
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31010
EOF
# 创建pod,service
kubectl apply -f test_nginx.yaml
[root@k8s-kubernetes-master ~]# kubectl get pod,svc -n test-dev -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP              NODE                   NOMINATED NODE   READINESS GATES
pod/nginx-deployment-5b7f6454f8-cr6f7   1/1     Running   0          87s   10.244.195.80   k8s-kubernetes-node1   <none>           <none>

NAME                    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/nginx-service   NodePort   10.1.48.179   <none>        80:31010/TCP   44s   app=nginx-pod

```

### 4. 创建HPA

```shell
# –cpu-percent=50 表示所有PodCPU使用率在超过50％扩容
# –min=1 --max=3 表示pod扩缩数量的范围
[root@k8s-kubernetes-master ~]# kubectl autoscale deployment nginx-deployment -n test-dev --cpu-percent=50 --min=1 --max=3
horizontalpodautoscaler.autoscaling/nginx-deployment autoscaled
# 查看当前hpa信息
[root@k8s-kubernetes-master ~]# kubectl get hpa -n test-dev
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   0%/50%    1         3         1          4h4m

```

打开2个终端分别观察pod的cpu和pod扩容副本的情况 

```shell
[root@k8s-kubernetes-master ~]# kubectl get hpa -n test-dev -w
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   0%/50%    1         3         1          4h42m

[root@k8s-kubernetes-master ~]# kubectl get pod -n test-dev -w
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5b7f6454f8-cr6f7   1/1     Running   0          20m
```

### 5. 下面进行压力测试

运行一个临时容器执行以下命令通过server ip进行访问

```shell
[root@k8s-kubernetes-master ~]# kubectl run busybox --rm -it --image=busybox /bin/sh
while true;do wget -q -O- 10.1.48.179;done
```

```shell
# 观察hpa和pod信息
[root@k8s-kubernetes-master ~]# kubectl get hpa -n test-dev -w
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   0%/50%    1         3         1          4h42m
nginx-deployment   Deployment/nginx-deployment   125%/50%   1         3         1          4h43m
nginx-deployment   Deployment/nginx-deployment   125%/50%   1         3         3          4h43m
[root@k8s-kubernetes-master ~]# kubectl get pod  -n test-dev -w
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-568bfb7b76-vcskz   1/1     Running   0          7m8s
nginx-deployment-568bfb7b76-gnrzc   0/1     Pending   0          0s
nginx-deployment-568bfb7b76-gnrzc   0/1     Pending   0          0s
nginx-deployment-568bfb7b76-ffg8d   0/1     Pending   0          0s
nginx-deployment-568bfb7b76-ffg8d   0/1     Pending   0          0s
nginx-deployment-568bfb7b76-ffg8d   0/1     ContainerCreating   0          0s
nginx-deployment-568bfb7b76-gnrzc   0/1     ContainerCreating   0          0s
nginx-deployment-568bfb7b76-ffg8d   0/1     ContainerCreating   0          2s
nginx-deployment-568bfb7b76-gnrzc   0/1     ContainerCreating   0          2s
nginx-deployment-568bfb7b76-gnrzc   1/1     Running             0          4s
nginx-deployment-568bfb7b76-ffg8d   1/1     Running             0          4s

[root@k8s-kubernetes-master ~]# kubectl get pod -n test-dev
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-568bfb7b76-ffg8d   1/1     Running   0          2m24s
nginx-deployment-568bfb7b76-gnrzc   1/1     Running   0          2m24s
nginx-deployment-568bfb7b76-vcskz   1/1     Running   0          27m

# 当退出busybox后等待一会大概5分钟左右会自动缩减到一个pod
[root@k8s-kubernetes-master ~]# kubectl get pod  -n test-dev
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-568bfb7b76-cg6ws   1/1     Running   0          16m
```

