---
title: "LeetCode 1984学生分数的最小差值"
date: 2022-08-27
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_1984/leetcode1984_thumb.jpg
coverImage: leetcode/leetcode_1984/leetcode1984_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- August 
tags:
- leetcode
- C++
- Java
- 数组
showSocial: false
---

leetcode第1984题，学生分数的最小差值。采用排序+滑动窗口的算法来完成。

<!--more-->
# LeetCode 1984学生分数的最小差值

给你一个 **下标从 0 开始** 的整数数组 `nums` ，其中 `nums[i]` 表示第 `i` 名学生的分数。另给你一个整数 `k` 。

从数组中选出任意 `k` 名学生的分数，使这 `k` 个分数间 **最高分** 和 **最低分** 的 **差值** 达到 **最小化** 。

返回可能的 **最小差值** 。



**示例 1：**

```
输入：nums = [90], k = 1
输出：0
解释：选出 1 名学生的分数，仅有 1 种方法：
- [90] 最高分和最低分之间的差值是 90 - 90 = 0
可能的最小差值是 0
```

**示例 2：**

```
输入：nums = [9,4,1,7], k = 2
输出：2
解释：选出 2 名学生的分数，有 6 种方法：
- [9,4,1,7] 最高分和最低分之间的差值是 9 - 4 = 5
- [9,4,1,7] 最高分和最低分之间的差值是 9 - 1 = 8
- [9,4,1,7] 最高分和最低分之间的差值是 9 - 7 = 2
- [9,4,1,7] 最高分和最低分之间的差值是 4 - 1 = 3
- [9,4,1,7] 最高分和最低分之间的差值是 7 - 4 = 3
- [9,4,1,7] 最高分和最低分之间的差值是 7 - 1 = 6
可能的最小差值是 2
```



**提示：**

- `1 <= k <= nums.length <= 1000`
- `0 <= nums[i] <= 105`



# 思路

进行排序+滑动窗口

首先将无序数组进行从小到大排序

开始遍历数组，使用一个大小固定为 `k`的滑动窗口在 `nums`上进行遍历，那么窗口中`k`名学生的最高分和最低分的差值在任意一次都是固定值，找到这个最小值即可



# 解题

```java
class Solution {
    //排序加滑动窗口
    public int minimumDifference(int[] nums, int k) {
        int len = nums.length;
        int min = Integer.MAX_VALUE;
        Arrays.sort(nums);
        for(int i = 0; i + k -1 < len; i++)
        {
            min = Math.min(min, nums[i + k - 1] - nums[i]);
        }
        return min;
    }
}
```

