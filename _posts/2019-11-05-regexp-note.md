---
layout: post
title: "[正则表达式] Basic"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - note
  - RegExp
---

> 一段时间没用连之前自己写的匹配模式都看不懂了。不过本来学得也不咋地，重新学习。示例使用Java Pattern/Matcher类验证。

这里有一份不知哪里来的[正则表达式手册](https://tool.oschina.net/uploads/apidocs/jquery/regexp.html)

### 元字符

|      |    |
| --- | --- | 
| `^` | 匹配字符串的开始位置 |
| `$` | 匹配字符串的结束位置 |
| `*` | 匹配前面的表达式0次或多次, "zo*"可匹配"z", "zo", "zoo"等, 等价于{0,} |
| `+` | 匹配前面的表达式1次或多次, "zo+"可匹配"zoo", 不可匹配"z", 等价于{1,} | 
| `?` | 匹配前面的表达式0次或1次, "zo"可匹配"z", "zo", 不可匹配"zoo", 等价于{0,1} |
| `?` |  (懒惰匹配)符号"?"跟在其他限制符如*, +, ?, {n}, {n,m}后时, 匹配尽可能短的字符串, 例如: "ooo,ooo", 模式"o+?"匹配单个"o"共6个, 而"o+"匹配得到两个"ooo" | 
| `.` | 匹配除"\n"以外的任意字符 | 
| `x|y` | 匹配x或y |


|      |    |     |
| --- | --- | --- |
| `\b` | 匹配单词边界-bound| "er\b"可以匹配"hacker die"中的er，不能匹配"verb"中的er |
| `\B` | 匹配非单词边界 | "er\b"可以匹配"verb"中的er，不能匹配"hacker die"中的er |
| `\d` | 匹配一个数字字符-digit | 等价于\[0-9] |
| `\D` | 匹配一个非数字字符 | 等价于\[^0-9] |
| `\s` | 匹配一个任意的空白字符-space | 等价于\[\f\n\r\t\v] (按顺序分为换页符, 换行, 回车, 制表符, 垂直制表符)|
| `\w` | 匹配一个包括下划线的任意单词字符-word | 等价于\[A-Za-z0-9_] 注意包含**下划线** |
| `\W` | 匹配任何非单词字符 | \[^A-Za-z0-9_] |
| `\*num*` | 正整数*num*表示第几个分组的引用 | 如`(s)-(a)-\2`匹配"s-a-a"，`\1`得到第一个分组,这里是"s",`\2`是第二个分组) |



### 获取分组

使用小括号捕获分组, 自左向右, 第一个分组为1, 第二个为2, 以此类推

|     |     |
| --- | ---- |
| `(pattern)` | 匹配并捕获这一分组，之后可以通过`$num`引用,若有嵌套, 则按照左括号出现的顺序捕获 |
| `(?:pattern)` | 匹配但不捕获 | 
| `(?<name>pt)` | 给分组命名,用`\k<name>`在模式中引用, 例如用`(?<myword>\w)\k<myword>`捕获两个连续的单词字符  |
| `(?=pattern)` | 正向预查, `pattern`部分不在匹配范围内 <br> 如`Java(?=6)`匹配"Java6, Java8"中的第一个"Java", 不包含"6" | 
| `(?!pattern)` | 正向否定预查 | 
| `(?<=pattern)` | 反向预查 <br> `(?<=J)a`匹配"Java6, Java7"的结果为J`a`va6, J`a`va7|
| `(?<!pattern)` | 反向否定预查 | 

正/反向预查部分例子来自于<https://my.oschina.net/lxpan/blog/27907>

Q: 为什么要使用`(?:pattern)`匹配但是不捕获分组?   
......为什么呢? 是不是存在可以不捕获但是必须分组的情况? 那么这种情况下就算捕获了会出现什么影响呢?

[这里][1]说捕获组是要占用内存的。那么答案应该是为了节省内存降低开销吧。

| Class Matcher Method |  |
| --- | --- |
| `String`  | `group(int group)` <br>  Returns the input subsequence captured by the given group during the previous match operation. |
| `String`  | `group(String name)` <br>  Returns the input subsequence captured by the given named-capturing group during the previous match operation. |

补充：使用Java进行验证的时候，从API可以看出，每个捕获分组都对应一个String，验证了上述说法。

--------------

> ~~实际上组号分配过程是要从左向右扫描两遍的：第一遍只给未命名组分配，第二遍只给命名组分配－－因此所有命名组的组号都大于未命名的组号~~

对于文本

    一段时间没用连之前自己写的匹配模式都看不懂了

模式`一段(.*)连(.*)匹配(.*)(?<n1>懂)(.*)`匹配得到`一段时间没用连之前自己写的匹配模式都看不懂了`

$1为`"时间没用"`, $2为`"之前自己写的"`, $3为`"模式都看不"`, $4为`"懂"`（与$\<n1\>相同。Java中获取命名分组使用`String group(String name)`, 获取分组数`groupCount()`），$5为`"了"`。与上面说法**不符**  

#### 嵌套分组捕获

前面说了，嵌套是按照左括号从左到右出现的先后来捕获的，下面是示例：

对于文本

    a1a2ba2  
    a1a2ba1  
    a1a2ba1a2  
    a1a2ba1a2b  

模式`((a1(a2))b)\1`匹配第4行`a1a2ba1a2b`  
模式`((a1(a2))b)\2`匹配第3、4行的`a1a2ba1a2`  
模式`((a1(a2))b)\3`匹配第1行`a1a2ba2`









[1]: https://dailc.github.io/2017/08/01/regularExpressionConcepts.html