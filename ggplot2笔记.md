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









参考资料：

(1) [Y叔博客：Use ggplot2](https://guangchuangyu.github.io/cn/2014/05/use-ggplot2/)
