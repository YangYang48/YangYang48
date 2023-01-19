---
title: "LeetCode 142环形链表 II"
date: 2022-08-16T09:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_142/leetcode142_thumb.jpg
coverImage: leetcode/leetcode_142/leetcode142_cover.jpg
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

leetcode第142题，环形链表2。采用双指针的算法来完成。

<!--more-->
# LeetCode 142环形链表 II

给定一个链表的头节点  `head` ，返回链表开始入环的第一个节点。 *如果链表无环，则返回 `null`。*

如果链表中有某个节点，可以通过连续跟踪 `next` 指针再次到达，则链表中存在环。 为了表示给定链表中的环，评测系统内部使用整数 `pos` 来表示链表尾连接到链表中的位置（**索引从 `0` 开始**）。如果 `pos` 是 -1，则在该链表中没有环。注意：`pos` **不作为参数进行传递**，仅仅是为了标识链表的实际情况。

**不允许修改** 链表。



**示例 1：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_142/leetcode142_1.png" thumbnail="/leetcode/leetcode_142/leetcode142_1.png" title="">}}

```
输入：head = [3,2,0,-4], pos = 1
输出：返回索引为 1 的链表节点
解释：链表中有一个环，其尾部连接到第二个节点。
```



**示例 2：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_142/leetcode142_2.png" thumbnail="/leetcode/leetcode_142/leetcode142_2.png" title="">}}

```
输入：head = [1,2], pos = 0
输出：返回索引为 0 的链表节点
解释：链表中有一个环，其尾部连接到第一个节点。
```



**示例 3：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_142/leetcode142_3.png" thumbnail="/leetcode/leetcode_142/leetcode142_3.png" title="">}}

```
输入：head = [1], pos = -1
输出：返回 null
解释：链表中没有环。
```



**提示：**

- 链表中节点的数目范围在范围 `[0, 104]` 内
- `-105 <= Node.val <= 105`
- `pos` 的值为 `-1` 或者链表中的一个有效索引



**进阶：**你是否可以使用 `O(1)` 空间解决此题？



# 思路

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_142/leetcode142_4.png" thumbnail="/leetcode/leetcode_142/leetcode142_4.png" title="">}}

对于环形链表有通用解法，使用**Floyd算法**。

1. 这里使用的还是双指针，与之前不同的是，这里的双指针的快慢指针，**快指针每次移动2个单位，慢指针每次移动1个单位**
2. 如果存在环，这两个指针必定会相遇，即指针的地址会相同，且会相差`m`个环的距离
3. 在这个基础上，一旦找到环。立刻把慢指针移动链表头部，这个时候并且继续移动两个快慢指针，**快慢指针每次移动1个单位**。
4. 直接再一次两个指针相遇，那么指针相遇的位置，即指针的位置则为环的开始。



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
    //快慢双指针相遇，慢指针移至链首，快慢指针以相同速度再次相遇的点为环的开始
    public ListNode detectCycle(ListNode head) {
        if(null == head) return null;
        ListNode slow = head;
        ListNode fast = head;
        boolean  existCycle = false;
        while(null != fast.next && null != fast.next.next)
        {
            fast = fast.next.next;
            slow = slow.next;
            if(fast == slow)
            {
                existCycle = true;
                break;
            }
        }

        if(existCycle)//存在环
        {
            slow = head;
            while(slow != fast)
            {
                slow = slow.next;
                fast = fast.next;

            }
            return slow;//返回环的起点
        }
        return null;
    }
}
```

