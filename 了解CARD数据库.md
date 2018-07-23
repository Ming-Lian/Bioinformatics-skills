<a name="content">目录</a>

[了解CARD数据库](#title)
- [简介](#introduction)
	- [研究微生物耐药性要回答的几个问题](#questions-to-answer)
	- [基于Ontology的知识组织框架：ARO](#ontology-base-organization-framework)
- [从下载数据理解Ontology式的知识组织框架](#understand-ontology)
- [用于耐药基因预测的数据](#datasets-used-for-prediction)








<h1 name="title">了解CARD数据库</h1>

<a name="introduction"><h2>简介 [<sup>目录</sup>](#content)</h2></a>

<a name="questions-to-answer"><h3>研究微生物耐药性要回答的几个问题 [<sup>目录</sup>](#content)</h3></a>

1、有哪些抗性基因？

2、它们的耐药性是通过什么机制实现的？

3、它们对哪些抗生素起作用，即靶点是什么？

<a name="ontology-base-organization-framework"><h3>基于Ontology的知识组织框架：ARO [<sup>目录</sup>](#content)</h3></a>

CARD数据库( http://arpcard.mcmaster.ca )核心是ARO(Antibiotic Resistance Ontology)， ARO包含了与抗生素抗性基因，抗性机制，抗生素和靶相关的term，如图所示。2017年发表的文章中，更新了数据库的相关功能，其中也提到了其他本体论，如用于描述抗生素抗性基因预测模块和参数的MO，定义不同term之间关系类型的RO，以及描述CARD中物种和菌株的NCBITaxon。

<img src=./picture/CARD-ARO.jpeg width=900 />

<a name="understand-ontology"><h2>从下载数据理解Ontology式的知识组织框架 [<sup>目录</sup>](#content)</h2></a>

Ontology式的知识组织框架，本质上就是具有层级结构的树状图，也就是由**节点**与**连线**组成的

目前，最新版本的CARD具有四种类型的Ontology

- **ARO**：The Antibiotic Resistance Ontology that serves as the primary organizing principle 
of the CARD
- **MO**：CARD's Model Ontology which describes the different antimicrobial resistance gene 
detection models encoded in the CARD and used by the Resistance Gene Identifier
- **RO**：CARD's Relationship Ontology which describes the different relationship types between 
ontology terms
- **NCBITaxon**：A slim version of the NCBI Taxonomy used by the CARD to track source pathogenof molecular sequences

以下，以ARO为例进行说明

1、节点数据

节点说明信息被打包到 `card-ontology.tar.bz2` 中，并提供了 OBO，CSV 和 JSON 三种格式的版本，不管哪种格式，基本都只包含对一个节点的三个说明信息：Accession ID，Name，Description

像这样

```
ARO:0000000	macrolide antibiotic	Macrolides are a group of drugs (typically antibiotics) that have a large macrocyclic lactone ring of 12-16 carbons to which one or more deoxy sugars, usually cladinose and desosamine, may be attached. Macrolides bind to the 50S-subunit of bacterial ribosomes, inhibiting the synthesis of vital proteins.
ARO:0000001	fluoroquinolone antibiotic	The fluoroquinolones are a family of synthetic broad-spectrum antibiotics that are 4-quinolone-3-carboxylates. These compounds interact with topoisomerase II (DNA gyrase) to disrupt bacterial DNA replication, damage DNA, and cause cell death.
ARO:0000002	tetracycline-resistant ribosomal protection protein	A family of proteins known to bind to the 30S ribosomal subunit. This interaction prevents tetracycline and tetracycline derivatives from inhibiting ribosomal function. Thus, these proteins confer elevated resistance to tetracycline derivatives as a ribosomal protection protein.
ARO:0000003	astromicin	Astromicin is an aminoglycoside antibiotic used to treat different types of bacterial infections. Astromicin works by binding to the bacterial 30S ribosomal subunit, causing misreading of mRNA and leaving the bacterium unable to synthesize proteins vital to its growth.
```

<a name="datasets-used-for-prediction"><h2>用于耐药基因预测的数据 [<sup>目录</sup>](#content)</h2></a>

在CARD数据库网站，点击Analyze选项可进入耐药基因预测界面。耐药基因预测分析可通过选择BLAST和RGI(Resistance Gene Identifier)两种模式来实现。
- BLAST是依赖NCBI中BLAST软件，将序列与CARD参考序列进行比对，获得相关的注释信息；

- RGI是CARD数据库团队开发的基于蛋白预测抗性基因序列的软件，即通过蛋白同源和蛋白变异来预测抗性基因序列。目前，RGI仅能够分析蛋白序列，如果有基因组序列或组装后的contigs提交上来，那么首先需要使用软件Prodigal来预测开放阅读框，然后RGI分析预测得到的蛋白序列。

用于耐药基因预测的数据集被打包成 `card-data.tar.bz2`

1、FASTA:

耐药基因预测是基于序列的，那么就需要提供保存序列的FASTA文件

由于RGI的预测基于多种预测模型，包括：

- protein homolog model
- protein knockout model
- protein over-expression model
- protein variant model

而每种模型对序列的要求都有所不同，所以开发人员对不同的预测模型整理了各自对应的序列集合，而且每种模型都需要分别整理核酸序列和蛋白质序列，所以得到以下系列序列文件：

```
nucleotide_fasta_protein_homolog_model.fasta
nucleotide_fasta_protein_knockout_model.fasta
nucleotide_fasta_protein_overexpression_model.fasta
nucleotide_fasta_protein_variant_model.fasta
protein_fasta_protein_homolog_model.fasta
protein_fasta_protein_knockout_model.fasta
protein_fasta_protein_overexpression_model.fasta
protein_fasta_protein_variant_model.fasta
```

2、MODELS:

模型相关的数据保存在 `card.json` 中，包含 CARD's AMR detection models 的完整数据，包括

> - reference sequences
> - SNP mapping data
> - model parameters
> - ARO classification





参考资料：

(1) [抗性基因数据库CARD介绍](https://www.sohu.com/a/221322638_99908466)

(2) CARD官方帮助文档
