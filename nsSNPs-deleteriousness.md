<a name="content">目录</a>

[nsSNPs致病性分析](#title)
- [1. 自己写脚本分析蛋白质保守性](#self-build-script-for-conservation)
- [2. nsSNV有害性评估工具测评](#compare-deleteriousness-prediction-methods)
	- [2.1. 简述](#compare-deleteriousness-prediction-methods-abstract)
	- [2.2. 数据集说明](#compare-deleteriousness-prediction-methods-datasets-description)
	- [2.3. 模型训练](#compare-deleteriousness-prediction-methods-training-models)






<h1 name="title">nsSNPs致病性分析</h1>

<a name="self-build-script-for-conservation"><h2>1. 自己写脚本分析蛋白质保守性 [<sup>目录</sup>](#content)</h2></a>

<p align="right">——仅以此纪念自己半路夭折的工具开发计划</p>

初衷：

> SNV有很多类型，其中落在编码区域的非同义突变 (nsSNPs) 会带来氨基酸序列的改变，很可能带来不良的影响，但是其影响程度与其**所处的位置**以及**氨基酸的理化性质**有关
> 
> 若非同义替换的发生位置是蛋白质的功能或结构的核心区域则更可能带来不良的影响，但是也不尽然，若突变前后的氨基酸的理化性质是否相近，比如分子大小相近，极性也很相似，那么也很可能是一个良性的变异
> 
> 因此氨基酸位置的保守性能很大程度上反映其所处位置的重要性

想法：

> **目的**：分析目的基因编码蛋白质序列各个氨基酸位点的保守性
> 
> **实现方法**：（1）获取与目的蛋白同源的蛋白质序列集合——从NCBI的Homogene数据库中获得某个基因（蛋白质）的同源基因（蛋白质）；（2）基于这些序列之间的多序列比对结果，推算出各个位点的保守性评分（计算方法在下文中说明）——分值在0-1之间，分值越低保守性越低；

1. **获取同源蛋白质序列及多序列比对结果**

	从NCBI的Homogene数据库中获得某个基因（蛋白质）的同源基因（蛋白质），获取这些同源蛋白的多序列比对结果，直接将网页上的多序列比对结果复制黏贴成纯文本`homogene.msa`

	<p align="center"><img src=./picture/nsSNPs-deleteriousness-HomoloGene-msa.png width=600 /></p>

	<p align="center">HomoloGene中的同源蛋白序列以多序列比对</p>
	
	将`homogene.msa`转换成FASTA格式：
	
	```
	# 读入上一步复制黏贴直接得到的原始多序列比对结果文件，除了空行之外，
	# 	每行都由四列组成（以任意空字符为分隔符），其中第一列为序列Id，
	# 	第三列为序列组成，则将这些序列读入并保存成哈希，以序列Id为key，
	# 	以序列组成为value
	# 文件读取结束后，即将每条序列以fasta格式输出

	$ perl -ne \
	'chomp;
	next if(/^$/);	# 跳过空行
	@F=split /\s+/;
	$Seq{$F[0]}.=$F[2];
	END{
		foreach $key (keys(%Seq)){
			print ">$key\n$Seq{$key}\n"
		}
	}' \
	homogene.msa >homogene.msa.fasta
	```

2. **计算保守性分值**

	用python脚本`ConservativeAnalysis.py`基于`homogene.msa.fasta`计算目标蛋白质序列各个位点的保守性，保守性的计算公式如下：

	<p align="center"><img src=./picture/conservative_formula-1.png width=600 /></p>

	这个公式其实用到了信息熵的思想，本质上是将总信息熵（或称最大信息熵）log<sub>2</sub>N，减去观测到的信息熵`-∑Plog2P`，若某一个位点的氨基酸组成多样性高，则保守性低，其观测信息熵就大，算出来的R值就小

	还可以用以下两种方式表示保守性：

	> - 相对保守性：
	> 
	> <p align="center"><img src=./picture/conservative_formula-2.png height=80 /></p>
	> 
	> - 保守性分位数：
	> <p align="center"><img src=./picture/conservative_formula-3.png height=80 /></p>

	Python脚本代码如下（脚本对上面提到的3中保守性分值都进行了计算）：

	```
	from Bio import SeqIO
	import sys
	import numpy as np
	import math
	
	# 用Biopython读入fasta序列
	def loadAASeq(infile):
		seq = {}
		for i in SeqIO.parse(infile,'fasta'):
			seq[i.id] = list(i.seq)
		return seq
	
	# 构建20种氨基酸和“-”gap占位符的字典，value为这些符号的编号
	def AAHash():
		AARow = {"A":0,
		"R":1,
		"N":2,
		"D":3,
		"C":4,
		"Q":5,
		"E":6,
		"G":7,
		"H":8,
		"I":9,
		"L":10,
		"K":11,
		"M":12,
		"F":13,
		"P":14,
		"S":15,
		"T":16,
		"W":17,
		"Y":18,
		"V":19,
		"-":20}
		return AARow
	
	# 对目标序列某一个位点的氨基酸组成进行频数统计
	def countFreq(seq,gaps,posi):
		'''
		seq:	[char] 目标序列名
		gaps:	[int] posi位置之前，目标序列存在的gap数
		posi:	[int] 当前位置,0-base
		'''
		if Seq[seq][posi] == '-':
			gaps += 1
		else:
			for key in Seq.keys():
				aa = Seq[key][posi]
				row = int(AA[aa])
				col = posi-gaps
				Count[row,col] += 1
		return gaps
	
	# 计算保守性分位数
	def Quantile(Vec):
		Vlen = len(Vec)
		SortIndex = Vec.argsort()
		
		# 计算每个数的秩序，当有多个数的值相等时，则用它们的最大秩序表示，如[1,2,2,5]，其中的两个2并列第3
		# 	主要思路为：若当前位置的值与上一个位置的值相同，说明存在数值并列情况，记下这个位置；
		#		若当前位置的值与上一个位置的值不相同，对之前的位置生成它们的秩序
		#	注意考虑第一个位置和最后一个位置的特殊情况：第一个位置没有更前的位置，最后一个位置没有更后的位置
		VRank = np.zeros(Vlen)
		forwardIndex = list()
		forwardValue = 0
		for i in range(0,Vlen):
			if Vec[SortIndex[i]] == forwardValue:
				forwardIndex.append(SortIndex[i])
				# 考虑最后一个位置的情况
				if i == Vlen-1:
					for j in forwardIndex:
						VRank[j] = i+1
			else:
				if len(forwardIndex) >= 2:
					for j in forwardIndex:
						VRank[j] = i
				else:
					# 考虑第一个位置的情况
					if i > 0:
						VRank[SortIndex[i-1]] = i
				# 考虑最后一个位置的情况
				if i == Vlen-1:
					VRank[SortIndex[i]] = i+1
				forwardIndex = [SortIndex[i]]
				forwardValue = Vec[SortIndex[i]]
		
		# 计算分位数
		PercValue = VRank*100/Vlen
		return PercValue
	
	if __name__ == '__main__':
		'''
		脚本共有两个输入参数：
		1. fasta格式的多序列比对结果
		2. 目标序列的id
		'''
		in_msa = sys.argv[1]
		nseq = sys.argv[2]
		Seq = loadAASeq(in_msa)
		AA = AAHash()
		seqlenAll = len(Seq[nseq]) # 多序列比对得到的比对整体的长度
		seqlenInt = seqlenAll - Seq[nseq].count('-') # 目标序列的氨基酸序列长度
		Count = np.zeros((21,seqlenInt),dtype = float) + 0.000001 # 加上一个很小的数是为了防止后面在执行对数计算时报错
		gaps = 0
		# 对每个位点进行频数统计，统计目标序列每个氨基酸位点的氨基酸组成频数分布，得到一个矩阵，行为20中氨基酸加“-”gap占位符，列为各个氨基酸位点	
		for i in range(0,seqlenAll):
			gaps = countFreq(nseq,gaps,i)
		colSum = sum(Count[...,0])
		# 每个位点氨基酸组成的频率
		CountRate = Count/colSum
		# 计算每个位点的保守性
		Shannon = -(CountRate*np.log2(CountRate)).sum(axis=0)
		Rindex = math.log2(21)-Shannon
		# 相对保守性
		Rrelat = (Rindex - Rindex.min())/(Rindex.max()-Rindex.min())
		# 保守性分位数，使用自定义的Quantile函数
		Rquant = Quantile(Rindex)
	```

	由于脚本中添加了详细的注释，这里就不展开讲解了

<a name="compare-deleteriousness-prediction-methods"><h2>2. nsSNV有害性评估工具测评 [<sup>目录</sup>](#content)</h2></a>

<a name="compare-deleteriousness-prediction-methods-abstract"><h3>2.1. 简述 [<sup>目录</sup>](#content)</h3></a>

测评的工具：

> - 11款功能影响预测工具 (function prediction scores)：
> 	1. PolyPhen-2
> 	2. SIFT
> 	3. MutationTaster
> 	4. Mutation Assessor
> 	5. FATHMM
> 	6. LRT
> 	7. PANTHER
> 	8. PhD-SNP
> 	9. SNAP
> 	10. SNPs&GO
> 	11. MutPred
> - 3款保守性评估工具 (conservation scores)：
> 	1. GERP++
> 	2. SiPhy
> 	3. PhyloP
> - 4款综合评估工具 (ensemble scores)：
> 	1. CADD
> 	2. PON-P
> 	3. KGGSeq
> 	4. CONDEL

评估的结论是：

> 独立的打分工具中，FATHMM表现最优；
> 
> 综合评估工具中，KGGSeq表现最优；

<a name="compare-deleteriousness-prediction-methods-datasets-description"><h3>2.2. 数据集说明 [<sup>目录</sup>](#content)</h3></a>

评估中所使用的数据集，共4个，如下表：

| Dataset | Training dataset | Testing dataset I | Testing dataset II | Testing dataset III|
|:---|:---|:---|:---|:---|
| TP | 14 191 | 120 | 6279 | 0 |
| TN| 22 001 | 124 | 13 240 | 10 164 |
| Total | 36 192 | 244 | 19 519 | 10 164 |
| Source | Uniprot database | Recent Nature Genetics publications for TP variants <br> <br>CHARGE database for TN variants | VariBench dataset II without mutations in training dataset | CHARGE database |

> - TP: true positive, number of deleterious mutations that were treated as TP observations in modeling. 
> 
> - TN: true negative, number of non-deleterious mutations that were treated as true negative observations in modeling.
> 
> TN包括常见变异（common variants，MMAF>1%，1000 Genomes project）和罕见变异（rare variants，MMAF ≤1%）
> 
> - Total: total number of mutations for each dataset (Total = TP + TN).


其中`Training dataset`作为训练集，用于训练SVM和LR（logistic回归）模型（这两个是作者在额外开发的工具），另外3个数据作为测试集，用于测试作者开发的工具与其他工具的性能

在整理测试数据集时，为了降低潜在的bias，从3个不同的数据来源搜集数据：

> - **Testing dataset I**
> 
> 	120 TP observations that are deleterious variants recently reported to cause Mendelian diseases, diseases caused by single-gene defects, with experimental support in 57 recent publications (after 1 January 2011) from the journal Nature Genetics
> 
> 	124 TN observations that are common neutral variants (MMAF >1%) newly discovered from participants free of Mendelian disease from the Atherosclerosis Risk in Communities (ARIC) study (32) via the Cohorts for Heart and Aging Research in Genomic Epidemiology (CHARGE) sequencing project 
> 
> - **Testing dataset II**
> 
> 	Testing dataset II is derived from the VariBench (13) dataset II, a benchmark dataset used for performance evaluation of nsSNV scoring methods. 
> 
> 	Because VariBench dataset II contains mutations that overlap our training dataset, we removed these mutations and curated 6279 deleterious variants as our TP observations and 13 240 common neutral variants (MMAF >1%) as our TN observations.
> 
> - **Testing dataset III**
> 
> 	只搜集罕见中性变异 (rare neutral mutations)
> 
> 	Testing dataset III that contains 10 164 rare neutral mutations (singletons) from 824 European Americans from the cohort random samples of the ARIC study (32) via the CHARGE sequencing project

同时，测试集的选择还要避免与待评估工具的训练数据发生重叠，由于测试集I和III都是从最新的研究项目和文献中整理的，测试集II一般被用作工具性能比较的benchmark，都不太可能被用作那些待评估工具的训练集

因此，这些测试集在构造上bias比较低

<a name="compare-deleteriousness-prediction-methods-training-models"><h3>2.3. 模型训练 [<sup>目录</sup>](#content)</h3></a>

对于每个nsSNV位点，SVM和LR模型能基于其他工具给出的**分值**和**千人基因组MMAF**给出综合性评分

分值数据采用9个工具的：SIFT, PolyPhen-2, GERP++, MutationTaster, Mutation Assessor, FATHMM, LRT, SiPhy and PhyloP

各个工具的分值数据的缺失情况：

|	Dataset	|	Training dataset (%)	|	Testing dataset I (%)	|	Testing dataset II (%)	|	Testing dataset III (%)	|
|:---|:---|:---|:---|:---|
|	SIFT	|	6.91	|	2.87	|	3.83	|	3.96	|
|	PolyPhen-2	|	3.79	|	2.87	|	0.55	|	0.02	|
|	LRT	|	10.49	|	13.93	|	7.97	|	11.33	|
|	MutationTaster	|	0.04	|	0.41	|	0.1	|	0.11	|
|	Mutation Assessor	|	1.51	|	3.69	|	2.23	|	2.67	|
|	FATHMM	|	4.05	|	5.73	|	3.48	|	6.69	|
|	GERP++	|	0	|	0	|	0	|	0.01	|
|	PhyloP	|	0	|	0	|	0	|	0	|
|	SiPhy	|	0	|	0	|	0	|	0.285	|
|	PON-P	|	NA	|	13.93	|	NA	|	NA	|
|	PANTHER	|	NA	|	47.95	|	NA	|	NA	|
|	PhD-SNP	|	NA	|	6.56	|	NA	|	NA	|
|	SNAP	|	NA	|	9.02	|	NA	|	NA	|
|	SNPs&GO	|	NA	|	15.98	|	NA	|	NA	|
|	MutPred	|	NA	|	7.37	|	NA	|	NA	|
|	KGGSeq	|	NA	|	0.82	|	0.05	|	0.82	|
|	CONDEL	|	NA	|	0.41	|	0.0003	|	0.32	|
|	CADD	|	0	|	0	|	0	|	0	|

对于缺失数据采用合适的数据填充方法

SVM模型的训练使用了3种核函数：linear，radial 和 polynomial kernel












---

参考资料：

(1) Dong C, Wei P, Jian X, et al. Comparison and integration of deleteriousness prediction methods for nonsynonymous SNVs in whole exome sequencing studies. Hum Mol Genet. 2014;24(8):2125-37. 
