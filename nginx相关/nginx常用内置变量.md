

# nginx常用内置变量

```nginx
1. $args     # 这个变量等于请求行中的参数。
2. $binary_remote_addr     # 远程地址的二进制表示
3. $body_bytes_sent    # 已发送的消息体字节数
4. $content_length     # 请求头中的Content-length字段
5. $content_type     # 请求头中的Content-Type字段
6. $document_uri     # 与$uri相同
7. $document_root     # 当前请求在root指令中指定的值
8. $host     # 请求主机头字段，否则为服务器名称
9. $http_user_agent     # 客户端agent信息
10. $http_cookie     # 客户端cookie信息
11. $http_referer    # 引用地址
12. $http_user_agent    # 客户端代理信息
13. $http_via    # 最后一个访问服务器的Ip地址
14. $http_x_forwarded_for    # 相当于网络访问路径
15. $query_string    # 与$args相同
16. $request_method     # 客户端请求的动作，通常为GET或POST
17. $limit_rate     # 这个变量可以限制连接速率
18. $request_body_file     # 客户端请求主体信息的临时文件名
19. $remote_addr     # 客户端的IP地址
20. $remote_port     # 客户端的端口
21. $remote_user     # 已经经过Auth Basic Module验证的用户名
22. $request        # 用户请求
23. $request_body_file        # 发往后端的本地文件名称
24. $request_filename        # 当前请求的文件路径，由root或alias指令与URI请求生成
25. $request_method        # 请求的方法，比如 GET 、POST 等
26. $request_uri        # 请求的URI，带参数
27. $query_string     # 与$args相同
28. $scheme     # HTTP方法（如http，https）
29. $server_protocol     # 请求使用的协议，通常是HTTP/1.0或HTTP/1.1
30. $server_addr     # 服务器地址，在完成一次系统调用后可以确定这个值
31. $server_name     # 服务器名称
32. $server_port     # 请求到达服务器的端口号
33. $request_uri     # 包含请求参数的原始URI，不包含主机名，如 /foo/bar.php?arg=baz
34. $uri     # 不带请求参数的当前URI，$uri不包含主机名，如 /foo/bar.html
```

