### 将nginx外部配置文件挂载到pod中

#### 1. 从pod中复制一份nginx配置文件

```shell
kubectl cp pro-dev/nginx-deployment-6dd84d7654-gf98w:etc/nginx/conf.d/default.conf /root/default.conf
# 修改配置文件在文件最后增加8080端口监听
server {
    listen       8080;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

#### 2. 创建configmap文件

```shell
# 这里-n pro-dev为pod运行的名称空间
kubectl create configmap nginx-config --from-file=default.conf -n pro-dev
# 查看创建的configmap信息
[root@k8s-kubernetes-master ~]# kubectl get cm -n pro-dev
NAME               DATA   AGE
kube-root-ca.crt   1      3d
nginx-config       1      19m
# 查看具体配置信息
[root@k8s-kubernetes-master ~]# kubectl describe cm nginx-config  -n pro-dev
Name:         nginx-config
Namespace:    pro-dev
Labels:       <none>
Annotations:  <none>

Data
====
default.conf:
----
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}

server {
    listen       8080;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}


BinaryData
====

Events:  <none>
```

#### 3. 在pod中绑定configmap信息，service配置多端口

```shell
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
        resources:
          requests:
            memory: "0.5Gi"
            cpu: "100m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html
          # 挂载nginx配置
          - name: confnginx
            mountPath: /etc/nginx/conf.d
            readOnly: true #只有
#            subPath: index.html
#          - name: config
#            mountPath: /etc/nginx/nginx.conf
#            subPath: nginx.conf
        readinessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 15
      volumes:
        - name: html
          persistentVolumeClaim:
            claimName: nginxhtmlpvc
        # 引入config配置
        - name: confnginx
          configMap:
            name: nginx-config

---
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
    name: webserver # name名用于端口区分
  - port: 8080
    targetPort: 8080
    nodePort: 31110
    name: weblisten
```

#### 4. 应用配置信息

```shell
kubectl apply -f nginx.yaml

[root@k8s-kubernetes-master ~]# kubectl get pod,svc -n pro-dev -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP               NODE                   NOMINATED NODE   READINESS GATES
pod/nginx-deployment-854f9f945d-b7qrd   1/1     Running   0          52s   10.244.130.139   k8s-kubernetes-node2   <none>           <none>
pod/nginx-deployment-854f9f945d-tznpc   1/1     Running   0          52s   10.244.195.75    k8s-kubernetes-node1   <none>           <none>

NAME                    TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)                       AGE   SELECTOR
service/nginx-service   NodePort   10.1.42.70   <none>        80:31002/TCP,8080:31110/TCP   13m   app=nginx-pod

[root@k8s-kubernetes-master ~]# curl 10.244.195.75:8080
---hello word---
# 使用节点ip+31110端口访问也能成功访问
```

