---
title: "LeetCode 121买卖股票的最佳时机"
date: 2022-10-30T08:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_121/leetcode121_thumb.jpg
coverImage: leetcode/leetcode_121/leetcode121_cover.jpg
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
- 双指针
- C++
- Java
showSocial: false
---

leetcode第121题，买卖股票的最佳时机。采用动态规划或者类似双指针来完成。

<!--more-->
# LeetCode 121买卖股票的最佳时机

给定一个数组 `prices` ，它的第 `i` 个元素 `prices[i]` 表示一支给定股票第 `i` 天的价格。

你只能选择 **某一天** 买入这只股票，并选择在 **未来的某一个不同的日子** 卖出该股票。设计一个算法来计算你所能获取的最大利润。

返回你可以从这笔交易中获取的最大利润。如果你不能获取任何利润，返回 `0` 。



**示例 1：**

```text
输入：[7,1,5,3,6,4]
输出：5
解释：在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。
注意利润不能是 7-1 = 6, 因为卖出价格需要大于买入价格；同时，你不能在买入前卖出股票。
```



**示例 2：**

```text
输入：prices = [7,6,4,3,1]
输出：0
解释：在这种情况下, 没有交易完成, 所以最大利润为 0。
```



**提示：**

- `1 <= prices.length <= 105`
- `0 <= prices[i] <= 104`



# 思路

可以跟之前的136题一样，使用暴力求解，但是不推荐。

这里可以使用动态规划。

这里可以看做把买股票作为一种亏损，买股票作为一种盈利。

```java
//定义DP数组，其中i代表的是天数，j可以有0，代表不持有股票/或者股票卖出，j可以为1代表目前还持有股票。
//DP[i][0],代表当前第i天不持有股票/或者股票卖出的总收益
//DP[i][1],代表当前第i天持有股票的总收益(持有肯定是负的)
DP[i][j]
DP[i][0] = Math.max(DP[i - 1][0], DP[i - 1][1] + nums[i]);
DP[i][1] = Math.max(DP[i - 1][1], -nums[i]);
```

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_121/leetcode121_1.png" thumbnail="/leetcode/leetcode_121/leetcode121_1.png" title="">}}

# 解题

```java
class Solution {
    public int maxProfit(int[] prices) {
        //判定是否满足要求
        if(prices.length < 2)
        {
            return 0;
        }

        //用动态规划的方式，并且构造一个二位的DP数组
        int[][] DP = new int[prices.length][2];
        //初始化DP数组
        DP[0][0] = 0;
        DP[0][1] = -prices[0];//股票持有，那么就是负数
        //从第二天开始遍历
        for(int i = 1; i < prices.length; i++)
        {
            DP[i][0] = Math.max(DP[i - 1][0], DP[i - 1][1] + prices[i]);
            DP[i][1] = Math.max(DP[i - 1][1], -prices[i]);
        }
        return DP[prices.length - 1][0];
    }
}
```

开始遍历数组来形成DP

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_121/leetcode121_2.png" thumbnail="/leetcode/leetcode_121/leetcode121_2.png" title="">}}

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_121/leetcode121_3.png" thumbnail="/leetcode/leetcode_121/leetcode121_3.png" title="">}}

当然，买卖股票其实就是找到一个低位点，然后看低位点的某一高点的距离，距离最大即为最佳买卖点。

# 解题优化

```java
class Solution {
    public int maxProfit(int[] prices) {
        //判定是否满足要求
        if(prices.length < 2)
        {
            return 0;
        }
        //类似双指针，使用遍历数组，指针A作为最小值，指针B用于和指针A保持最大距离
        int min = prices[0];
        int max = 0;
        for(int i = 1; i < prices.length; i++)
        {
            if(min > prices[i])
            {
                min = prices[i];
            }else
            {
                max = Math.max(max, prices[i] - min);
            }
        }
        return max;
    }
}
```

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_121/leetcode121_4.png" thumbnail="/leetcode/leetcode_121/leetcode121_4.png" title="">}}