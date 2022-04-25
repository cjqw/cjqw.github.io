---
layout: post
title: 如何愉快地写一个parser
date: 2018-01-23 23:39 +0800
tags: Clojure Instaparse parser
---


预告
----

有没有想过这样写一个 parser:

-   不要写 scanner
-   不要写 AST 生成
-   不要优先级，左结合右结合
-   不用担心左递归/右递归
-   最最重要的是无需设计复杂的语法

Q & A
=====

啥时候填坑
----------

在可预知的未来是没空了（要无限期延迟了）。 下面通过 QA
简略介绍一下这东西怎么弄。

到底怎么写
----------

用 clojure 的一个库 instaparse。
[简单的教程](https://github.com/Engelberg/instaparse) 在这里。

显然简单单词的 scanner 已经嵌在语法里了。

clojure 没有类，区分 token 全靠 vector 第一个元素，自然不要写 AST。 （用
pprint 把结果打出来就可以看得很清楚了）

clojure 的入门教程的话，个人推荐
[braveclojure](http://www.braveclojure.com/)。

这东西当 scanner 只能处理正则，我要搞 comment 怎么办
----------------------------------------------------

parse 之前先搞个 comment-parser 抠出 comment，删除之。（不要笑）

怎么处理优先级，结合性
----------------------

优先级：

``` {.c}
expr0 = item ('+' item)*
expr1 = expr0 ('*' expr0)*
```

结合性：

``` {.c}
expr = expr | expr '+' item (*左结合*)
expr = expr | item '+' expr (*右结合*)
```

你知道按上面这种方式 grammar 有多难写吗
---------------------------------------

我知道。但是那是针对普通 parser geneator 而言的。 instaparse
完全可以将设计 grammar 的复杂性降至最低。

怎么降
------

关键是多道 parser。

为什么一定要一个 parser 解决所有问题？

以 c 语言的 parser 为例，你可以先抠出所有
comment，再抠出变量定义，每条定义用一个单独 parser 处理。
抠出函数体，把每条语句分开，再慢慢写一个 expression-parser。

c 语言各种 x.y , \*p,各种优先级很难处理对吧？写一个 variable-parser 把
x.y.z 之类变量处理掉， 在用一个 parser 处理运算符。

x.(a+b) 怎么处理？ 多来几次 parse-variable 咯。

还觉得设计 grammar 很难嘛？

时间复杂度？
------------

妥善设计语法的话是 O(n) 的。当然像上面说的那么暴力肯定最坏是 O(n^2^)
的。

另外有个坑点，你的语法最好不要有歧义，不然 instaparse 会把所有可能的 AST
算出来然后返回其中一个。 当然你也可以用 parses 函数得到所有
AST。具体情况官方的 README 里写的很清楚了。
这种情况最坏可能到指数复杂度，如果你的 parser
慢如蜗牛，可以找找看哪有歧义。

一般的使用场景中是不用担心速度太慢的。有几个原因：

-   使用多道 parser 导致底层的 n
    规模很小，你只要保证顶部数层设计良好即可
-   如果你 parse
    的东西是手打的，你总不会坐下来敲几万行代码然后还层层嵌套吧，有个东西叫代码规范
-   如果是其他程序的输出，你为啥不能找个好点的办法输出，甚至直接输出 AST

个人以为，类似 antlr 或者 jcup
之类的工具变态般的追求复杂度主要是因为以前计算机跑太慢了。
除了单片机等特殊设备上，我实在找不出什么场景不能用 instaparse 代替 antlr
的。 如果你找到了，欢迎告诉我。

上面的 parse 方法好暴力，有没有更高明的方法
-------------------------------------------

不知道，懒得想。

这东西只能 clojure 用，我用不了啊
---------------------------------

clojure 是基于 jvm 的，你可以在任意操作系统使用 clojure。

clojure 是可以直接和 java 交互，创建 java 的类的。具体方法可以看
[braveclojure](http://www.braveclojure.com/java/)。

至于和其他语言的交互，你可以输出 AST。写一个 AST 的 parser
应该是不用费脑子的。

我能用这东西干什么
------------------

你可以用它做文本处理，比如我一开始想拓展 org-mode 让它能搞出 latex
的某些功能（比如 abstract）， 后来发现已经有 solution
了，所以这个东西就无限坑掉了。另外你可以写些
pydoc，cdoc，javadoc，cljdoc 之类的东西，虽然大部分已经有 solution
了，但是你可以在这些工具基础上加一点更复杂的东西。 比如把 org-mode 的
babel 搞到 pydoc 里面，当然我只是举个例子，能有多大用我也不知道。

今天偶然翻到了[这篇文章](http://www.yinwang.org/blog-cn/2015/09/19/parser)
，写得很好。

parser 这个东西确实单独看用处不大，甚至一般用 python
随便敲几行代码就解决问题了。 更何况各种 parser generator
可以很容易的构造一个 parser，里面的各种算法其实也没什么难的。
当年编译原理一半以上都是讲 parser 现在想起来都是泪。 当时大作业是写到
parser 就算 80 分的，基本上写到类型检查的都没几个人，
现在想来编译原理真是白学了。

最后还是拿别人的程序举个栗子
----------------------------

感谢 kulesa 的 [organum-cljs](https://github.com/gmorpheme/organum)
。多道 parse 的想法就是看了这位的代码才知道的。

源代码主要看[这里](https://github.com/kulesa/organum-cljs/blob/master/src/organum/core.cljs)
。这是一个 org-mode 的 parser，而且已经坑掉了。

说一下这个的一些不足：

-   作者写的时候似乎不知道可以设置 ：whitespace，很多地方定义了 ws，br
    之类的东西。
-   大量使用/，很多地方用|就可以解决问题了。
-   设计的不是很好，后面一大半代码都在干 fix-tree。
-   我忘记他有没有在 fix 里把 section 搞成树形了，他的 parser
    是没法区分多级标题的
-   parse table 居然用了两个 parser，一个 is-table，一个 org-tables

说一下怎么处理多级标题，定义一个生成 n 级标题的 parser 的函数，
搞个无穷的 lazy-seq 存 parser，然后递归 parse 就行了。 当然也可以把标题
parse 一下然后回过头 fix-tree。
