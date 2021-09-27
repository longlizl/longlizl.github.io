## docker-compose配置语法

```shell
version: '3'
services:
  web-nginx:
    image: nginx
    container_name: nginx_web
    restart: always
    ports:
      - "80:80"
    volumes:
      - weblog:/var/log
      - webhtml:/usr/share/nginx/html
    depends_on:
      - mysql
     networks:
      - nginxnw
  
  web-tomcat:
    image: tomcat
    container_name: tomcat_web
    restart: always
    ports:
      - "8082:8080"
    volumes:
      - tomcatlog:/usr/local/tomcat/logs
      - tomcatapp:/usr/local/tomcat/webapps
    networks:
      - tomcatnw

  mysql:
    image: mysql:5.735
    container_name: mysql_server
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=123456
    privileged: true
    ports:
      - "3306:3306"
    volumes:
      - mysqlconf:/etc/mysql/conf.d
      - mysqldata:/var/lib/mysql
     networks:
      - nginxnw

# 申明数据卷和网络
volumes:
  weblog:
  webhtml:
  tomcatlog:
  tomcatapp:
networks:
  nginxnw:
  tomcatnw:
```

