---
title: "寻X的平方根"
date: 2022-11-01T08:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/others/others3/others3_thumb.jpg
coverImage: leetcode/others/others3/others3_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- November
tags:
- leetcode
- C++
- Java
- 二分查找
- 牛顿迭代
showSocial: false
---

X的平方根。采用二分查找或者牛顿迭代方法。

<!--more-->
# X的平方根

在不使用`sqrt(x)`函数的情况下，得到x的平方根的整数部分

**示例：**

```text
输入：141
输出：11
```

# 思路

这类题目通常用暴力求解是比较快速的，但是这里涉及到(0, x-1)的区间，可以考虑使用**二分查找**、**牛顿迭代**方式，会比暴力求解更加快速



# 解题

`二分法`

解法一：二分查找
`x`的平方根肯定在`(0, x)`之间，使用二分查找定位该数字，该数字的平方一定是最接近x的，平方值如果大于`x`、则往左边找，如果小于等于`x`则往右边找
找到0和`x`的最中间的数`m`,如果`m * m > x`,则`m`取`x / 2`到`x`的中间数字，直到`m * m < X`,`m`则为平方根的整数部分如果`m * m <= x`,则取0到`x / 2`的中间值，知道两边的界限重合，找到最大的整数，

则为x平方根的整数部分时间复杂度：`O(IogN)`

```java
public static int binarySearch(int num)
{
    int index = -1;
    int left = 0;
    int right = num;
    while(left <= right)
    {
        int mid = (left + right) / 2;
        if(mid * mid <= num)
        {
            index = mid;
            left = mid + 1;
        }else
        {
            right = mid - 1;
        }
    }
    return index;
}
```

`牛顿迭代`

1. 通过两个分解的因子，相加起来取平均值，将这个平均值方和目标数比较。
2. 如果这个平均值没有满足要求，那么把平均值带入原式中，求的目标数与上一步的平均值相除的结果和上一步平均值结果之和的平均值。生成一个新的平均值，然后再次求解平均值方和目标数的对比。
3. 以此类推，直到得到最终结果

```java
public static int newTon(int num)
{
    if(0 == num)
    {
        return 0;
    }
    return (int)sqrt(num, num);
}

public static double sqrt(double i, int num)
{
    double res = (i + num / i) / 2;
    if(res == i)
    {
        return i;
    }else
    {
        return sqrt(res, num);
    }
}
```

