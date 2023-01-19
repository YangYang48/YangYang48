---
title: "LeetCode 206反转链表"
date: 2022-08-17T06:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_206/leetcode206_thumb.jpg
coverImage: leetcode/leetcode_206/leetcode206_cover.jpg
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

leetcode第206题，反转链表。采用迭代的算法来完成。

<!--more-->
# LeetCode 206反转链表

给你单链表的头节点 `head` ，请你反转链表，并返回反转后的链表。

**示例 1：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_206/leetcode206_1.png" thumbnail="/leetcode/leetcode_206/leetcode206_1.png" title="">}}

```
输入：head = [1,2,3,4,5]
输出：[5,4,3,2,1]
```



**示例 2：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_206/leetcode206_2.png" thumbnail="/leetcode/leetcode_206/leetcode206_2.png" title="">}}

```
输入：head = [1,2]
输出：[2,1]
```



**示例 3：**

```
输入：head = []
输出：[]
```



**提示：**

- 链表中节点的数目范围是 `[0, 5000]`
- `-5000 <= Node.val <= 5000`



**进阶：**链表可以选用迭代或递归方式完成反转。你能否用两种方法解决这道题？



# 思路

遍历链表，用另外一个新链表来存储每一次的结点

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_206/leetcode206_3.png" thumbnail="/leetcode/leetcode_206/leetcode206_3.png" title="">}}

1. 链表开始遍历的时候，构造一个新链表，新链表指向`null`
2. 链表每移动一个结点，就把当前结点的指向修改成指向新链表中的结点
3. 直到链表移动到`null`结束，那么新链表就为反转的链表



> 1.  需要一个变量保存当前的遍历结点，用于判断是否到最后一个节点，`current = head;` `current = current.next;`
> 2. `nextTmp`变量用于保存下一个结点，因为一旦将当前结点的指向修改，就找不到下一个结点。`nextTmp = current.next;`
> 3. `preTmp`变量用于保存当前结点，因为下一个结点无法主动找到上一个结点,`preTmp = current;`并且`preTmp = current;`操作要在`current = current.next;`

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_206/leetcode206_5.png" thumbnail="/leetcode/leetcode_206/leetcode206_5.png" title="">}}

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
    //让当前指针的指向改成往前，需要用另外一个链表的保存
    public ListNode reverseList(ListNode head) {
        if(null == head) return null;
        ListNode preNode = null;//保存每一个链表结点的值
        ListNode cur = head;//链表的当前指向结点
        while(null != cur)
        {
            ListNode next = cur.next;
            cur.next = preNode;
            preNode = cur;
            cur = next;
        }
        return preNode;
    }
}
```



# 递归

也可以采用递归的方式

1. 传入的是`head`链表头部。需要翻转链表，意味着第二个节点的`next`指向1，`head.next.next = head;`
2. 然后再将1结点的`next`指向`null`，`head.next = null;`，避免形成环形链表
3. 上面是1结点和2结点的翻转，并且可以将这种方式推广到剩余的结点，也可以两两翻转

按照上面的操作，会发现按顺序执行会找不到第二个结点后面的结点，因为可以**逆序**操作，从最后一个结点，一个个倒退回到第一个结点

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_206/leetcode206_4.png" thumbnail="/leetcode/leetcode_206/leetcode206_4.png" title="">}}

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
    //让当前指针的指向改成往前，需要用另外一个链表的保存
    public ListNode reverseList(ListNode head) {
        if(null == head || null == head.next) return head;

        ListNode newlist = reverseList(head.next);
        head.next.next = head;
        head.next = null;
        return newlist;
    }
}
```

