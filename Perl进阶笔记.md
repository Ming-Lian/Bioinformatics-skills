<a name="content">目录</a>

[Perl进阶笔记](#title)
- [pod文档](#pod2doc)
- [Getopt::Long](#getopt-long)
- [perl单行](#perl-command-one-line)






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

