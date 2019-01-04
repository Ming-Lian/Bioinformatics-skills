<a name="content">目录</a>

[流程编程语言](#title)
- [1. WDL](#wdl)
	- [1.1. 编写WDL脚本](#coding-wdl-script)
		- [1.1.1. task之间的多种关系](#relationship-among-tasks)
		- [1.1.2. 多次调用同一个task时避免冲突的解决方案](#run-one-task-serveral-times-same-time)
	- [1.2. 运行WDL脚本](#run-wdl-script)





<h1 name="title">流程编程语言</h1>

<a name="wdl"><h2>1. WDL [<sup>目录</sup>](#content)</h2></a>

WDL是GATK官方推荐的workflow语言

```
workflow myWorkflow {
    call myTask
}
task myTask {
    command {
        echo "hello world"
    }
    output {
        String out = read_string(stdout())
    }
}
```

每个脚本包含1个workflow， workflow由多个task构成。 在workflow中，通过call调用对应的task。每个task在workflow代码块之外单独定义。

<table>
<th>
	Task结构
</th>
<th>
	Workflow结构
</th>
<tbody>
<tr>
	<td>
		<p align="center"><img src=./picture/Workflow-Programming-Languages-task-structure.png width=500 /></p>
	</td>
	<td>
		<p align="center"><img src=./picture/Workflow-Programming-Languages-workflow-structure.png width=600 /></p>
	</td>
</tr>
</tbody>
</table>

<a name="coding-wdl-script"><h3>1.1. 编写WDL脚本 [<sup>目录</sup>](#content)</h3></a>

<a name="relationship-among-tasks"><h4>1.1.1. task之间的多种关系 [<sup>目录</sup>](#content)</h4></a>

1\. 一对一的依赖关系

<p align="center"><img src=./picture/Workflow-Programming-Languages-task-relationship-one2one-1.jpg width=600 /></p>

或

<p align="center"><img src=./picture/Workflow-Programming-Languages-task-relationship-one2one-2.jpg width=600 /></p>

```
workflow LinearChain {
  File firstInput
  call stepA { input: in=firstInput }
  call stepB { input: in=stepA.out }
  call stepC { input: in=stepB.out }
}
task stepA {
  File in
  command { programA I=${in} O=outputA.ext }
  output { File out = "outputA.ext" }
}
task stepB {
  File in
  command { programB I=${in} O=outputB.ext }
  output { File out = "outputB.ext" }
}
task stepC {
  File in
  command { programC I=${in} O=outputC.ext }
  output { File out = "outputC.ext" }
}
```

2\. 多对多的依赖关系

<p align="center"><img src=./picture/Workflow-Programming-Languages-task-relationship-many2many.jpg width=600 /></p>


3\. 3. 平行关系

<p align="center"><img src=./picture/Workflow-Programming-Languages-task-relationship-parallel.jpg width=600 /></p>

```
workflow ScatterGather {
  Array[File] inputFiles
  scatter (oneFile in inputFiles) {
    call stepA { input: in=oneFile }
  }
  call stepB { input: files=stepA.out }
}
task stepA {
  File in
  command { programA I=${in} O=outputA.ext }
  output { File out = "outputA.ext" }
}
task stepB {
  Array[File] files
  command { programB I=${files} O=outputB.ext }
  output { File out = "outputB.ext" }
}
```

<a name="run-one-task-serveral-times-same-time"><h4>1.1.2. 多次调用同一个task时避免冲突的解决方案 [<sup>目录</sup>](#content)</h4></a>

task和函数还是有一定的区别，函数可以在代码中多次调用，但是**task多次调用会有风险**。下面的示意图中，stepA 运行两次，一次作为stepB的输入，一次作为stepC的输入。如果stepA的两次调用并行执行，当执行完之后，在传递给下一个task时，由于存在两个同名的stepA, stepB和stepC 就会无法正确接受参数。

<p align="center"><img src=./picture/Workflow-Programming-Languages-task-relationship-oneTask-multiTimes.jpg width=600 /></p>

WDL中提供了解决方案，叫做task alias, 为task起一个别名，示例如下：

```
call stepA as firstSample { input: in=firstInput }
call stepA as secondSample { input: in=secondInput }
```

在WDL脚本中, 理论上每个task 只可以调用1次，如果希望多次调用，必须借助task alias。

<a name="run-wdl-script"><h3>1.2. 运行WDL脚本 [<sup>目录</sup>](#content)</h3></a>

运行wdl脚本，需要两个文件：

- cromwell.jar，用于得到输入参数的列表，用法如下：

	```
	$ java -jar womtools.jar inputs myWorkflow.wdl > myWorkflow_inputs.json
	```
	
	用json格式存存储，这一步得到的只是一个模板，需要编辑这个文件，将对应的参数替换成实际的参数

- womtools.jar

	```
	$ java -jar Cromwell.jar run myWorkflow.wdl —inputs myWorkflow_inputs.json
	```








---

参考资料：

(1) [【简书】GATK官方推荐的workflow语言-WDL](https://www.jianshu.com/p/41f377e20ff7)
