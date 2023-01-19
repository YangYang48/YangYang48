---
title: "Leetcode 88合并两个有序数组"
date: 2022-08-15T08:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_88/leetcode88_thumb.jpg
coverImage: leetcode/leetcode_88/leetcode88_cover.jpg
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
- 双指针
- 数组
showSocial: false
---

leetcode第88题，合并两个有序数组。采用双指针的算法来完成。

<!--more-->
# Leetcode 88合并两个有序数组

给你两个按 **非递减顺序** 排列的整数数组 `nums1` 和 `nums2`，另有两个整数 `m` 和` n` ，分别表示 `nums1` 和 `nums2` 中的元素数目。

请你 合并 `nums2` 到 `nums1` 中，使合并后的数组同样按 **非递减顺序** 排列。

注意：最终，合并后数组不应由函数返回，而是存储在数组 `nums1` 中。为了应对这种情况，`nums1` 的初始长度为 `m + n`，其中前 `m` 个元素表示应合并的元素，后` n` 个元素为 0 ，应忽略。`nums2` 的长度为 `n` 。

 

**示例 1：**

```
输入：nums1 = [1,2,3,0,0,0], m = 3, nums2 = [2,5,6], n = 3
输出：[1,2,2,3,5,6]
解释：需要合并 [1,2,3] 和 [2,5,6] 。
合并结果是 [1,2,2,3,5,6] ，其中斜体加粗标注的为 nums1 中的元素。
```

**示例 2：**

```
输入：nums1 = [1], m = 1, nums2 = [], n = 0
输出：[1]
解释：需要合并 [1] 和 [] 。
合并结果是 [1] 。
```

**示例 3：**

```
输入：nums1 = [0], m = 0, nums2 = [1], n = 1
输出：[1]
解释：需要合并的数组是 [] 和 [1] 。
合并结果是 [1] 。
注意，因为 m = 0 ，所以 nums1 中没有元素。nums1 中仅存的 0 仅仅是为了确保合并结果可以顺利存放到 nums1 中。
```



**提示：**

- `nums1.length` == `m` + `n`
- `nums2.length` == `n`
- 0 <= `m`, `n` <= 200
- 1 <= `m` + `n` <= 200
- -109 <= `nums1[i]`, `nums2[j]` <= 109



**进阶：**你可以设计实现一个时间复杂度为 `O(m + n)` 的算法解决此问题吗？



# 思路

可以直接利用函数的快速排序。

1. 将`nums2`的元素放到`nums1`中，形成一个新的只有`nums1`的序列
2. 然后对`nums`的序列快速排序



# 解题

```java
class Solution {
    //合并成一个数组，然后快速排序
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        for(int i = 0; i < n; i++)
        {
            nums1[m + i] = nums2[i];
        }
        Arrays.sort(nums1);
    }
}
```



其实还可以利用递增的关系（双指针）

1. 每个数组头部头有一个指针指向第一位，创建一个临时数组
2. 比较双指针所在的值，会出现三种结果。如果相等，默认取`nums2`的元素，并且指针往后移动一位；如果不相等，取指针指向小的那位，并且指针往后移动一位。
3. 直到两个指针全部移动完成



```java
class Solution {
    //双指针做法，增加一个临时数组
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int k = m + n;
        int[] tmp = new int[k];
        for(int index = 0, nums1Index = 0, nums2Index = 0; index < k; index++)
        {
            if(nums1Index >= m)//nums1数组元素取完
            {
                tmp[index] = nums2[nums2Index++];
            }else if(nums2Index >= n)//nums2数组元素取完
            {
                tmp[index] = nums1[nums1Index++];
            }else if(nums1[nums1Index] < nums2[nums2Index])
            //nums1元素小于nums2元素
            {
                tmp[index] = nums1[nums1Index++];
            }else//nums2元素小于nums1元素
            {
                tmp[index] = nums2[nums2Index++];
            }
        }
        //重新赋值给nums1
        for(int i = 0; i < k; i++)
        {
            nums1[i] = tmp[i];
        }
    }

}
```



由于这里`nums1`存在一个`m`+`n`的元素存放，那么其实可以利用上这个空间，不需要临时`tmp`数组

还是利用双指针做法

1. 每个数组头尾部有一个指针指向最后一位
2. 比较双指针所在的值，会出现三种结果。如果相等，默认取`nums2`的元素，并且指针往前移动一位；如果不相等，取指针指向大的那位，并且指针往前移动一位。把元素存放到`nums1`的数组中
3. 直到两个指针全部移动完成



```java
class Solution {
    //双指针做法，利用nums1的空间，从后往前降序排列
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int k = m + n;
        for(int index = k-1, nums1Index = m - 1, nums2Index = n -1;
            index >= 0; index--)
        {
            if(nums1Index < 0)//nums1元素已经取完
            {
                nums1[index] = nums2[nums2Index--];
            }else if(nums2Index < 0)//nums2元素已经取完
            {
                break;
            }else if(nums1[nums1Index] > nums2[nums2Index])
            //nums1元素值大于nums2,取nums1值
            {
                nums1[index] = nums1[nums1Index--];
            }else
            {
                //nums2元素值大于等于nums1，取nums2值
                nums1[index] = nums2[nums2Index--];
            }
        }
    }

}
```

