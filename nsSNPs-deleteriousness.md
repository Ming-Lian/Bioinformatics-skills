<a name="content">目录</a>

[nsSNPs致病性分析](#title)
- [1. 自己写脚本分析蛋白质保守性](#self-build-script-for-conservation)
- [2. nsSNV有害性评估工具测评](#compare-deleteriousness-prediction-methods)
	- [2.1. 简述](#compare-deleteriousness-prediction-methods-abstract)
	- [2.2. 数据集说明](#compare-deleteriousness-prediction-methods-datasets-description)
	- [2.3. 模型训练](#compare-deleteriousness-prediction-methods-training-models)
- [3. 现有工具及其原理](#exist-tools)
	- [3.1. SIFT](#sift)
	- [3.2. Polyphen2](#polyphen2)
	- [3.3. CADD](#cadd)
	- [3.4. DANN](#dann)
	- [3.5. MetaSVM](#metasvm)
	- [3.6. dbNSFP数据库：整合多种nsSNP预测工具的结果](#dbnsfp)





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

<a name="exist-tools"><h2>3. 现有工具及其原理 [<sup>目录</sup>](#content)</h2></a>

<a name="sift"><h3>3.1 SIFT [<sup>目录</sup>](#content)</h3></a>

<p align="center"><img src=./picture/conservative_exist-tools-SIFT-1.png width=800 /></p>

<p align="center"><img src=./picture/conservative_exist-tools-SIFT-2.png width=800 /></p>

算法说明：

```
For a given protein sequence, SIFT compiles a dataset of functionally related protein sequences by searching a protein database
 using the PSI-BLAST algorithm6. It then builds an alignment from the homologous sequences with the query sequence. 

In the second step of the algorithm, SIFTscans each position in the alignment and calculates the probabilities for all possible
 20 amino acids at that position. These probabilities are normalized by the probability of the most frequent amino acid and are
 recorded in a scaled probability matrix. 

SIFT predicts a substitution to affect protein function if the scaled probability, also termed the SIFTscore, lies below a certain
 threshold value. Generally, a highly conserved position is intolerant to most substitutions, whereas a poorly conserved position
 can tolerate most substitutions
```

计算某个位点保守性的公式：

<p align="center"><img src=./picture/conservative_exist-tools-SIFT-3.png width=300 /></p>

P<sub>ca</sub>的计算方法——基于伪计数得到校正的PSSM：

<p align="center"><img src=./picture/conservative_exist-tools-SIFT-4.gif width=300 /></p>

> N<sub>c</sub>：实际得到的同源序列数
> g<sub>ca</sub>：目标序列c位点出现a氨基酸的实际频率
> B<sub>c</sub>：伪计数，未观察到的同源序列数
> f<sub>ca</sub>：目标序列c位点出现a氨基酸的伪计数频率

<p align="center"><img src=./picture/conservative_exist-tools-SIFT-5.png width=600 /></p>

由伪计数得到的 B<sub>c</sub> 和 f<sub>ca</sub> 是怎么确定的？

> 对于氨基酸组成多样的位点，SIFT倾向于使用更大的伪计数，因为越多样则被漏掉的同源序列可能就越多

得到的PSSM的形式：

<p align="center"><img src=./picture/conservative_exist-tools-SIFT-6.png width=600 /></p>

<a name="polyphen2"><h3>3.2 Polyphen2 [<sup>目录</sup>](#content)</h3></a>

PolyPhen同时结合序列和结果上的信息，主要的假设就是说有一些氨基酸的改变可能会影响蛋白的折叠，影响蛋白的的相互作用区间，影响它的稳定性 ，而蛋白结构如果有改变，那蛋白的功能就更可能会发生改变，所以它整合了序列和三维结构的一些特征 

<p align="center"><img src=./picture/conservative_exist-tools-polyphen2-1.jpeg width=600 /></p>

**Sequence-based features**

> （1）通过已有的蛋白质注释数据库（如UniProtKB/Swiss-Prot），鉴定某个替换 (substitution) 是否落在某个特殊的区域/位置
> 
> 特殊位点包括：
> 
> - DISULFID, CROSSLNK bond or
> - BINDING, ACT_SITE, LIPID, METAL, SITE, MOD_RES, CARBOHYD, NON_STD site
> 
> 特殊区域包括：
> 
> TRANSMEM, INTRAMEM, COMPBIAS, REPEAT, COILED, SIGNAL, PROPEP 
> 
> （2）另外还会计算替换前后PSIC值的差值
> 
> PSIC值的计算方法类似于PSSM的计算方法，即在UniRef100数据库中利用BLAST搜索与qury序列高度同源的序列，然后将这些序列进行多序列比对，基于多序列比对结果得到profile matrix，其中行表示一个特定的氨基酸位点，列表示一种氨基酸，像这样：
> 
> <p align="center"><img src=./picture/conservative_exist-tools-SIFT-6.png width=600 /></p>
> 
> 这个矩阵的每个元素（profile score）的计算公式为：
> 
> <p align="center"><img src=./picture/nsSNPs-deleteriousness-tools-polyphen2-PSIC-1.png height=70 /></p>
> 
> 其中i表示矩阵的行号，j表示矩阵的列号，PSIC<sub>i,j</sub>表示矩阵第i行，第j列的PSIC值，P(aa=A<sub>j</sub>| posi=i) 表示在query序列第i个氨基酸位点出现A<sub>j</sub>氨基酸的概率，P(aa=A<sub>j</sub>) 表示任意位点出现A<sub>j</sub>氨基酸的概率
> 
> 若在qury序列第i个氨基酸位点，发生了A<sub>m</sub>到A<sub>n</sub>的非同义突变，则
> 
> <p align="center"><img src=./picture/nsSNPs-deleteriousness-tools-polyphen2-PSIC-2.png height=30 /></p>
> 
> 若ΔPSIC是一个比较大的正数，说明这种突变发生的概率很低，这种突变很可能是一个有害突变

**Structural features**

> 找到这个蛋白的三维结构，或者这个三维结构没有，但是有一个和你这个蛋白序列比较相类似的另外一个蛋白结构有，那你可以做一个同源建模，来预测它的三维结构
> 
> 然后基于这个三维结构计算该位点相关的结构参数 (structural parameters)，PolyPhen
2利用DSSP数据库来获得下面的结构参数：
>
> - Secondary structure (according to the DSSP nomenclature)
> - Solvent accessible surface area (absolute value in Å²)
> - Phi-psi dihedral angles 

使用的预测算法为Naive Bayes

训练集有两种：

> - **HumDiv**
> 
> compiled from all damaging alleles with known effects on the molecular function causing human Mendelian diseases, present in the UniProtKB database, together with differences between human proteins and their closely related mammalian homologs, assumed to be non-damaging
> 
> - **HumVar**
> 
> consisted of all human disease-causing mutations from UniProtKB, together with common human nsSNPs (MAF>1%) without annotated involvement in disease, which were treated as non-damaging. 

基于两种不同类型的训练集训练得到两种不同的预测模型，适用于不同类型nsSNP的预测

> - **HVAR**：should be used for diagnostics of Mendelian diseases, which requires distinguishing mutations with drastic effects from all the remaining human variation, including abundant mildly deleterious alleles.The authors recommend calling "probably damaging" if the score is between 0.909 and 1, and "possibly damaging" if the score is between 0.447 and 0.908, and "benign" is the score is between 0 and 0.446.
> - **HDIV**： be used when evaluating rare alleles at loci potentially involved in complex phenotypes, dense mapping of regions identified by genome-wide association studies, and analysis of natural selection from sequence data. The authors recommend calling "probably damaging" if the score is between 0.957 and 1, and "possibly damaging" if the score is between 0.453 and 0.956, and "benign" is the score is between 0 and 0.452.

一般突变看HVAR

<a name="cadd"><h3>3.3 CADD [<sup>目录</sup>](#content)</h3></a>

CADD —— Combined Annotation Dependent Depletion

这个工具出行的历史任务是，在此之前，大多数SNV有害性或可容忍性 （deleteriousness）的评估都是基于单个因素，而CADD对多种特征都进行了整合

> While many variant annotation and scoring tools are around, most annotations tend to exploit a single information type (e.g. conservation) and/or are restricted in scope (e.g. to missense changes). Thus, a broadly applicable metric that objectively weights and integrates diverse information is needed. Combined Annotation Dependent Depletion (CADD) is a framework that integrates multiple annotations into one metric by contrasting variants that survived natural selection with simulated mutations.

CADD独创了一种打分算法，来衡量变异位点的有害程度。对于一组变异位点，CADD 结合等位基因的**多态性**，变异的**致病性**等多个因素，构建了一套模型，对每个变异位点进行评估，并给出一个具体的得分，简称`C-Scores`。 统计模型直接给出的打分叫做`RawScore`, 这个值越高，代表该变异位点是一个有害突变的概率越高。

对于不同组的变异位点，比如对于1000G和ESP两批变异位点而言，由于各因素的差异，其模型是不同的，`RawScore`在不同模型间是无法直接比较的。所以提出了`scaled C-scores`的概念。对`RawScores`进行从大到小排序，采用`-10*log10(rank/total)`的公式计算出`scaled C-scores`。由于这个公式和`phread`的定义方式类似，所以`scaled C-scores`也叫做`PHREAD`。

在分析潜在的致病变异位点时，通常会对PHREAD进行过滤。官方推荐阈值为10,15,20都可以，但是更加推荐结合C-Scores和其他实验证据来对变异位点的致病性进行评估，而不是单纯的进行一个数值过滤。

<a name="dann"><h3>3.4 DANN [<sup>目录</sup>](#content)</h3></a>

DANN利用神经网络算法评估变异位点的有害程度

DANN软件可以看作是CADD的改进版本，改进了预测的算法，效果比CADD有所提高。

CADD软件的核心是**支持向量机SVM算法**，这个算法在机器学习领域是一个常用的算法之一，对于具有线性关系的特征具有具有较好的性能，但是对于非线性关系的特征，其性能就相对差点。DANN采用了**神经网络算法**，更容易捕获非线性关系的特征，所以效果上比CADD要好一点。

<p align="center"><img src=./picture/conservative_exist-tools-DANN-AUC.jpg width=800 /></p>

<p align="center">Bioinformatics. 2015 Mar 1; 31(5): 761–763.</p>

可以看到，两幅图中，DANN的AUC都比SVM的要大，说明DANN相比CADD确实是性能更好。

<a name="metasvm"><h3>3.5 MetaSVM [<sup>目录</sup>](#content)</h3></a>

分为三步：

> (1) perform imputation for whole-exome variants and fill out missing scores for SIFT, PolyPhen, MutationAssessor and so on.
> 
> (2) Normalize all scores to 0-1 range 
> 
> (3) use a radial SVM model to train prediction model using all available scores and some population genetics parameters, and then apply the model on whole-exome variants. 

简单来说，就是结合SIFT, PolyPhen 和 MutationAssessor 的预测分值，训练SVM模型来预测

<a name="dbnsfp"><h3>3.5 dbNSFP数据库：整合多种nsSNP预测工具的结果 [<sup>目录</sup>](#content)</h3></a>

网址：https://sites.google.com/site/jpopgen/dbNSFP

整合了20种nsSNP的功能预测算法与6种保守性评估方法得到的分值

> - 功能预测：
> 
> SIFT, Polyphen2-HDIV, Polyphen2-HVAR, LRT, MutationTaster2, MutationAssessor, FATHMM, MetaSVM, MetaLR, CADD, VEST3, PROVEAN, FATHMM-MKL coding, fitCons, DANN, GenoCanyon, Eigen coding, Eigen-PC, M-CAP, REVEL, MutPred
> 
> - 保守性评估：
> 
> PhyloP x 2, phastCons x 2, GERP++ and SiPhy

|	Score (dbtype)	|	# variants in LJB23 build hg19	|	Categorical Prediction	|
|:---|:---|:---|
|	SIFT (sift)	|	77593284	|	D: Deleterious (sift<=0.05); T: tolerated (sift>0.05)	|
|	PolyPhen 2 HDIV (pp2_hdiv)	|	72533732	|	D: Probably damaging (>=0.957), P: possibly damaging (0.453<=pp2_hdiv<=0.956); B: benign (pp2_hdiv<=0.452)	|
|	PolyPhen 2 HVar (pp2_hvar)	|	72533732	|	D: Probably damaging (>=0.909), P: possibly damaging (0.447<=pp2_hdiv<=0.909); B: benign (pp2_hdiv<=0.446)	|
|	LRT (lrt)	|	68069321	|	D: Deleterious; N: Neutral; U: Unknown	|
|	MutationTaster (mt)	|	88473874	|	A" ("disease_causing_automatic"); "D" ("disease_causing"); "N" ("polymorphism"); "P" ("polymorphism_automatic"	|
|	MutationAssessor (ma)	|	74631375	|	H: high; M: medium; L: low; N: neutral. H/M means functional and L/N means non-functional	|
|	FATHMM (fathmm)	|	70274896	|	D: Deleterious; T: Tolerated	|
|	MetaSVM (metasvm)	|	82098217	|	D: Deleterious; T: Tolerated	|
|	MetaLR (metalr)	|	82098217	|	D: Deleterious; T: Tolerated	|
|	GERP++ (gerp++)	|	89076718	|	higher scores are more deleterious	|
|	PhyloP (phylop)	|	89553090	|	higher scores are more deleterious	|
|	SiPhy (siphy)	|	88269630	|	higher scores are more deleterious	|

dbNSFP的数据已经被整合进ANNOVAR中了，目前的最新版本为`dbnsfp33a`

```
# 注释数据下载
$ annotate_variation.pl -downdb -webfrom annovar -buildver hg19 dbnsfp33a humandb/

# 同时获得所有dnNSFP的注释
$ table_annovar.pl ex1.avinput humandb/ -protocol dbnsfp33a -operation f -build hg19 -nastring .

# 获得单一dnNSFP的注释，需要先从ANNOVAR的官方服务器上下载对应某个dnNSFP的注释的文件，以SIFT为例
$ annotate_variation.pl -filter -dbtype ljb23_sift -buildver hg19 -out ex1 example/ex1.avinput humandb/
```













---

参考资料：

(1) Dong C, Wei P, Jian X, et al. Comparison and integration of deleteriousness prediction methods for nonsynonymous SNVs in whole exome sequencing studies. Hum Mol Genet. 2014;24(8):2125-37. 

(2) Predicting the effects of coding non-synonymous variants on protein function using the SIFT algorithm, Nature Protocols 4, - 1073 - 1081 (2009)

(3) Predicting Deleterious Amino Acid Substitutions, Genome Res. 2001 May; 11(5): 863.874.

(4) [PolyPhen-2官网](http://genetics.bwh.harvard.edu/pph2/dokuwiki/overview)

(5) [CADD官网](https://cadd.gs.washington.edu/)

(6) [【简书】CADD数据库简介](https://www.jianshu.com/p/9b73eddd171f)

(7) [【简书】DANN：利用神经网络算法评估变异位点的有害程度](https://www.jianshu.com/p/d6e6d52f1e3d)

(8) Quang D, Chen Y, Xie X. DANN: a deep learning approach for annotating the pathogenicity of genetic variants. Bioinformatics. 2014;31(5):761-3. 

(9) [dbNSFP 官网](https://sites.google.com/site/jpopgen/dbNSFP)

(10) [ANNOVAR document](http://annovar.openbioinformatics.org/en/latest/)


