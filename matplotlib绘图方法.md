# matplotlib 绘图方法汇总
绘制如下图所示多子图图表：
![image](http://orxe6lzm4.bkt.clouddn.com/YouDao/1513169385808.png)

代码如下：
```python
import matplotlib.pyplot as plt
# specifies the parameters of our graphs
fig = plt.figure(figsize=(18,6), dpi=1600) 
alpha=alpha_scatterplot = 0.2 
alpha_bar_chart = 0.55

# lets us plot many diffrent shaped graphs together 
ax1 = plt.subplot2grid((2,3),(0,0))
# plots a bar graph of those who surived vs those who did not.               
df.Survived.value_counts().plot(kind='bar', alpha=alpha_bar_chart)
# this nicely sets the margins in matplotlib to deal with a recent bug 1.3.1
ax1.set_xlim(-1, 2)
# puts a title on our graph
plt.title("Distribution of Survival, (1 = Survived)")    

plt.subplot2grid((2,3),(0,1))
plt.scatter(df.Survived, df.Age, alpha=alpha_scatterplot)
# sets the y axis lable
plt.ylabel("Age")
# formats the grid line style of our graphs                          
plt.grid(b=True, which='major', axis='y')  
plt.title("Survival by Age,  (1 = Survived)")

ax3 = plt.subplot2grid((2,3),(0,2))
df.Pclass.value_counts().plot(kind="barh", alpha=alpha_bar_chart)
ax3.set_ylim(-1, len(df.Pclass.value_counts()))
plt.title("Class Distribution")

plt.subplot2grid((2,3),(1,0), colspan=2)
# plots a kernel density estimate of the subset of the 1st class passangers's age
df.Age[df.Pclass == 1].plot(kind='kde')    
df.Age[df.Pclass == 2].plot(kind='kde')
df.Age[df.Pclass == 3].plot(kind='kde')
 # plots an axis lable
plt.xlabel("Age")    
plt.title("Age Distribution within classes")
# sets our legend for our graph.
plt.legend(('1st Class', '2nd Class','3rd Class'),loc='best') 

ax5 = plt.subplot2grid((2,3),(1,2))
df.Embarked.value_counts().plot(kind='bar', alpha=alpha_bar_chart)
ax5.set_xlim(-1, len(df.Embarked.value_counts()))
# specifies the parameters of our graphs
plt.title("Passengers per boarding location")
```

代码中涉及的函数有：
#### plt.figure()
创建一个图表，可以通过参数指定图表的属性
```
fig = plt.figure(figsize=(18,8), dpi=1600)
# figsize: 指定图表的尺寸
# dpi：    指定图表的像素(dot per inch)
```

#### plt.subplot2grid()
在同一个表中创建多个子表
```python
ax1 = plt.subplot2grid((2, 3), (0, 0))
# 两个元组参数分别表示子图的个数（两行三列）和当前图标在整个表中的位置索引（第0行第0列）
```

#### Pandas.DataFrame.plot(kind='', alpha=)
pandas库的绘图操作，可以将dataframe绘制成特定的图形，kind参数指定图表的类型，包括：
```
kind:
    line[default] 线条
    bar, barh     柱状图
    hist          直方图
    box           箱型图
    kde           kernel density estimate(拟合)
    area
    scatter       散点图
    hexbin        六角直方图
    pie           饼状图
alpha:
    透明[0.0 - 1.0]
```

#### plt.title()
为图表添加标题

#### plt.scatter()
绘制散点图，参数分别为点的x，y坐标
```python
plt.scatter(x, y)
```
