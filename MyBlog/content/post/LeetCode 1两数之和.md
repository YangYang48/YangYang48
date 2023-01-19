---
title: "LeetCode 1两数之和"
date: 2022-08-14
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_1/leetcode1_thumb.jpg
coverImage: leetcode/leetcode_1/leetcode1_cover.png
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
- 双指针
showSocial: false
---

leetcode第1题，两数之和。采用双指针的算法来完成。

<!--more-->
# 0 前言

从这里开始不定期更新leetcode算法题，其中穿插题目解析，主要用于掌握数据结构基础为主。



# 1两数之和

给定一个整数数组 `nums `和一个整数目标值 `target`，请你在该数组中找出 和为目标值 `target`  的那两个整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。



示例 1：

```
输入：nums = [2,7,11,15], target = 9
输出：[0,1]
解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。
```


示例 2：

```
输入：nums = [3,2,4], target = 6
输出：[1,2]
```


示例 3：

```
输入：nums = [3,3], target = 6
输出：[0,1]
```

提示：

- 2 <= `nums.length` <= 104
- -109 <= `nums[i]` <= 109
- -109 <= `target` <= 109
- 只会存在一个有效答案

进阶：你可以想出一个时间复杂度小于 `O(n2)` 的算法吗？



# 2解题思路（双指针）

对这个数组排序，升序排序，并在数组头尾设置双指针

双指针移动，移动的时候计算

```c
/*L指针向右移动，R指针向左移动，遍历所有
**a[L] + a[R]是否等于T
**只会出现三种场景
**/
a[L] + a[R] == T		//找到这个等于，算法结束
a[L] + a[R] < T			//只能移动L指针
a[L] + a[R] > T			 //只能移动R指针
```


# 3解题


```c++
//最终答案
class Solution {
public:
    vector<int> twoSum(vector<int>& a, int t) {
        int n = a.size();
        vector<int> idexs;
        for(int i = 0; i < n; i++)
        {
            idexs.push_back(i);
        }

        //不能直接排序，会丢失下标
        sort(idexs.begin(), idexs.end(), [a, idexs](int i, int j){
            return a[idexs[i]] < a[idexs[j]];
        });

        int l = 0, r = n - 1;
        vector<int> ret;
        while(l < r){
            int sum = a[idexs[l]] + a[idexs[r]];
            if(sum == t)
            {
                ret.push_back(idexs[l]);
                ret.push_back(idexs[r]);
                break;
            }else if(sum < t)
            {
                l++;
            }else if(sum > t)
            {
                r--;
            }
        }
        return ret;
    }
};
```



