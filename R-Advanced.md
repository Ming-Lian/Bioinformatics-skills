<a name="content">目录</a>

[R语言进阶笔记](#title)
- [用pheatmap画热图](#use-pheatmap)
- [R中的并行化方法](#r-parallel)
- [R语言向量化技术](#r-vectorization-technology)
- [标量化和向量化的逻辑运算](#scalarization-and-vectorization-for-logic-operation)



<h1 name="title">R语言进阶笔记</h1>

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
annotation_col<-data.frame(...) # 需要是factor类型
rownames(annotation_col)<-colnames(data)	# 需要将数据框的行名设成输入数据的列名
pheatmap(data,...,annotation_col=annotation_col)
```

5、添加标题，设置main参数

```
pheatmap(data,...,main="title")
```

<a name="r-parallel"><h2>R中的并行化方法 [<sup>目录</sup>](#content)</h2></a>

R的并行化是在 `*apply` 系列函数的基础上产生的，所以在介绍R的并行化之前，有必要对 `*apply` 系列函数作一个简单的说明，下面只对`apply( )`进行说明

> 函数语法格式：`apply(x, margin, fun, ...)`
> 
> - `x`：一个data.frame或者是一个matrix
> 
> - `margin`：选择1或者2，1表示行，2表示列
> 
> - `fun`：一个函数对象，可以是R自带的，也可以是用户自定义的函数
> 
> - `...`：传递给函数fun的其他变量
> 
> 例如：`apply(x,1,sum)`，将变量x逐行传递给函数sum，进行求和，得到的是变量x每行的和的列表

一个任务之所以能够进行并行化处理，是因为该任务可以被拆分成许多个相互独立的小任务，每个小任务的求解对其他任务没有任何影响，因此可以对它们进行分而治之，最后将每个小任务求解结果进行汇总，即简单地合并，apply函数实现的就是这样的任务，将一个比较大的变量X按照行（margin=1）或列（margin=2）进行拆分，然后对每个行分别独立地进行求解

但是`*apply`系列函数的分治策略并没有进行并行化，是一个只利用了一个线程的串行任务，但是由于它本身的分治属性，对它进行简单地改造和封装，就可以实现高效地并行化，这便是`parallel`包

```
library(parallel)

# 初始化一个并行化的集群
## 设置集群使用的核心数
if (is.na(Args[4])){
	no_cores <- detectCores() - 10
} else {
	no_cores <- as.integer(Args[4])
}
## 按照指定的核心数，初始化一个集群对象
cl <- makeCluster(no_cores)

# 调用parApply函数，执行并行化任务
out <- parApply(cl,data,1,fun)

# 任务结束后，关闭集群对象
stopCluster(cl)
```

<a name="r-vectorization-technology"><h2>R语言向量化技术 [<sup>目录</sup>](#content)</h2></a>

什么是向量化计算呢？其实你可以简单的理解成这样：当我们在使用函数或者定义函数的时候发现，我们只能对单个数据进行运算。而这点显然不能满足我们的需求。那么如何使函数可以计算多个数据呢？这时就要采用向量化计算的方法了。

举一个简单的例子来理解向量化技术：

首先，我们先定义一个判断是否为偶数的函数，返回值为布尔值：

```
func <- function(x){
  if(x %% 2 == 0){
    ret <- TRUE
  } else{
    ret <- FALSE}
  return(ret)
}
```

当我们使用这个函数时发现他只能对单个数据进行运算，而多个数据时则会报错：

```
> func(c(123,34,12,43))
[1] FALSE
Warning message:
In if (x%%2 == 0) { :
  the condition has length > 1 and only the first element will be used
>
```

这个时候如果想对一个向量的每一个元素都执行该函数，如何实现？

- 第一种方法就是，使用循环操作
	
	```
	for(i in c(123,34,12,43)){
		func(i)
	}
	```

	当然这样实现完全没有问题，但是一点也不优雅

- 第二种方法，使用R语言中带有的向量化技术去实现



<a name="scalarization-and-vectorization-for-logic-operation"><h2>标量化和向量化的逻辑运算 [<sup>目录</sup>](#content)</h2></a>

R语言的逻辑运算有两种：“与”运算和“或”运算

- “与”运算：`&&`、`&`
- “或”运算：`||`、`|`

其中，运算符“逻辑与”和“逻辑或”存在两种形式，`&`和`|`作用在对象中的每一个元素上并且返回和比较次数相等长度的逻辑值；`&&`和`||`只作用在对象的第一个元素上

