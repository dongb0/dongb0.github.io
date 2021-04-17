---
layout: post
title: "[c++/STL] vector - traits"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - c++
  - source code
---

### allocator_traits

本来只是想看看迭代器对应 pointer 应该如何获取以及编译器如何判断相应类型的，没想到是个深坑，一个类套一个类，跟套娃似的，花了好长时间才捋清楚思路。这过程中还 （~~被迫~~） 学了一些c++11的新特性，好歹是大致看懂了（~~好像也没有~~）。



首先看 vector 的 iterator 类型，是``

    //代码

然后我们找 _Vector_alloc 

    //_Vector_alloc 定义

然后在这里我们看到，原来 vector 的 iterator 类型是 _Vector_iterator<>

    //_Vector_iterator<> 定义

记住，从 vector 传到 _Vector_iterator<> 的类型是 _Vector_val<_Vec_base_types<_Ty, _Alloc>::_Val_types>，而 _Vector_val 定义的 pointer 类型又是来自传进 _Vector_val<> 的实参，（见下面定义中`typedef pointer`一行）

这里也有 iterator，不过没确定实际用处

    template<class _Val_types>
      class _Vector_val
        : public _Container_base
      {	// base class for vector to hold data
    public:
      typedef _Vector_val<_Val_types> _Myt;

      typedef typename _Val_types::value_type value_type;
      typedef typename _Val_types::size_type size_type;
      typedef typename _Val_types::difference_type difference_type;
      typedef typename _Val_types::pointer pointer;
      typedef typename _Val_types::const_pointer const_pointer;
      typedef typename _Val_types::reference reference;
      typedef typename _Val_types::const_reference const_reference;

      typedef _Vector_iterator<_Myt> iterator;
      typedef _Vector_const_iterator<_Myt> const_iterator;
      //...
      };

也就是说我要去找 _Vec_base_types<_Ty, _Alloc>>::_Val_types 的 pointer 类型是怎么定义的。

    template<class _Ty,
      class _Alloc0>
      struct _Vec_base_types
      {	// types needed for a container base
      typedef _Alloc0 _Alloc;
      typedef _Vec_base_types<_Ty, _Alloc> _Myt;

      typedef _Wrap_alloc<_Alloc> _Alty0;
      typedef typename _Alty0::template rebind<_Ty>::other _Alty;


      typedef typename _If<_Is_simple_alloc<_Alty>::value,
        _Simple_types<typename _Alty::value_type>,
        _Vec_iter_types<typename _Alty::value_type,
          typename _Alty::size_type,
          typename _Alty::difference_type,
          typename _Alty::pointer,
          typename _Alty::const_pointer,
          typename _Alty::reference,
          typename _Alty::const_reference> >::type
        _Val_types;
      };

                                                            //说成定义还是声明？
这里乍一看好像又是个套娃（实际上还真的是），不过已经逐渐接近真相了。这里定义的 _Val_types 的类型就决定了后面层层 pointer 套娃的类型。

首先我们来看 _If<> 的定义

    template<bool,
      class _Ty1,
      class _Ty2>
      struct _If
      {	// type is _Ty2 for assumed false
      typedef _Ty2 type;
      };

    template<class _Ty1,
      class _Ty2>
      struct _If<true, _Ty1, _Ty2>
      {	// type is _Ty1 for assumed true
      typedef _Ty1 type;
      };

配合注释，我们知道这里 _If 的作用是，当第一个参数 bool 值为 true 时，将 type 定义为第二个参数的类型；否则 type 的类型与第三个参数类型相同。而 _Is_simple_alloc<_Alty> 的作用就是完成对 _Alty 是否为简单类型的判断。

    template<class _Alty>
      struct _Is_simple_alloc
        : _Cat_base<is_same<typename _Alty::size_type, size_t>::value
        && is_same<typename _Alty::difference_type, ptrdiff_t>::value
        && is_same<typename _Alty::pointer, typename _Alty::value_type *>::value
        && is_same<typename _Alty::const_pointer, const typename _Alty::value_type *>::value
        && is_same<typename _Alty::reference, typename _Alty::value_type&>::value
        && is_same<typename _Alty::const_reference, const typename _Alty::value_type&>::value>
      {	// tests if allocator has simple addressing
      };

    typedef integral_constant<bool, true> true_type;
    typedef integral_constant<bool, false> false_type;

    template<class _Ty1,
      class _Ty2>
      struct is_same
        : false_type
      {	// determine whether _Ty1 and _Ty2 are the same type
      };

    template<class _Ty1>
      struct is_same<_Ty1, _Ty1>
        : true_type
      {	// determine whether _Ty1 and _Ty2 are the same type
      };

    template<bool _Val>
      struct _Cat_base
        : integral_constant<bool, _Val>
      {	// base class for type predicates
      };

      	// TEMPLATE CLASS integral_constant
    template<class _Ty, _Ty _Val>
      struct integral_constant
      {	// convenient template for integral constant types
      static constexpr _Ty value = _Val;
      //...
      };

