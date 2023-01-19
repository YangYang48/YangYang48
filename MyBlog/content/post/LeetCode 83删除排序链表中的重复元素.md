---
title: "LeetCode 83删除排序链表中的重复元素"
date: 2022-08-16T07:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_83/leetcode83_thumb.jpg
coverImage: leetcode/leetcode_83/leetcode83_cover.jpg
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
- 链表
showSocial: false
---

leetcode第83题，删除排序链表中的重复元素。采用指针指向越过下一个结点的算法来完成。

<!--more-->
# LeetCode 83删除排序链表中的重复元素

给定一个已排序的链表的头 `head` ， *删除所有重复的元素，使每个元素只出现一次* 。返回 *已排序的链表* 。



**示例 1：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_83/leetcode83_1.png" thumbnail="/leetcode/leetcode_83/leetcode83_1.png" title="">}}

```
输入：head = [1,1,2]
输出：[1,2]
```



**示例 2：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_83/leetcode83_2.png" thumbnail="/leetcode/leetcode_83/leetcode83_2.png" title="">}}

```
输入：head = [1,1,2,3,3]
输出：[1,2,3]
```



**提示：**

- 链表中节点数目在范围 `[0, 300]` 内
- `-100 <= Node.val <= 100`
- 题目数据保证链表已经按升序 **排列**



# 思路

1. 在一条链表中，如果当前节点值和结点下一个值相等，说明下一个结点是重复的
2. 当前结点越过当前结点的下一个结点，指向当前结点的下下的结点。
3. 后续的结点也通过同样的方式去判定，直到达到链表尾部



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
    public ListNode deleteDuplicates(ListNode head) {
        if(null == head) return null;
        ListNode currentNode = head;
        while(null != currentNode && null != currentNode.next)
        {
            if(currentNode.val == currentNode.next.val)
            {
                //存在相同的需要继续判断
                currentNode.next = currentNode.next.next;
            }else{
                currentNode = currentNode.next;
            }
            
        }
        return head;
    }
}
```



也可以使用递归的操作

1. 在一条链表中，如果当前节点值和结点下一个值相等，说明下一个结点是重复的
2. 当前结点越过当前结点的下一个结点，实际上转化成去掉一个结点的链表的删除重复元素的问题
3. 递归删除，直到达到链表尾部



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
    public ListNode deleteDuplicates(ListNode head) {
        if(null == head || null == head.next) return head;
        head.next = deleteDuplicates(head.next);
        if(head.next.val == head.val)
        {
            return head.next;
        }else
        {
            return head;
        }
        
    }
}
```

