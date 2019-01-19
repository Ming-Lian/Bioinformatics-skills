<a name="content">目录</a>

[SAM/BAM相关的进阶知识](#title)
- [1. samtools和picard的排序问题](#sort-by-samtools-and-picard)
- [2. SAM文件中FLAG值的理解](#understand-flg-value)
- [3. SAM文件中那些未比对的reads](#unmaped-reads)



<h1 name="title">SAM/BAM相关的进阶知识</h1>

<a name="sort-by-samtools-and-picard"><h2>1. samtools和picard的排序问题 [<sup>目录</sup>](#content)</h2></a>

`samtools`和`picard`都有对SAM/BAM文件进行排序的功能，一般都是基于坐标排序（还提供了`-n`选项来设定用reads名进行排序），先是对chromosome/contig进行排序，再在chromosome/contig内部基于start site从小到大排序，对start site排序很好理解，可是对chromosome/contig排序的时候是基于什么标准呢？

**基于你提供的`ref.fa`文件中的chromosome/contig的顺序**。当你使用比对工具将fastq文件中的reads比对上参考基因组后会生成SAM文件，SAM文件包含头信息，其中有以`@SQ`开头的头信息记录，reference中有多少条chromosome/contig就会有多少条这样的记录，而且它们的顺序与`ref.fa`是一致的。

> SAM/BAM文件的头信息：
> 
> ```
> @HD     VN:1.3  SO:coordinate
> @SQ     SN:chr1 LN:195471971
> @SQ     SN:chr2 LN:182113224
> @SQ     SN:chr3 LN:160039680
> @SQ     SN:chr4 LN:156508116
> @SQ     SN:chr5 LN:151834684
> @SQ     SN:chr6 LN:149736546
> @SQ     SN:chr7 LN:145441459
> @SQ     SN:chr8 LN:129401213
> @SQ     SN:chr9 LN:124595110
> @SQ     SN:chr10        LN:130694993
> @SQ     SN:chr11        LN:122082543
> @SQ     SN:chr12        LN:120129022
> @SQ     SN:chr13        LN:120421639
> @SQ     SN:chr14        LN:124902244
> @SQ     SN:chr15        LN:104043685
> @SQ     SN:chr16        LN:98207768
> @SQ     SN:chr17        LN:94987271
> @SQ     SN:chr18        LN:90702639
> @SQ     SN:chr19        LN:61431566
> @SQ     SN:chrX LN:171031299
> @SQ     SN:chrY LN:91744698
> @SQ     SN:chrM LN:16299
> @RG     ID:ERR144849    LB:ERR144849    SM:A_J  PL:ILLUMINA
> ```
> 
> `ref.fa`中chromosome/contig的排列顺序：
> 
> ```
> >chr1
> >chr2
> >chr3
> >chr4
> >chr5
> >chr6
> >chr7
> >chr8
> >chr9
> >chr10
> >chr11
> >chr12
> >chr13
> >chr14
> >chr15
> >chr16
> >chr17
> >chr18
> >chr19
> >chrX
> >chrY
> >chrM
> ```
> 
> 它们的顺序一致


当使用samtools或picard对SAM/BAM文件进行排序时，这些工具就会读取头信息，按照头信息指定的顺序来排chromosome/contig。所以进行排序时需要提供包含头信息的SAM/BAM文件。

那么**普通情况下我们的chromosome/contig排序情况是什么样的?**

> 一般情况下我们获取参考基因组序列文件的来源有三个：
> 
> - NCBI
> - ENSEMBEL
> - UCSC Genome Browser
> 
> 这里以UCSC FTP下载源为例：
> 
> <p align="center"><img src=./picture/Advanced-knowledge-SAM-1-1.png width=600 ></p>
> 
> 这是一个压缩文件，使用`tar zxvf chromFa.tar.gz`解压后，会得到多个fasta文件，每条chromosome/contig一个fasta文件：chr1.fa, chr2.fa ...
> 
> 之后我们会将它们用`cat *.fa >ref.fa`合并成一个包含多条chromosome/contig的物种参考基因组序列文件
> 
> 用`grep ">" ref.fa`可以查看合并后发ref.fa文件中染色体的排列顺序为：
> 
> ```
> >chr10
> >chr11
> >chr12
> >chr13
> >chr14
> >chr15
> >chr16
> >chr17
> >chr18
> >chr19
> >chr1
> >chr1_GL456210_random
> >chr1_GL456211_random
> >chr1_GL456212_random
> >chr1_GL456213_random
> >chr1_GL456221_random
> >chr2
> >chr3
> >chr4
> ```
> 
> 这和我们平时想象的染色体的排列顺序是不是有一些出入？难道不应该是从chr1开始到chr22，最后是chrX和chrY这样的顺序吗？
> 
> 想象归想象，实际上它是按照字符顺序进行的，chr11就应该排在chr2前面

一般情况下在进行SAM文件的排序时，染色体的排序到底是按照哪种规则进行排序的，不是一个很重要的问题，也不会对后续的分析产生影响，但是在执行GATK流程时，GATK对染色体的排序是有要求的，必须按照从chr1开始到chr22，最后是chrX和chrY这样的顺序，否则会报错

面对这样变态的要求，我们怎么解决？

在构造ref.fa文件时，让它按照从chr1开始到chr22，最后是chrX和chrY这样的顺序进行组织就可以了：

```
for i in $(seq 1 22) X Y M;
do cat chr${i}.fa >> hg19.fasta;
done
```

<a name="understand-flg-value"><h2>2. SAM文件中FLAG值的理解 [<sup>目录</sup>](#content)</h2></a>

FLAG列在SAM文件的第二列，这是一个很重要的列，包含了很多mapping过程中的有用信息，但很多初学者在学习SAM文件格式的介绍时，遇到FLAG列的说明，常常会一头雾水

<p align="center"><img src=./picture/Advanced-knowledge-SAM-2-1.png width=200 ></p>

what?还二进制，这也太反人类的设计了吧！

不过如果你站在开发者的角度去思考这个问题，就会豁然开朗

在mapping过程中，我们想记录一条read的mapping的信息包括：

- 这条read是read1 (forward-read) 还是read2 (reverse-read)？
- 这条read比对上了吗？与它对应的另一头read比对上了吗？
- ...

这些信息总结起来总共包括以下12项：

| 序号 | 简写 | 说明 |
|:---:|:---|:---|
|	1	|	PAIRED	|	paired-end (or multiple-segment) sequencing technology	|
|	2	|	PROPER_PAIR	|	each segment properly aligned according to the aligner	|
|	3	|	UNMAP	|	segment unmapped	|
|	4	|	MUNMAP	|	next segment in the template unmapped	|
|	5	|	REVERSE	|	SEQ is reverse complemented	|
|	6	|	MREVERSE	|	SEQ of the next segment in the template is reversed	|
|	7	|	READ1	|	the first segment in the template	|
|	8	|	READ2	|	the last segment in the template	|
|	9	|	SECONDARY	|	secondary alignment	|
|	10	|	QCFAIL	|	not passing quality controls	|
|	11	|	DUP	|	PCR or optical duplicate	|
|	12	|	SUPPLEMENTARY	|	supplementary alignment	|

而每一项又只有两种情况，是或否，那么我可以用一个12位的二进制数来记录所有的信息，每一位表示某一项的情况，这就是原始FLAG信息的由来，但是二进制数适合给计算机看，不适合人看，需要转换成对应的十进制数，也就有了我们在SAM文件中看到的FLAG值

但是FLAG值所包含信息的解读还是要转换为12位的二进制数

<p align="center"><img src=./picture/Advanced-knowledge-SAM-2-2.png width=800 ></p>

<a name="unmaped-reads"><h2>3. SAM文件中那些未比对的reads [<sup>目录</sup>](#content)</h2></a>

SAM格式文件的第3和第7列，可以用来判断某条reads是否比对成功到了基因组的染色体，左右两条reads是否比对到同一条染色体

<p align="center"><img src=./picture/Advanced-knowledge-SAM-3-1.png width=800 ></p>

有两个方法可以提取未比对成功的测序数据：

> - SAM文件的第3列是*的(如果是PE数据，需要考虑第3,7列)
> 
> ```
> $ samtools view sample.bam | perl -alne '{print if $F[2] eq "*" or $F[6] eq "*" }' sample.unmapped.sam
> ```
>  
> - 或者SAM文件的flag标签包含0x4的
> 
> ```
> # 小写的f是提取，大写的F是过滤
> $ samtools view -f4 sample.bam sample.unmapped.sam
> ```
> 
> 虽然上面两个方法得到的结果是一模一样的，但是这个perl脚本运行速度远远比不上上面的samtools自带的参数

在未比对成功的测序数据可以分成3类：

> - 仅reads1没有比对成功；
> 
> 该提前条件包括：
> 
>> - 该read是read1，对应于二进制FLAG的第7位，其十进制值为64；
>> - 该read未成功比对到参考基因组，对应于二进制FLAG的第3位，其十进制值为8；
>> - 另一配对read成功比对到参考基因组,对应于二进制FLAG的第4位，其十进制值为0；
> 
> ```
> # 
> $ samtools view -f 
> ```
> 
> - 仅reads2没有比对成功；
> - 两端reads都没有比对成功



---

参考资料：

(1) [【简书】从零开始完整学习全基因组测序数据分析：第5节 理解并操作BAM文件](https://www.jianshu.com/p/364e640d3c9f)

(2) [【生信技能树】【直播】我的基因组（十五）:提取未比对的测序数据](https://mp.weixin.qq.com/s?__biz=MzAxMDkxODM1Ng==&mid=2247483782&idx=1&sn=a2c7bac48e03195b646ae34f3b82ac29&chksm=9b48413dac3fc82b61f9868c4385609af3eba9bf68dc026ab78a3f14980ce750cb35623195ce&scene=21#wechat_redirect)





