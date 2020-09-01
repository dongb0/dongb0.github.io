---
layout: post
title: "[c++/STL] vector - swap()"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
tags:
  - c++
  - source code
---

> 第一次翻阅源码，能力有限，难免有错漏和理解不当之处，如果出错还请多多包涵

今天突然想了解一下 STL 的 vector 调用内部的`swap()`时候效率如何，具体是怎么实现的，这样以后用的时候能知其所以然，比较安心。

~~然后还想顺便验证一下 STL 在传入不同类型的实参时，如何判断对应的指针、引用等的类型的。~~ 下次。

我先是翻了翻侯捷的『STL源码剖析』，vector 的部分并没有介绍`swap()`，也没有贴出代码。
然后决定找 vs 的 `vector` 文件看看。


### `template<class _Ty,	class _Alloc = allocator<_Ty> >`<br>`void vector<_Ty, _Alloc>::swap(_Myt& _Right)`

首先找到 vector 的 `swap()` 函数

    // 文件路径：Visual studio2015/VC/2015/include/vector   line 1545
    // typedef 
    void swap(_Myt& _Right)
      _NOEXCEPT_OP(_Alty::propagate_on_container_swap::value
        || _Alty::is_always_equal::value)
      {	// exchange contents with _Right
      if (this != &_Right)
        {	// (maybe) swap allocators, swap control information
        _Pocs(this->_Getal(), _Right._Getal());
        this->_Swap_all(_Right);
        _Swap_adl(this->_Myfirst(), _Right._Myfirst());
        _Swap_adl(this->_Mylast(), _Right._Mylast());
        _Swap_adl(this->_Myend(), _Right._Myend());
        }
      }

`_Pocs(this->_Getal(), _Right._Getal());`是用于交换 `_Alloc` 的，应该无关紧要就不看了。
`_Myfirst()`获取 vector 使用空间的头指针，`_Mylast()`获取目前使用空间的尾指针，`_Myend()`获取已开辟空间的尾指针。

定义在` `中，由` `创建。


而`_Swap_adl()`函数定义如下：

    template<class _Ty> inline
      void _Swap_adl(_Ty& _Left, _Ty& _Right)
        _NOEXCEPT_OP(_Is_nothrow_swappable<_Ty>::value)
      {	// exchange values stored at _Left and _Right, using ADL
      swap(_Left, _Right);
      }

再往下定位不到函数定义了，但是应该是个交换参数`_Left`和`_Right`的简单函数。所以这三次调用 `_Swap_adl()` 交换了 vector 的头尾指针和 end 指针。

那么还有`_Swap_all()`函数我们看看是做什么的。这个函数也是 `vector` 继承自 `_Vector_alloc` 的。拆开几层继承套娃之后，最后来到了这个地方。

    inline void _Container_base12::_Swap_all(_Container_base12& _Right)
      {	// swap all iterators
      //...debug info
      _Container_proxy *_Temp = _Myproxy;
      _Myproxy = _Right._Myproxy;
      _Right._Myproxy = _Temp;

      if (_Myproxy != 0)
        _Myproxy->_Mycont = (_Container_base12 *)this;
      if (_Right._Myproxy != 0)
        _Right._Myproxy->_Mycont = (_Container_base12 *)&_Right;
      }

根据注释，这个函数完成的功能是交换两个迭代器。但是对几个新出现的模板类功能和含义不是很清楚，我们再稍微看一下他们的定义。

    struct _Container_base12
      {	// store pointer to _Container_proxy
    public:
      _Container_proxy *_Myproxy;

      _Iterator_base12 **_Getpfirst() const
        {	// get address of iterator chain
        return (_Myproxy == 0 ? 0 : &_Myproxy->_Myfirstiter);
        }

      void _Swap_all(_Container_base12&);	// swap all iterators
      //...
      };

        // CLASS _Container_proxy
    struct _Container_proxy
      {	// store head of iterator chain and back pointer
      _Container_proxy()
        : _Mycont(0), _Myfirstiter(0)
        {	// construct from pointers
        }

      const _Container_base12 *_Mycont;
      _Iterator_base12 *_Myfirstiter;
      };

      struct _Iterator_base12
	      {	// store links to container proxy, next iterator
      public:
        	_Container_proxy *_Myproxy;
	        _Iterator_base12 *_Mynextiter;
          //...
        };

可以看到 `_Container_base12` 包装了一个 `_Container_proxy` 的指针，而 `_Container_proxy` 则包含一个迭代器链的头指针。同时，通过定义我们还能看到 `_Vector_iterator` 最后正是继承自 `_Iterator_base12`。（那部分代码懒得贴了）

