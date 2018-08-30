<a name="content">目录</a>

[Perl进阶笔记](#title)
- [pod文档](#pod2doc)
- [Getopt::Long](#getopt-long)
- [perl单行](#perl-command-one-line)
- [使用Hash遇到的坑](#notice-when-using-hash)





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

