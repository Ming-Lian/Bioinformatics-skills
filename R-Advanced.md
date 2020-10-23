<a name="content">目录</a>

[R语言笔记](#title)
- [1. 用pheatmap画热图](#use-pheatmap)
- [2. R中的并行化方法](#r-parallel)
    - [2.1. parallel包](#r-parallel-parallel-package)
    - [2.2. foreach包](#r-parallel-foreach-package)
- [3. R语言向量化技术](#r-vectorization-technology)
- [4. 标量化和向量化的逻辑运算](#scalarization-and-vectorization-for-logic-operation)
- [5. Divide-Conquer：矩阵的分块计算与子块计算结果合并](#divide-conquer-in-calc-large-matrix)
- [6. match函数：到底谁match谁，傻傻分不清](#how-to-use-match-function)
- [7. 逐行读入文件](#read-one-line-each-time)
- [8. 关于超大矩阵运算的思考](#think-of-vast-matrix)
- [9. optparse的使用](#usage-of-optparse)
- [10. 数字以科学计数法输出](#output-number-in-scientific-method)
- [11. reshape2包](#use-reshape2)
  - [11.1. melt：将短数据框转化为长数据框](#reshape2-melt)
  - [11.2. dcast：将长数据框转化为短数据框](#reshape2-dcast)
- [12. 计算偏离系数](#calculate-Asymmetry-coefficient)
- [13. R包安装的各种报错以及解决](#deal-with-package-install)
- [14. dplyr包的使用](#use-dplyr)
  - [14.1. dplyr的管道操作%>%](#dplyr-pipe)
  - [14.2. summarise 和 mutate 函数使用](#dplyr-summarise-and-mutate)
  - [14.3. 其他数据筛选与修改操作](#dplyr-other-utility)
- [15. 用do.call整理lapply的结果](#reshape-output-from-lapply)



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

R并行计算的必要性：

> R 的内存使用方式和计算模式限制了 R 处理大规模数据的能力
>
> R 采用的是内存计算模式（In-Memory），被处理的数据需要预取到主存（RAM）中，优点是计算效率高、速度快，但缺点是这样一来能处理的问题规模就非常有限（小于 RAM 的大小）
>
> 另一方面，R 的核心（R core）是一个单线程的程序

现代的多核处理器上，R 无法有效地利用所有的计算内核

并行计算技术正是为了在实际应用中解决单机内存容量和单核计算能力无法满足计算需求的问题而提出

<p align="center"><img src=./picture/Advanced-R-programing-r-parallel-1.png width=600 /></p>

R 中的并行计算模式大致可以分为隐式和显示两种

- 隐式并行计算

    隐式计算对用户隐藏了大部分细节，用户不需要知道具体数据分配方式 ，算法的实现或者底层的硬件资源分配。系统会根据当前的硬件资源来自动启动计算核心

    这种模式对于大多数用户来说是最喜闻乐见的。我们可以在完全不改变原有计算模式以及代码的情况下获得更高的性能

- 显示并行计算

    显式计算则要求用户能够自己处理算例中数据划分，任务分配，计算以及最后的结果收集，因此，显式计算模式对用户的要求更高，用户不仅需要理解自己的算法，还需要对并行计算和硬件有一定的理解

    值得庆幸的是，现有 R 中的并行计算框架，如 parallel (snow,multicores)，Rmpi 和 foreach 等采用的是映射式并行模型（Mapping），使用方法简单清晰，极大地简化了编程复杂度

    <p align="center"><img src=./picture/Advanced-R-programing-r-parallel-2.png width=600 /></p>



<a name="r-parallel-parallel-package"><h3>2.1. parallel包 [<sup>目录</sup>](#content)</h3></a>

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

注意：

> 如果有环境里面的外置变量（自己定义）那么需要额外插入，复制到不同核上面，而且如果有不同包里面的函数，都要额外加载、复制多份给不同的电脑核心

```R
xx <- 1

clusterExport(cl, "xx")
```

<a name="r-parallel-foreach-package"><h3>2.2. foreach包 [<sup>目录</sup>](#content)</h3></a>

初始化的过程有些不同，你需要register注册集群：

```R
library(foreach)
library(doParallel)

cl<-makeCluster(no_cores)
registerDoParallel(cl)
```

记得最后要结束集群：

```R
stopImplicitCluster()
```

oreach函数可以使用参数.combine控制你汇总结果的方法：

```R
> foreach(exponent = 2:4, .combine = c)  %dopar%  base^exponent
[1]  4  8 16

> foreach(exponent = 2:4, .combine = rbind)  %dopar%   base^exponent
         [,1]
result.1    4
result.2    8
result.3   16

> foreach(exponent = 2:4, .combine = list, .multicombine = TRUE)  %dopar%   base^exponent
[[1]]
[1] 4

[[2]]
[1] 8

[[3]]
[1] 16
```

注意到最后list的`combine`方法是默认的。在这个例子中用到一个`.multicombine`参数，他可以帮助你避免嵌套列表。比如说`list(list(result.1, result.2), result.3)`：

```R
> foreach(exponent = 2:4, .combine = list)  %dopar%   base^exponent
[[1]]
[[1]][[1]]
[1] 4

[[1]][[2]]
[1] 8


[[2]]
[1] 16
```

注：


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

```R
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
library(optparse)

option_list <- list(
	make_option(c("-v", "--verbose"), action="store_true", default=TRUE, help="Print extra output [default]"),
	make_option(c("-q", "--quietly"), action="store_false", dest="verbose", help="Print little output"),
	make_option(c("-c", "--count"), type="integer", default=5, help="Number of random normals to generate [default %default]", metavar="number"),
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

<a name="output-number-in-scientific-method"><h2>10. 数字以科学计数法输出 [<sup>目录</sup>](#content)</h2></a>

```R
format(0.00001, scientific=TRUE, digit=2)
```

<a name="use-reshape2"><h2>11. reshape2包 [<sup>目录</sup>](#content)</h2></a>

<a name="reshape2-melt"><h3>11.1. melt：将短数据框转化为长数据框 [<sup>目录</sup>](#content)</h3></a>

reshape2::melt常用于将短表格转换成长表格，以用于ggplot绘图

短表格示例：

| Id | Var1 | Var2 | Var3 |
|:---:|:---:|:---:|:---:|
| Id1 | a1 | b1 | c1 |
| Id2 | a2 | b2 | c2 |
| Id3 | a3 | b3 | c3 |

其对应的长表格为：

| Id | VAR | VALUE |
|:---:|:---:|:---:|
| Id1 | Var1 | a1 |
| Id1 | Var2 | b1 |
| Id1 | Var3 | c1 |
| Id2 | Var1 | a2 |
| Id2 | Var2 | b2 |
| Id2 | Var3 | c2 |
| Id3 | Var1 | a3 |
| Id3 | Var2 | b3 |
| Id3 | Var3 | c3 |

假设原始的短表格保存在数据框变量X中，现要将其转换成以Id列为唯一记录识别项，其他项作为测量项的长表格Y，则可以通过下面的命令实现：

```
Y <- melt(X, id.vars=1)
```

<a name="reshape2-dcast"><h3>11.1. dcast：将长数据框转化为短数据框 [<sup>目录</sup>](#content)</h3></a>

reshape2::melt常用于将长表格转换成短表格

长表格示例：

| Id | VAR | VALUE |
|:---:|:---:|:---:|
| Id1 | Var1 | a1 |
| Id1 | Var2 | b1 |
| Id1 | Var3 | c1 |
| Id2 | Var1 | a2 |
| Id2 | Var2 | b2 |
| Id2 | Var3 | c2 |
| Id3 | Var1 | a3 |
| Id3 | Var2 | b3 |
| Id3 | Var3 | c3 |

其对应的长表格为：

| Id | Var1 | Var2 | Var3 |
|:---:|:---:|:---:|:---:|
| Id1 | a1 | b1 | c1 |
| Id2 | a2 | b2 | c2 |
| Id3 | a3 | b3 | c3 |

假设原始的短表格保存在数据框变量X中，现要将其转换成以Id列为唯一记录识别项，其他项作为测量项的长表格Y，则可以通过下面的命令实现：

```
Y <- dcast(X, Id~VAR, fun.aggregate = sum, value.var = 'VALUE')
```

<a name="calculate-Asymmetry-coefficient"><h2>12. 计算偏离系数 [<sup>目录</sup>](#content)</h2></a>

$$\gamma_1=\frac{\mu_3}{\mu_2^{3 \over 2}}$$

其中$\mu_2$和$\mu_3$分别表示第二、三中心元素，其计算方法为：

$$\mu_k=\frac{1}{N}\sum_{i=1}^{N}(x_i-\overline{x})^k$$

当分布完全对称时，$\gamma_1=0$，当$\gamma_1<0$或者$\gamma_1>0$则分别表示左偏和右偏

```R
> library(e1071)                    # load e1071 
> duration = faithful$eruptions     # eruption durations 
> moment(duration, order=3, center=TRUE) 
[1] -0.6149 
```



<a name="deal-with-package-install"><h2>13. R包安装的各种报错以及解决 [<sup>目录</sup>](#content)</h2></a>

（1）报错关键信息：`Error in if (nzchar(SHLIB_LIBADD)) SHLIB_LIBADD else character() : argument is of length zero`

参数缺失的报错，找到R安装目录下 `R/etc` 下是否有 `Makeconf` 这个文件，如果是Conda下的R，路径应该为：`Conda_path/lib/R/etc/Makeconf`

先查看这个文件是否存在，如果存在，看看是否是空文件，如果是空文件，将以下内容复制到这个文件中：

```
# etc/Makeconf.  Generated from Makeconf.in by configure.
#
# ${R_HOME}/etc/Makeconf
#
# R was configured using the following call
# (not including env. vars and site configuration)
# configure  '--prefix=/miniconda3/envs/tcga' '--host=x86_64-conda_cos6-linux-gnu' '--build=x86_64-conda_cos6-linux-gnu' '--enable-shared' '--enable-R-shlib' '--with-blas=-lblas' '--with-lapack=-llapack' '--disable-prebuilt-html' '--enable-memory-profiling' '--with-tk-config=/miniconda3/envs/tcga/lib/tkConfig.sh' '--with-tcl-config=/miniconda3/envs/tcga/lib/tclConfig.sh' '--with-x' '--with-pic' '--with-cairo' '--with-readline' '--with-recommended-packages=no' '--without-libintl-prefix' 'LIBnn=lib' 'build_alias=x86_64-conda_cos6-linux-gnu' 'host_alias=x86_64-conda_cos6-linux-gnu' 'PKG_CONFIG_PATH=/miniconda3/envs/tcga/lib/pkgconfig' 'CC=x86_64-conda_cos6-linux-gnu-cc' 'CFLAGS=-march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix' 'LDFLAGS=-Wl,-O2 -Wl,--sort-common -Wl,--as-needed -Wl,-z,relro -Wl,-z,now -Wl,--disable-new-dtags -Wl,--gc-sections -Wl,-rpath,/miniconda3/envs/tcga/lib -Wl,-rpath-link,/miniconda3/envs/tcga/lib -L/miniconda3/envs/tcga/lib -Wl,-rpath-link,/miniconda3/envs/tcga/lib' 'CPPFLAGS=-DNDEBUG -D_FORTIFY_SOURCE=2 -O2 -isystem /miniconda3/envs/tcga/include -I/miniconda3/envs/tcga/include -Wl,-rpath-link,/miniconda3/envs/tcga/lib' 'CPP=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/_build_env/bin/x86_64-conda_cos6-linux-gnu-cpp' 'FC=x86_64-conda_cos6-linux-gnu-gfortran' 'CXX=x86_64-conda_cos6-linux-gnu-c++' 'CXXFLAGS=-fvisibility-inlines-hidden  -fmessage-length=0 -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix' 'OBJC=x86_64-conda_cos6-linux-gnu-cc'

## This fails if it contains spaces, or if it is quoted
include $(R_SHARE_DIR)/make/vars.mk

AR = x86_64-conda_cos6-linux-gnu-ar
BLAS_LIBS = -lblas
C_VISIBILITY = -fvisibility=hidden
CC = x86_64-conda_cos6-linux-gnu-cc
CFLAGS = -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
CPICFLAGS = -fpic
CPPFLAGS = -DNDEBUG -D_FORTIFY_SOURCE=2 -O2 -isystem /miniconda3/envs/tcga/include -I/miniconda3/envs/tcga/include -Wl,-rpath-link,/miniconda3/envs/tcga/lib
CXX = x86_64-conda_cos6-linux-gnu-c++ -std=gnu++11
## Not used by anything in R, in particular not for the .cc.d rule
## but used via R CMD config by several packages
CXXCPP = $(CXX) -E
CXXFLAGS = -fvisibility-inlines-hidden  -fmessage-length=0 -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
CXXPICFLAGS = -fpic
CXX11 = x86_64-conda_cos6-linux-gnu-c++
CXX11FLAGS = -fvisibility-inlines-hidden  -fmessage-length=0 -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
CXX11PICFLAGS = -fpic
CXX11STD = -std=gnu++11
CXX14 = x86_64-conda_cos6-linux-gnu-c++
CXX14FLAGS = -fvisibility-inlines-hidden  -fmessage-length=0 -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
CXX14PICFLAGS = -fpic
CXX14STD = -std=gnu++14
CXX17 = x86_64-conda_cos6-linux-gnu-c++
CXX17FLAGS = -fvisibility-inlines-hidden  -fmessage-length=0 -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
CXX17PICFLAGS = -fpic
CXX17STD = -std=gnu++17
CXX20 =
CXX20FLAGS =  $(LTO)
CXX20PICFLAGS =
CXX20STD =
CXX_VISIBILITY = -fvisibility=hidden
DYLIB_EXT = .so
DYLIB_LD = $(CC)
DYLIB_LDFLAGS = -shared -fopenmp# $(CFLAGS) $(CPICFLAGS)
DYLIB_LINK = $(DYLIB_LD) $(DYLIB_LDFLAGS) $(LDFLAGS)
ECHO = echo
ECHO_C =
ECHO_N = -n
ECHO_T =
F_VISIBILITY = -fvisibility=hidden
## FC is the compiler used for all Fortran as from R 3.6.0
FC = x86_64-conda_cos6-linux-gnu-gfortran
FCFLAGS = -fopenmp -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
## additional libs needed when linking with $(FC), e.g. on some Oracle compilers
FCLIBS_XTRA =
FFLAGS = -fopenmp -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix $(LTO)
FLIBS =  -lgfortran -lm -lgomp -lquadmath -lpthread
FPICFLAGS = -fpic
FOUNDATION_CPPFLAGS =
FOUNDATION_LIBS =
JAR = /usr/bin/jar
JAVA = /usr/bin/java
JAVAC = /usr/bin/javac
JAVAH = /usr/bin/javah
## JAVA_HOME might be used in the next three.
## They are for packages 'JavaGD' and 'rJava'
JAVA_HOME = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre
JAVA_CPPFLAGS = -I$(JAVA_HOME)/../include -I$(JAVA_HOME)/../include/linux
JAVA_LIBS = -L$(JAVA_HOME)/lib/amd64/server -ljvm
JAVA_LD_LIBRARY_PATH = $(JAVA_HOME)/lib/amd64/server
LAPACK_LIBS = -llapack
LDFLAGS = -Wl,-O2 -Wl,--sort-common -Wl,--as-needed -Wl,-z,relro -Wl,-z,now -Wl,--disable-new-dtags -Wl,--gc-sections -Wl,-rpath,/miniconda3/envs/tcga/lib -Wl,-rpath-link,/miniconda3/envs/tcga/lib -L/miniconda3/envs/tcga/lib -Wl,-rpath-link,/miniconda3/envs/tcga/lib
## we only need this is if it is external, as otherwise link to R
LIBINTL=
LIBM = -lm
LIBR0 = -L"$(R_HOME)/lib$(R_ARCH)"
LIBR1 = -lR
LIBR = -L"$(R_HOME)/lib$(R_ARCH)" -lR
LIBS = -L/miniconda3/envs/tcga/lib -lpcre2-8 -llzma -lbz2 -lz -lrt -ldl -lm -liconv -licuuc -licui18n
## needed by R CMD config
LIBnn = lib
LIBTOOL = $(SHELL) "$(R_HOME)/bin/libtool"
LTO =
## needed to build applications linking to static libR
MAIN_LD = $(CC)
MAIN_LDFLAGS = -Wl,--export-dynamic -fopenmp
RPATH_LDFLAGS = -Wl,-rpath,$(abs_top_builddir)/lib -Wl,-rpath,/miniconda3/envs/tcga/lib
MAIN_LINK = $(MAIN_LD) $(MAIN_LDFLAGS) $(LDFLAGS) $(RPATH_LDFLAGS)
MKINSTALLDIRS = "$(R_HOME)/bin/mkinstalldirs"
OBJC = x86_64-conda_cos6-linux-gnu-cc
OBJCFLAGS = -g -O2 -fobjc-exceptions $(LTO)
OBJC_LIBS =
OBJCXX =
R_ARCH =
RANLIB = x86_64-conda_cos6-linux-gnu-ranlib
SAFE_FFLAGS = -fopenmp -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O2 -ffunction-sections -pipe -isystem /miniconda3/envs/tcga/include -fdebug-prefix-map=/home/conda/feedstock_root/build_artifacts/r-base_1595316670215/work=/usr/local/src/conda/r-base-4.0.2 -fdebug-prefix-map=/miniconda3/envs/tcga=/usr/local/src/conda-prefix -msse2 -mfpmath=sse
SED = /miniconda3/envs/tcga/bin/sed
SHELL = /bin/sh
SHLIB_CFLAGS =
SHLIB_CXXFLAGS =
SHLIB_CXXLD = $(CXX)
SHLIB_CXXLDFLAGS = -shared
SHLIB_CXX11LD = $(CXX11) $(CXX11STD)
SHLIB_CXX11LDFLAGS = -shared
SHLIB_CXX14LD = $(CXX14) $(CXX14STD)
SHLIB_CXX14LDFLAGS = -shared
SHLIB_CXX17LD = $(CXX17) $(CXX17STD)
SHLIB_CXX17LDFLAGS = -shared
SHLIB_CXX20LD = $(CXX20) $(CXX20STD)
SHLIB_CXX20LDFLAGS = -shared
SHLIB_EXT = .so
SHLIB_FFLAGS =
SHLIB_LD = $(CC)
SHLIB_LDFLAGS = -shared# $(CFLAGS) $(CPICFLAGS)
SHLIB_LIBADD = 
## We want to ensure libR is picked up from $(R_HOME)/lib
## before e.g. /usr/local/lib if a version is already installed.
SHLIB_LINK = $(SHLIB_LD) $(SHLIB_LDFLAGS) $(LIBR0) $(LDFLAGS)
SHLIB_OPENMP_CFLAGS = -fopenmp
SHLIB_OPENMP_CXXFLAGS = -fopenmp
SHLIB_OPENMP_FFLAGS =
STRIP_STATIC_LIB = x86_64-conda_cos6-linux-gnu-strip --strip-debug
STRIP_SHARED_LIB = x86_64-conda_cos6-linux-gnu-strip --strip-unneeded
TCLTK_CPPFLAGS = -I/miniconda3/envs/tcga/include -I/miniconda3/envs/tcga/include
TCLTK_LIBS = -L/miniconda3/envs/tcga/lib -ltcl8.6 -L/miniconda3/envs/tcga/lib -ltk8.6 -lX11
YACC = yacc

## Legacy settings: these might be used in a src/Makefile
SHLIB_FCLD = $(FC)
SHLIB_FCLDFLAGS = -shared


## for linking to libR.a
STATIC_LIBR = # -Wl,--whole-archive "$(R_HOME)/lib$(R_ARCH)/libR.a" -Wl,--no-whole-archive $(BLAS_LIBS) $(FLIBS)  $(LIBINTL) -lreadline  $(LIBS)

## These are recorded as macros for legacy use in packages
## set on AIX, formerly for old glibc (-D__NO_MATH_INLINES)
R_XTRA_CFLAGS =
##  was formerly set on HP-UX
R_XTRA_CPPFLAGS =  -I"$(R_INCLUDE_DIR)" -DNDEBUG
## currently unset
R_XTRA_CXXFLAGS =
## used for gfortran in R > 3.6.0
R_XTRA_FFLAGS = -fno-optimize-sibling-calls

## CXX98 is no longer supported, but packages may use it.
SHLIB_CXX98LD = @SHLIB_CXX98LD@
SHLIB_CXX98LDFLAGS = @SHLIB_CXX98LDFLAGS@

## SHLIB_CFLAGS SHLIB_CXXFLAGS SHLIB_FFLAGS are apparently currently unused
## SHLIB_CXXFLAGS is undocumented, there is no SHLIB_FCFLAGS
ALL_CFLAGS =  $(PKG_CFLAGS) $(CPICFLAGS) $(SHLIB_CFLAGS) $(CFLAGS)
ALL_CPPFLAGS =  -I"$(R_INCLUDE_DIR)" -DNDEBUG $(PKG_CPPFLAGS) $(CLINK_CPPFLAGS) $(CPPFLAGS)
ALL_CXXFLAGS =  $(PKG_CXXFLAGS) $(CXXPICFLAGS) $(SHLIB_CXXFLAGS) $(CXXFLAGS)
ALL_OBJCFLAGS = $(PKG_OBJCFLAGS) $(CPICFLAGS) $(SHLIB_CFLAGS) $(OBJCFLAGS)
ALL_OBJCXXFLAGS = $(PKG_OBJCXXFLAGS) $(CXXPICFLAGS) $(SHLIB_CXXFLAGS) $(OBJCXXFLAGS)
ALL_FFLAGS = -fno-optimize-sibling-calls $(PKG_FFLAGS) $(FPICFLAGS) $(SHLIB_FFLAGS) $(FFLAGS)
## can be overridden by R CMD SHLIB
P_FCFLAGS = $(PKG_FFLAGS)
ALL_FCFLAGS = -fno-optimize-sibling-calls $(P_FCFLAGS) $(FPICFLAGS) $(SHLIB_FFLAGS) $(FCFLAGS)
## LIBR here as a couple of packages use this without SHLIB_LINK
ALL_LIBS = $(PKG_LIBS) $(SHLIB_LIBADD) $(LIBR)# $(LIBINTL)

.SUFFIXES:
.SUFFIXES: .c .cc .cpp .d .f .f90 .f95 .m .mm .M .o

.c.o:
	$(CC) $(ALL_CPPFLAGS) $(ALL_CFLAGS) -c $< -o $@
.c.d:
	@echo "making $@ from $<"
	@$(CC) -MM $(ALL_CPPFLAGS) $< > $@
.cc.o:
	$(CXX) $(ALL_CPPFLAGS) $(ALL_CXXFLAGS) -c $< -o $@
.cpp.o:
	$(CXX) $(ALL_CPPFLAGS) $(ALL_CXXFLAGS) -c $< -o $@
.cc.d:
	@echo "making $@ from $<"
	@$(CXX) -M $(ALL_CPPFLAGS) $< > $@
.cpp.d:
	@echo "making $@ from $<"
	@$(CXX) -M $(ALL_CPPFLAGS) $< > $@
.m.o:
	$(OBJC) $(ALL_CPPFLAGS) $(ALL_OBJCFLAGS) -c $< -o $@
.m.d:
	@echo "making $@ from $<"
	@$(OBJC) -MM $(ALL_CPPFLAGS) $< > $@
.mm.o:
	$(OBJCXX) $(ALL_CPPFLAGS) $(ALL_OBJCXXFLAGS) -c $< -o $@
.M.o:
	$(OBJCXX) $(ALL_CPPFLAGS) $(ALL_OBJCXXFLAGS) -c $< -o $@
.f.o:
	$(FC) $(ALL_FFLAGS) -c $< -o $@
## @FCFLAGS_f9x@ are flags needed to recognise the extensions
.f95.o:
	$(FC) $(ALL_FCFLAGS) -c  $< -o $@
.f90.o:
	$(FC) $(ALL_FCFLAGS) -c  $< -o $@
```

注意：上面需要修改的位置：

> - 将每个匹配上`/miniconda3/envs/tcga/`的位置，修改成你当前conda的安装路径
>
> - 将`SED = /miniconda3/envs/tcga/bin/sed`中sed的路径设置为当前sed的实际路径

（2）报错信息：`ERROR: failed to lock directory 'C:/Users/MSI-NB/Documents/R/win-library/4.0' for modifying`

这时如果查看对应安装包的文件夹（例子中是win-library/4.0），会发现多了一个叫做“00LOCK-rlang”（或者直接叫“00LOCK”）的文件夹。

错误产生原因：

> 出于防止其他安装过程干扰和暂存旧版本的目的，R在安装X包时会先建立并锁定一个叫00LOCK-X的临时文件夹。安装完毕后如果由于某种原因该临时文件夹没有被删除的话，下次更新可能会因为锁定失败而gg。

解决方案：

> 1. `install.packages()` 加上`INSTALL_opts = '--no-lock'`：
> 
> ```R
> install.packages("your_package", INSTALL_opts = '--no-lock')
> ```
> 
> 2. 删除00LOCK-rlang文件夹，后续照常安装即可

（3）报错信息：

```
../x86_64-conda_cos6-linux-gnu/bin/ld: cannot find -llapack
../x86_64-conda_cos6-linux-gnu/bin/ld: cannot find -lblas
```

本质上就是少了两个基本库工具，如果是在conda环境下，可以用下面的命令安装：

```shell
conda install -c conda-forge lapack
```

<a name="use-dplyr"><h2>14. dplyr包 [<sup>目录</sup>](#content)</h2></a>

<a name="dplyr-summarise-and-mutate"><h3>14.1. summarise 和 mutate 函数使用 [<sup>目录</sup>](#content)</h3></a>

summarise和mutate是R中plyr包的两个函数。plyr是R中用来数据汇总的超强R包

summarise和mutate函数都可以对一个数据框的某一列(而不是整个数据框)进行修改和汇总，两者的主要区别在于返回结果的方式不同，其中summarise函数返回一个只包含修改或汇总后数据的数据框，而mutate函数则返回一个由原始数据和修改或汇总后数据两部分构成的数据框

例如：

```R
require(plyr)
set.seed(1) # 保证每次产生的数据框的唯一性
dfx <- data.frame(
  group = c(rep('A', 8), rep('B', 15), rep('C', 6)),
  sex = sample(c("M", "F"), size = 29, replace = TRUE),
  age = sample(20:30, size = 29, replace = TRUE),
  worktime = sample(1:5, size = 29, replace = TRUE)
)
### 数据修改
summarise(dfx, age = age + 1) # 返回一个只含一列age的数据框
mutate(dfx, age = age + 1) # 返回一个和dfx列数一样的4列数据框，但age列的数值已经修改
### 数据汇总
summarise(dfx, mean.age = mean(age), sd.age = sd(age)) # 返回一个只含汇总结果的2列数据框
mutate(dfx, mean.age = mean(age), sd.age = sd(age)) # 返回一个由dfx和汇总结果组成的4列数据框
```

<a name="dplyr-pipe"><h3>14.2. dplyr的管道操作%>% [<sup>目录</sup>](#content)</h3></a>

`％>％`来自dplyr包的管道函数，类似于Shell命令中的管道`|`

其作用是将前一步的结果直接传参给下一步的函数，从而省略了中间的赋值步骤，可以大量减少内存中的对象，节省内存

举一个简单的例子：

> 比如我们要算f(x)=sin((x+1)^2)在x=4的值，可以分为以下三步：
> 
> （1）计算`a = x+1`的值；
> 
> （2）计算`b = a^2`的值；
> 
> （3）计算`c = sin(b)`的值
> 
> 这样一来，c就是我们需要的最终结果了
>
> 用R语言管道传参，只需要这样写：
> 
> ```R
> f1 <- function(x){return(x+1)}
> f2 <- function(x){return(x^2)}
> f3 <- function(x){return(sin(x))}
> a <- 1
> b <- a %>% f1 %>% f2 %>% f3
> 
> print(b)
> [1] -0.7568025
> ```

管道传参的具体用法：

> `a %>% f(b)`等同于`f(a,b)`；
> 
> `b %>% f(a,.,c)`等同于`f(a,b,c)`；

应用实例：

> 给定以下格式的数据框：
> 
> ```
>          date hour min second
> 1  2017-06-22    8  35     34
> 2  2017-06-23   20  26     27
> 3  2017-06-24    4  56     10
> 4  2017-06-25    3   4     16
> 5  2017-06-26    1  30     52
> 6  2017-06-27   12  22      9
> 7  2017-06-28    5   5     35
> 8  2017-06-29    2   7      4
> 9  2017-06-30   15  39     54
> 10 2017-07-01   14  13      3
> 11 2017-07-02   22   1     29
> 12 2017-07-03    7  60     11
> 13 2017-07-04    6  41      8
> 14 2017-07-05   21  11     40
> 15 2017-07-06   16  33     60
> ```
>
> 将其变成标准时间格式:
>
> ```
>              datetime
> 1   2017-06-22 8:35:34
> 2  2017-06-23 20:26:27
> 3   2017-06-24 4:56:10
> 4    2017-06-25 3:4:16
> 5   2017-06-26 1:30:52
> 6   2017-06-27 12:22:9
> 7    2017-06-28 5:5:35
> 8     2017-06-29 2:7:4
> 9  2017-06-30 15:39:54
> 10  2017-07-01 14:13:3
> 11  2017-07-02 22:1:29
> 12  2017-07-03 7:60:11
> 13   2017-07-04 6:41:8
> 14 2017-07-05 21:11:40
> 15 2017-07-06 16:33:60
> ```
> 
> 使用`unite`函数和管道轻松搞定：
> 
> ```R
> dat1 <- dat %>% unite(datehour, date, hour, sep = ' ') %>% unite(datetime, datehour, min, second, sep = ':')
> ```

<a name="dplyr-other-utility"><h3>14.3. 其他数据筛选与修改操作 [<sup>目录</sup>](#content)</h3></a>

数据筛选：

`filter`：根据指定的条件筛选

`select`：选择指定列

<a name="reshape-output-from-lapply"><h2>15. 用do.call整理lapply的结果 [<sup>目录</sup>](#content)</h2></a>

`lapply`实现的功能是将输入的list的每个元素注意进行处理，并将处理后的每个元素的结果分别保存为一个list元素，即输入为list，输出也是list

此时，如果想要将输出的list形式的结果，整理成比较规整的形式，例如，原先的list中的每个元素是一个数据框，需要将它们按照行进行拼接，即使用`rbind`时实现这个拼接操作，得到一个拼接后的大数据框，可以这样：

```R
do.call('rbind', lapply(list, fun))
```

即，通过`do.call`，将list的每一个元素逐一传递给`rbind`，作为它的多个输入参数，以`rbind(list[1], list[2], ...)`的形式来执行

---

参考资料：

(1) [在R中如何逐行读取CSV文件并将内容识别为正确的数据类型？](https://oomake.com/question/1544925)

(2) [CSDN·卡西莫多的礼物《R语言 optparse的使用》](https://blog.csdn.net/qq_35696312/article/details/88188379)

(3) [R︱并行计算以及提高运算效率的方式(parallel包、clusterExport函数、SupR包简介)](https://blog.csdn.net/sinat_26917383/article/details/52719232)

(4) [统计之都《R 与并行计算》](https://cosx.org/2016/09/r-and-parallel-computing)

(5) [【Ben Solomon】Analyzing TCR sequence Data with Atchley Factors](https://bsolomon.us/post/2018/01/13/TCR_analysis_with_Atchyley_factors.html)

(6) [R Tutorial eBook: Skewness](http://www.r-tutor.com/elementary-statistics/numerical-measures/skewness)

(7) [summarise 和 mutate 函数使用](http://xukuang.github.io/blog/2014/06/dots-in-plyr-of-r-packages/)

(8) [[ERROR] R包安装的 non-zero exit](https://zhuanlan.zhihu.com/p/46077702)

(9) [【R语言】无法安装一些包](https://d.cosx.org/d/154773-154773)

(10) [【知乎】R语言ERROR解读｜failed to lock directory](https://zhuanlan.zhihu.com/p/264363714)
