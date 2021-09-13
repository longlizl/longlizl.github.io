# 处理表格可视化模块

> 1. pyecharts
> 2. matplotlib

1.   使用pyecharts处理表格以html页面展示

   下面以折线图为例（其他图语法可以参考官网[pyecharts](https://pyecharts.org/)）

   ```python
   # 数据可视化模块
   from pyecharts.charts import Bar
   from pyecharts import options as opts
   from pyecharts.charts import Line
   import os
   # excel处理模块 读：xlrd(read)，写：xlrd(write)
   import xlrd
   # 读取excel文件
   data = xlrd.open_workbook(r'C:\Users\15507\Desktop\各公司按周统计超速.xlsx')
   # 通过索引读取sheet表
   # table = data.sheets()[1]
   sheet_list = data.sheet_names()
   # print(sheet_list)
   for x in range(1,len(sheet_list)):
       table = data.sheets()[x]
       # 通过表名读取
       # table = data.sheet_by_name('xxx-xxx物流有限公司')
       # 打印表的行数,列数
       # print(table.nrows,table.ncols)
       # 存放时间，超速，振动，负载，点检，离位列表
       time_list = []
       cs_list = []
       zd_list = []
       fz_list = []
       dj_list = []
       lw_list = []
       # 循环遍历表格每行数据
       for i in range(1,table.nrows):
           # print(table.row_values(i))
           time_list.append(table.row_values(i)[0])
           cs_list.append(table.row_values(i)[2])
           zd_list.append(table.row_values(i)[3])
           fz_list.append(table.row_values(i)[4])
           # dj_list.append(table.row_values(i)[6])
           # lw_list.append(table.row_values(i)[7])
       # print(time_list)
       # print(cs_list)
       # print(fz_list)
       # print(fz_list)
       # print(dj_list)
       # print(lw_list)
       # 折线图
       line = Line()
       line.add_xaxis(time_list)
       # 设置折线宽度
       line.add_yaxis('超速',cs_list,linestyle_opts=opts.LineStyleOpts(width=2))
       line.add_yaxis('振动',zd_list,linestyle_opts=opts.LineStyleOpts(width=2))
       line.add_yaxis('负载',fz_list,linestyle_opts=opts.LineStyleOpts(width=2))
       # line.add_yaxis('点检',dj_list)
       # line.add_yaxis('离位',lw_list)
       line.set_colors(['red','green','blue'])
       # 配置全局参数 标题，x轴标题倾斜度.datazoom_opts设置x轴可缩放，orient="vertical"为Y轴缩放 默认不加参数为x轴
       line.set_global_opts(opts.TitleOpts(title="xxxx", subtitle=sheet_list[x]), \
                            xaxis_opts=opts.AxisOpts(name_rotate=60,axislabel_opts={"rotate":45},min_=0,max_=24), \
                            datazoom_opts=opts.DataZoomOpts())
   
       # 循环遍历工作簿里各sheet并生成html文件
       if sheet_list[x]:
           line.render(sheet_list[x])
           os.replace(sheet_list[x],sheet_list[x] + '.html')
   # print(data.sheet_names())
   ```

   

2. 使用matplotlib处理表格可视化

   ```python
   ############### 柱状图 #################
   # import pandas as pd
    import matplotlib.pyplot as plt
    # 设置可视化图中文字体（不设置中文会是乱码）
    plt.rcParams['font.sans-serif'] = ['SimHei']
    # 处理坐标为负数时显示乱码问题
    plt.rcParams['axes.unicode_minus'] = False
    # 读取成绩表第一个sheet（默认为第一个sheetname这里写的是索引号也可以写成sheet_name='成绩'）
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # 打印行和列数
    print(data.shape)
    # 对分数这列降序排序
    data.sort_values(by='分数',inplace=True,ascending=False)
    # 设置x轴和y轴
    plt.bar(data.姓名,data.分数,label='成绩')
    # 设置标签成绩展示(默认右上) loc='upper left'(左上)
    plt.legend()
    # 设置x轴和y轴标签
    plt.xlabel('姓名')
    plt.ylabel('分数')
    # 设置x坐标轴标签倾斜度
    plt.xticks(data.姓名,rotation=45)
    # 设置y轴坐标取值范围
    plt.ylim([-30,120])
    # 设置图标大标题
    plt.title('学生成绩表',fontsize=18,fontweight='bold',color='green',loc='center')
    # 自动适应布局
    plt.tight_layout()
    # 将读取的内容写入另一个表
    data.to_excel(r'D:\xlsx\成绩bak.xlsx',index=False)
    # print(data)
    # 打印索引分数列索引为0的数
    print(data.分数[0])
    # 展示柱状图
    plt.show()
   
   ############### 条形图水平 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    # 读取成绩表第一个sheet（这里写的是索引号也可以写成sheet_name='成绩'）
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # 对分数这列降序排序
    data.sort_values(by='分数',inplace=True,ascending=False)
    plt.bar(x=0,height=0.5,bottom=data.姓名,width=data.分数,color='red',orientation='horizontal')
    plt.show()
   
   ############### 折线图 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # grid=True参数显示网格
    # data.plot(x='姓名',y='分数',grid=True)
    # 多y轴值
    data.plot(x='姓名',y=['年龄','分数'],grid=True,kind='line')
    plt.show()
   
   ############### 条形图垂直展示 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # grid=True参数显示网格
    # data.plot(x='姓名',y='分数',grid=True)
    # 多y轴值
    data.plot(x='姓名',y=['年龄','分数'],grid=True,kind='bar')
    # 自动适应布局
    plt.tight_layout()
    plt.show()
   
   ############### 水平图 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # grid=True参数显示网格
    # data.plot(x='姓名',y='分数',grid=True)
    # 多y轴值
    data.plot(x='姓名',y=['年龄','分数'],grid=True,kind='barh')
    # 自动适应布局
    plt.tight_layout()
    plt.show()
   
   ############### 散点图 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # grid=True参数显示网格
    # data.plot(x='姓名',y='分数',grid=True)
    # 多y轴值
    data.plot(x='姓名',y='年龄',grid=True,kind='scatter',color='red')
    # 自动适应布局
    plt.tight_layout()
    plt.show()
   
   ############### 饼图 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    data.sort_values(by='年龄',inplace=True,ascending=False)
    data.index = ['小李','小红','小明','小花','小孙','小田','小张']
    # 多y轴值
    data.plot(y='年龄',kind='pie',legend=False)
    # 自动适应布局
    plt.tight_layout()
    plt.show()
   
   ############### 多图排列 #################
    import pandas as pd
    import matplotlib.pyplot as plt
    plt.rcParams['font.sans-serif'] = ['SimHei']
    data = pd.read_excel(r'D:\xlsx\成绩.xlsx',sheet_name=0)
    # grid=True参数显示网格
    # data.plot(x='姓名',y='分数',grid=True)
    # layout=(1,2) 1行2列 ，figsize=(10, 5)长度和宽度
    data.plot(x='姓名',y=['年龄','分数'],subplots=True,layout=(1,2),figsize=(10, 5),kind='line')
    plt.show()
   ```

   

