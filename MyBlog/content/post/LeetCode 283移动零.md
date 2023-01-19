---
title: "LeetCode 283移动零"
date: 2022-08-15T09:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_283/leetcode283_thumb.jpg
coverImage: leetcode/leetcode_283/leetcode283_cover.jpg
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

leetcode第283题，移动零。采用双指针的算法来完成。

<!--more-->
# LeetCode 283移动零

给定一个数组 `nums`，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。

**请注意** ，必须在不复制数组的情况下原地对数组进行操作。

 

**示例 1:**

```c
输入: nums = [0,1,0,3,12]
输出: [1,3,12,0,0]
```

**示例 2:**

```
输入: nums = [0]
输出: [0]
```



**提示:**

- 1 <= `nums.length` <= 104
- -231 <= `nums[i]` <= 231 - 1

**进阶：**你能尽量减少完成的操作次数吗？



# 思路（双指针）

1. 一个指针指向`nums`数组的头部，用于遍历数组，一个指针指向`nums`数组的头部，用于非0元素的指向。
2. 用于遍历数组的指针指向了非0元素，那么将直接赋值到非0元素指向的指针，并且两个指针同时往后移动一位。
3. 用于遍历数组的指针指向了0元素，那么自己往后移动一个元素。
4. 如果用于遍历数组的指针遍历完了数组，那么对于非0元素的指向的指针之后在数组的元素，全部赋值为0。



# 解题

```java
class Solution {
    //使用双指针
    public void moveZeroes(int[] nums) {
        if(nums.length == 0)
        {
            return;
        }

        //第一次遍历，j指针记录非0的个数，只要是非0统统都赋值给nums[j]
        int j = 0;//用于记录非0元素的指针
        for(int i = 0; i < nums.length; i++)
        {
            if(nums[i] != 0)
            {
                //如果是非0元素，比较是否是双指针指向同一元素，如果是那么不交换
                if(i != j)
                {
                    nums[j] = nums[i];
                }
                j++;
            }
        }
        //非0元素遍历完，剩下全是0
        for(int i = j; i < nums.length; i++)
        {
            nums[i] = 0;
        }

    }
}
```



# 思路（双指针+交换）

这里参考了快速排序的思想，快速排序首先要确定一个待分割的元素做中间点x，然后把所有小于等于x的元素放到x的左边，大于x的元素放到其右边。
这里我们可以用0当做这个中间点，把不等于0(注意题目没说不能有负数)的放到中间点的左边，等于0的放到其右边。
这的中间点就是0本身，所以实现起来比快速排序简单很多，我们使用两个指针i和j，只要`nums[i]`!=0，我们就交换`nums[i]`和`nums[j]`



```java
class Solution {
    //双指针+交换
	public void moveZeroes(int[] nums) {
		if(nums == null) {
			return;
		}
		//两个指针i和j
		int j = 0;
		for(int i = 0; i < nums.length; i++) {
			//当前元素!=0，就把其交换到左边，等于0的交换到右边
			if(nums[i] != 0) {
                //如果是非0元素，比较是否是双指针指向同一元素，如果是那么不交换
                if (i != j)
                {
				    nums[j] = nums[i];
				    nums[i] = 0;
                }
                j++;
			}
		}
	}
}
```

