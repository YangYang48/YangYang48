---
title: "LeetCode 509斐波那契数"
date: 2022-10-30T06:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_509/leetcode509_thumb.jpg
coverImage: leetcode/leetcode_509/leetcode509_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- October
tags:
- 动态规划
- 循环
- 递归
- C++
- Java
showSocial: false
---

leetcode第509题，斐波那契数。一般可以直接使用循环或者递归的方式(类似一维DP数组)，这里采用动态规划来完成。

<!--more-->
# LeetCode 509斐波那契数

**斐波那契数** （通常用 `F(n)` 表示）形成的序列称为 **斐波那契数列** 。该数列由 `0` 和 `1` 开始，后面的每一项数字都是前面两项数字的和。也就是：

```text
F(0) = 0，F(1) = 1
F(n) = F(n - 1) + F(n - 2)，其中 n > 1
```

给定 `n` ，请计算 `F(n)` 。



**示例 1：**

```text
输入：n = 2
输出：1
解释：F(2) = F(1) + F(0) = 1 + 0 = 1
```



**示例 2：**

```text
输入：n = 3
输出：2
解释：F(3) = F(2) + F(1) = 1 + 1 = 2
```



**示例 3：**

```text
输入：n = 4
输出：3
解释：F(4) = F(3) + F(2) = 2 + 1 = 3
```



**提示：**

- `0 <= n <= 30`



# 思路

这道题可以用循环或者是递归的方式。

但是也可以通过动态规划来操作，如何构建一个让前面的状态去影响后面的状态的动态规划数组（`DP`）。

```c++
F(n) = F(n - 1) + F(n - 2);//其中n > 1
```



# 解题

```java
class Solution {
    public int fib(int n) {
        //动态规划的方式
        if(n <= 1)
        {
            return n;
        }
        int[] DP = new int[n + 1];
        DP[0] = 0;
        DP[1] = 1;
        for(int i = 2; i < n + 1; i++)
        {
            DP[i] = DP[i - 1] + DP[i - 2];
        }
        return DP[n];
    }
}
```

