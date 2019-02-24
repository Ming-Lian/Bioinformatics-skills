<a name="content">目录</a>

[shell+R双重并行化——加速分析过程](#title)
- [问题描述](#description)
- [shell并行化](#shell-parallel)
- [R并行化](#r-parallel)
- [shell+R双重并行化实现案例](#example)




<h1 name="title">shell+R双重并行化——加速分析过程</h1>

<a name="description"><h2>问题描述 [<sup>目录</sup>](#content)</h2></a>

在进行宏基因组shotgun数据分析时，数据集分case和control两组，每组11个样本，通过Gene Profiling得到400多万行的profile（以RPKM方式进行定量），得到的proflie文件有600+MB

接着想用wilcox检验来筛选两组gene abundance存在差异的基因，结果跑了一天一夜没跑出结果

在排除脚本问题后，基本确定问题是数据量太大，计算效率太低导致的，在解决这个问题过程中总结出了一种加速技巧：**shell+R的双重并行化**

<a name="shell-parallel"><h2>shell并行化 [<sup>目录</sup>](#content)</h2></a>

我目前所知道的在linux环境下的并行化实现方式有两种：

（1）利用Slurm排队系统

可以使用`--array=1-11%4`参数来指定任务队列编号为1-11，每次最多并行执行4个任务

有两种书写格式：

```
1. 作为sbatch的参数
$ sbatch --array=1-11%4 tasks.sh

2. 写在shell脚本内部
如：
	#!/bin/bash
	#SBATCH --array=1-11%4
	...
```

（2）使用`do{...} &;done;wait`语句

该语句的格式为：

```
while read i
do
	{
	...
	} &
done
wait

或

for((i=0;i<n;i++)
do
	{
	...
	} &
done
wait
```

例如：

```
for((i=0;i<10;i++))
do
	{
	echo $i
	sleep 5
	} &
done
wait
```

上面的命令如果不使用并行化，则总的执行时间为10*5=50秒，而并行之后只需要5秒

可以在该命令模块外面再套一层循环，来**控制每次并行化的任务数量**，这个需求在大多数情况下是非常必要的：若总的任务很多，如果不对每一次并行化的任务数进行控制，而是一股脑地全部提交并行任务，非常容易到达服务器的负荷上限，那么等着你的很可能就是其他用户的骂声一片和服务器管理员的警告了！

例如，从1到100，每次输出5个数，每次输出后间隔5秒后再继续下一轮的输出：

```
for((i=0;5*i<100;i++))
do
	for((j=5*i+1;j<5*i+6;j++))
	do
		{
		echo $j
		sleep 5
		} &
	done
	wait
done
```




<a name="r-parallel"><h2>R并行化 [<sup>目录</sup>](#content)</h2></a>

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



<a name="example"><h2>shell+R双重并行化实现案例 [<sup>目录</sup>](#content)</h2></a>

有了上面提到的shell与R的并行化的实现方法，那么我们就可以将其运用于本文一开始提到的问题

并行化的思路：

> （1）先将原始的大profile文件进行拆分，若每m行拆成一个subprofile，可以拆成n个subprofiles
> 
> （2）对这n个subprofiles，按照每i个执行一次并行化任务（**shell并行化**），给每个并行化任务j个核心（**R并行化**）

1. **shell脚本**

	脚本名：`ParWilcox.sh`
	```
	# 参数说明：
	# - Profile files
	# - rows per sub-profile
	# - maximum tasks per run
	# - outdir
	
	profiles=$1
	nrow_per=$2
	num_tasks=$3
	outdir=$4
	if [ ! -d $outdir ]
	then
		mkdir -p $outdir
	fi
	if [ -f $outdir/SubProfiles.list ]
	then
		rm $outdir/SubProfiles.list
	fi
	
	##################################需要修改的变量##################################
	workdir=/Path/To/Workdir
	################################################################################

	# 分割原始Profiles
	nline=$(wc -l $profiles | awk '{printf $1}')
	for((i=0;i*nrow_per<nline;i++))
	do
		awk -v j=$i -v n=$nrow_per 'NR==1 || (NR>j*n+1 && NR<=(j+1)*n+1)' $profiles >$outdir/SubProfiles-${i}
		echo "SubProfiles-${i}" >>$outdir/SubProfiles.list
	done
	
	# 并行化执行多个sub-profile的wilcox检验
	num_SubProfiles=$(wc -l $outdir/SubProfiles.list | awk '{printf $1}')
	for((i=0;i*num_tasks<num_SubProfiles;i++))
	do
		awk -v j=$i -v n=$num_tasks 'NR>j*n && NR<=(j+1)*n' $outdir/SubProfiles.list > $outdir/current_subprofiles.List
		while read subprofile
		do
			{
			Rscript $workdir/Script/wilcox-sided-parallel.R $outdir/$subprofile $workdir/Gene_abundance/SampleGroup.txt $outdir/statOut.$subprofile 2
			} &
		done < $outdir/current_subprofiles.List
		wait
	done
	cat $outdir/statOut.* >$outdir/statOut
	rm $outdir/statOut.*
	```

2. **被shell脚本调用的执行wilcox检验的R脚本**

	脚本名：`wilcox-sided-parallel.R`

	```
	# 三个参数，按顺序分别为：
	# - profiles matrix file
	# - sample group file
	# - outfile name
	# - threads number（默认为服务器总线程-10）

	Args <- commandArgs(T)

	library(parallel)

	# Initiate cluster
	if (is.na(Args[4])){
		no_cores <- detectCores() - 10
	} else {
		no_cores <- as.integer(Args[4])
	}
	cl <- makeCluster(no_cores)

	data <- read.table(Args[1],head=T,row.names=1,sep='\t')
	group <- read.table(Args[2],head=T,sep='\t')
	
	# 找出profiles中每列代表的样本所属的组
	col_group1 <- colnames(data) %in% sub('-','.',paste(group$Sample[group$Group==1],'.rpkm',sep=''))
	col_group2 <- colnames(data) %in% sub('-','.',paste(group$Sample[group$Group==2],'.rpkm',sep=''))
	
	# 对单个基因进行wilcox检验，并输出统计检验结果，输出格式如下：
	# 	geneName,pvalue,group1Median,group2Median,direction
	wilcox_fun <- function(data,col_group1,col_group2){
		g1 <- as.numeric(data[col_group1])
		g2 <- as.numeric(data[col_group2])
		# 执行wilcox检验
		stat <- wilcox.test(g1,g2,paired = FALSE, exact=NULL, correct=TRUE, alternative="two.sided")
		if(is.na(stat$p.value)){
			stat$p.value <- 1
		}
		# 对有统计学意义的基因进行判断，是上调"up"还是下调"down"或者是不变"-"（对于组2，即不吃药组）
		if(stat$p.value < 0.1  & median(g1) < median(g2)){
			G12 <- c(ifelse(is.na(stat$p.value),1,stat$p.value),median(g1),median(g2),'down')
		}
		else if(stat$p.value < 0.1  & median(g1) > median(g2)){
			G12 <- c(ifelse(is.na(stat$p.value),1,stat$p.value),median(g1),median(g2),'up')
		}
		else if(stat$p.value < 0.1  & median(g1) == median(g2)){
			G12 <- c(ifelse(is.na(stat$p.value),1,stat$p.value),median(g1),median(g2),'-')
		}
		else{
			G12 <- c()
		}
		G12
	}
	
	statOut <- parApply(cl,data,1,wilcox_fun,col_group1,col_group2)
	stopCluster(cl)

	# 删除为NULL的列表元素
	for(i in names(statOut)){
		if(is.null(statOut[[i]])){
			statOut[[i]] <- NULL
		}
	}

	# 将结果写入文件中
	if(!is.null(statOut)){
		# 转换成数据框
		statOut <- as.data.frame(t(as.matrix(as.data.frame(statOut))))
		colnames(statOut) <- c('pvalue','group1Median','group2Median','direction')
		# 写入文件
		write.table(statOut,Args[3],sep = '\t',row.names = T,col.names = T,quote = F)
	}
	```

执行方法如下：

```
$ bash ParWilcox.sh <raw profile> <rows per sub-profile> <tasks per run> <threads>
```

脚本中用到的样本分组文件，如下：

```
Sample	Group
NP-003A	1
NP-017A	1
NP-018A	1
...
NP-007A	2
NP-010A	2
NP-011A	2
...
```
