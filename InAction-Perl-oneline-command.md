<a name="content">目录</a>

[Perl单行实战笔记](#title)
- [实战](#inaction)
	- [1. 计算metagenome shotgun中每种菌的测序深度](#depth-for-meta-bins)
- [附录](#appendix)
	- [1. bioawk的用法](#bioawk)


<h1 name="title">Perl单行实战笔记</h1>

<a name="depth-for-meta-bins"><h2>实战 [<sup>目录</sup>](#content)</h2></a>

<a name="depth-for-meta-bins"><h3>1. 计算metagenome shotgun中每种菌的测序深度 [<sup>目录</sup>](#content)</h3></a>

问题描述：

> 对宏基因组（例如肠道微生物）进行全基因组shotgun测序，对其中已知的高丰度菌进行mapping，从而统计每种菌的测序深度
> 
> 现在你有的文件：
> 1. mapping到各个菌的reads的fasta文件
> 2. 每种菌的refence序列，为fasta格式
> 
> 文件夹结构：
> 
> ```
> workdir
> |----bins （mapping到各个菌的reads的fasta文件，一种菌一个文件）
> 		|----Acetanaerobacterium_sp._Contig_Acetanaerobacterium_sp._GCA_900289145.1
> 		|----Acholeplasma_sp._Scaffold_Acholeplasma_sp._GCA_000432255.1
> 		...
> |----ref （每种菌的refence序列，一种菌一个文件）
> 		|----Acetanaerobacterium_sp._Contig_Acetanaerobacterium_sp._GCA_900289145.1
> 		|----Acholeplasma_sp._Scaffold_Acholeplasma_sp._GCA_000432255.1
> 		...
> ```

注：该实战内容涉及到`bioawk`的使用，如果未使用过`bioawk`请点 [这里](#bioawk) 进行学习

首先，得到mapping到每种菌的reads的总长：

```
$ ls bins | while read i;
do
	echo -ne "$i\t";
	bioawk -c fastx '{len+=length($seq)}END{print len}' bins/$i;
done >mappingLen.stat
```

然后计算每种菌的参考基因组长度：

```
$ ls ref | while read i;
do
	echo -ne "$i\t";
	bioawk -c fastx '{print length($seq)}' ref/$i;
done >refLen.stat
```

最后，用perl单行进行测序深度的计算，即对各个菌求`mappingLen/refLen`：

```
# 这里本来应该写成一行，不过为了方便理解，添加了换行和缩进

$ perl -e \
'open F1,$ARGV[0];
open F2,$ARGV[1];

# 逐行读入第一个文件，即前面得到的mappingLen.stat，保存成哈希，key为菌名，value为mappingLen
while(<F1>){
	chomp;
	@line=split /\t/;
	$ccs{$line[0]}=$line[1];
}

# 逐行读入第二个文件，即前面得到的refLen.stat，保存成哈希，key为菌名，value为refLen
while(<F2>){
	chomp;
	@line=split /\t/;
	$ref{$line[0]}=$line[2];
}

# 对每个菌计算depth
for $assem (keys %ccs){
	if($ref{$assem}){
		$depth=$ccs{$assem}/$ref{$assem};
		print "$assem\t$depth\n";
	}
}' \
mappingLen.stat refLen.stat >ref_depth.stat
```

<a name="appendix"><h2>附录 [<sup>目录</sup>](#content)</h2></a>

<a name="bioawk"><h3>1. bioawk的用法[<sup>目录</sup>](#content)</h3></a>

**bioawk是什么？**

> 曾经在github上有讨论awk的使用，做生信的人都反映awk是很好，但用起来总有那么一点不顺手，毕竟不是为生信而生嘛！
> 
> 李恒看到了，拿来awk的源代码修修补补，最终bioawk诞生

**bioawk能干嘛？**

> 由于bioawk是从awk的基础上衍生出来的，那么从awk出发，对bioawk进行类比就能明白bioawk在干些什么了
> 
> - 首先回答awk在干些什么
> 
> awk的作用是逐行读入文本文件，然后对读入的行按照分隔符（默认是任意的空字符，包括制表符`\t`和空格符`\0`）进行分割，并将分隔开的字符串保存到`$1`，`$2`等中；
> 
> - 类比awk，bioawk在干些什么
> 
> 对于fasta文件，bioawk是逐条记录读入（fasta文件的一条记录包括以`>`起始的序列名和序列），并且按照文件属性自动将每条记录的各个组分打散，保存到对应的变量中，对于fasta文件，它的两部分内容会被打散然后保存到`$name`和`$seq`两个变量中，之后就可以按照对普通变量的处理方法对它们进行操作了；
> 
> bioawk不仅可以处理fasta文件，还可以处理fastq、bed、vcf、sam等格式的文件，处理的逻辑跟上面的相似，只是在提供不同格式的文件时，需要用`-f`参数告诉bioawk这个文件的格式；

bioawk的安装很简单，分两种：

> - 用conda安装
> 
>	```
>	$ conda install bioawk
>	```
> 
> - 源码安装
> 
> 	bioawk的官方地址：`https://github.com/lh3/bioawk`
> 	
> 	可以通过`git clone`或	`wget`的方法获得源码，解压后，进入目录执行`make`即完成源码的编译













---

参考资料：

(1) [【简书】一个神奇的小软件bioawk](https://www.jianshu.com/p/27605abc1cfb)
