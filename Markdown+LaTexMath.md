<a name="content">目录</a>

[Markdown+LaTex](#title)
- [数学公式的编辑与显示](#edit-and-present-math-formula)
- [编辑器的选择](#chose-proper-editor)
- [数学公式与LaTex Math](#math-formula-and-latex-math)
	- [内联与块状](#inner-insert-and-block-insert)
	- [LaTex Math的基本语法](#syntax-of-latex-math)
	- [复杂数学公式](#difficult-math-formula)
- [LaTex Math的跨平台显示](#cross-platform-of-latex-math)

<h1 name="title">Markdown+LaTex</h1>

实践出一整套便于互联网传播分享的数学公式跨平台编辑、跨环境显示是非常有必要的，如果还是停留在Word或PDF时代，那**数学就会被限制在文档或图片里而无法通过最流行的网页方式进行传播**，而且Word、PDF等文件处理软件里的数学公式编辑既麻烦，而且最重要的是与编程脱节。

<a name="edit-and-present-math-formula"><h2>数学公式的编辑与显示 [<sup>目录</sup>](#content)</h2></a>

常用的文本编辑用Markdown就能高效地完成，但是Markdown有一个实现不了的功能——编辑数学公式，当然也不是说完全不能解决，折中一下，编辑成图片，以图片形式插入也行，但这毕竟不太高效和方便，也不利于传播，而在出版界用得比较多的LaTex语法虽然在数字的编辑上有一套成熟的解决方案，LaTex math，但是在普通文本的编辑上，它的语法格式与Markdown相比又显得太过复杂

那么，有没有一种方法，可以分别将Markdown的文本编辑优势和LaTex的数学公式编辑优势相互结合起来呢？

有，就是Markdown+LaTex

与LaTex文档的比较

> 虽然很多数学学术论文整个文档就像使用Markdown一样是直接使用的LaTex语法来编辑的，但是仔细比对之后发现直接用LaTex语法来写整个文档来，它的效果和Markdown + LaTex Math 方式没有太大的区别。
> 
> 但是LaTex的语法、编辑器、配置、中文支持等都要比Markdown要复杂的多，而且也不及Markdown已经非常成熟的生态（包括工具链、社区等）。

<a name="chose-proper-editor"><h2>编辑器的选择 [<sup>目录</sup>](#content)</h2></a>

Markdown的编辑器非常多，对于很多初学者来说，个人比较推荐使用VS Code

> - VS Code汉化比较方便，想让更多人学会使用Python来学数学，有一个中文界面还是比较重要的；而且VS Code是跨平台的，Mac、Windows都可以上手；
> 
> - VS Code是一款极为优秀的代码编辑器，说起优秀，应该算是目前最为推荐的编辑器之一（可能没有之一）；
> 
> - VS Code插件丰富，Python的编译、Markdown的编写与预览、LaTex Math的显示等工具链相当完备；

<a name="math-formula-and-latex-math"><h2>数学公式与LaTex Math [<sup>目录</sup>](#content)</h2></a>

<a name="inner-insert-and-block-insert"><h3>内联与块状 [<sup>目录</sup>](#content)</h3></a>

在介绍数学公式之前，我们需要先来了解一下**内联**与**块状**的概念

所谓内联就是我们可以把数学符号嵌入到文字段落里面，比如：

```
函数式：$f(x)=\frac{P(x)}{Q(x)}$  
```

如果我们需要输出的数学公式比较复杂，或者我们需要凸出并独立显示公式，这个时候我们就需要使用到公式的块状输出，块状输出的语法使用4个美元符号`$$数学公式$$`，我们来看案例。

```
$$f(x)=\frac{P(x)}{Q(x)}$$ 
```

使用块状输出，函数会居中显示，值得一提的是我们在使用块状输出数学公式时，**在Markdown里需要换行来写公式**。

<a name="syntax-of-latex-math"><h3>LaTex Math的基本语法 [<sup>目录</sup>](#content)</h3></a>

- **希腊字母**

```
$\Gamma$
$\iota$
$\sigma$
$\phi$
$\upsilon$
$\Pi$
$\Bbbk$
$\heartsuit$
$\int$
$\oint$
```

- **三角函数**

```
$\tan$
$\sin$
$\cos$
```

- **运算符**

```
$+$
$-$
$=$
$>$
$<$
$\times$
$\div$
$\equiv$
$\leq$
$\geq$
$\neq$  
```

- **集合符号**

```
$\cup$
$\cap$
$\in$
$\notin$
$\ni$
$\subset$
$\subseteq$
$\supset$
$\supseteq$
$\infty$
```

- **指数输出**

```
$x^3+x^9$  
$x^y$ 
```

- **n次方根输出**

```
$\sqrt{3x-1}+\sqrt[5]{2y^5-4}$
```

- **输出分数**

语法为`\frac{分子}{分母}`

```
$$\frac{x}{2y} +\frac{x-y}{x+y} $$
```

- **累加求和输出**

求和公式比较复杂，会涉及到上标和下标，在输出指数`^`时我们可以把它看成是上标，使用`_`来输出下标

```
$$\sum_{n=1}^\infty k$$
```

- **极限的输出**

输出极限就会使用到下标

```
$$\lim\limits_{x \to \infty} \exp(-x) = 0$$
```

- **输出矩阵**

使用`\begin{matrix}`和`\end{matrix}`围住即可输出矩阵，矩阵之间用`$`来空格，用`\\`来换行

```
$$
  \begin{matrix}
   1 & 2 & 3 \\
   4 & 5 & 6 \\
   7 & 8 & 9
  \end{matrix} 
$$
```

<a name="difficult-math-formula"><h3>复杂数学公式 [<sup>目录</sup>](#content)</h3></a>

- **分段函数的编写**

分段函数是非常复杂的，这时候会用到LaTex的cases语法，用`\begin{cases}`和`\end{cases}`围住即可，中间则用`\\`来分段，具体我们来看下面的例子

```
$$
X(m,n)=
\begin{cases}
x(n),\\
x(n-1)\\
x(n-1)
\end{cases}
$$
```

<a name="cross-platform-of-latex-math"><h2>LaTex Math的跨平台显示 [<sup>目录</sup>](#content)</h2></a>

**（1）知乎、简书**

简书的Markdown编辑器可以比较完美的支持Markdown语法以及Markdown Math语法，可以直接把用VS Code写的Markdown文件里的内容复制粘贴过去，然后进行一些简单的修改就可以了

知乎自带数学公式的插入，如果直接导入Markdown文件显示会出现一些问题，需要把数学公式用知乎自带的Tex编辑器重新书写，只需要把`$$`删除即可


**（2）网页**

由于我们的网页可以不用Markdown，用HTML替换Markdown排版语法就可以，所以我们只需要专注于如何在网页上显示数学公式即可。比较完美的解决方案是使用`mathjax`，我们只需要在`<head>`标签内插入`mathjaxjs`即可。

```
<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML" async>
</script>
```

要注意的是内联式的语法会有些不同，不再是`$符号与公式$`，而是：`\(符号与公式\)`

```
<p>
  当 \(a \ne 0\)时,  \(ax^2 + bx + c = 0\) 会有两个解，它们是：
  $$x = {-b \pm \sqrt{b^2-4ac} \over 2a}.$$
</p>
```

**（3）公众号**

微信公众号封闭且奇葩，美化微信公众号的排版虽然用的是html和css语法，但是有很多需要注意的地方，因此排版也相对来说比较麻烦

要想让数学公式在公众号上显示就比较麻烦，微信公众号是不支持LaTex语法的，所以需要把公式做成图片，其他不支持LaTex的自媒体平台也可以这么处理


---

参考资料：

(1) [【简书】HackWeek《使用Markdown输出LaTex数学公式》](https://www.jianshu.com/p/d9a5a1c694b4)

