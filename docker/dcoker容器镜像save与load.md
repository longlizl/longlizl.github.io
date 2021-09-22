1. 导出镜像save

   docker save [options] images [images...]

   ```shell
   [root@cs ~]# docker images
   REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
   tomcat       vv6       8d1d0c5d815a   13 minutes ago   680MB
   tomcat       latest    bb832de23021   3 days ago       680MB
   nginx        latest    ad4c705f24d3   8 days ago       133MB
   ```

   ```shell
   docker save bb832de23021 > tomcatv1.tar
   ```

   将一个运行的容器保存为镜像

   ```shell
   [root@cs ~]# docker ps
   CONTAINER ID   IMAGE     COMMAND             CREATED          STATUS          PORTS                                       NAMES
   a3c04c98efa2   tomcat    "catalina.sh run"   29 minutes ago   Up 29 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   tomcat_web
   
   ```

   ```shell
   # docker commit [OPTIONS] CONTAINER/id [REPOSITORY[:TAG]]
   docker commit -a '1550789579@qq.com' -m 'add test web' tomcat_web tomcat:vv6
   ```

   

2. 导入镜像load

   ```shell
   [root@cs ~]# docker load  <  tomcatv1.tar
   Loaded image ID: sha256:bb832de230212b71e43b7b6b7e12b830f5b329d21ab690d59f0fd85b01045574
   
   [root@cs ~]# docker images
   REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
   tomcat       vv6       8d1d0c5d815a   29 minutes ago   680MB
   tomcat       latest    bb832de23021   3 days ago       680MB
   nginx        latest    ad4c705f24d3   8 days ago       133MB
   ```

   

