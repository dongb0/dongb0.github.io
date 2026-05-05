---
layout: post
title: "[C++] move && forward"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - C++
---

> 一些CPP八股，左值右值反复看反复忘记，还是得整理理解后才进入脑子。

## Prerequisite

#### 右值引用

这是为了支持移动语义引入的概念，用`&&`表示。右值引用只能绑定在“将要销毁的对象”上。比如：

```
int i =42;
int &r = i;
int &&rr = i;       // 错误，不能将右值引用绑定在左值上
int &r2 = i * 42    // 错误，左值引用不能绑定在右值
const int &r3 = i * 42; // 正确，const引用可以绑定在右值上
int &&rr2 = i * 42; // 正确，可以将rr2绑定在乘法结果
```

## move

std::move 的本质是一个模板函数，它等价于做了一次 static_cast，将左值 x 强制转换为右值引用（Rvalue Reference, T&&）。

当一个变量初始化时接收的参数是右值引用，编译器会自动寻找匹配的移动构造函数；如果找不到则会继续使用拷贝构造函数，此时用move就无法带来性能提升。注意：这里存在一个误区，即我将参数声名为右值，是否就可以不用std::move函数而直接触发移动呢？

```
void processData(MyClass&& ref) {
    // 问：下面这行代码，调用的是拷贝构造，还是移动构造？
    MyClass newObj(ref); 
}
```

答案是：不行。具有名字的右值引用，本身也是一个左值，上述代码依然触发拷贝构造，非常的反人类。只有使用std::move返回一个匿名的右值引用，才会触发移动构造语义。


## 完美转发（forward）

一句话概括forward的作用：在模版中，将原参数（左值/右值）的类型原封不动转发给下一层函数调用。来看个代码例子理解一下：

```
void doWork(const MyClass& arg) {
    std::cout << "接收到左值，执行拷贝工作" << std::endl;
}

void doWork(MyClass&& arg) {
    std::cout << "接收到右值，执行移动工作" << std::endl;
}

template <typename T>
void wrapper(T&& arg) {
    // 问：这里应该怎么传给 doWork？
    doWork(arg); 
}

MyClass a;
wrapper(a);             // 传入左值。内部 doWork 匹配左值版本（符合预期）
wrapper(std::move(a));  // 传入右值。内部 doWork 依然匹配左值版本！右值属性丢失了
```

正如前文所说，具名右值引用也是一个左值，因此最后匹配的版本统统都是左值版本。而使用完美转发后，则可以解决该问题：

```
template <typename T>
void wrapper(T&& arg) {
    // 使用 std::forward 完美转发
    doWork(std::forward<T>(arg)); 
}
```

此时使用 wrapper(std::move(a)) 也能正确的调用右值版本，使用移动构造了。

另外为什么强调forward是用在模版中，因为普通函数里，参数的具体类型是确定的。比如上面的wrapper不用模版会是这样：

```
// 情况 A：明确只接收右值
void ordinaryWrapper(MyClass&& arg) {
    // 因为签名写死了是 &&，所以我【明确知道】外面的调用者传进来的是个右值/临时对象。
    // 既然我知道它本来是个右值，我现在要转发给别人，我直接无脑用 std::move 就行了！
    // 根本不需要什么“完美转发”或者“条件判断”。
    doWork(std::move(arg)); 
}

// 情况 B：明确只接收左值
void ordinaryWrapper(const MyClass& arg) {
    // 签名写死了是 const &，说明外面的东西不能被移动。
    // 我直接原样传下去就行。
    doWork(arg); 
}
```

而用模版后，因为用了万能右值引用T &&，调用wrapper时具体的类型是左值还是右值，是编译器在模板实例化时推导的。

## 何时需要自定义移动构造函数

1. 结构体里使用了裸指针；
2. 非内存类型的资源，比如FILE* 或者一个 int fd 套接字描述符

C++ 有一个著名的经验法则：如果你发现自己需要为一个类手动编写“析构函数”、“拷贝构造函数”或“拷贝赋值运算符”中的任何一个，那么你通常也需要写另外两个，并且极大概率也需要手动编写“移动构造函数”和“移动赋值运算符”。

所以裸指针等用符合RAII的智能指针代替，尽量用标准库的容器，能够避免需要手动编写拷贝/移动构造函数的情况。