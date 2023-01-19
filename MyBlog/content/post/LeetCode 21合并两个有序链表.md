---
title: "LeetCode 21合并两个有序链表"
date: 2022-08-16T06:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_21/leetcode21_thumb.jpg
coverImage: leetcode/leetcode_21/leetcode21_cover.jpg
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
- 链表
showSocial: false
---

leetcode第21题，合并两个有序链表。采用双指针的算法来完成。

<!--more-->
# LeetCode 21合并两个有序链表

将两个升序链表合并为一个新的 **升序** 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。

**示例 1：**

 {{< image classes="fancybox center fig-100" src="/leetcode/leetcode_21/leetcode21_1.png" thumbnail="/leetcode/leetcode_21/leetcode21_1.png" title="">}}

```
输入：l1 = [1,2,4], l2 = [1,3,4]
输出：[1,1,2,3,4,4]
```

**示例 2：**

```
输入：l1 = [], l2 = []
输出：[]
```

**示例 3：**

```
输入：l1 = [], l2 = [0]
输出：[0]
```

**提示：**

- 两个链表的节点数目范围是 [0, 50]
- -100 <= `Node.val` <= 100
- `l1` 和 `l2` 均按 **非递减顺序** 排列



# 思路

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_21/leetcode21_2.png" thumbnail="/leetcode/leetcode_21/leetcode21_2.png" title="">}}

可以同之前数组一样采用双指针的形式

1. 每个链表头部有一个指针
2. 比较双指针所在的值，会出现三种结果。如果相等，默认取链表2中的元素，P指向这个元素，并且链表P和指针往后移动一位；如果不相等，取指针指向小的那位，并且指针往后移动一位。
3. 直到两个指针全部移动完成



# 解题

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        if(list1 == null) return list2;
        if(list2 == null) return list1;
        ListNode list3 = new ListNode(0);
        ListNode p = list3;
        while(list1 != null && list2 != null)
        {
            if(list1.val >= list2.val)
            {
                p.next = list2;
                list2 = list2.next;
            }else
            {
                p.next = list1;
                list1 = list1.next;
            }
            p = p.next;
        }

        if(list1 == null)
        {
            p.next = list2;
        }

        if(list2 == null)
        {
            p.next = list1;
        }

        return list3.next;
    }
}
```



递归操作

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_21/leetcode21_3.png" thumbnail="/leetcode/leetcode_21/leetcode21_3.png" title="">}}

两条链表合并问题经过一次转换为短链表的合并问题

可以同之前数组一样采用双指针+递归的形式

1. 每个链表头部有一个指针
2. 比较双指针所在的值，会出现三种结果。如果相等，默认取链表2中的元素，P指向这个元素，并且链表P和指针往后移动一位；如果不相等，取指针指向小的那位，并且指针往后移动一位。
3. 后移之后这个问题转换成两个短链表之间的合并问题（递归）
4. 直到短链表合并出现一条链表，然后两个指针全部移动完成



```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        if(list1 == null) return list2;
        if(list2 == null) return list1;
        if(list1.val >= list2.val)
        {
            //前面的list2.next代表list2的下一个指向为mergeTwoLists的ListNode
            //后面的list2.next代表list2的指针向后移动一位
            list2.next = mergeTwoLists(list1, list2.next);
            return list2;
        }else
        {
            list1.next = mergeTwoLists(list1.next, list2);
            return list1;
        }
    }
}
```