看命名大概也能猜到是用来判断 _Alty::pointer 与 _Alty::value_type * 等是否为同一类型。我调整了部分代码的缩进（主要是换行），让它看起来更清晰一点。

最后，终于到决定 pointer 类型的地方了。如果是 _Is_simple_alloc<> 的话，_Vec_base_types<_Ty, _Alloc>>::_Val_types 的类型将会是 _Simple_types，那么 _Simple_types 是如何定义的呢？我们来看代码。

    template<class _Value_type>
      struct _Simple_types
      {	// wraps types needed by iterators
      typedef _Value_type value_type;
      typedef size_t size_type;
      typedef ptrdiff_t difference_type;
      typedef value_type *pointer;
      typedef const value_type *const_pointer;
      typedef value_type& reference;
      typedef const value_type& const_reference;
      };

很明显我们可以看到，pointer 的类型被定义为 `value_type *`，也就是原生指针。

我暂时还不清楚能否忽略 _Wrap_alloc<>::rebind() 但是要继续了解关于这个类的功能好像还要花费相当多的时间，下次新开一文继续吧。


https://stackoverflow.com/questions/14148756/what-does-template-rebind-do


mark  
https://stackoverflow.com/questions/12362363/why-is-allocatorrebind-necessary-when-we-have-template-template-parameters








