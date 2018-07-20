<a name="content">目录</a>

[MCScanX基因组共线性分析](#title)
- [安装](#install)
- [输入文件的准备](#prepare-input)
- [使用MCScanX分析基因组共线性区块](#run-mcscanx)


<h1 name="title">MCScanX基因组共线性分析</h1>

<a name="install"><h2>安装 [<sup>目录</sup>](#content)</h2></a>

下载MCScanX的安装包MCScanX.zip，可以在linux系统和Mac OS进行编译：

```
$ unzip MCscanX.zip
$ cd MCScanX
$ make
```

MCScanX是为32位系统开发的，若直接在64位系统上安装会出现以下报错信息：

```
genek@GenekServer:~/software/MCScanX$ make
g++ struct.cc mcscan.cc read_data.cc out_utils.cc dagchainer.cc msa.cc permutation.cc -o MCScanX
msa.cc: In function ‘void msa_main(const char*)’:
msa.cc:289:22: error: ‘chdir’ was not declared in this scope
     if (chdir(html_fn)<0)
                      ^
makefile:2: recipe for target 'mcscanx' failed
make: *** [mcscanx] Error 1
```

这个时候需要对源代码进行修改。只需要给 `msa.h`, `dissect_multiple_alignment.h`, 和 `detect_collinear_tandem_arrays.h` 这三个文件前面添加

```
#include <unistd.h>
```

就没问题了

<a name="prepare-input"><h2>输入文件的准备 [<sup>目录</sup>](#content)</h2></a>

我们需要两个物种的蛋白信息，还有对应蛋白在基因组中的位置。注意！这两个文件分别命名为`*.blast`，`*.gff`，前缀需一致

推荐从Ensembl下载蛋白质信息，它的注释信息非常全。

```
$ wget -c ftp://ftp.ensembl.org/pub/grch37/release-93/fasta/homo_sapiens/pep/Homo_sapiens.GRCh37.pep.all.fa.gz
```

查看其中的一条序列的记录：

```
>ENSP00000452494.1 pep chromosome:GRCh38:14:22449113:22449125:1 gene:ENSG00000228985.1 transcript:ENST00000448914.1 gene_biotype:TR_D_gene transcript_biotype:TR_D_gene gene_symbol:TRDD3 description:T cell receptor delta diversity 3 [Source:HGNC Symbol;Acc:HGNC:12256]
```

可以看到，该蛋白质名ENSP00000452494.1，位于14号染色体上，起始位置22449113，终止位置22449125

BLAST建库

```
$ makeblastdb -in Homo_sapiens.GRCh37.pep.all.fa -dbtype prot -parse_seqids -out Homo_sapiens.GRCh37.pep.all
```

BLAST比对

```
$ blastp -query Homo_sapiens.GRCh37.pep.all.fa -db Homo_sapiens.GRCh37.pep.all -out HumanPep.blast -evalue 1e-10 -num_threads 16 -outfmt 6 -num_alignments 5
```

按照软件说明书的要求，-evalue 1e-10是将e value卡在1e-10，-num_alignments 5是取最好的5个比对结果，-outfmt 6是输出格式为tab分隔的比对结果

蛋白质基因在基因组上的位置文件为gff文件。这里的gff文件不是平常用到的那种，需要修改一下。文件是一个tab分隔的，只有4列：

```
sp# gene starting_position ending_position
```

sp是两个字母，后面的#号表示染色体编号，这个文件可以基于FASTA文件的注释行信息，自己生成一下好了。

```
$ awk '/^>/ {gsub(">","",$1);split($3,location,":");print "hsap" location[3] "\t" $1 "\t" location[4] "\t" location[5]}' Homo_sapiens.GRCh37.pep.all.fa >HumanPep.gff
```

<a name="run-mcscanx"><h2>使用MCScanX分析基因组共线性区块 [<sup>目录</sup>](#content)</h2></a>

```
$ MCScanX HumanPep
```

程序输出的结果有两个xyz.collinearity文件和xyz.html目录

- xyz.collinearity

成对的共线性区域，如下图所示：

![](./picture/MCScanX1.png)

- xyz.html目录




参考资料：

(1) [基因课：MCScanX 安装bug](http://www.genek.tv/article/45)

(2) [biochen：使用MCScanX分析基因组共线性区块](http://blog.biochen.com/archives/878)
