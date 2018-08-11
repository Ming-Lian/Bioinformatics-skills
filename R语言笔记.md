<a name="content">目录</a>

[R语言笔记](#title)
- [用pheatmap画热图](#use-pheatmap)




<h1 name="title">R语言笔记</h1>

<a name="use-pheatmap"><h2>用pheatmap画热图 [<sup>目录</sup>](#content)</h2></a>

需要输入的数据为matrix，所以若原始数据为第一列是行名的data.frame，则需要将其转换为matrix

```
> data_matrix<-as.matrix(data[,-1])
> rownames(data_matrix)<-data$col1
```

用默认参数绘图：

```
pheatmap(data)#默认参数
```

<p align="center"><img src=./picture/Rnote-heatmap-pheatmap-1.png width=600 /></p>

pheatmap默认对column和row都进行聚类，基于欧式距离进行聚类

1、 若不想进行聚类，则可以通过`cluster_rows`或`cluster_cols`参数设置对应维度上的聚类是开启还是关闭

```
pheatmap(data,...,cluster_rows=F,cluster_cols=F)
```

2、 均一化后再绘制热图，需要设置scale参数，"row","column" or "none"默认是"none"

```
pheatmap(data,scale="column",...）
pheatmap(data,scale="row",...）
```

3、隐藏行名或列名，设置show_colnames或show_rownames参数

```
pheatmap(data,show_colnames=F,...）
```

4、给列（一般一列可以看作是一个样本）添加分组信息，需要创建一个用于保存分组信息的数据框

```
annotation_col<-data.frame(...)
rownames(annotation_col)<-colnames(data)	# 需要将数据框的行名设成输入数据的列名
pheatmap(data,...,annotation_col=annotation_col)
```

5、添加标题，设置main参数

```
pheatmap(data,...,main="title")
```
