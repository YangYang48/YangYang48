---
title: "字符串匹配 KMP(Knuth-Morris-Pratt)"
date: 2022-10-29
thumbnailImagePosition: left
thumbnailImage: leetcode/string/index/KMP/KMP_thumb.jpg
coverImage: leetcode/string/index/KMP/KMP_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- October
tags:
- 字符串匹配
- KMP
- C++
- Java
showSocial: false
---

经典的字符串匹配算法，KMP(Knuth-Morris-Pratt)算法。

<!--more-->
# 字符串匹配 KMP(Knuth-Morris-Pratt)

## 预先处理模式Pattern字符串

因此当已知字符串p时，先对它进行预处理。

理论：对p[1], p[1..2]，p[1..i]..p[1..n]逐一处理（1<= i <=n），找到每个p[1..i]的前后缀相交的最长字符串，得到这个字符串的长度x。然后存入kmp数组。（但这样花费太多时间）

**实际：求解Pattern的kmp或next_s的方法，使用的是递归的方法**

 

由此得到一套字符串P的最大共有字符串的长度集合，即kmp数组。

例如：

下图给出了关于模式 P = “ababababca”的kmp(即前缀和后缀集合，共有的字符串集合中，最长的字符串的的长度的值)的表格，称为部分(即部分字符串)匹配表（Partial Match Table）。

![img](https://img2018.cnblogs.com/blog/1276550/201910/1276550-20191015094602732-1555013603.png)



计算过程

kmp[0] = 0，匹配a 仅一个字符，前缀和后缀为空集，共有元素最大长度为 0；

kmp[1] = 0，匹配ab 的前缀 a，后缀 b，不匹配，共有元素最大长度为 0；

kmp[2] = 1，aba，前缀 **a** ab，后缀 ba **a**，共有元素最大长度为 1；

kmp[3] = 2，abab，前缀 a **ab** aba，后缀 bab **ab** b，共有元素最大长度为 2；

kmp[4] = 3，ababa，前缀 a ab **aba** abab，后缀 baba **aba** ba a，共有元素最大长度为 3；

kmp[5] = 4，ababab，前缀 a ab aba **abab** ababa，后缀 babab **abab** bab ab b，共有元素最大长度为 4；

kmp[6] = 5，abababa，前缀 a ab aba abab **ababa** ababab，后缀 bababa **ababa** baba aba ba a，共有元素最大长度为 5；

kmp[7] = 6，abababab，前缀 .. **ababab** ..，后缀 .. **ababab** ..，共有元素最大长度为 6；

kmp[8] = 0，ababababc，前缀和后缀不匹配，共有元素最大长度为 0；

kmp[9] = 1，ababababca，前缀 .. **a** ..，后缀 .. **a** ..，共有元素最大长度为 1；

之后就可以利用这个表了。



> 另外，比较到发生不匹配时，需要在匹配表找kmp[j-1], 所以为了编程方便，将kmp数组向后移动一个位置，产生一个next数组, 使用这个next数组即可。⚠️这本身只是为了让代码看起来更优雅。无其他意义。反而对初学者来说，不好理解。



> 优化代码使用next数组：
>
> next[j]是什么？
>
> next_s的意义：**代表当前字符j之前的字符串中，有多大长度的相同前缀后缀（可称为最长前缀/后缀）。**
>
> 例如：next[j] = k ,代表j之前的字符串中，最大前缀/后缀的长度为k。
>
>  
>
> 本例子next_s = [-1, 0,0,1,2,3,4,0], 其实就是在kmp数组头部插入了一个元素-1, 或者说整体向后移动一个位置。
>
> 因为代码j = kmp[j - 1]，所以使用j = next_s[j]，但需要对第一个字符就不匹配的情况改代码：
>
> 特殊情况：
>
> *pattern只有一个字母"x"时，不匹配，我们设置next[0]等于-1。*
>
> *当p = 'x'， p[0]不等于text[0]， 只能是穷举法了，每轮i+1，j不变。*
>
> *因此要修改一下条件判断 ： if j == -1 || text[j] == pattern[j]*
>
>  
>
> - *因为j等于-1，所以判断true, 于是i和j都加+1。 那么下一轮i =1, j= 0, 继续比较。*
> - *当j = 0的情况时，如果不匹配，那么j = next_s[j] ,即j等于-1。 然后下一轮 , 又是true。*