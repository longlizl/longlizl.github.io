需求：需要将表格中报告人那列名字，从已准备好的存有上千人名字的文本中随机取出一个来替换

```python
# 导入相关模块
import pandas as pd
import os
import random
# 展示所有行和列
pd.set_option('display.max_columns',None)
pd.set_option('display.max_rows',None)
# 打开文本文件将字符串以','分隔转换成列表
with open(r'C:\Users\15507\Desktop\name.txt','r',encoding='utf-8') as f:
    for line in f.readlines():
        str_name = line
        name_list = str_name.split(',')
# print(name_list[3])
# 返回一个有6个元素的列表
#b = random.sample(name_list,6)
# print(a)
table = pd.read_excel(r'C:\Users\15507\Desktop\交付清单\运维支持工作日报(2021.9.11 - 2021.9.17) .xlsx',sheet_name=0,index_col=None)
# print(table)
for i in table.index:
    # 值为NaN的单元格跳过
    if pd.isnull(table.报告人.at[i]):
        continue
    # 从列表中取出一个随机名字
    a = random.choice(name_list)
    # 随机赋值
    table.报告人.at[i] = a
    # print(table.报告人.at[i])
# 将表中‘关键字’字段设置为索引，保存时不生成新的索引列
table.set_index('关键字',inplace=True)
table.to_excel(r'c:\Users\15507\Desktop\运维支持工作日报(2021.9.11 - 2021.9.17)_bak.xlsx')
```
