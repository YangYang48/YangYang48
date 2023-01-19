---
title: "LeetCode 141环形链表"
date: 2022-08-16T08:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_141/leetcode141_thumb.jpg
coverImage: leetcode/leetcode_141/leetcode141_cover.jpg
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
- Floyd算法
- 环形链表
showSocial: false
---

leetcode第141题，环形链表。采用双指针的算法来完成。

<!--more-->
# LeetCode 141环形链表

给你一个链表的头节点 `head` ，判断链表中是否有环。

如果链表中有某个节点，可以通过连续跟踪 `next` 指针再次到达，则链表中存在环。 为了表示给定链表中的环，评测系统内部使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 0 开始）。**注意：`pos` 不作为参数进行传递** 。仅仅是为了标识链表的实际情况。

*如果链表中存在环* ，则返回 `true` 。 否则，返回 `false` 。



**示例 1：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_141/leetcode141_1.png" thumbnail="/leetcode/leetcode_141/leetcode141_1.png" title="">}}

```
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```



**示例 2：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_141/leetcode141_2.png" thumbnail="/leetcode/leetcode_141/leetcode141_2.png" title="">}}

```
输入：head = [1,2], pos = 0
输出：true
解释：链表中有一个环，其尾部连接到第一个节点。
```



**示例 3：**

```
输入：head = [1], pos = -1
输出：false
解释：链表中没有环。
```



**提示：**

- 链表中节点的数目范围是 `[0, 104]`
- `-105 <= Node.val <= 105`
- `pos` 为 `-1` 或者链表中的一个 **有效索引** 。



**进阶：**你能用 `O(1)`（即，常量）内存解决此问题吗？



# 思路

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_141/leetcode141_3.png" thumbnail="/leetcode/leetcode_141/leetcode141_3.png" title="">}}

可以用hash表，但是引入hash表，空间复杂度为o(n)

对于环形链表有通用解法，使用**Floyd算法**。

1. 这里使用的还是双指针，与之前不同的是，这里的双指针的快慢指针，**快指针每次移动2个单位，慢指针每次移动1个单位**
2. 如果存在环，这两个指针必定会相遇，即指针的地址会相同，且会相差`m`个环的距离



# 解题

```java
/**
 * Definition for singly-linked list.
 * class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    //Folyd算法，采用快慢指针
    public boolean hasCycle(ListNode head) {
        if(head == null) return false;
        ListNode slow = head;
        ListNode fast = head;
        while(null != fast.next && null != fast.next.next)
        {
            slow = slow.next;
            fast = fast.next.next;
            if(slow == fast)
            {
                return true;
            }
        }
        return false;
    }
}
```

