---
title: "LeetCode 338比特位计数"
date: 2022-10-25
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_338/leetcode338_thumb.jpg
coverImage: leetcode/leetcode_338/leetcode338_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- October
tags:
- 位运算
- 奇偶性
- C++
- Java
showSocial: false
---

leetcode第338题，比特位计数。采用位运算或者奇偶性来完成。

<!--more-->
# LeetCode 338比特位计数

给你一个整数 `n` ，对于 `0 <= i <= n` 中的每个 `i` ，计算其二进制表示中 **`1` 的个数** ，返回一个长度为 `n + 1` 的数组 `ans` 作为答案。



**示例 1：**

```text
输入：n = 2
输出：[0,1,1]
解释：
0 --> 0
1 --> 1
2 --> 10
```



**示例 2：**

```text
输入：n = 5
输出：[0,1,1,2,1,2]
解释：
0 --> 0
1 --> 1
2 --> 10
3 --> 11
4 --> 100
5 --> 101
```



**提示：**

- `0 <= n <= 105`



**进阶：**

- 很容易就能实现时间复杂度为 `O(n log n)` 的解决方案，你可以在线性时间复杂度 `O(n)` 内用一趟扫描解决此问题吗？
- 你能不使用任何内置函数解决此问题吗？（如，C++ 中的 `__builtin_popcount` ）



# 思路

使用位运算。

```tetx
//清除最低位的1
x = x & (x - 1)
```

`15跟14相比相差最低位的1`

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_338/leetcode338_1.png" thumbnail="/leetcode/leetcode_338/leetcode338_1.png" title="">}}

`14跟12相比相差最低位的1`

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_338/leetcode338_2.png" thumbnail="/leetcode/leetcode_338/leetcode338_2.png" title="">}}

`12跟8相比相差最低位的1`

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_338/leetcode338_3.png" thumbnail="/leetcode/leetcode_338/leetcode338_3.png" title="">}}

`8跟0相比相差最低位的1`

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_338/leetcode338_4.png" thumbnail="/leetcode/leetcode_338/leetcode338_4.png" title="">}}

也就是说实际上15位中有几个1，可以是等价于14位中有几个1再加上1；同理14位中有几个1，可以是等价于12位中有几个1再加上1；12位中有几个1，可以是等价于8位中有几个1再加上1；8位中有几个1，可以是等价于0位中有几个1再加上1。



# 解题

```java
class Solution {
    public int[] countBits(int n) {
        int[] bitArray = new int[n + 1];
        //初始值为0
        bitArray[0] = 0;
        for(int i = 1; i < n + 1; i++)
        {
            //有递归的思想
            bitArray[i] = bitArray[i & (i - 1)] + 1;
        }
        return bitArray;
    }
}
```



# 思路2

可以用奇偶的特性

```c++
//当奇数的时候
//9 1001
//8 1000
bitArray[i] = bitArray[i - 1];
//当偶数的时候
//例如
//8 1000
//4 0100
//2 0010
//1 0001
bitArray[i] = bitArray[i >> 1];
```



# 解题

```java
class Solution {
    public int[] countBits(int n) {
        int[] bitArray = new int[n + 1];
        //初始值为0
        bitArray[0] = 0;
        for(int i = 1; i < n + 1; i++)
        {
            //使用奇偶性
            bitArray[i] = ((i & 1) == 1) ?
                          bitArray[i - 1] + 1 :
                          bitArray[i >> 1];
        }
        return bitArray;
    }
}
```

