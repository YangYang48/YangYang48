---
title: "LeetCode 136只出现一次的数字"
date: 2022-10-24
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_136/leetcode136_thumb.jpg
coverImage: leetcode/leetcode_136/leetcode136_cover.jpg
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

leetcode第136题，只出现一次的数字。采用位运算来完成。

<!--more-->
# LeetCode 136只出现一次的数字

给定一个**非空**整数数组，除了某个元素只出现一次以外，其余每个元素均出现两次。找出那个只出现了一次的元素。

**说明：**

你的算法应该具有线性时间复杂度。 你可以不使用额外空间来实现吗？

**示例 1:**

```text
输入: [2,2,1]
输出: 1
```

**示例 2:**

```text
输入: [4,1,2,1,2]
输出: 4
```



# 思路

使用位运算。

题干中提到除了某个元素只出现一次以外，其余每个元素均出现两次。

> 在异或运算的时候，相同的两个数异或一定为0



# 解题

```java
class Solution {
    public int singleNumber(int[] nums) {
        int res = 0;
        for(int num : nums)
        {
            res = res ^ num;
        }
        return res;
    }
}
```

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_136/leetcode136_1.png" thumbnail="/leetcode/leetcode_136/leetcode136_1.png" title="">}}
