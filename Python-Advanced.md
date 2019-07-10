<a name="content">目录</a>

[Python进阶笔记](#title)
- [1. argparse的使用](#usage-of-argparse)
- [2. subprocess的使用](#usage-of-subprocess)



<h1 name="title">Python进阶笔记</h1>

<a name="usage-of-argparse"><h2>1. argparse的使用 [<sup>目录</sup>](#content)</h2></a>

该示例例来自**黄树嘉**大佬的github项目 [cmdbtools](https://github.com/ShujiaHuang/cmdbtools/blob/master/cmdbtools/cmdbtools.py)

```python
import argparse    # 导入命令行解析的库文件

# 为了别人执行代码的时候用--help看出来怎么使用这些代码
argparser = argparse.ArgumentParser(description='Manage authentication for CMDB API and do querying from command line.') 
# 添加子命令
commands = argparser.add_subparsers(dest='command', title='Commands')

# 分别定义各个子命令，并未各个子命令设置参数

# 1. 定义子命令login
login_command = commands.add_parser('login', help='Authorize access to CMDB API.')
login_command.add_argument('-k', '--token', type=str, required=True, dest='token',help='CMDB API access key(Token).') # 给子命令添加'-k'参数

# 2. 定义子命令logout
logout_command = commands.add_parser('logout', help='Logout CMDB.')

# 3. 定义子命令token
token_command = commands.add_parser('print-access-token', help='Display access token for CMDB API.')

# 4. 定义子命令annotate
annotate_command = commands.add_parser('annotate', help='Annotate input VCF.',
		description='Input VCF file. Multi-allelic variant records in input VCF must be split into multiple bi-allelic variant records.')
annotate_command.add_argument('-i', '--vcffile', metavar='VCF_FILE', type=str, required=True, dest='in_vcffile',help='input VCF file.')

# 5. 定义子命令query_variant
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
argparser.parse_args()
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

下面换一种思路来学习`argparse`：给出要实现的帮助文档，如何通过编写代码实现

帮助文档以上面的为例：

<p align="center"><img src=./picture/Helpdoc-in-Perl-Shell-and-Python.png /></p>

上面用红色框将帮助文档的内容分成了三个在实现过程中相对独立的部分

（1）脚本基本语法和描述信息

在创建好`argparse`对象之后，就设定好了这一部分的信息

```
argparser = argparse.AugumentParser(description='Manage authentication for CMDB API and do querying from command line.')
.
.
.
argparser.parse_args()
```

描述信息在`description`参数中添加

从脚本基本语法可以看出：

```
usage: cmdbtools [-h]
                {login,logout,print-access-token,annotate,query-variant} ...
```

在主程序的parser下方还创建了附属的子parser，可以通过`argparser.add_subparsers`创建附属的子parsers对象（例子中将子parsers对象命名为commands）：

```
commands = argparser.add_subparsers(dest='command', title='Commands')
```

创建好子parsers对象之后，还需要创建其下所附属的具体的每一个子parser，使用`command.add_parser`：

```python
login_command = commands.add_parser('login', help='Authorize access to CMDB API.')

logout_command = commands.add_parser('logout', help='Logout CMDB.')

...
```

（2）默认的可选参数`-h/--help`

这个默认的可选参数不需要单独创建，在创建`argparse`对象后就会默认内置了

（3）添加具体的参数

例如，子parser `login`下的帮助文档：

```
usage: cmdbtools login [-h] -k TOKEN

optional arguments:
  -h, --help            show this help message and exit
  -k TOKEN, --token TOKEN
                        CMDB API access key(Token).
```

则可以通过下面的代码添加对应的参数解析：

```python
login_command.add_argument('-k', '--token', type=str, required=true, dest='token', help='CMDB API access key(Token).')
```

<a name="usage-of-subprocess"><h2>2. subprocess的使用 [<sup>目录</sup>](#content)</h2></a>

在Python中我们可以通过`os.system`来以控制台的形式运行程序，但当涉及到需要进行进程间通信时，就需要用到subprocess模块

subprocess模块中只定义了一个类: **Popen**，可以使用Popen来创建进程，并与进程进行复杂的交互。它的构造函数如下：

```python
subprocess.Popen(
    args,
    bufsize=0,
    executable=None, 
    stdin=None, 
    stdout=None, 
    stderr=None, 
    preexec_fn=None, 
    close_fds=False, 
    shell=False, 
    cwd=None, 
    env=None, 
    universal_newlines=False, 
    startupinfo=None, 
    creationflags=0)
```

参数说明：

> - `args`：可以是字符串或者序列类型（如：list，元组），用于指定进程的可执行文件及其参数
>
>    如果是序列类型，第一个元素通常是可执行文件的路径。我们也可以显式的使用executeable参数来指定可执行文件的路径
>
> - `stdin, stdout, stderr`：分别表示程序的标准输入、输出、错误句柄。他们可以是PIPE，文件描述符或文件对象，也可以设置为None，表示从父进程继承
>
> - `shell`：（默认为 False）指定是否使用 shell 执行程序
>
> - `env`：用于指定子进程的环境变量，是字典类型。如果`env = None`，子进程的环境变量将从父进程中继承

Popen的方法：

> - `Popen.poll( )`
>
>    用于检查子进程是否已经结束。设置并返回returncode属性
>
> - `Popen.wait( )`
>
>    等待子进程结束。设置并返回returncode属性
>
> - `Popen.communicate(input=None)`
>
>   与子进程进行交互，向stdin发送数据，或从stdout和stderr中读取数据
>
>   可选参数input指定发送到子进程的参数。`Communicate()`返回一个元组：`(stdoutdata, stderrdata)`
>
>   注意：如果希望通过进程的 stdin 向其发送数据，在创建Popen对象的时候，参数 stdin 必须被设置为 PIPE。同样，如果希望从 stdout 和 stderr 获取数据，必须将 stdout 和 stderr 设置为PIPE
>
> - `Popen.send_signal(signal)`
>
>   向子进程发送信号
>
> - `Popen.terminate( )`
>
>   停止(stop)子进程
>
> - `Popen.kill( )`
>
>   杀死子进程

subprocess的用法：

（1）简单用法

```python
p = subprocess.Popen("dir", shell=True) # shell参数根据你要执行的命令的情况来决定，上面是dir命令，就一定要shell=True了

p.wait() # 得到命令的返回值，0表示执行成功
```

（2）进程通讯

如果想得到进程的输出，管道是个很方便的方法，这样：

```python
p = subprocess.Popen("dir", shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

(stdoutput,erroutput) = p.communicate()
```

`p.communicate`会一直等到进程退出，并将标准输出和标准错误输出返回，这样就可以得到子进程的输出了

上面，标准输出和标准错误输出是分开的，也可以合并起来，只需要将stderr参数设置为`subprocess.STDOUT`就可以了

```python
p=subprocess.Popen("dir", shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)

(stdoutput, erroutput) = p.communicate()
```

（3）类似shell管道的方式连接起上下游多个命令

```python
p1 = subprocess.Popen('cat ff', shell=True, stdout=subprocess.PIPE)

p2 = subprocess.Popen('tail -2', shell=True, stdin=p1.stdout, stdout=subprocess.PIPE)
```

---

参考资料：

(1) [Python3.7 - Argparse模块讲解](https://www.jianshu.com/p/00425f6c0936)

(2) [subprocess模块用法](http://www.calvinneo.com/2018/07/17/subprocess_usage/)

(3) [CSDN·imzoer《python中subprocess学习》](https://blog.csdn.net/imzoer/article/details/8678029)

(4) [官方中文文档《subprocess --- 子进程管理》](https://docs.python.org/zh-cn/3/library/subprocess.html)
