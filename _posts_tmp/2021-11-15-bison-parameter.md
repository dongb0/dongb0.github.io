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

%parse-param {Parser *parser} 给yyparse添加入参，但.tab.h的函数声明没有引用定义Parser的头文件；
解决方法：在.y文件里用%code requires {#include "parser.h"}

默认情况flex读取标准输入，现在想让flex从string读取输入
解决方法：yy_scan_string()  yy_delete_buffer()