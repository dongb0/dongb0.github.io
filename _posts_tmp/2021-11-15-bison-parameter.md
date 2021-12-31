---
layout: post
title: "[sql parser] - flex and bison"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
---

遇到问题：flex生成yylex函数进行词法解析，bison身成yyparse进行语法分析，默认无参数；进行sql解析时需要通过入参的指针向其他（analize模块）返回解析得到的AST节点，

解决方法：.y文件设置%param参数