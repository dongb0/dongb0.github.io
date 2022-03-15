---
layout: post
title: "[c++] 自定义Key类型哈希函数"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
  - programming
  - c++
---

> 之前搜索过好几次都记不住c++自定义类型用作unordered_map/unordered_set键时，哈希函数应该怎么写，java的倒是记得要重载hash和equals两个函数。所以还是一并做个总结好了。

### C++自定义键值类型

其实是先搜到了[这个博客](1)，感觉已经介绍得足够好了。总结来说其实也是需要hash函数和equals函数的，但是我的自定义类型一般用默认的比较函数就够了，所以只要重载一下hash函数。

// 函数对象

不过这引出了我新的疑问，C++中 `==` 运算符，对于自定义的struct或者class，是如何运作的。

首先是 unordered_map 模板的定义：

    // CLASS TEMPLATE unordered_map
    template <class _Kty, class _Ty, class _Hasher = hash<_Kty>, class _Keyeq = equal_to<_Kty>,
        class _Alloc = allocator<pair<const _Kty, _Ty>>>
    class unordered_map : public _Hash<_Umap_traits<_Kty, _Ty, _Uhash_compare<_Kty, _Hasher, _Keyeq>, _Alloc, false>>

这里我们关心的当然是第三个参数 `hash<_Kty>` 和第四个参数 `equal_to<_Kty>` 的默认实现。

//  `hash<_Kty>` 有点难懂

而 `equal_to<_Kty>` 的实现比较直观，重载的 `()` 中（也许应该叫函数对象）直接通过 `==` 比较两个实例是否相等。

    // STRUCT TEMPLATE equal_to
    template <class _Ty = void>
    struct equal_to {
        _CXX17_DEPRECATE_ADAPTOR_TYPEDEFS typedef _Ty _FIRST_ARGUMENT_TYPE_NAME;
        _CXX17_DEPRECATE_ADAPTOR_TYPEDEFS typedef _Ty _SECOND_ARGUMENT_TYPE_NAME;
        _CXX17_DEPRECATE_ADAPTOR_TYPEDEFS typedef bool _RESULT_TYPE_NAME;

        constexpr bool operator()(const _Ty& _Left, const _Ty& _Right) const {
            return _Left == _Right;
        }
    };





[1]: https://blog.csdn.net/y109y/article/details/82669620