那么通过浏览这几个结构体的定义，我们彻底弄清楚了`_Swap_all()`函数就是交换两个迭代器（通过交换`_Container_proxy`的指针）。但是 vector 内平时并不需要存放 iterator 类型，所以这个函数对 vector 应该也没起作用。

`_Myfirst, _Mylast, _Myend`三个指针是在 vector 的父类中创建的，与迭代器没什么关系。
这与『STL源码剖析』中介绍的 SGI STL 实现版本差不多。但是 SGI STL 版本可就清晰多了，

    //摘自『STL源码剖析』P117
    template <class T, class Alloc = alloc>
    class vector{
    public:
      typedef T value_type;
      typedef value_type* iterator;
    ...
    }

看看人家的可读性，这能节省多少理解时间啊。枯了。

本来一个迭代器应该包含一个数据和一个next指针，但是因为 vector 不需要 next 指针，所以这里 iterator 唯一的作用就是调用 `vector<T>::begin()` 或者 `vector<T>::end()` 时，把 `_Myfirst, _Mylast` 封装在 iterator 中返回。我们得到封装在 iterator 中的头尾指针之后，可以使用 iterator 提供的一系列重载运算符。但是话说，vector 的迭代器需要的操作无非就`operator*, operator->, operator++, operator--, operator+, operator-, operator+=, operator-=`，这些不是人家原生指针就支持的吗，所以 SGI 版本才直接这么用了啊，真是让人脑阔疼。

    iterator begin() _NOEXCEPT
      {	// return iterator for beginning of mutable sequence
      return (iterator(this->_Myfirst(), &this->_Get_data()));
      }
      
      //  typedef _Vector_alloc<_Vec_base_types<_Ty, _Alloc> > _Mybase;
      //	typedef typename _Mybase::iterator iterator;

      //  记得 iterator 来自这里
      template<class _Alloc_types>
        class _Vector_alloc
        {	// base class for vector to hold allocator
        typedef _Vector_iterator<_Vector_val<_Val_types> > iterator;
          ...
        };

      //  附上_Vector_iterator的构造函数
      //  _Vector_iterator的父类还有继承关系，不过构造函数大致与此相同
      _Vector_iterator(pointer _Parg, const _Container_base *_Pvector)
        : _Mybase(_Parg, _Pvector)
        {	// construct with pointer _Parg
        }

size的计算也跟iterator无关。

    size_type size() const _NOEXCEPT
      {	// return length of sequence
      return (this->_Mylast() - this->_Myfirst());
      }

如果是 List 或者 Map，iterator 的表现就会大不相同了。


### 结论

综上，我们知道了 vector 的 `swap()` 函数就像上面代码描述的那样，仅把两个 vector 的迭代器（指针）和一些控制信息进行了交换，没有重新分配内存，因此效率上是有保证的。

然后我们来验证一下

    int main()
    {
      vector<int> vec1 = { 1,3,5,7,9 }, vec2 = { 2,4,6,8,10 };

      cout << "vec1.begin(): " << vec1.begin()._Ptr << endl;
      cout << "vec1.end():   " << vec1.end()._Ptr << endl;
      cout << "vec2.begin(): " << vec2.begin()._Ptr << endl;
      cout << "vec2.end():   " << vec2.end()._Ptr << endl;

      cout << "After swap()\n";

      vec1.swap(vec2);

      cout << "vec1.begin(): " << vec1.begin()._Ptr << endl;
      cout << "vec1.end():   " << vec1.end()._Ptr << endl;
      cout << "vec2.begin(): " << vec2.begin()._Ptr << endl;
      cout << "vec2.end():   " << vec2.end()._Ptr << endl;
      return 0;
    }

  ![ouput](/img/in-post/post-vec-swap/vector-output.jpg)

能看到 vec1 和 vec2 的 `begin()` 和 `end()` 的指针值如同预期的那样交换了。 

顺带一提，vector 还重载了传入两个 vector 作为参数的`swap()`，最后还是调用上面的`swap()`成员函数。

    template<class _Ty,
      class _Alloc> inline
      void swap(vector<_Ty, _Alloc>& _Left, vector<_Ty, _Alloc>& _Right)
        _NOEXCEPT_OP(_NOEXCEPT_OP(_Left.swap(_Right)))
      {	// swap _Left and _Right vectors
      _Left.swap(_Right);
      }

那么，关于 vector 的介绍暂时就到这里，后续可能会把容器的坑再填一填，看一看 `sort()` 之类的常用算法（也可能不会）。

The End

-------------