------------

    //Line 466~494
    template<class _Val_types>
      class _Vector_val
        : public _Container_base
      {	// base class for vector to hold data
    public:
      typedef _Vector_val<_Val_types> _Myt;

      typedef typename _Val_types::value_type value_type;
      typedef typename _Val_types::size_type size_type;
      typedef typename _Val_types::difference_type difference_type;
      typedef typename _Val_types::pointer pointer;
      typedef typename _Val_types::const_pointer const_pointer;
      typedef typename _Val_types::reference reference;
      typedef typename _Val_types::const_reference const_reference;

      typedef _Vector_iterator<_Myt> iterator;
      typedef _Vector_const_iterator<_Myt> const_iterator;

      _Vector_val()
        : _Myfirst(),
        _Mylast(),
        _Myend()
        {	// initialize values
        }

      pointer _Myfirst;	// pointer to beginning of array
      pointer _Mylast;	// pointer to current end of sequence
      pointer _Myend;	// pointer to end of array
      };



    //xememory0

    template<class _Alloc>
      struct _Wrap_alloc
        : public _Alloc
      {	// defines traits for allocators
      typedef _Alloc _Mybase;
      typedef allocator_traits<_Alloc> _Mytraits;




		// TEMPLATE STRUCT _Get_pointer_type
    template<class _Ty>
      struct _Get_pointer_type
      _GET_TYPE_OR_DEFAULT(pointer,
        typename _Ty::value_type *);


      	// TYPE TESTING MACROS
    struct _Wrap_int
      {	// wraps int so that int argument is favored over _Wrap_int
      _Wrap_int(int)
        {	// do nothing
        }
      };

    template<class _Ty>
      struct _Identity
      {	// map _Ty to type unchanged, without operator()
      typedef _Ty type;
      };

      //声明_Fn函数，该函数返回值是_Identity<>，c++11新特性尾置返回类型（C++ Primer 中文版第五版 P206）
      //Q: 只声明无定义，能否正常使用？

      #define _GET_TYPE_OR_DEFAULT(TYPE, DEFAULT) \
      { \
      template<class _Uty> \
        static auto _Fn(int) \
          -> _Identity<typename _Uty::TYPE>; \
    \
      template<class _Uty> \
        static auto _Fn(_Wrap_int) \
          -> _Identity<DEFAULT>; \
    \
      typedef decltype(_Fn<_Ty>(0)) _Decltype; \
      typedef typename _Decltype::type type; \
      }

参考了[这里的解释](https://www.jianshu.com/p/b22a01d81ce8)


根据
~~~
struct _Get_pointer_type
	_GET_TYPE_OR_DEFAULT(pointer,
		typename _Ty::value_type *);
~~~
会根据传入参数确定返回的究竟是


      		// TEMPLATE CLASS allocator
    template<class _Ty>
      class allocator
      {	// generic allocator for objects of class _Ty
    public:
      static_assert(!is_const<_Ty>::value,
        "The C++ Standard forbids containers of const elements "
        "because allocator<const T> is ill-formed.");

      typedef void _Not_user_specialized;

      typedef _Ty value_type;

      typedef value_type *pointer;
      typedef const value_type *const_pointer;

      typedef value_type& reference;
      typedef const value_type& const_reference;

      typedef size_t size_type;
      typedef ptrdiff_t difference_type;

      ...
      
      };



    template<class _Ty,
    class _Alloc0>
    struct _Vec_base_types
    {	// types needed for a container base
    typedef _Alloc0 _Alloc;
    typedef _Vec_base_types<_Ty, _Alloc> _Myt;

    typedef _Wrap_alloc<_Alloc> _Alty0;
    typedef typename _Alty0::template rebind<_Ty>::other _Alty;


    //_Is_simple_alloc<>又有好几层继承关系，简单概括来说是如果_Alty是简单类型（简单指_Alty内定义的pointer为原生指针，引用为普通引用），那么value的值是true，否则为false。考虑要不要贴定义（折叠）

    //class _If   第一个参数bool的值为真，则将type设为第1个参数的类型；否则type设为第2个参数的类型
    template<bool,
      class _Ty1,
      class _Ty2>
      struct _If
      {	// type is _Ty2 for assumed false
      typedef _Ty2 type;
      };

    template<class _Ty1,
      class _Ty2>
      struct _If<true, _Ty1, _Ty2>
      {	// type is _Ty1 for assumed true
      typedef _Ty1 type;
      };


    typedef typename _If<_Is_simple_alloc<_Alty>::value, 
      _Simple_types<typename _Alty::value_type>,
      _Vec_iter_types<typename _Alty::value_type,
        typename _Alty::size_type,
        typename _Alty::difference_type,
        typename _Alty::pointer,
        typename _Alty::const_pointer,
        typename _Alty::reference,
        typename _Alty::const_reference> >::type
      _Val_types;
    };

因此我们大多数时候使用 vector 的时候，像int，long，double一类的（自定义结构体呢？）都会使用value_type *，也就是原生指针，作为 vector::iterator 的类型。

而根据 STL源码剖析 一书***，只有在自定义结构体（类）中包含指针时，编译器才会转到第二类（也分编译器）

    template<class _Value_type>
      struct _Simple_types
      {	// wraps types needed by iterators
      typedef _Value_type value_type;
      typedef size_t size_type;
      typedef ptrdiff_t difference_type;
      typedef value_type *pointer;
      typedef const value_type *const_pointer;
      typedef value_type& reference;
      typedef const value_type& const_reference;
      };


C++模板函数的匹配规则我现在也搞不太清楚了，*C++ Primer* 没提会根据返回值的类型推断，但是我们学校老师编的课本上好像又写有。当初 C++ 老师是本硕博数学系出身的，实在是太难了。这里就当作编译器能够根据返回值的模板类来推断吧，以后要是找到解释了再补充。

妈呀真的是累死我了，我只是想验证一下 vecotr 的 iterator 是不是指针类型，结果自己找起来真的费劲。现在终于可以用














mark
https://stackoverflow.com/questions/22368300/what-does-this-return-exactly


  //今日迷惑代码
	reference operator*() const
		{	// return designated object
		return ((reference)**(_Mybase *)this);
		}

//迷惑代码解析

this is a pointer to the current object.

\*this is a reference to the current object, i.e. this dereferenced.

\*\*this is the returned value of the overloaded unary operator* function called on the current object.

If the object returned from \*\*this has an overloaded operator&() function, then &\*\*this evaluates to the returned value of (\*\*this).operator&(). Otherwise, &\*\*this is the pointer to the returned value of the overloaded unary operator* function called on the current object.

Example:

    #include <iostream>

    struct A
    {
      int b;
      int a;
      int& operator*() {return a;}

      int* test()
      {
          return &**this;
      }
    };

    int main()
    {
      A a;
      std::cout << "Address of a.a: " << a.test() << std::endl;
      std::cout << "Address of a.a: " << &(*a) << std::endl;
      std::cout << "Address of a.a: " << &(a.a) << std::endl;
      return 0;
    }

Sample output:

    Address of a.a: 0x7fffbc200754
    Address of a.a: 0x7fffbc200754
    Address of a.a: 0x7fffbc200754