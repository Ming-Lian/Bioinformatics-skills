<a name="content">目录</a>

[在Perl、Shell和Python中传参与输出帮助文档](#title)
- [Perl](#perl)
	- [Perl中getopt传参](#getopt-in-perl)
	- [Perl中输出帮助文档](#helpdoc-in-perl)
	- [实现实例](#example-perl)
- [Shell](#shell)
	- [Shell中的getopt传参](#getopt-in-shell)
	- [Shell中输出帮助文档](#helpdoc-in-shell)
	- [实现实例](#example-shell)
- [Python](#python)
	- [Python中的getopt传参](#getopt-in-python)
	- [Python中的argparse传参](#argparse-in-python)
	- [Python中输出帮助文档](#helpdoc-in-python)
	- [实现实例](#example-python)






<h1 name="title">在Perl、Shell和Python中传参与输出帮助文档</h1>

基于本人对多种编程语言的粗浅了解，不论是哪种编程语言它的参数传递方式主要分为下面两类：

> - **直接传递**（以Perl为例进行说明）
> 
> 在调用脚本时，直接传递参数，如：`./script.pl a b c`
> 
> 然后在脚本中用`@ARGV`变量获取这些参数
> 
> - **getopt方法**
> 
> 这种方法是大多数专业程序中都会用到的方法，使用的时候在参数名前加连接符，后面接上参数的实际赋值，例如：`-a 1`
> 
> 不过getopt方法的参数解析方式略显复杂，下面会在具体的语言中进行逐一说明

直接传递的传参方式的优点是编写和使用起来很方便，但缺点很明显，参数的顺序是固定的，不能随意改变，每回使用时都需要确定各个参数分别是什么，而且一般采用这种传参方式的人是不会编写帮助文档的，所以一旦忘了只能查看源代码

getopt方法的优点是，传参方式灵活，而且采用这种传参方式的程序员一般都会在程序中添加帮助文档，因此这种传参方式对用户是非常友好的，但是对于程序员来说，则意味着他或她不得不多写好几行代码——所以一个好的程序员头顶凉凉是可以理解的~

以下我们只介绍第二种传参方法

<a name="perl"><h2>Perl [<sup>目录</sup>](#content)</h2></a>

<a name="getopt-in-perl"><h3>Perl中getopt传参 [<sup>目录</sup>](#content)</h3></a>

Perl中的这个功能需要通过调用`Getopt::Long`模块实现

```
use Getopt::Long;
```

然后使用GetOptions函数承接传递的参数：

```
my ($var1,$var2,$var3,$var4); # 若使用"use strict"模式，则需要提前定义变量
GetOptions(
	"i:s"=>\$var1,
	"o:s"=>\$var2,
	"n:i"=>\$var3,
	"m:i"=>\$var4
	);
```

这样，你就可以通过以下的方式进行灵活的Perl脚本参数传递了：

```
$ perl perlscript.pl -i var1 -o var2 ...
```

<a name="helpdoc-in-perl"><h3>Perl中输出帮助文档 [<sup>目录</sup>](#content)</h3></a>

可以使用POD文档实现在Perl中输出帮助文档，想了解更多关于POD文档的知识，请点 [这里](http://www.runoob.com/perl/perl-embedded-documentation.html)

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

<a name="example-perl"><h3>实现实例 [<sup>目录</sup>](#content)</h3></a>

```
#!/usr/bin/perl
use strict;
use warnings;
use Getopt::Long;
use POSIX;

# 帮助文档
=head1 Description

	This script is used to split fasta file, which is too large with thosands of sequence

=head1 Usage

	$0 -i <input> -o <output_dir> [-n <seq_num_per_file>] [-m <output_file_num>]
	
=head1 Parameters

	-i	[str]	Input raw fasta file
	-o	[str]	Output file to which directory
	-n	[int]	Sequence number per file, alternate chose paramerter "-n" or "-m", if set "-n" and "-m" at the same time, only take "-n" parameter
	-m	[int]	Output file number (default:100)
=cut

my ($input,$output_dir,$seq_num,$file_num);
GetOptions(
	"i:s"=>\$input,
	"o:s"=>\$output_dir,
	"n:i"=>\$seq_num,
	"m:i"=>\$file_num
	);

die `pod2text $0` if ((!$input) or (!$output_dir));

.
.
.
```

<a name="shell"><h2>Shell [<sup>目录</sup>](#content)</h2></a>

<a name="getopt-in-shell"><h3>Shell中的getopt传参 [<sup>目录</sup>](#content)</h3></a>

Shell中的这个功能可以通过getopts函数实现

```
getopts [option[:]] [DESCPRITION] VARIABLE
```

> `option`：表示为某个脚本可以使用的选项
> 
> `":"`：如果某个选项（option）后面出现了冒号（":"），则表示这个选项后面可以接参数（即一段描述信息DESCPRITION）
> 
> `VARIABLE`：表示将某个选项保存在变量VARIABLE中

```
while getopts ":a:b:c:" opt
do
    case $opt in
        a)
        echo "参数a的值$OPTARG"
        ;;
        b)
        echo "参数b的值$OPTARG"
        ;;
        c)
        echo "参数c的值$OPTARG"
        ;;
        ?)
        echo "未知参数"
        exit 1;;
    esac
done
```

<a name="helpdoc-in-shell"><h3>Shell中输出帮助文档 [<sup>目录</sup>](#content)</h3></a>

在Shell中编辑一个`helpdoc( )`的函数即可实现输出帮助文档

```
helpdoc(){
    cat <<EOF
Description:

    .
	.
	.

Usage:

    $0 -a <argv1> -b <argv2> -c <argv3> ...

Option:

    .
	.
	.
EOF
}
```

将你想要打印出来的帮助信息写在`cat <<EOF`和`EOF`之间

之所以使用`EOF`来编写是因为，只是一种所见即所得的文本编辑形式（这个不好解释，请自行百度）

当你要打印帮助文档时，直接调用执行`helpdoc( )`函数即可

```
# 当没有指定参数时，即参数个数为0时，输出帮助文档并退出程序执行
if [ $# = 0 ]
then
	helpdoc()
	exit 1
fi
```

<a name="example-shell"><h3>实现实例 [<sup>目录</sup>](#content)</h3></a>

```
helpdoc(){
    cat <<EOF
Description:

    This shellscript is used to run the pipeline to call snp using GATK4
    - Data merge: merge multi-BAM files coresponding to specified strain
    - Data pre-processing: Mark Duplicates + Base (Quality Score) Recalibration
    - Call variants per-sample
    - Filter Variants: hard-filtering

Usage:

    $0 -S <strain name> -R <bwa index> -k <known-site> -i <intervals list>

Option:

    -S    strain name, if exist character "/", place "/" with "_" (Required)
    -R    the path of bwa index (Required)
    -k    known-sites variants VCF file
    -i    intervals list file,must be sorted (Required)
EOF
}

workdir="/work_dir"

# 若无指定任何参数则输出帮助文档
if [ $# = 0 ]
then
    helpdoc
    exit 1
fi

while getopts "hS:k:R:i:" opt
do
    case $opt in
        h)
            helpdoc
            exit 0
            ;;
        S)
            strain=$OPTARG
            # 检测输入的strain名是否合格：是否含有非法字符"/"
            if [[ $strain =~ "/" ]]
            then
                echo "Error in specifing strain name, if exist character \"/\", place \"/\" with \"_\""
                helpdoc
                exit 1
            fi
            if [ ! -d $workdir/SAM/$strain ]
            then
                echo "There is no such folder coresponding to $strain"
                helpdoc
                exit 1
            fi
            ;;
        R)
            index=$OPTARG
            ;;
        k)
            vcf=$OPTARG
            if [ ! -f $vcf ]
            then
                echo "No such file: $vcf"
                helpdoc
                exit 1
            fi
            ;;    
        i)
            intervals=$OPTARG
            if [ ! -f $bed ]
            then
                echo "No such file: $intervals"
                helpdoc
                exit 1
            fi
            ;;
        ?)
            echo "Unknown option: $opt"
            helpdoc
            exit 1
            ;;
    esac
done

.
.
.
```

<a name="python"><h2>Python [<sup>目录</sup>](#content)</h2></a>

<a name="getopt-in-python"><h3>Python中的getopt传参 [<sup>目录</sup>](#content)</h3></a>

Python中的这种功能需要通过`getopt`模块实现

```
import getopt
```

Python脚本获得成对的参数名和参数值后，会分别把它们保存在一个字典变量中，参数名为key，参数值为value

```
opts,args = getopt.getopt(argv,"hi:o:t:n:",["ifile=","ofile=","time="])
```

getopt函数的使用说明：

> `argv`：使用`argv`过滤掉第一个参数（它是执行脚本的名字，不应算作参数的一部分）
>  
> `"hi:o:t:n:"`：使用短格式分析串，当一个选项只是表示开关状态时，即后面不带附加参数时，在分析串中写入选项字符。当选项后面是带一个附加参数时，在分析串中写入选项字符同时后面加一个":" 号
> 
> `["ifile=","ofile=","time="]`： 使用长格式分析串列表，长格式串也可以有开关状态，即后面不跟"=" 号。如果跟一个等号则表示后面还应有一个参数 

然后通过条件判断的方法对参数进行解析：

```
for opt,arg in opts:
	if opt in ("-h","--help"):
		print(helpdoc)
		sys.exit()
	elif opt in ("-i","--ifile"):
		infile = arg
	elif opt in ("-t","--time"):
		sleep_time = int(arg)
	.
	.
	.
```

<a name="argparse-in-python"><h3>Python中的argparse传参 [<sup>目录</sup>](#content)</h3></a>

该示例例来自**黄树嘉**大佬的github项目 [cmdbtools](https://github.com/ShujiaHuang/cmdbtools/blob/master/cmdbtools/cmdbtools.py)

```
import argparse    //导入命令行解析的库文件

// 为了别人执行代码的时候用--help看出来怎么使用这些代码
argparser = argparse.ArgumentParser(description='Manage authentication for CMDB API and do querying from command line.') 
// 添加子命令
commands = argparser.add_subparsers(dest='command', title='Commands')

// 分别定义各个子命令，并未各个子命令设置参数

// 1. 定义子命令login
login_command = commands.add_parser('login', help='Authorize access to CMDB API.')
login_command.add_argument('-k', '--token', type=str, required=True, dest='token',help='CMDB API access key(Token).') // 给子命令添加'-k'参数

// 2. 定义子命令logout
logout_command = commands.add_parser('logout', help='Logout CMDB.')

// 3. 定义子命令token
token_command = commands.add_parser('print-access-token', help='Display access token for CMDB API.')

// 4. 定义子命令annotate
annotate_command = commands.add_parser('annotate', help='Annotate input VCF.',
		description='Input VCF file. Multi-allelic variant records in input VCF must be split into multiple bi-allelic variant records.')
annotate_command.add_argument('-i', '--vcffile', metavar='VCF_FILE', type=str, required=True, dest='in_vcffile',help='input VCF file.')

// 5. 定义子命令query_variant
query_variant_command = commands.add_parser('query-variant',
											help='Query variant by variant identifier or by chromosome name and chromosomal position.',
											description='Query variant by identifier chromosome name and chromosomal position.')
query_variant_command.add_argument('-c', '--chromosome', metavar='name', type=str, dest='chromosome',help='Chromosome name.', default=None)
query_variant_command.add_argument('-p', '--position', metavar='genome-position', type=int, dest='position',help='Genome position.', default=None)
query_variant_command.add_argument('-l', '--positions', metavar='File-contain-a-list-of-genome-positions',
									type=str, dest='positions',
									help='Genome positions list in a file. One for each line. You can input single '
										'position by -c and -p or using -l for multiple poisitions in a single file, '
										'could be .gz file',
									default=None)
```

输出的主命令的帮助文档如下：

```
$ cmdbtools --help
usage: cmdbtools [-h]
                {login,logout,print-access-token,annotate,query-variant} ...

Manage authentication for CMDB API and do querying from command line.

optional arguments:
 -h, --help            show this help message and exit

Commands:
 {login,logout,print-access-token,annotate,query-variant}
   login               Authorize access to CMDB API.
   logout              Logout CMDB.
   print-access-token  Display access token for CMDB API.
   annotate            Annotate input VCF.
   query-variant       Query variant by variant identifier or by chromosome
                       name and chromosomal position.
```

<a name="helpdoc-in-python"><h3>Python中输出帮助文档 [<sup>目录</sup>](#content)</h3></a>

在Python中创建一个字符串变量`helpdoc`即可实现输出帮助文档

```
helpdoc = '''
Description

	...

Usage

	python pyscript.py -i/--ifile <input file> -o/--ofile <output file> -t/--time <int> ...

Parameters

	-h/--help
		Print helpdoc
	-i/--ifile
		Input file, including only one column with sampleId
	-o/--ofile
		Output file, including two columns, the 1st column is sampleId, the 2nd column is attribute information
	-t/--time
		Time for interval (seconds, default 5s)
	...
'''
```

在需要时将这个变量打印出来即可：

```
try:
	opts,args = getopt.getopt(argv,"hi:o:t:n:",["ifile=","ofile=","time="])
	if len(opts) == 0:
		print("Options Error!\n\n"+helpdoc)
		sys.exit(2)
except getopt.GetoptError:
	print("Options Error!\n\n"+helpdoc)
	sys.exit(2)
```

<a name="example-python"><h3>实现实例 [<sup>目录</sup>](#content)</h3></a>

```
import getopt

.
.
.

if __name__ == '__main__':
	...
	helpdoc = '''
Description

    This script is used to grab SRA sample attributes information based on SampleId

Usage

    python webspider_ncbiBiosample.py -i/--ifile <input file> -o/--ofile <output file> -t/--time <int> -n/--requests-number <int>

Parameters

    -h/--help
        Print helpdoc
    -i/--ifile
        Input file, including only one column with sampleId
    -o/--ofile
        Output file, including two columns, the 1st column is sampleId, the 2nd column is attribute information
    -t/--time
        Time for interval (seconds, default 5s)
    -n/--requests-number
        Setting the requests number between interval (default 10)
'''

    # 获取命令行参数
    try:
        opts,args = getopt.getopt(argv,"hi:o:t:n:",["ifile=","ofile=","time="])
        if len(opts) == 0:
            print("Options Error!\n\n"+helpdoc)
            sys.exit(2)
    except getopt.GetoptError:
        print("Options Error!\n\n"+helpdoc)
        sys.exit(2)

    # 设置参数
    for opt,arg in opts:
        if opt in ("-h","--help"):
            print(helpdoc)
            sys.exit()
        elif opt in ("-i","--ifile"):
            infile = arg
        elif opt in ("-o","--ofile"):
            outfile = arg
            # 若指定的输出文件已经存在，让用户决定覆盖该文件，还是直接退出程序
            if os.path.exists(outfile):
                keyin = input("The output file you specified exists, rewrite it?([y]/n: ")
                if keyin in ("y","Y",""):
                    os.remove(outfile)
                elif keyin in ("n","N"):
                    print("The output file existed!\n")
                    sys.exit(2)
                else:
                    print("Input error!\n")
                    sys.exit(2)
        elif opt in ("-t","--time"):
            sleep_time = int(arg)
        elif opt in ("-n","--requests-number"):
            requestNum = int(arg)

```


---

参考资料：

(1) [本人github笔记：Perl进阶笔记](https://github.com/Ming-Lian/Bioinformatics-skills/blob/master/Perl%E8%BF%9B%E9%98%B6%E7%AC%94%E8%AE%B0.md)

(2) [本人github笔记：实用小脚本](https://github.com/Ming-Lian/Bioinformatics-skills/blob/master/%E5%AE%9E%E7%94%A8%E5%B0%8F%E8%84%9A%E6%9C%AC.md)

(3) [本人github笔记：Linux (Raspbian) 操作进阶——Shell编程](https://github.com/Ming-Lian/Hello-RaspberryPi/blob/master/Linux%E6%93%8D%E4%BD%9C%E8%BF%9B%E9%98%B6.md#shell-programing)

(4) [Python 命令行参数和getopt模块详解](https://www.cnblogs.com/kex1n/p/5975722.html)
