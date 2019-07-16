<a name="content">目录</a>

[R语言笔记](#title)
- [1. 用pheatmap画热图](#use-pheatmap)
- [2. R中的并行化方法](#r-parallel)
- [3. R语言向量化技术](#r-vectorization-technology)
- [4. 标量化和向量化的逻辑运算](#scalarization-and-vectorization-for-logic-operation)
- [5. Divide-Conquer：矩阵的分块计算与子块计算结果合并](#divide-conquer-in-calc-large-matrix)
- [6. match函数：到底谁match谁，傻傻分不清](#how-to-use-match-function)
- [7. 逐行读入文件](#read-one-line-each-time)
- [8. 关于超大矩阵运算的思考](#think-of-vast-matrix)
- [9. optparse的使用](#usage-of-optparse)




<h1 name="title">R语言笔记</h1>

<a name="use-pheatmap"><h2>1. 用pheatmap画热图 [<sup>目录</sup>](#content)</h2></a>

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

<a name="r-parallel"><h2>2. R中的并行化方法 [<sup>目录</sup>](#content)</h2></a>

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

<a name="r-vectorization-technology"><h2>3. R语言向量化技术 [<sup>目录</sup>](#content)</h2></a>

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



<a name="scalarization-and-vectorization-for-logic-operation"><h2>4. 标量化和向量化的逻辑运算 [<sup>目录</sup>](#content)</h2></a>

R语言的逻辑运算有两种：“与”运算和“或”运算

- “与”运算：`&&`、`&`
- “或”运算：`||`、`|`

其中，运算符“逻辑与”和“逻辑或”存在两种形式，`&`和`|`作用在对象中的每一个元素上并且返回和比较次数相等长度的逻辑值；`&&`和`||`只作用在对象的第一个元素上

<a name="divide-conquer-in-calc-large-matrix"><h2>5. Divide-Conquer：矩阵的分块计算与子块计算结果合并 [<sup>目录</sup>](#content)</h2></a>

在对一个大矩阵执行相关性计算或Jaccard Index的计算时，其实执行的是矩阵任意两行（这里假设要进行分析的对象是矩阵的每个行）之间的两两的计算，若这个矩阵的规模非常庞大，有n行时，计算的时间复杂度就是$O(n^2)$，这个时候可以采用并行化策略来加速这个进程（参考上文的 [2. R中的并行化方法](#r-parallel)）：

```
StatOut <- parApply(cl, data, 1, fun, data)
```

这样就会实现将一个 `n vs. n` 的问题拆分成 n 个可以并行解决的 `1 vs. n` 的问题，当然通过设置线程数为$m,\,(m\le n)$，使得每次只并行执行m个 `1 vs. n` 的问题

然后再在函数`fun`内部再定义一个并行化计算来进一步并行化加速上面产生的 `1 vs. n` 的计算：

```
fun <- function(vec, data){
	...
	parApply(cl, data, 1, fun2, vec)
}
```

在这个函数内部实现了将一个 `1 vs. n` 的问题拆分成 n 个可以并行解决的 `1 vs. 1` 的问题

这样就实现了两步并行化，这样在保证硬件条件满足的情况下，的确能显著加快分析速度

但是并行化技术会带来一个问题是，虽然时间开销减少了，但是**空间开销显著增加了**

> 比如，第一次并行化，将一个 `n vs. n` 的问题拆分成 $\frac{n}{m}$ 次可以并行解决的 m个 `1 vs. n` 的问题，则需要在每一次并行化任务中拷贝一个 `1 vs. n` 的计算对象，即原始有n行的矩阵被拷贝了m次，则相应的缓存空间也增加了m倍，很显然内存的占用大大增加了

空间开销显著增加带来的后果就是，很容易导致运行内存不足程序运行中断的问题，那该怎么解决这个问题呢？

可以采用分治方法（Divide-Conquer)，将原始大矩阵，按照行拆分成多个小的子块，对每个子块执行计算，从而得到每个子块的运算结果，最后再讲每个子块的结果进行合并：

```
n_row <- nrow(data)
nblock <- 10 # 用于设定子块的数，用户可以自行指定
# 该列表变量用于保存每个子块的计算结果，若总共拆成了m个子块，
# 因为要进行任意两子块的计算，因此会有mxm个子块计算结果，因
# 此该列表要保存的数据为mxm维度的数据，每个元素为一个计算结
# 果，它们都是矩阵
StatOutList <- vector("list", nblock) 

# 1. 开始进行分块计算
print("[Divide-Conquer]Start carry out Divide-Conquer for Large Observed Matrix")
print("##################################################")
for(i in 1:nblock){
	for(j in 1:nblock){
		nrow_start <- (i-1)*nrow_block+1
		nrow_end <- i*nrow_block
		# 并行化计算
		if(!is.na(Args[2])){
			print(paste("[Divide-Conquer]Start carry out statistic Jaccard Index parallel for block: ",i,"-",j,sep=''))
			StatOutList[[i]] <- append(StatOutList[[i]], parApply(cl, data[nrow_start:nrow_end,], 1 , JaccardIndexSer, data))
			print(paste("[Divide-Conquer]Finish run parallel for block: ",i,"-",j,sep=''))
		# 串行计算
		}else{
			print(paste("[Divide-Conquer]Start carry out statistic Jaccard Index serially for block: ",i,"-",j,sep=''))
			StatOutList[[i]] <- append(StatOutList[[i]], apply(data, 1 , JaccardIndexSer, data))
			print(paste("[Divide-Conquer]Finish run serially for block: ",i,"-",j,sep=''))
		}
	}
}

# 2. 结束分治方法的分块计算
if(!is.na(Args[2])){
	print("##################################################")
	print("[Divide-Conquer]Finish parallel running for statistic Jaccard Index!")
	stopCluster(cl)
}else{
	print("##################################################")
	print("[Divide-Conquer]Finish serial running for statistic Jaccard Index!")
}

# 3. 开始进行子块结果的合并
print("[Divide-Conquer]Start bind sub-block statout")
StatOut <- vector("list", nblock)
# 先对列进行合并
for(i in 1:nblock){
	for(block in StatOutList[[i]]){
		StatOut[[i]] <- cbind(StatOut[[i]], block)
	}
}
# 再对行进行合并
StatOutMerge <- data.frame()
for(block in StatOut){
	StatOutMerge <- rbind(StatOutMerge, block)
}
```

<a name="how-to-use-match-function"><h2>6. match函数：到底谁match谁，傻傻分不清 [<sup>目录</sup>](#content)</h2></a>

match函数的一般应用场景：



<a name="read-one-line-each-time"><h2>7. 逐行读入文件 [<sup>目录</sup>](#content)</h2></a>

类似于其他编程语言中的操作，先要创建一个文件句柄：

```R
f <- file(filename,'r')
```

然后逐行读入：

```R
row_current <- readLines(f, n=1) # 用参数n控制一次读入的行数，这里设为1
```

读入的文本信息一般来说是以制表符形式进行分割的格式，此时可以将读入的行基于分隔符进行打散

```R
Row_current <- strsplit(row_current, '\t')
```

<a name="think-of-vast-matrix"><h2>8. 关于超大矩阵运算的思考 [<sup>目录</sup>](#content)</h2></a>

此处不讨论分布式计算的问题，因为本狗（科研狗）也不懂~

首先，对我所谓的“超大矩阵”作一个简单的定义：

它的“大”首先体现在**矩阵文件的大小**上，一般单个文件就要大于4G，那么如果我们以R中常用的`read.table`这个系列的函数去读取这个文件的话，由于`read.table`是一次性将文件内容全部读入内存当中，所以：

- 大文件不仅意味着**大内存消耗**（实际的内存消耗要远大于原始文本文件的文件大小，例如原始文本文件可能只有4G，读到内存当中可能就变成了100G）；

- 也意味着**较长的读写等待时间**，一个文件光将它读入等待的时间就得按照小时计算，那后面的活还干不干了！

- 读入后的每一步矩阵操作都是慢动作。即使克服了内存消耗的问题（一般的服务缓存空间起码都有500G+，所以还是能够应付咱们的内存消耗问题的），也耐住了寂寞等到了把整个文件读入内存的时刻，但是问题才刚刚开始，要知道对这么一个庞然大物，每一步操作都是慢动作，所以你面对的是等待，等待，还是等待！

总而言之，处理大矩阵，巨大的存储成本和时间成本都是极大的，而其中最重要也最不能让人容忍的就是时间成本，我的时间非常宝贵，我的青春等不起！

那么，有什么办法可以解决哪怕是缓解这个问题呢？

其实，无非就是从两个角度来思考这个问题：

> - 缓解内存消耗
>
> 	我不一次性地读入，我**将原始文件按照行或者按照列进行拆分**
>	
> 	到底是按行还是按列拆分，具体得依据处理的数据，若矩阵的分析运算是以列为单位进行的，而且列于列之间相对独立，那么就按照列来拆，或者如果是以行为单位进行的，那么就按照行来拆，两种条件都不满足，那就没招了
>
>	拆分玩之后，就得到了若干个小矩阵，将这些小矩阵分批次地读入并分析然后写出，最后将这些结果进行汇总合并
>
> 	拆分（行拆分）的极端情况就是**将文件逐行读入**，大家都知道在perl和python中有逐行读入的操作，其实R也有这种操作，而且操作原理和方法和它们都差不多，都是创建文件句柄对象，然后利用`readLines`函数实现逐行读取
>
> 	将文件逐行读入是行拆分的极端情况，那么有没有方法实现逐列读入呢？很遗憾，没有，考虑到计算机进行文本读取的实际过程就知道，这样的操作其实是不存在的，那些咱们平时用到的像`cut`这样的命令看起来好像的确是能进行行的提取操作，但是其底层的实现还是需要将原始文件从头到尾扫描一遍，然后将对应的列提出来——但也不是完全没有办法，咱们将原始文件转置一下，不就可以通过对转置后的文件进行逐行读取，从而实现对原始文件的逐列读取了吗？不过转置操作也是相当耗时间的
>
> - 降低时间成本
>
>	降低时间成本最有效的办法就是采用**并行化**的策略，利用多线程进行并行化处理，则时间消耗就会成倍地降低
>
> 	但是并行化操作是有前提条件的：
>
>	> 并行化操作的本质是将一个大问题拆解成若干个互不干扰的小问题，则大问题的解就是小问题的解的汇总，例如，对一个矩阵求它的各个列的和，很明显每一列的和只与本列相关，完全不受其他列的影响，所以完全可以每一列独立进行，最后进行汇总
> 	>
> 	> 其实像这种场景是比较多的

<a name="usage-of-optparse"><h2>9. optparse的使用 [<sup>目录</sup>](#content)</h2></a>

`optparse`可以用来实现类似在Python中的`argparse`一样的作用，使得我们写的R脚本可以在命令行中通过添加参数的方式来运行，同时有帮助文档可以来帮助使用者快速上手

```R
library（optparse)

option_list <- list(
	make_option(c("-v", "--verbose"), action="store_true", default=TRUE, help="Print extra output [default]"),
	make_option(c("-q", "--quietly"), action="store_false", dest="verbose", help="Print little output"),
	make_option(c("-c", "--count"), type="integer", default=5, help="Number of random normals to generate [default %default]",        metavar="number"),
	make_option("--generator", default="rnorm", help = "Function to generate random deviates [default \"%default\"]"),
	make_option("--mean", default=0, help="Mean if generator == \"rnorm\" [default %default]"),
	make_option("--sd", default=1, metavar="standard deviation", help="Standard deviation if generator == \"rnorm\" [default %default]")
)

opt <- parse_args(OptionParser(option_list=option_list))

# print some progress messages to stderr if "quietly" wasn't requested
if ( opt$verbose ) {
	write("writing some verbose output to standard error...\n", stderr())
}
# do some operations based on user input
if( opt$generator == "rnorm") {
	cat(paste(rnorm(opt$count, mean=opt$mean, sd=opt$sd), collapse="\n"))
}
else {
	cat(paste(do.call(opt$generator, list(opt$count)), collapse="\n"))
}
cat("\n")
```


---

参考资料：

(1) [在R中如何逐行读取CSV文件并将内容识别为正确的数据类型？](https://oomake.com/question/1544925)

(2) [CSDN·卡西莫多的礼物《》](https://blog.csdn.net/qq_35696312/article/details/88188379)
