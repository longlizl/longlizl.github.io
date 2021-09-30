处理服务器日志文件并根据条件筛选出所需要的数据写入另一个文件中

> 我们从下面日志格式中筛选出符合条件batteryVoltage > 4 的日志文件

![image-20210930152512125](https://longlizl.github.io/python/images/1.png)

```python
import os
with open(r'C:\Users\15507\Desktop\info.log',encoding='utf-8') as f:
    with open(r'C:\Users\15507\Desktop\info1.txt','w',encoding='utf-8') as f1:
        for lines in f.readlines():
            if 'batteryVoltage' not  in lines:
                continue
            else:
                # print(lines.index('batteryVoltage'))
                if int(lines[310]) > 4:
                    f1.writelines(lines)
```

> 打开处理好的文件可以看到需要的数据已经帅选出来了

![image-20210930153634291](https://longlizl.github.io/python/images/2.png)

