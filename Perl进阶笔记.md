<a name="content">目录</a>

[Perl进阶笔记](#title)
- [pod文档](#pod2doc)
- [Getopt::Long](#getopt-long)
- [perl单行](#perl-command-one-line)
- [使用Hash遇到的坑](#notice-when-using-hash)
- [安装Perl模块](#install-perl-module)
	- [使用CPAN模块自动安装](#use-cpan-module)
	- [手工安装](#install-manually)
	- [非root用户的另一个解决方案](#install-way-for-unroot)
- [循环匹配](#loop-matching)



<h1 name="title">Perl进阶笔记</h1>

<a name="pod2doc"><h2>pod文档 [<sup>目录</sup>](#content)</h2></a>

使用pod文档可以实现程序usage说明

```
=head1 part1

	doc in part1

=head2 part2

	doc in part2

.
.
.

=cut	# pod文档结束的标志
```

注意：每个=标签上下必须隔一行，否则就会错误解析。

用`pod2doc $0`可以将程序中的文档打印出来，不过一般用在程序内部，当程序参数设定错误时打印pod文档：

```
die `pod2doc $0` if (...);
```

<a name="getopt-long"><h2>Getopt::Long [<sup>目录</sup>](#content)</h2></a>

首先需要在脚本开头加上对该模块的引用

```
use Getopt::Long;
```

使用GetOptions函数承接传递的参数：

```
my ($var1,$var2,$var3,$var4); # 若使用"use strict"模式，则需要提前定义变量
GetOptions(
	"i:s"=>\$var1,
	"o:s"=>\$var2,
	"n:i"=>\$var3,
	"m:i"=>\$var4
	);
```

<a name="perl-command-one-line"><h2>perl单行 [<sup>目录</sup>](#content)</h2></a>

执行`perl -h`可以查看perl的所有参数使用说明，使用perl单行需要使用到`-e`参数

```
$ perl -e 'print "hello world!\n"'
```

perl单行常用的场景为进行文本的逐行读取并操作，即隐式地开启`while(<>)`，需要使用`-n`参数

```
perl -ne
	`BEGIN{}
	...
	END{}'
filename
```

BEGIN 和 END 区块根据需要进行添加

若需要在逐行读取的同时，自动将行中的元素打散 (split)，默认以`\s`（空字符，即空格或制表符）作为分割符，则需要使用`-a`参数，相当于在执行了`@F = split $_`，打散后的元素会保存在`@F`中

<a name="notice-when-using-hash"><h2>使用Hash遇到的坑 [<sup>目录</sup>](#content)</h2></a>

在编写perl脚本的过程中，我们常常会将读入文件一行中的某两项（一行可能有多列，列与列之间用制表符`\t`隔开）作为相对应的两项，分别作为Hash的键（key)和值（value)，由于Hash这种数据结构要求key是唯一，而value可以重复，因此一般将唯一的那一项作为key，另一项作为value

但是，由于字符串首末端空字符的存在会导致一个意想不到的情况：

创建了两个Hash，让它们的key是一一对应的，而各自存储的value不同，当时在某些key字符串首末端混入了空字符，例如Hash1有一个key为`"KEY"`，Hash2有一个key为`"KEY "`，它们本来应该是一样的，但是由于空字符的存在，它们现在不一样了

这时候的解决方法是在构建Hash之前，不论实际的字符串的首末端有没有空字符串，都尝试将这些空字符串去掉：

```
# 假设读入的文件只有用制表符隔开的两列
while(<IN>){
	chomp;
	@recorder = split /\t/;
	$recorder[0] =~ s/(^\s+)|(\s+$)//g;	# 去除开头和末尾的空字符串
	$recorder[1] =~ s/(^\s+)|(\s+$)//g;	# 去除开头和末尾的空字符串
	$hash{$recorder[0]} = $recorder[1];
}
```

<a name="install-perl-module"><h2>安装Perl模块 [<sup>目录</sup>](#content)</h2></a>

查看perl模块的安装目录，主要就是@INC这个默认变量 

```
perl -e '{print "$_\n" foreach @INC}'
```

查看已安装的Perl模块：

```
# 查看系统中安装的Perl模块
find  `perl -e 'print "@INC"'` -name '*.pm'
# 查看当前环境下所有的模块（一般为用户自己安装的）
instmodsh
```

查询单个perl模块的安装路径：

```
perldoc -l Getopt::Long
```

查看安装的perl模块的版本号

```
perl -MGetopt::Long -e 'print Getopt::Long->VERSION. "\n"'
```

装Perl模块有两种方法


- 自动安装 (使用CPAN模块自动完成下载、编译、安装的全过程)
- 手工安装 (去CPAN网站下载所需要的模块，手工编译、安装)

<a name="use-cpan-module"><h3>使用CPAN模块自动安装 [<sup>目录</sup>](#content)</h3></a>

安装前需要先联上网，**有无root权限均可**

```
$ perl -MCPAN -e shell
```

```
cpan>help
cpan>m
cpan>install Net::Server
cpan>quit
```

> - 查询：cpan[1]> d /模块名字或者部分名字/
> 
> 查询结果中会给出所有含有模块名字或者部分名字的模块，选择您所需要的模块进行下载
> 
> - 下载安装：cpan[1]> install 模块名字
> 
> 同时会自动安装很多依赖的模块，非常方便。

<a name="install-manually"><h3>手工安装 [<sup>目录</sup>](#content)</h3></a>

> 一般情况下不推荐这种安装方式，但是总是会有**迫不得已**的时候，而且尝试这种方式，能加深对perl模块的理解。

比如从 CPAN下载了Net-Server模块0.97版的压缩文件Net-Server-0.97.tar.gz，假设放在/usr/local/src/下。

```
cd /usr/local/src
tar xvzf Net-Server-0.97.tar.gz
cd Net-Server-0.97
perl Makefile.PL
make test
```



如果测试结果报告`all test ok`，你就可以放心地安装编译好的模块了。

<a name="install-way-for-unroot"><h3>非root用户的另一个解决方案 [<sup>目录</sup>](#content)</h3></a>

手动下载local::lib, 这个perl模块，然后自己安装在指定目录，也是能解决模块的问题！

下载之后解压，进入：

```
perl Makefile.PL --bootstrap=~/.perl ##这里设置你想把模块放置的目录
make test && make install
echo 'eval $(perl -I$HOME/.perl/lib/perl5 -Mlocal::lib=$HOME/.perl)' >> ~/.bashrc ##目录与前面要一致
```

等待几个小时即可！！！

添加好环境变量之后，就可以用

```
perl -MCPAN -Mlocal::lib -e 'CPAN::install(LWP)'
```

或有更简单的写法：

```
cpanm --local-lib=~/perl5 local::lib && eval $(perl -I ~/perl5/lib/perl5/ -Mlocal::lib)
```

<a name="loop-matching"><h2>循环匹配 [<sup>目录</sup>](#content)</h2></a>

在李恒的github博客 [On the definition of sequence identity](http://lh3.github.io/2018/11/25/on-the-definition-of-sequence-identity) 当中看到这样一个Perl单行代码：

```
$ perl -ane 'if(/NM:i:(\d+)/){$n=$1;$l=0;$l+=$1 while/(\d+)[MID]/g;print(($l-$n)/$l,"\n")}'
```

这行代码的目的是为了计算SAM文件中每条记录BLAST identity

> BLAST identity是根据你比对到的碱基除去比对所涉及到的columns数目，换句话来说就是比对涉及到所有的碱基数目
> 
> 例如这样的双序列比对：
> 
> ```
> Ref+:  1 CCAGTGTGGCCGATaCCCcagGTtgGC-ACGCATCGTTGCCTTGGTAAGC 49
>          |||||||||||||| |||   ||  || ||||||||||||||||||||||
> Qry+:  1 CCAGTGTGGCCGATgCCC---GT--GCtACGCATCGTTGCCTTGGTAAGC 45
> ```
> 
> 它们的BLAST identity就是`43/50=86%`

那么要计算SAM文件中每条reads的BLAST identity，总长可以通过叠加CIGAR中对应的`M/I/D`的数目得到，比对到的碱基数目等于总长减去NMtag（比对不上的碱基位置的标记）

<p align="center"><img src=/picture/Advanced-Perl-SAM-NMtag.png width=600 /></p>

李恒的这行代码中有一部分一开始没有读懂，就是下图红框中的那部分：

<p align="center"><img src=/picture/Advanced-Perl-lh-code.png width=600 /></p>

其实这是一种简写方式，正规完整且更容易读懂的形式可以写成下面这样：

```
# 这里为了更好看，添加了适当的换行和缩进

$ perl -ane \
'if(/NM:i:(\d+)/){
	$n=$1;
	$l=0;
	while(/(\d+)[MID]/g){
		$l+=$1;
	}
	print(($l-$n)/$l,"\n");
}
'
```

`while(/(\d+)[MID]/g)`中的正则表达式`/(\d+)[MID]/g`，引起了我极大的好奇：它是在正则表达式后面添加了一个`g`字符，即开启了全局匹配，又由于是在`while( )`中进行的正则匹配，等于是开启了循环匹配，即对于CIGAR字符串`18M3D22M`，正则表达式`/(\d+)[MID]/g`，先会匹配上`18M`，然后会匹配上`3D`，最后匹配上`22M`

很有意思的用法

---

参考资料：

(1) [生信菜鸟团：perl模块安装大全](http://www.bio-info-trainee.com/2451.html)

(2) [Heng Li's blog: On the definition of sequence identity](#http://lh3.github.io/2018/11/25/on-the-definition-of-sequence-identity)

(3) [【简书】生信杂谈：怎样定义sequences比对的相似度？](https://www.jianshu.com/p/9539f44ecf61)
