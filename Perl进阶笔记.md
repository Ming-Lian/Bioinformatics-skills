<a name="content">目录</a>

[Perl进阶笔记](#title)
- [1. pod文档](#pod2doc)
- [2. 用Getopt::Long模块传参](#getopt-long)
- [3. perl单行](#perl-command-one-line)
	- [3.1. 向perl单行传参](#passing-parameters-in-perl-command-one-line)
	- [3.2. perl单行里的坑](#trap-in-perl-command-one-line)
- [4. 使用Hash遇到的坑](#notice-when-using-hash)
- [5. Hash中的排序操作](#sort-for-hash)
- [6. 安装Perl模块](#install-perl-module)
	- [6.1. 使用CPAN模块自动安装](#use-cpan-module)
	- [6.2. 手工安装](#install-manually)
	- [6.3. 非root用户的另一个解决方案](#install-way-for-unroot)
- [7. 循环匹配](#loop-matching)
- [8. chomp带来空行匹配的失败](#chomp-result-in-failure-blank-line)
- [9. perl中的grep函数](#grep-in-perl)
- [10. 计算数组的长度：千万不要用length](#calc-array-length)
- [11. 在哈希中设置键值为数组](#hash-key-point-to-arrary)
- [12. 文件句柄引用](#filehandle-quote)





<h1 name="title">Perl进阶笔记</h1>

<a name="pod2doc"><h2>1. pod文档 [<sup>目录</sup>](#content)</h2></a>

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

<a name="getopt-long"><h2>2. 用Getopt::Long模块传参 [<sup>目录</sup>](#content)</h2></a>

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

<a name="perl-command-one-line"><h2>3. perl单行 [<sup>目录</sup>](#content)</h2></a>

执行`perl -h`可以查看perl的所有参数使用说明，使用perl单行需要使用到`-e`参数

```
$ perl -e 'print "hello world!\n"'
```

perl单行常用的场景为进行文本的逐行读取并操作，即隐式地开启`while(<>)`，需要使用`-n`参数

```
perl -ne
	'BEGIN{}
	...
	END{}'
filename
```

BEGIN 和 END 区块根据需要进行添加

若需要在逐行读取的同时，自动将行中的元素打散 (split)，默认以`\s`（空字符，即空格或制表符）作为分割符，则需要使用`-a`参数，相当于在执行了`@F = split $_`，打散后的元素会保存在`@F`中

<a name="passing-parameters-in-perl-command-one-line"><h3>3.1. 向perl单行传参 [<sup>目录</sup>](#content)</h3></a>








<a name="trap-in-perl-command-one-line"><h3>3.2. perl单行里的坑 [<sup>目录</sup>](#content)</h3></a>

在执行以下形式的perl单行时，总是报错

```perl
$ perl -e \
'...;
open I,"<$infile" or die "Can't open $infile: $!";
open O,">$outfile" or die "Can't make file $outfile: $!";
...;' \
<infile> <outfile>
```

报错信息总是显示：`No such file or directory`

反复地检查，愣是没找出脚本错在哪

然后将这个单行保存成perl脚本再运行——不报错又能正常运行了，什么鬼？难以理解

后面终于发现问题出在哪了：`Can't`的`'`字符和perl单行命令区块起始的那个`'`字符配对了，命令区块提前结束了

<a name="notice-when-using-hash"><h2>4. 使用Hash遇到的坑 [<sup>目录</sup>](#content)</h2></a>

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

<a name="sort-for-hash"><h2>5. Hash中的排序操作 [<sup>目录</sup>](#content)</h2></a>

对key进行排序

其基本的语法结构为：

```
sort <排序规则> <排序对象>
```

若要对keys进行排序则排序对象就是keys，所以最后一项要写成`keys %hash`

```
# 按value排序
## 对hash的keys按hash value排序（按ASCII码排序）
sort { $hash{$a} cmp $hash{$b} } keys %hash
## 对hash的keys按hash value排序（按数字大小排序）
sort { $hash{$a} <=> $hash{$b} } keys %hash

# 按key排序
# 对hash的keys按hash key排序
sort {$a<=>$b} keys %hash
```

<a name="install-perl-module"><h2>6. 安装Perl模块 [<sup>目录</sup>](#content)</h2></a>

查看perl模块的安装目录，主要就是@INC这个默认变量 

```
perl -e '{print "$_\n" foreach @INC}'
```

若要临时添加perl模块的安装目录，则在perl脚本中shebang（`#!/usr/bin/perl`）后紧接着添加`push(@INC,"...");`命令，若是在perl单行中，则写成`BEGIN{push(@INC,"...");}`

若是要永久添加perl模块的安装目录，则修改PERL5LIB环境变量即可：

```
export PERL5LIB=/PATH/TO/LIB
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

<a name="use-cpan-module"><h3>6.1. 使用CPAN模块自动安装 [<sup>目录</sup>](#content)</h3></a>

首先你得已经安装了CPAN，若没有执行以下命令：

```
# yum install perl-CPAN
```

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

<a name="install-manually"><h3>6.2. 手工安装 [<sup>目录</sup>](#content)</h3></a>

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

<a name="install-way-for-unroot"><h3>6.3. 非root用户的另一个解决方案 [<sup>目录</sup>](#content)</h3></a>

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

<a name="loop-matching"><h2>7. 循环匹配 [<sup>目录</sup>](#content)</h2></a>

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

<p align="center"><img src=./picture/Advanced-Perl-SAM-NMtag.png width=600 /></p>

李恒的这行代码中有一部分一开始没有读懂，就是下图红框中的那部分：

<p align="center"><img src=./picture/Advanced-Perl-lh-code.png width=600 /></p>

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

<a name="chomp-result-in-failure-blank-line"><h2>8. chomp带来空行匹配的失败 [<sup>目录</sup>](#content)</h2></a>

我常用正则表达式`/^\s+$/`来进行文件中空行的匹配

在用perl单行进行文本处理，我喜欢用一个固定的格式：

```
$ perl -ne 'chomp;...' <input>
```

chomp在这里的作用是在每读入一行后，去除末尾的换行符

此时如果你要匹配的空行是`/^\n/`形式的，即这行只有一个换行符，被chomp处理过之后这一行就变成了`/^$/`形式的空字符的行，此时如果再用正则表达式`/^\s+$/`进行匹配，就会匹配失败

**那么遇到这种情况应该怎么匹配呢？**

使用`/^\s?$/`即可，元字符`?`的作用是匹配前面的字符0到多次

<a name="grep-in-perl"><h2>9. perl中的grep函数 [<sup>目录</sup>](#content)</h2></a>

grep有2种表达方式：

```
grep BLOCK LIST
grep EXPR, LIST
```

> `BLOCK`表示一个code块，通常用{ }表示；
>
> `EXPR`表示一个表达式，通常是正则表达式。原文说EXPR可是任何东西，包括一个或多个变量，操作符，文字，函数，或子函数调用；
>
> `LIST`是要匹配的列表

grep的工作原理：

> grep对列表里的每个元素进行BLOCK或EXPR匹配，它遍历列表，并临时设置元素为`$_`
>
> 在列表上下文里，grep返回匹配命中的所有元素，结果也是个列表。在标量上下文里，grep返回匹配命中的元素个数。

实例一：

```perl
open FILE "<myfile" or die "Can't open myfile: $!";
print grep /terrorism|nuclear/i <FILE>;
```

打开一个文件myfile，然后查找包含terrorism或nuclear的行。`<FILE>`返回一个列表，它包含了文件的完整内容。可能你已发现，如果文件很大的话，这种方式很耗费内存，因为文件的所有内容都拷贝到内存里了

实例二：

```perl
foreach my $infile (grep { !/^\./ && -f "$indir/$_" } readdir(DIR)){
	...
}
```

`readdir(DIR)`读入指点文件夹句柄`DIR`下的所有文件（包括以`.`开头的隐藏文件）的文件名，构成一个文件名列表 (list)，然后每一次读入一个文件名保存到临时变量`$_`中，传递给grep处理，先用`!/^\./`判断文件名是否以`.`开头，保证该文件不是隐藏文件，接着，通过`-f "$indir/$_"`判断该文件是否存在

实例三：找出某个元素在数组中的位置（索引值）

```perl
($index) = grep {$A[$_] -eq $Str} 0..$#A ;
```

注意：`$index`两侧的括号很重要，因为grep的返回值会受到承接变量的上下文环境的影响，若承接变量是一个不带括号的标量，则以为着这是一个标量上下文，则grep的返回值为符合条件的匹配的元素的个数，若承接变量是一个数组或者一个带括号的标量，则说明这是一个列表上下文，则grep返回值是匹配匹配的元素值的列表

<a name="calc-array-length"><h2>10. 计算数组的长度：千万不要用length [<sup>目录</sup>](#content)</h2></a>

在计算perl数组的长度时，本能地就想到了`length`函数，结果输出的结果跟预期的不同，当时我就原地爆炸了，what？怎么会这样？

脚本要实现的功能很简单，就是把数组中的元素按顺序输出，且元素之间用制表符隔开

```perl
for($i=0;$i<length(@F);$i++){
	print "$F[$i]\t";
}
```

当然还有一个更简单的实现方法：

```perl
print join("\t",@F);
```

先不管哪种方法更简便，就说第一种用法吧——它的预期输出应该是：数组中的元素按顺序输出，且元素之间用制表符隔开，也就是说如果我的数字中有10个元素，那么应该输出10个制表符分隔的元素，结果却输出了前2个元素

后面才知道用`length`函数不能得到数组的长度，它是用来计算字符串的长度的，那么，将它错误地用到数组长度的计算上，会出现什么结果呢？为什么上面的例子中，输出的是前2个元素呢？

> 因为`length`是用来计算字符串的长度的，也就是说它操作的对象是一个标量，当你给它一个数组时，即 `length(@F)` ，它会将这个数组标量化，相当于执行了`scalar @F`，得到了这个数组的长度为10，然后再对这个10执行`length`，即 `length(10)` ，最后得到的当然是2了

<a name="hash-key-point-to-arrary"><h2>11. 在哈希中设置键值为数组 [<sup>目录</sup>](#content)</h2></a>

有下面形式的原始数据：

```
Chicago, USA 
Frankfurt, Germany 
Berlin, Germany 
Washington, USA 
Helsinki, Finland 
New York, USA 
```

在perl中，一种比较合适的保持这些数据的数据结构为哈希：

```
Finland: Helsinki
Germany: Berlin, Frankfurt
USA:  Chicago, New York, Washington
```

也就是说：将国家的名称设置为一个哈希结构的键，和国家名称对应的健值是这些国家内的城市的一个数组

但是，这种方法能否在Perl中实现呢？

很遗憾，在Perl 4中，哈希的值不能是列表，它们只能是字符串。所以你必须把所有的城市名合并成一个字符串。当需要输出时，你再把这个字符串分拆成一个列表，然后对列表排序，最后将列表中的数据转成字符串输出

那怎么实现呢？

可以使用Perl中的“引用”功能来实现

```perl
$H{$a} = \@A; # 将键名为为$a的键指向数组@A的首地址，即它的键值为数组@A的首地址，虽然键值还是一个标量，但是实际上类似实现了让键值为数组的功能

# 获得了这个数组的首地址后就可以对这个数组进行操作了，比如：
push(@{$H{$a}}, $b); # @{$H{$a}}实现对这个键值里保持的数组首地址的解引用，则push函数的第一个参数就是对应的数组本身
```

<a name="filehandle-quote"><h2>12. 文件句柄引用 [<sup>目录</sup>](#content)</h2></a>

perl中使用`open`命令来创建一个文件句柄，`open`命令的基本语法结构为：

```
open FILEHANDLE,MODE,LIST
```

> - `FILEHANDLE`：创建的文件句柄对象，习惯上使用大写字母的句柄方式（即所谓的裸字文件句柄，bareword filehandle）：`open H,"<$file"`；其实也可以考虑使用变量文件句柄的形式：`open $fh,"<$file`
>
> - `MODE`：文件句柄的读写模式，`'<'`为只读模式（默认模式），`'>'`为写入模式，`'>>'`为追加模式
>
> - `LIST`：文件路径

裸字文件句柄和变量文件句柄用法是完全一致的，能用裸字文件句柄的地方都可以替换为变量文件句柄：

```perl
while(<H>){...}

while(<$fh>){...}
```

只是需要注意的是，使用变量文件句柄的方式，在`say/print`输出的时候，指定文件句柄时需要使用大括号包围，以免产生歧义：

```perl
print {$data_fh} "your output content";
```






---

参考资料：

(1) [生信菜鸟团：perl模块安装大全](http://www.bio-info-trainee.com/2451.html)

(2) [Heng Li's blog: On the definition of sequence identity](#http://lh3.github.io/2018/11/25/on-the-definition-of-sequence-identity)

(3) [【简书】生信杂谈：怎样定义sequences比对的相似度？](https://www.jianshu.com/p/9539f44ecf61)

(4) [perl中grep的详细用法](https://www.cnblogs.com/techhacker/p/3741832.html)

(5) 陈卫华师兄的perl脚本

(6) [perl---(数组和哈希)引用](https://blog.csdn.net/zll01/article/details/4506206)

