---
title: "LeetCode 53最大子数组和"
date: 2022-10-30T07:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_53/leetcode53_thumb.jpg
coverImage: leetcode/leetcode_53/leetcode53_cover.jpg
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
- C++
- Java
showSocial: false
---

leetcode第53题，最大子数组和。采用动态规划来完成。

<!--more-->
# LeetCode 53最大子数组和

给你一个整数数组 `nums` ，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

**子数组** 是数组中的一个连续部分。



**示例 1：**

```text
输入：nums = [-2,1,-3,4,-1,2,1,-5,4]
输出：6
解释：连续子数组 [4,-1,2,1] 的和最大，为 6 。
```



**示例 2：**

```text
输入：nums = [1]
输出：1
```

**示例 3：**

```text
输入：nums = [5,4,-1,7,8]
输出：23
```

**提示：**

- `1 <= nums.length <= 105`
- `-104 <= nums[i] <= 104`



**进阶：**如果你已经实现复杂度为 `O(n)` 的解法，尝试使用更为精妙的 **分治法** 求解。



# 思路

可以暴力求解，但是时间复杂度是o(n^2^)，不推荐

可以用动态规划的思想。

首先，这个的选择是当前元素是否加到前面子序列中，而这个选择会影响后面元素的大小。

```c++
//DP[i]的定义，包括下标i之前的最大连续子序列和
DP[i] = max(nums[i], (DP[i - 1] + nums[i]));
```

遍历一次就可以得到最大的子序列和，时间复杂度o(n)。

# 解题

```java
class Solution {
    public int maxSubArray(int[] nums) {
        //动态规划的方式求解
        int numsLen = nums.length;
        int[] DP = new int[numsLen];
        //第一个元素，直接赋值即可，不需要在动态规划
        DP[0] = nums[0];
        //定一个变量收集最大值
        int max = DP[0];
        for(int i = 1; i < numsLen; i++)
        {
            DP[i] = Math.max(nums[i],
                             nums[i] + DP[i - 1]);
            if(max < DP[i])
            {
                max = DP[i];
            }
        }
        return max;
    }
}
```

改进版本，比一定需要`DP`数组，这个的`DP`数组元素只跟`nums`当前元素和`DP`的过去元素相关。将`nums`和`DP`合并成一个数组，也不会因此影响。


{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_53/leetcode53_1.png" thumbnail="/leetcode/leetcode_53/leetcode53_1.png" title="">}}

```c++
DP[i] = max(nums[i], (DP[i - 1] + nums[i]));
```

# 解题改进

```java
class Solution {
    public int maxSubArray(int[] nums) {
        //动态规划的方式求解,只需要使用一个nums数组
        //定一个变量收集最大值
        int max = nums[0];
        int pre = max;
        for(int i = 1; i < nums.length; i++)
        {
            pre = Math.max(nums[i],
                           nums[i] + pre);
            if(max < pre)
            {
                max = pre;
            }
        }
        return max;
    }
}
```

