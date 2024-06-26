---
layout: post
title: "latexdiff生成显示修订的latex文件"
date: 2024-06-26
tags: [Latex]
comments: true
author: jtyang
---

本文重点是解决在编译latexdiff 产生的文件遇到的bug.
关于latexdiff 的环境变量配置等不属于本文内容.

## 基本流程
``` bash
# open a win cmd in Admin, then do
latexdiff old.tex new.tex > diff.tex
# bbl file, the field option is necessary
latexdiff --append-textcmd=field old.bbl new.bbl > diff.bbl
# then you can compile the tex file normally, by a GUI soft or CMD
```

## Debug

尽管生成文件的过程可能很顺利, 但是latexdiff 在生成时可能会出现bug, 就个人经验而言, 特别需要注意的是修改过大纲的部分, 比如`\section`的添加或者修改, 以及表格部分. 主要是关于花括号不匹配的问题.

在编译时, 报错可能不会显示具体的行号, 这时可以提前在某行插入`\end{document}` 来提前结束编译, 帮助查找报错行.

生成出来的`diff.bbl`文件逻辑关系复杂, 很难debug, 还没有找到好的解决办法, 如不能解决只好使用`new.bib, new.bbl`暂时编译.
