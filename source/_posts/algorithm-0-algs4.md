---
title: 算法系列-0-使用javac-algs4和java-algs4
categories:
  - 算法
tags:
  - algorithm
  - java
date: 2018-02-27 00:02:46
---

在教程中会使用 javac-algs4 和 java-algs4 命令进行编译和执行 java 文件，classpath 默认有 algs4.jar。所以需要自定义 javac 和 java

## 效果

- windows
```
E:\de_learn\algorithms\homework\dequeue_ramdom>javac-algs4 Permutation.java

E:\de_learn\algorithms\homework\dequeue_ramdom>java-algs4 Permutation 3 < queues\distinct.txt
RandomizedQueue{C, B, A}
```

- linux
```
$ javac-algs4 PercolationStats.java 
$ java-algs4 PercolationStats 200 100
mean                    = 0.5937762499999999
stddev                  = 0.0098221928257679
95% confidence interval = [0.5918511002061494, 0.5957013997938504]

```

<!-- more -->

{% note warning %}
思路很简单，首先尝试使用 javac 和 java 命令，直接添加 classpath。
然后自定义 cmd 命令，将 classpath 带上。
{% endnote %}

## java javac 添加 classpath

- windows 下用分号 ";" 作为分隔符

```
javac -cp E:\de_project\git\AlgorithmsSedgewick\algs4.jar;E:\de_project\git\AlgorithmsSedgewick\stdlib.jar; *.java
或者
javac -classpath E:\de_project\git\AlgorithmsSedgewick\algs4.jar;E:\de_project\git\AlgorithmsSedgewick\stdlib.jar; *.java
```

- linux 下用冒号 ":" 作为分隔符

```
javac -cp /home/sealde/Document/de_file/algorithms/homework/jar/algs4.jar:/home/sealde/Document/de_file/algorithms/homework/jar/stdlib.jar: *.java
```

## windows 下进行自定义命令

- 设置 ALGS4 环境变量（可以不设置，只是为了方便）
```
ALGS4=E:\de_project\git\AlgorithmsSedgewick\algs4.jar;E:\de_project\git\AlgorithmsSedgewick\stdlib.jar
```

- 编写 bat 脚本，脚本功能为添加自定义的命令。符号意义如下：
    - doskey 相当于 linux 的 alias，@ 不显示命令
    - %ALGS4% 从系统环境变量取值
    - $* 指还有参数，这个没有深究
    - 分号确保有以分号结束 classpath

```
@doskey java-algs4 = java -classpath %ALGS4%; $*
@doskey javac-algs4 = javac -classpath %ALGS4%; $*
```

- 添加注册表信息，为了 cmd 启动时自动运行上面的脚本
    - Win+R ==》regedit ==》 HKEY_LOCAL_MACHINE\\Software\\Microsoft\\Command Processor ==》 新建字符串值，名为AutoRun ==》 值为E:\\de_learn\\algorithms\\bin\\algs4.bat ==》 保存退出
    
## linux 下进行自定义命令

- 设置 ALGS4 环境变量（可以不设置，只是为了方便）；并添加 alias
```
$ vim ~/.bashrc

ALGS4="/home/sealde/Document/de_file/algorithms/homework/jar/"
alias javac-algs4="javac -cp $ALGS4/stdlib.jar:$ALGS4/algs4.jar:"
alias java-algs4="java -cp $ALGS4/stdlib.jar:$ALGS4/algs4.jar:"
```