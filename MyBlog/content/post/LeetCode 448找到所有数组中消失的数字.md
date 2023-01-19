---
title: "LeetCode 448找到所有数组中消失的数字"
date: 2022-08-15T10:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_448/leetcode448_thumb.jpg
coverImage: leetcode/leetcode_448/leetcode448_cover.png
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

leetcode第448题，找到所有数组中消失的数字。采用数组元素改变(加或者减)的算法来完成。

<!--more-->
# LeetCode 448找到所有数组中消失的数字

给你一个含 `n` 个整数的数组 `nums` ，其中 `nums[i]` 在区间 [1, n] 内。请你找出所有在 [1, n] 范围内但没有出现在 `nums` 中的数字，并以数组的形式返回结果。

 

**示例 1：**

```
输入：nums = [4,3,2,7,8,2,3,1]
输出：[5,6]
```

**示例 2：**

```
输入：nums = [1,1]
输出：[2]
```



**提示：**

- `n` == `nums.length`
- 1 <= `n` <= 105
- 1 <= `nums[i]` <= `n`

**进阶：**

你能在不使用额外空间且时间复杂度为 `O(n)` 的情况下解决这个问题吗? 你可以假定返回的数组不算在额外空间内。



# 思路

给数组中的元素加某个值或者把数组中的元素变成负数

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_448/leetcode448_1.png" thumbnail="/leetcode/leetcode_448/leetcode448_1.png" title="">}}

1. 移动指针从数组头部开始，去除指针所指的元素，让其减1使得变成从[0, n-1]的下标。
2. 找到指针所指的元素相同的下标，把下标所指的元素值改成负数。
3. 从数组头部开始遍历整个数组，每次找到对应指针所指的元素相同的下标所指的元素，改成负数。如果已经改成负数，则忽略。
4. 再次遍历整个数组，查看没有成为负数的下标，这些值在数组中没有出现过，对应下标+1则为[1,n]的返回结果



# 解题

```java
class Solution {
    //对整个数组相加或者相减，第二次遍历找出没有发生变化的元素下标
    public List<Integer> findDisappearedNumbers(int[] nums) {
        int n = nums.length;
        for(int i = 0; i < n; i++)
        {
            int x = (nums[i] - 1) % n;//这里的取余操作还原本来的值
            nums[x] += n;
        }
        List<Integer> res = new ArrayList<Integer>();
        for(int i = 0; i < n; i++)
        {
            if(nums[i] <= n)
            {
                res.add(i + 1);
            }
        }
        return res;
    }
}
```

