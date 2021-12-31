---
layout: post
title: "[algorithm/hash] 加密哈希与非加密哈希"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
---

> 之前大概知道MD5和SHA-1/SHA-2都是某种哈希算法，但是看了实现感觉跟数据结构课上学到的哈希函数不是同一个东西，最近才知道哈希算法分为加密/非加密哈希，故挖此坑。~~（另外之前略微了解过几种相似哈希算法，不知道是不是还能开个新坑）~~

## 二者的区别（我猜的）

首先我们要明确的一点是，加密哈希和非加密哈希都是哈希算法，都是接收原始数据输入，输出一个哈希值。区别在于，加密哈希算法必须要满足几点安全性要求，才能被称为一个加密哈希算法[^1]：

0. 同样的输入必须计算得到相同的hash输出
1. 难以找到两组不同的数据 m1，m2， 使得 hash(m1) == hash(m2)
2. 难以通过 hash(m1) 倒推得到原始数据 m1
3. 对原始数据内容的微小改变，会计算得到一个完全不相干(变化巨大)的哈希值

为了满足上述的要求，加密哈希算法需要设计更复杂的计算步骤，因此通常计算速度也比非加密哈希要慢。最理想的加密哈希算法当然希望能够将第1和第2点中的“难以”替换成“无法”，但是目前应该还未设计出这样完美的算法，而是将出现这种情况的概率降到尽可能小，小到能够近似的认为满足上述安全性要求。

而相对的，非加密哈希算法可以把关注点放在如何减少计算开销上，计算过程往往相对简单，因此比加密哈希算法更适合用在hash table、校验和等场景。当然你在hash table中使用加密哈希算法也不是不可以，毕竟都是哈希函数，只是插入和查找的性能会明显下降而已。


## 非加密哈希

非加密哈希大致分类参照[]某博客和wiki hash function，可分为***种类型。在此将介绍FNV，siphash，murmur3等几种常用的 （听说wyHash性能不错)


### FNV 

  
  python 3.4之前使用  ，之后使用SipHash [] /////引证

  实现比较简单，wiki上FNV-1的伪代码如下[^2]：

      algorithm fnv-1 is
      hash := FNV_offset_basis

      for each byte_of_data to be hashed do
          hash := hash × FNV_prime
          hash := hash XOR byte_of_data

      return hash 

  简单用C++实现32位的FNV_1函数大概如下：
5
      const int FNV_prime_32 = 16777619;
      unsigned int FNV_1(vector<char> &data) {
        unsigned int hash = 2166136261; 
        for (const char& d : data) {
          hash ^= d;
          hash *= FNV_prime_32;
        }
        return hash;
      }


FNV目前有3个版本，分别是FNV-0，FNV-1，FNV-1a。其中FNV-0与FNV-1的差别只在于FNV_0的FNV_offset_basis初值为0，但是并不推荐使用FNV-0，因为如果输入的数据全为0，FNV-0得到的hash值也为0。

目前FNV-0的唯一用途是用于计算任意的一组数据来获取FNV-1和FNV-1a的offset_basis初值（wiki上的示例是计算字符串`"chongo <Landon Curt Noll> /\../\"`的哈希，得到的正是2166136261）。而之所以使用这串字符的理由是[^3]

> That string was used because the person testing FNV with non-zero  
   offset_basis values was looking at an email message from Landon

其他不同位数的FNV_offset_basis和FNV_prime详见[wiki](1)中的表。

参照[FNV文档](2)中关于FNV安全性的章节，其中解释FNV算法是非加密算法的原因有3点：

1. A cryptographic hash should not have a state in which it can stick for a plausible input pattern，可能是指加密哈希算法在接受（处理）某种模式的输入时不应一直保持同一个状态（同一个哈希值），而FNV-0在处理全0的输入时，哈希值一直为0，直到读取到非0数据。
2. Diffusion？/////待更新
3. 加密哈希算法应该使暴力破解原数据这一过程十分耗时，但FNV并不能满足，即通过暴力搜索依然能较快的从哈希值得到原始数据


//  超过64位的用C++怎么实现？返回的应该需要是一个char数组，但是乘法要怎么做？8位的乘法好像还好，可以用int记录进位，最后的

### siphash


### murmur3


### 各种非加密哈希算法的碰撞率

## 加密哈希

包括MD5，SHA族的哈希算法，，，等都属于



[1]: https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function#FNV_hash_parameters
[2]: https://datatracker.ietf.org/doc/html/draft-eastlake-fnv-17.html#page-108

[^1]: [加密哈希算法 - wiki](https://en.wikipedia.org/wiki/Cryptographic_hash_function)
[^2]: [FNV-1a - wiki](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function#FNV-1_hash)
[^3]: [FNV 计算offset_basis字符串由来](https://datatracker.ietf.org/doc/html/draft-eastlake-fnv-17.html#section-2.2)

