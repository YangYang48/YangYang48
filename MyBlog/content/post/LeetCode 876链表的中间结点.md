---
title: "LeetCode 876链表的中间结点"
date: 2022-08-17T08:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_876/leetcode876_thumb.jpg
coverImage: leetcode/leetcode_876/leetcode876_cover.png
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

leetcode第876题，链表的中间结点。采用双指针的算法来完成。

<!--more-->
# LeetCode 876链表的中间结点

给定一个头结点为 `head` 的非空单链表，返回链表的中间结点。

如果有两个中间结点，则返回第二个中间结点。



**示例 1：**

```
输入：[1,2,3,4,5]
输出：此列表中的结点 3 (序列化形式：[3,4,5])
返回的结点值为 3 。 (测评系统对该结点序列化表述是 [3,4,5])。
注意，我们返回了一个 ListNode 类型的对象 ans，这样：
ans.val = 3, ans.next.val = 4, ans.next.next.val = 5, 以及 ans.next.next.next = NULL.
```



**示例 2：**

```
输入：[1,2,3,4,5,6]
输出：此列表中的结点 4 (序列化形式：[4,5,6])
由于该列表有两个中间结点，值分别为 3 和 4，我们返回第二个结点。
```



**提示：**

- 给定链表的结点数介于 `1` 和 `100` 之间。



# 思路

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_876/leetcode876_1.png" thumbnail="/leetcode/leetcode_876/leetcode876_1.png" title="">}}

这题本质还是双指针

1. 采用两个快慢指针，慢指针每次移动一个单位，快指针每次移动两个单位
2. 快指针当前结点或者是快指针后一个结点为null，那么当前慢指针指向的位置即为中间结点



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
    //快慢双指针
    public ListNode middleNode(ListNode head) {
        if(null == head) return null;
        ListNode fast = head;
        ListNode slow = head;

        while(null != fast && null != fast.next)
        {
            fast = fast.next.next;
            slow = slow.next;
        }
        return slow;
    }
}
```

