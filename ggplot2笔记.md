<a name="content">目录</a>

[ggplot2笔记](#title)
- [直方图：geom_histogram](#histogram)
- [柱状图：geom_bar](#bar)
- [控制整体图形属性](#whole-plot-style)
	- [标尺（Scale）]($scale)
	- [标题](#plot-title)
	- [图例](#legend)
    - [简约主题设置](#theme-setting)
- [统计变换（Statistics）](#statistics)
- [坐标系统（Coordinante）](#coordinante)
- [分面（Facet）](#facet)
- [画辅助线](#auxiliary-line)
- [使用 RColorBrewer 扩展调色板](#use-RColorBrewer)
- [坐标轴截断](#axis-break-based)
  - [ggplot + grid](#axis-break-based-on-ggplot-and-grid)
  - [ggplot + ggpubr](#axis-break-based-on-ggplot-and-ggpubr)
- [改变横坐标顺序](#reorder-x-axis)
- [特殊图形的绘制](#plot-special)
  - [作分组箱线图并添加均值点连线及显著性程度标注](#boxplot-plus-siginfo)
    - [从头实现（不推荐）](#boxplot-plus-siginfo-mannual)
    - [使用ggsiginf包](#boxplot-plus-siginfo-use-ggsiginf)
  - [使用ggbiplot画PCA图](#use-ggbiplot-plot-PCA)
  - [使用gglorenz画洛伦兹曲线](#plot-lorenz-curve)

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

图例的安放:用 `theme` 和 `guides` 来调整legend的位置和布局：

原图

<p align="center"><img src=./picture/ggplot2-use-RColorBrewer-4.png width=700 /></p>

将图例放在图下方，且为了防止它按照默认的组织方式——按两列展开，这样会影响整体布局的美观，可以按照指定行数去展开：

```
... + theme(legend.position="bottom") +
  guides(fill=guide_legend(nrow=2))
```

<p align="center"><img src=./picture/ggplot2-use-RColorBrewer-5.png width=700 /></p>

<a name="theme-setting"><h3>简约主题设置 [<sup>目录</sup>](#content)</h3></a>

```
panel.background = element_blank()
panel.border = element_blank()
axis.line= element_line(colour = 'black')
```







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

<a name="auxiliary-line"><h2>画辅助线 [<sup>目录</sup>](#content)</h2></a>

若想添加与x轴平行的辅助线，则可以使用`geom_hline`函数：

```R
geom_hline(aes(yintercept=0.1), lintype='dashed')
```





<a name="use-RColorBrewer"><h2>使用 RColorBrewer 扩展调色板 [<sup>目录</sup>](#content)</h2></a>

通过运行 `display.brewer.all()` ，可以查看RColorBrewer中提供的标准配色方案：

<p align="center"><img src=./picture/ggplot2-use-RColorBrewer-1.png width=700 /></p>

则在用ggplot2绘图时只需要在其中加上这样一句：

```R
... + scale_fill_brewer(palette="Set1") + ...
```

通过设置`palette`参数来选择合适的配色方案

有时会遇到这样的情况：需要的颜色种类和提供的配色方案中所具有的颜色种类不匹配，一般是需要的颜色种类对于所能提供的，此时就可能报错：

```R
ggplot(mtcars) + 
  geom_bar(aes(factor(hp), fill=factor(hp))) +
  scale_fill_brewer(palette="Set2")

Warning message:
In RColorBrewer::brewer.pal(n, pal) : n too large, allowed maximum for palette Set2 is 8
Returning the palette you asked for with that many colors
```

<p align="center"><img src=./picture/ggplot2-use-RColorBrewer-2.png width=700 /></p>

此时，RColorBrewer为我们提供了一种通过使用构造函数`colorRampPalette`插入现有调色板来生成更大调色板的方法

```R
colourCount = length(unique(mtcars$hp))
getPalette = colorRampPalette(brewer.pal(9, "Set1"))
 
ggplot(mtcars) + 
  geom_bar(aes(factor(hp)), fill=getPalette(colourCount)) + 
  theme(legend.position="right")
```

<p align="center"><img src=./picture/ggplot2-use-RColorBrewer-3.png width=700 /></p>

虽然我们解决了颜色不足的问题，但其他有趣的事情发生了：虽然所有的柱子都回来了并且涂上了不同颜色，但是我们也失去了颜色图例。我故意添加主题（`legend.position = ...`）来展示这个事实：尽管有明确的位置请求，但图例不再是图形的一部分

原因在于：fill参数移动到柱状图aes函数之外 - 这有效地从ggplot的美学数据集中删除了fill信息。因此，legend没有应用到任何东西上

要修复它，请将`fill`放回到`aes`中并使用`scale_fill_manual()`定义自定义调色板：

```R
ggplot(mtcars) + 
  geom_bar(aes(factor(hp), fill=factor(hp))) + 
  scale_fill_manual(values = getPalette(colourCount))
```
<p align="center"><img src=./picture/ggplot2-use-RColorBrewer-4.png width=700 /></p>

<a name="axis-break"><h2>坐标轴截断 [<sup>目录</sup>](#content)</h2></a>

<a name="axis-break-based-on-ggplot-and-grid"><h3>ggplot + grid [<sup>目录</sup>](#content)</h3></a>

本质上就是通过在使用ggplot绘图过程中，使用`coord_cartesian`来截取指定范围内的图形部分，多次截取则得到多个子图，然后使用grid打开一个画布，将之前截取的多个子图在一个画布上拼凑出来

```R
#使用 coord_cartesian() 分割作图结果
split1 <- bar_plot + coord_cartesian(ylim = c(0, 4)) + 
    theme(legend.position='none')
split2 <- bar_plot + coord_cartesian(ylim = c(4, 20)) + 
    theme(axis.text.x = element_blank(), axis.ticks.x = element_blank(), legend.position = 'none')
split3 <- bar_plot + coord_cartesian(ylim = c(60, 90)) + 
    theme(axis.text.x = element_blank(), axis.ticks.x = element_blank(), legend.position = c(0.95, 0.7))
```

<p align="center"><img src=./picture/ggplot2-axis-break-using-ggplot-grid.png /></p>

```R
library(grid)
grid.newpage()
plot_site1 <- viewport(x = 0.008, y = 0, width = 0.994, height = 0.4, just = c('left', 'bottom'))
plot_site2 <- viewport(x = 0, y = 0.4, width = 1, height = 0.3, just = c('left', 'bottom'))
plot_site3 <- viewport(x = 0, y = 0.7, width = 1, height = 0.3, just = c('left', 'bottom'))
print(split1, vp = plot_site1)
print(split2, vp = plot_site2)
print(split3, vp = plot_site3)
```

<p align="center"><img src=./picture/ggplot2-axis-break-using-ggplot-grid-2.png /></p>

<a name="axis-break-based-on-ggplot-and-ggpubr"><h3>ggplot + ggpubr [<sup>目录</sup>](#content)</h3></a>

如何实现从A图到B图的转化呢？

<p align="center"><img src=./picture/ggplot2-axis-break-using-ggplot-ggpubr-1.jpg width=600/></p>

只需要三步，画下面一半，画上面一半，拼起来即可

（1）画下面一半

```
#导入包
library(ggplot2)
library(ggpubr)
#数据
data <- data.frame(x = c("Alpha","Bravo","Charlie","Delta"),y=c(200,20,10,15))
#画下面
p1 <- ggplot(data,aes(x=x,y=y,fill=x)) + geom_bar(stat='identity',position=position_dodge()) +
  labs(x=NULL,y=NULL,fill=NULL)+    #可自定义标签名字
  coord_cartesian(ylim = c(0,25))   #设置下面一半的值域
```

<p align="center"><img src=./picture/ggplot2-axis-break-using-ggplot-ggpubr-2.png width=300/></p>

（2）画上面一半

```
p2 <- ggplot(data,aes(x=x,y=y,fill=x)) + geom_bar(stat='identity',position=position_dodge()) +
  labs(x=NULL,y=NULL,fill=NULL) +   #不要标签
  theme(axis.text.x = element_blank(),axis.ticks.x = element_blank()) +     #去掉X轴和X轴的文字
  coord_cartesian(ylim = c(195,205)) +  #设置上面一半的值域
  scale_y_continuous(breaks = c(195,205,5)) #以5为单位划分Y轴
```

<p align="center"><img src=./picture/ggplot2-axis-break-using-ggplot-ggpubr-3.png width=300/></p>

（3）拼起来

```
ggarrange(p2,p1,heights=c(1/5, 4/5),ncol = 1, nrow = 2,common.legend = TRUE,legend="right",align = "v") 
```

<p align="center"><img src=./picture/ggplot2-axis-break-using-ggplot-ggpubr-4.png width=300/></p>

> 排序为 p2 p1，即上面的图放上面，下面的图放下面
>
> `heights`: 两个图高度所占的比例，根据实际情况进行修改
>
> `align`: 这个参数很重要，对齐参数将上下两个图对齐，`h` 为水平对齐，`v` 为垂直对齐

<a name="reorder-x-axis"><h2>改变X轴标签顺序 [<sup>目录</sup>](#content)</h2></a>

<p align="center"><img src=./picture/R-advanced-reorder-x-axis-1.png/></p>

若想将上图的X轴标签的排列顺序改成下面的形式：

<p align="center"><img src=./picture/R-advanced-reorder-x-axis-2.png/></p>

只需将其所对应的factor类型的列的level进行设置即可，如下：

```
tt.data$group <- factor(tt.data$group, levels=c("B", "A", "C","D"), ordered=TRUE)
```

<a name="plot-specail"><h2>特殊图形的绘制 [<sup>目录</sup>](#content)</h2></a>

<a name="boxplot-plus-siginfo"><h3>作分组箱线图并添加均值点连线及显著性程度标注 [<sup>目录</sup>](#content)</h3></a>

<a name="boxplot-plus-siginfo-mannual"><h4>从头实现（不推荐） [<sup>目录</sup>](#content)</h4></a>

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

<a name="boxplot-plus-siginfo-use-ggsiginf"><h4>使用ggsiginf包 [<sup>目录</sup>](#content)</h4></a>

官方文档：https://cran.r-project.org/web/packages/ggsignif/vignettes/intro.html

简单使用——就两步：

> (1) 导入需要的包
>
> ```R
> library(ggplot2)
> library(ggsignif)
> ```
>
> (2) 画图
>
> ```R
> ggplot(iris, aes(x=Species, y=Sepal.Length)) + 
>  geom_boxplot() +
>  geom_signif(comparisons = list(c("versicolor", "virginica")), map_signif_level=TRUE)
> ```
>
> <p align="center"><img src=./picture/ggplot2-boxplot-plus-siginfo-5.png width=400 /></p>

高级用法：

> - （对连个比较对象）全部自行设置显著性标注
>
>   ```R
>   geom_signif(y_position=c(5.3, 8.3), xmin=c(0.8, 1.8), xmax=c(1.2, 2.2), annotation=c("**", "NS"), tip_length=0)
>   ```
>
> - 设定多个两两比较
>
>   ```R
>   comparisons=lists(c('c1', 'c2'), c('c1', 'c3'), ...)
>   ```

test参数提供两种统计检验方法：t-test与wilcox

<a name="use-ggbiplot-plot-PCA"><h3>使用ggbiplot画PCA图 [<sup>目录</sup>](#content)</h3></a>

注意：目前该R包已停止更新维护了，它对ggplot的版本有要求，目前已知可使用的版本为`ggplot2=3.0.0`，若想安装特点版本的ggplot2：

```
install_version('ggplot2',version='3.0.0')
```

安装

```R
library(devtools)
install_github("vqv/ggbiplot")
```

基本使用：

```R
library(ggbiplot)
data(wine)
wine.pca <- prcomp(wine, scale. = TRUE)
ggbiplot(wine.pca, obs.scale = 1, var.scale = 1,
  groups = wine.class, ellipse = TRUE, circle = TRUE) +
  scale_color_discrete(name = '') +
  theme(legend.direction = 'horizontal', legend.position = 'top')
```

<p align="center"><img src=./picture/ggplot2-use-ggbiplot-plot-PCA.png width=700 /></p>

<a name="plot-lorenz-curve"><h3>使用gglorenz画洛伦兹曲线 [<sup>目录</sup>](#content)</h3></a>

```
library(ggplot2)
library(gglorenz)

Distr1 <- c( A=137, B=499, C=311, D=173, E=219, F=81)
x <- data.frame(Distr1)

ggplot(x, aes(Distr1)) + 
  stat_lorenz() + 
  geom_abline(color = "grey")
```


---

参考资料：

(1) [Y叔博客：Use ggplot2](https://guangchuangyu.github.io/cn/2014/05/use-ggplot2/)

(2) [生信杂谈：ggplot2作分组箱线图并添加均值点连线及显著性程度标注](https://mp.weixin.qq.com/s/inzA7B3vfVSMm66gQi7RTA)

(3) [使用 ggplot2 和 RColorBrewer 扩展调色板](https://www.cnblogs.com/shaocf/p/9600340.html)

(4) [ggbiplot README](https://github.com/vqv/ggbiplot)

(5) [ggplot2中绘制截断坐标轴的方法](https://blog.csdn.net/enyayang/article/details/98181357)

(6) [R语言—ggplot2画图如何截断 y 轴](https://www.jianshu.com/p/79331cb85a03)

(7) [ggplot2调整X轴标签顺序](https://www.jianshu.com/p/437906c6b063)

(8) [How to plot a nice Lorenz Curve for factors in R (ggplot ?)
](https://stackoverflow.com/questions/22679493/how-to-plot-a-nice-lorenz-curve-for-factors-in-r-ggplot)
