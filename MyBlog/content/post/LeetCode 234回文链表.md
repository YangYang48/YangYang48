---
title: "LeetCode 234回文链表"
date: 2022-08-17T07:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_234/leetcode234_thumb.png
coverImage: leetcode/leetcode_234/leetcode234_cover.png
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

leetcode第234题，回文链表。采用双指针+迭代的算法来完成。

<!--more-->
# LeetCode 234回文链表

给你一个单链表的头节点 `head` ，请你判断该链表是否为回文链表。如果是，返回 `true` ；否则，返回 `false` 。

 

**示例 1：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_234/leetcode234_1.png" thumbnail="/leetcode/leetcode_234/leetcode234_1.png" title="">}}

```
输入：head = [1,2,2,1]
输出：true
```

**示例 2：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_234/leetcode234_2.png" thumbnail="/leetcode/leetcode_234/leetcode234_2.png" title="">}}

```
输入：head = [1,2]
输出：false
```



**提示：**

- 链表中节点数目在范围`[1, 105]` 内
- `0 <= Node.val <= 9`

 

**进阶：**你能否用 `O(n)` 时间复杂度和 `O(1)` 空间复杂度解决此题？



# 思路

可以直接构建一个数组来存放这些元素，但不满足空间复杂度`o(1)`的要求

还是采用双指针的方式，但是还是有区别。

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_234/leetcode234_3.png" thumbnail="/leetcode/leetcode_234/leetcode234_3.png" title="">}}

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_234/leetcode234_4.png" thumbnail="/leetcode/leetcode_234/leetcode234_4.png" title="">}}

1. 根据回文链表特性，前后半部分是对称的，所以后半部分可以做链表反转，如题目`LeetCode 206`的反转链表一样
2. 引入快慢双指针，**慢指针每次移动一个单位，快指针每次移动两个单位**，如果快指针当前或者快指针的后面一个元素为`null`，那么记录下慢指针当前的位置。
3. 判断奇偶性，快指针不为null，那么链表长度是奇数，慢指针往后移动一位，快指针为null，那么链表长度为偶数，慢指针不操作。
4. **反转慢指针当前位置到链表尾部的结点**，并将快指针移动到链表首部，同时快慢指针遍历元素，且快慢指针**每次移动一个单位**，直到慢指针指向链表尾部找不到和快指针不相同的元素，那么该链表为回文链表。



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
    public boolean isPalindrome(ListNode head) {
        if(null == head) return false;

        ListNode fast = head;
        ListNode slow = head;

        while(null != fast && null != fast.next)
        {
            fast = fast.next.next;
            slow = slow.next;
        }
        //说明是奇数链表
        if(null != fast)
        {
            slow = slow.next;//移动慢指针
        }

        fast = head;
        slow = reverseList(slow);

        while(null != slow)
        {
            if(fast.val != slow.val)
            {
                return false;
            }

            fast = fast.next;
            slow = slow.next;
        }
        return true;

    }

    //反转链表不要使用递归，会创建大量的临时变量和遍历
    public ListNode reverseList(ListNode head) {
        if(null == head) return null;
        ListNode preNode = null;//保存每一个链表结点的值
        //ListNode cur = head;//链表的当前指向结点
        while(null != head)
        {
            ListNode next = head.next;
            head.next = preNode;
            preNode = head;
            head = next;
        }
        return preNode;
    }
}
```

