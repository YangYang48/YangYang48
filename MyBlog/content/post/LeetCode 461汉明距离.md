---
title: "LeetCode 461汉明距离"
date: 2022-10-26
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_461/leetcode461_thumb.jpg
coverImage: leetcode/leetcode_461/leetcode461_cover.png
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
- C++
- Java
showSocial: false
---

leetcode第461题，汉明距离。采用位运算来完成。

<!--more-->
# LeetCode 461汉明距离

两个整数之间的 [汉明距离](https://baike.baidu.com/item/汉明距离) 指的是这两个数字对应二进制位不同的位置的数目。

给你两个整数 `x` 和 `y`，计算并返回它们之间的汉明距离。



**示例 1：**

```text
输入：x = 1, y = 4
输出：2
解释：
1   (0 0 0 1)
4   (0 1 0 0)
       ↑   ↑
上面的箭头指出了对应二进制位不同的位置。
```



**示例 2：**

```text
输入：x = 3, y = 1
输出：1
```



**提示：**

- `0 <= x, y <= 231 - 1`



# 思路

使用位运算。

异或的特性，不同的位数都为1。

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_461/leetcode461_1.png" thumbnail="/leetcode/leetcode_461/leetcode461_1.png" title="">}}

可以把题目转换为

```text
两个数做了异或操作之后，求结果中有几个1。
```



# 解题

```java
class Solution {
    public int hammingDistance(int x, int y) {
        int distance = 0;
        //定义两数的异或变量
        int twoNumxor = x ^ y;

        while(twoNumxor != 0)
        {
            twoNumxor &= (twoNumxor - 1);
            distance++;
        }
        return distance;
    }
}
```

