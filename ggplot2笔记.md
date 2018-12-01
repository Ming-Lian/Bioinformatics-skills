<a name="content">目录</a>

[ggplot2笔记](#title)
- [直方图：geom_histogram](#histogram)
- [柱状图：geom_bar](#bar)
- [控制整体图形属性](#whole-plot-style)
	- [标尺（Scale）](#scale)
	- [标题](#plot-title)
	- [图例](#legend)
- [统计变换（Statistics）](#statistics)
- [坐标系统（Coordinante）](#coordinante)
- [分面（Facet）](#facet)
- [作分组箱线图并添加均值点连线及显著性程度标注](#boxplot-plus-siginfo)



<h1 name="title">ggplot2笔记</h1>

<a name="histogram"><h2>直方图：geom_histogram [<sup>目录</sup>](#content)</h2></a>

分组直方图的排列方式

- `position="dodge"`

```
ggplot(small)+geom_histogram(aes(x=price, fill=cut), position="dodge")
```

<p align="center"><img src=./picture/ggplot2-position-1.png width=600 /></p>

- `position="fill"`

```
ggplot(small)+geom_histogram(aes(x=price, fill=cut), position="fill")
```

<p align="center"><img src=./picture/ggplot2-position-2.png width=600 /></p>

<a name="bar"><h2>柱状图：geom_bar [<sup>目录</sup>](#content)</h2></a>

柱状图两个要素，一个是分类变量，一个是数目，也就是柱子的高度。数目在这里不用提供，因为ggplot2会通过x变量计算各个分类的数目。

当然你想提供也是可以的，通过stat参数，可以让geom_bar按指定高度画图，比如以下代码：

```
ggplot()+geom_bar(aes(x=c(LETTERS[1:3]),y=1:3), stat="identity")
```

区分柱状图和直方图：

> 柱状图和直方图是很像的，直方图把连续型的数据按照一个个等长的分区（bin）来切分，然后计数，画柱状图。而柱状图是分类数据，按类别计数

<a name="whole-plot-style"><h2>控制整体图形属性 [<sup>目录</sup>](#content)</h2></a>

<a name="scale"><h3>标尺（Scale） [<sup>目录</sup>](#content)</h3></a>

在对图形属性进行映射之后，使用标尺可以控制这些属性的显示方式，比如坐标刻度，可能通过标尺，将坐标进行对数变换 `scale_y_log10()`；比如颜色属性，也可以通过标尺，进行改变 `scale_colour_manual(values=rainbow(7))`。

<a name="plot-title"><h3>标题 [<sup>目录</sup>](#content)</h3></a>

设置图像标题：`ggtitle()`

设置X轴标题：`xlab()`

设置Y轴标题：`ylab()`

```
p + ggtitle("Price vs Cut")+xlab("Cut")+ylab("Price")
```

同时设置图像标题、X轴标题、Y轴标题：`labs`

```
labs(x="Cut",y="Price",title="Price vs Cut")
```

标题居中：`theme(plot.title=element_text(hjust=0.5))`

<a name="legend"><h3>图例 [<sup>目录</sup>](#content)</h3></a>

隐藏图例：`theme(legend.position="none")`

<a name="statistics"><h2>统计变换（Statistics） [<sup>目录</sup>](#content)</h2></a>

统计变换对原始数据进行某种计算，然后在图上表示出来

对散点图上加一条回归线 `stat_smooth()`：

```
ggplot(small, aes(x=carat, y=price))+geom_point()+scale_y_log10()+stat_smooth()
```

<p align="center"><img src=./picture/ggplot2-statistic-smooth.png width=600 /></p>

<a name="coordinante"><h2>坐标系统（Coordinante） [<sup>目录</sup>](#content)</h2></a>

坐标系统控制坐标轴，可以进行变换，例如XY轴翻转，笛卡尔坐标和极坐标转换，以满足我们的各种需求。

坐标轴翻转由`coord_flip()`实现

```
ggplot(small)+geom_bar(aes(x=cut, fill=cut))+coord_flip()
```

<p align="center"><img src=./picture/ggplot2-oordinante-flip.png width=600 /></p>

转换成极坐标可以由`coord_polar()`实现：

```
ggplot(small)+geom_bar(aes(x=factor(1), fill=cut))+coord_polar(theta="y")
```

<p align="center"><img src=./picture/ggplot2-oordinante-polar.png width=600 /></p>

饼图实际上就是柱状图，只不过是使用极坐标而已，柱状图的高度，对应于饼图的弧度

<a name="facet"><h2>分面（Facet） [<sup>目录</sup>](#content)</h2></a>

分面可以让我们按照某种给定的条件，对数据进行分组，然后分别画图 `facet_wrap(~cut)`

在统计变换一节中，提到如果按切工分组作回归线，显然图会很乱，有了分面功能，我们可以分别作图

```
ggplot(small, aes(x=carat, y=price))+geom_point(aes(colour=cut))+scale_y_log10() +facet_wrap(~cut)+stat_smooth()
```

<p align="center"><img src=./picture/ggplot2-facet.png width=600 /></p>

<a name="boxplot-plus-siginfo"><h2>作分组箱线图并添加均值点连线及显著性程度标注 [<sup>目录</sup>](#content)</h2></a>

<p align="center"><img src=./picture/ggplot2-boxplot-plus-siginfo-1.png width=600 /></p>

**简单来说，就是需要自己计算获得均值和显著性水平，然后利用ggplot2的图层叠加画出想要的效果**

先产生测试数据：

```
Group1 <- data.frame(A=runif(100,15,90),
	B=rnorm(100,30,3),
	C=runif(100,30,60),
	D=rexp(100,0.1),
	E=rpois(100,10),
	group=1)
Group2 <- data.frame(A=runif(100,0,100),
	B=rnorm(100,50,2),
	C=runif(100,40,70),
	D=rexp(100,0.3),
	E=rpois(100,20),
	group=2)
b <- rbind(Group1,Group2)

	数据长这样：
	         A        B        C          D group
	1 54.22714 29.68265 58.79358  4.0479914     1
	2 65.97848 32.77361 52.92247  0.5999792     1
	3 44.70824 35.41883 56.19580  1.7838666     1
	4 78.40606 34.67005 52.77656 12.8923074     1
	5 51.26713 28.62068 43.12467  2.7550948     1
	6 16.67529 30.38058 38.92073  0.1057046     1
```

将数据框宽格式变长格式方便制图：

```
library(data.table)
b <- melt(b,id.vars = c("group"))
b$group<-as.factor(b$group)

	转换后长这样：
	  group variable    value
	1     1        A 54.22714
	2     1        A 65.97848
	3     1        A 44.70824
	4     1        A 78.40606
	5     1        A 51.26713
	6     1        A 16.67529
```

由于要做每个箱线图的均值点及均值连线，需要获得每个组每个属性的均值，并且定义每个组每个属性的X坐标为固定值

```
# 每个组每个属性的均值
c1 <- tapply(b[b$group==1,"value"],b[b$group==1,"variable"],mean)
c2 <- tapply(b[b$group==2,"value"],b[b$group==2,"variable"],mean)
c3 <- rbind(data.frame(variable=names(c1),value=c1,group=1),
	data.frame(variable=names(c2),value=c2,group=2))
c3$group<-as.factor(c3$group)


	   variable     value group
	A         A 52.039882     1
	B         B 30.269285     1
	C         C 45.875787     1
	D         D  9.376955     1
	A1        A 49.942698     2
	B1        B 50.089608     2
	C1        C 54.646858     2
	D1        D  3.240988     2


# 定义每个组每个属性的X坐标为固定值
c3$variable2 <- NA
c3[c3$group==1&c3$variable=="A","variable2"] <- 0.795
c3[c3$group==1&c3$variable=="B","variable2"] <- 1.795
c3[c3$group==1&c3$variable=="C","variable2"] <- 2.795
c3[c3$group==1&c3$variable=="D","variable2"] <- 3.795
c3[c3$group==1&c3$variable=="E","variable2"] <- 4.795

c3[c3$group==2&c3$variable=="A","variable2"] <- 1.185
c3[c3$group==2&c3$variable=="B","variable2"] <- 2.185
c3[c3$group==2&c3$variable=="C","variable2"] <- 3.185
c3[c3$group==2&c3$variable=="D","variable2"] <- 4.185
c3[c3$group==2&c3$variable=="E","variable2"] <- 5.185

	   variable     value group variable2
	A         A 52.039882     1     0.795
	B         B 30.269285     1     1.795
	C         C 45.875787     1     2.795
	D         D  9.376955     1     3.795
	A1        A 49.942698     2     1.185
	B1        B 50.089608     2     2.185
	C1        C 54.646858     2     3.185
	D1        D  3.240988     2     4.185
```

做两组每个属性的箱线图：

```
p1 <- ggplot(b)+
	# 绘制箱型图
  geom_boxplot(aes(x=variable,y=value,fill=group),
	width=0.6,
	position = position_dodge(0.8),
	outlier.size = 0,
	outlier.color = "white")+
	# 设置箱型图的填充类型及图例
  scale_fill_manual(values = c("red", "blue"),
	breaks=c("1","2"),
	labels=c("Group 1","Group 2"))+
	# 画出每个组每个属性的均值的散点
  geom_point(data=c3,
	aes(x=variable2,
	y=value,
	color=group),
	shape=15,
	size=1)+
	# 画出每个组每个属性的均值的散点的连线
  geom_line(data=c3,
	aes(x=variable2,y=value,color=group),
	size=1,
	linetype = "dotted")+
  labs(x="",y="")+
  scale_y_continuous(limits = c(0,110),breaks=seq(0,110,5)) +
	# 绘制显著性标识
  geom_signif(stat="identity",
			data=data.frame(x=c(0.795,1.795,2.795,3.795), 
					xend=c(1.185, 2.185,3.185,4.185),
					y=c(106,66,70,30),
					annotation=c("***", " *** ","  ***  ","    **    ")),
			aes(x=x,xend=xend, y=y, yend=y, annotation=annotation)) +
	# 主题风格设置
  theme_bw()+
  theme(
    legend.position = "top",
    legend.background=element_blank(),
    legend.key = element_blank(),
    legend.margin=margin(0,0,0,0,"mm"),
    axis.text.x=element_text(size=rel(1.1),face="bold"),
    axis.line.x = element_line(size = 0.5, colour = "black"),
    axis.line.y = element_line(size = 0.5, colour = "black"),
    legend.text=element_text(size=rel(1.1)),
    legend.title=element_blank(),
    panel.border = element_blank(),
    panel.grid = element_blank()
  )+
	# 移除颜色图例，保留填充图例
  guides(color=FALSE)
```

<p align="center"><img src=./picture/ggplot2-boxplot-plus-siginfo-2.png width=600 /></p>

然后做两组整体水平的箱线图：

```
p2<-ggplot(b)+
	# 画箱型图
  geom_boxplot(aes(x=group,y=value,fill=group),
	width=0.8,
	position=position_dodge(1))+
  stat_summary(fun.y = mean, geom = "point", aes(x=group,y=value,color=group),shape=15)+
  scale_fill_manual(values = c("red", "blue"),
	breaks=c("1","2"),
	labels=c("Group 1","Group 2"))+
  scale_x_discrete(breaks=c("1","2"),
	labels=c("Group 1","Group 2"))+
  scale_y_continuous(limits = c(0,110),breaks=seq(0,110,5)) +
	# 绘制显著性标识
  geom_signif(stat="identity",
              data=data.frame(x=c(1), xend=c(2),
                              y=c(106), annotation=c("**")),
              aes(x=x,xend=xend, y=y, yend=y, annotation=annotation))+
  theme_bw()+
  theme(
    legend.position = "top",
    legend.background=element_blank(),
    legend.key = element_blank(),
    legend.margin=margin(0,0,0,0,"mm"),
    axis.text.x=element_text(size=rel(1.1),face="bold"),
    axis.line.x = element_line(size = 0.5, colour = "black"),
    axis.line.y = element_blank(),
    axis.ticks.y = element_blank(),
    axis.text.y = element_blank(),
    axis.title = element_blank(),
    legend.text=element_text(size=rel(1.1)),
    legend.title=element_blank(),
    plot.margin = margin(11.5,0,7,0,"mm"),
    panel.border = element_blank(),
    panel.grid = element_blank()
  )+
  guides(fill=F,color=F)
```

<p align="center"><img src=./picture/ggplot2-boxplot-plus-siginfo-3.png width=200 /></p>


合并两个图像：

```
# 如果没有安装则install.packages("gridExtra")
library(gridExtra)
# 合并两个图：
grid.arrange(p1, p2, nrow=1, ncol=2,widths=c(3.5,1),heights=c(4))
```

<p align="center"><img src=./picture/ggplot2-boxplot-plus-siginfo-4.png width=600 /></p>

---

参考资料：

(1) [Y叔博客：Use ggplot2](https://guangchuangyu.github.io/cn/2014/05/use-ggplot2/)

(2) [生信杂谈：ggplot2作分组箱线图并添加均值点连线及显著性程度标注](https://mp.weixin.qq.com/s/inzA7B3vfVSMm66gQi7RTA)
