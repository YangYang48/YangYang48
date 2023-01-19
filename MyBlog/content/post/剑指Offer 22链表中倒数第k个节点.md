---
title: "LeetCode之剑指Offer 22链表中倒数第k个节点"
date: 2022-08-18
thumbnailImagePosition: left
thumbnailImage: leetcode/swordtooffer_22/swordtooffer22_thumb.jpg
coverImage: leetcode/swordtooffer_22/swordtooffer22_cover.jpg
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

leetcode之剑指Offer22题，链表中倒数第k个节点。采用双指针的算法来完成。

<!--more-->
# 剑指Offer 22链表中倒数第k个节点

输入一个链表，输出该链表中倒数第k个节点。为了符合大多数人的习惯，本题从1开始计数，即链表的尾节点是倒数第1个节点。

例如，一个链表有 `6` 个节点，从头节点开始，它们的值依次是 `1、2、3、4、5、6`。这个链表的倒数第 `3` 个节点是值为 `4` 的节点。



**示例：**

```
给定一个链表: 1->2->3->4->5, 和 k = 2.

返回链表 4->5.
```



# 思路

第一思考哈希表，通过[key, value]的形式保存[位置, 对应结点],但是空间复杂度是o(n)。

第二思考，本质是也可以是求正序的`n + 1 - k`的结点，可以先遍历一次求得链表总长度，然后在遍历求得对应结点

第三思考，由于第二思考是两次循环，如果引入双指针可以解决

{{< image classes="fancybox center fig-100" src="/leetcode/swordtooffer_22/swordtooffer22_1.png" thumbnail="/leetcode/swordtooffer_22/swordtooffer22_1.png" title="">}}

1. 存在两个快慢指针，快指针会先移动`k - 1`个结点
2. 之后，两个快慢指针同时移动，快慢指针每次移动一个单位，直到快指针的下一个结点为null



# 解题

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    //双指针
    public ListNode getKthFromEnd(ListNode head, int k) {
        if(null == head || k < 1) return null;
        ListNode fast = head;
        ListNode slow = head;
        
        int count = -1;
        boolean flag = false;
        while(null != fast)
        {
            fast = fast.next;
            count++;
            if(count == k - 1)
            {
                flag = true;
                break;
            }
        }
        //如果k比总长度要大，那么也找不到
        if(!flag)
        {
            return null;
        }

        while(null != fast)
        {
            fast = fast.next;
            slow = slow.next;
        }
        return slow;
    }
}
```

