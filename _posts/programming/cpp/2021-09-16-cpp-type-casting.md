---
layout: post
title: "[c++] 显式类型转化"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: true
tags:
  - programming
  - c++
---

> 这里是当时做 bustub 顺手整理的笔记，后来补发的，内容主要摘自 「C++ Primer Plus」第六版

c++一共有4种类型转换符
- dynamic_cast
- const_cast
- static_cast
- reinterpret_cast

##### dynamic_cast

要求被赋值的变量必须是被强制转换变量的基类类型，即用于从子类指针转换为父类指针（向上转换）。
```
class A{ };
class B : public A { };
...
B *pt_b = new B();
A *pt_a = dynamic_cast<A *>(pt_b);  // OK
pt_b = dynamic_cast<B *>(pt_a);  // error: dynamic_cast : "A"不是多态类型
```
也就是说 dynamic_cast 转换和赋值的对象都得是struct/class（的指针或引用），否则编译报错：
```
int* pt_int = nullptr;
char* pt_char = nullptr;
pt_int = dynamic_cast<int*>(pt_char);  // error: "int *": dynamic_cast的目标类型无效，目标类型必须是指向已定义类的指针或引用
```

##### const_cast

只能用来给变量 **添加或者去除** `const/volatile`属性。看例子：
```
int *pt1 = new int[10];
const int *pt2 = pt1;
int *pt3 = nullptr;
pt3 = pt2; // 如果不用强制类型转换，会报错: 无法从"const int *"转为"int *"
pt3 = const_cast<int *>(pt2); // OK
```
但不能用 const_cast 来改变`const/volatile`以外的其他属性：
```
char *pt4 = const_cast<char *>(pt2); // error: const_cast不能更改基础属性，不能把基础类型int *转为char *
```
const_cast可能的应用场景：某个值大多数时候是常量， 但有时也需要修改，可以将其声明为const，在需要时使用const_cast

##### static_cast
c++ primer plus(第6版)的介绍为：`static_cast< type_name >(expression)`
只有 type_name 可以隐式转换为 expression 所属的类型，或者 expression 隐式转换为 type_name 时，上述转换才合法。
比如可以用于父类和子类指针的互相转换，int转为double，float转为long等。但不能将类的指针转化为其他没有继承关系的类，如：
```
class A{ };
class B : public A { };
class C {};
...
A *pt_a = new A();
B *pt_b = static_cast<B *>(pt_a); // OK, 父类A指针转为子类B指针
C *pt_c = static_cast<C *>(pt_a); // error
pt_a = static_cast<A *>(new C()); // error
```


##### reinterpret_cast
与C风格的强制类型转换功能类似，但不能去除const（C的强制转换可以），其余转换皆可（通过编译，但有可能运行时才出现错误）。
```
int *pt1 = new int[10];
const *int pt2 = pt1;
int *pt3 = reinterpret_cast<int *>(pt2); // error，不能去除const属性
int *pt4 = (int *) pt2;  // OK
```

----------------