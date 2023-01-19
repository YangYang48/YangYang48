---
title: "LeetCode 160相交链表"
date: 2022-08-16T10:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_160/leetcode160_thumb.jpg
coverImage: leetcode/leetcode_160/leetcode160_cover.jpg
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

leetcode第160题，相交链表。采用双指针的算法来完成。

<!--more-->
# LeetCode 160相交链表

给你两个单链表的头节点 `headA` 和 `headB` ，请你找出并返回两个单链表相交的起始节点。如果两个链表不存在相交节点，返回 `null` 。

图示两个链表在节点 `c1` 开始相交**：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_160/leetcode160_1.png" thumbnail="/leetcode/leetcode_160/leetcode160_1.png" title="">}}

题目数据 **保证** 整个链式结构中不存在环。

**注意**，函数返回结果后，链表必须 **保持其原始结构** 。

**自定义评测：**

**评测系统** 的输入如下（你设计的程序 **不适用** 此输入）：

`intersectVal` - 相交的起始节点的值。如果不存在相交节点，这一值为 `0`
`listA` - 第一个链表
`listB` - 第二个链表
`skipA` - 在 `listA` 中（从头节点开始）跳到交叉节点的节点数
`skipB` - 在 `listB` 中（从头节点开始）跳到交叉节点的节点数

评测系统将根据这些输入创建链式数据结构，并将两个头节点 `headA` 和 `headB` 传递给你的程序。如果程序能够正确返回相交节点，那么你的解决方案将被 **视作正确答案** 。



**示例 1：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_160/leetcode160_2.png" thumbnail="/leetcode/leetcode_160/leetcode160_2.png" title="">}}

```
输入：intersectVal = 8, listA = [4,1,8,4,5], listB = [5,6,1,8,4,5], skipA = 2, skipB = 3
输出：Intersected at '8'
解释：相交节点的值为 8 （注意，如果两个链表相交则不能为 0）。
从各自的表头开始算起，链表 A 为 [4,1,8,4,5]，链表 B 为 [5,6,1,8,4,5]。
在 A 中，相交节点前有 2 个节点；在 B 中，相交节点前有 3 个节点。
```



**示例 2：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_160/leetcode160_3.png" thumbnail="/leetcode/leetcode_160/leetcode160_3.png" title="">}}

```
输入：intersectVal = 2, listA = [1,9,1,2,4], listB = [3,2,4], skipA = 3, skipB = 1
输出：Intersected at '2'
解释：相交节点的值为 2 （注意，如果两个链表相交则不能为 0）。
从各自的表头开始算起，链表 A 为 [1,9,1,2,4]，链表 B 为 [3,2,4]。
在 A 中，相交节点前有 3 个节点；在 B 中，相交节点前有 1 个节点。
```



**示例 3：**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_160/leetcode160_4.png" thumbnail="/leetcode/leetcode_160/leetcode160_4.png" title="">}}

```
输入：intersectVal = 0, listA = [2,6,4], listB = [1,5], skipA = 3, skipB = 2
输出：null
解释：从各自的表头开始算起，链表 A 为 [2,6,4]，链表 B 为 [1,5]。
由于这两个链表不相交，所以 intersectVal 必须为 0，而 skipA 和 skipB 可以是任意值。
这两个链表不相交，因此返回 null 。
```



**提示：**

- `listA` 中节点数目为 `m`
- `listB` 中节点数目为 `n`
- `1 <= m, n <= 3 * 104`
- `1 <= Node.val <= 105`
- 0 <= `skipA` <= m
- 0 <= `skipB` <= n
- 如果 `listA` 和 `listB` 没有交点，`intersectVal` 为 0
- 如果 `listA` 和 `listB` 有交点，`intersectVal` == `listA[skipA] `== `listB[skipB]`



**进阶：**你能否设计一个时间复杂度 `O(m + n)` 、仅用 `O(1)` 内存的解决方案？



# 思路

可以采用暴力循环，但时间复杂度为o(m*n)，也可以采用hash表，但是空间复杂度为o(n)，不满足题目要求

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_160/leetcode160_5.png" thumbnail="/leetcode/leetcode_160/leetcode160_5.png" title="">}}

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_160/leetcode160_6.png" thumbnail="/leetcode/leetcode_160/leetcode160_6.png" title="">}}

还是使用双指针的做法，相当于两个指针都遍历了两条链表的非公共部分区域

1. 每个链表头部有一个指针
2. 比较指针所指向的地址，会出现三种结果。如果相等，说明两个链表相交；如果不相等，那么两个指针都移向后一位。如果其中指针指向为null，那么这个指针重新指向另一条链表的头部。
3. 直到两个指针全部移动完成



# 解题

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    //双指针的做法，相当于两个指针都遍历了两条链表的非公共部分区域
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        if(null == headA || null == headB) return null;
        ListNode pA = headA;
        ListNode pB = headB;
        while(pA != pB)
        {
            if(pA == null)
            {
                pA = headB;
            }else
            {
                pA = pA.next;
            }

            if(pB == null)
            {
                pB = headA;
            }else
            {
                pB = pB.next;
            }
        }
        return pA;
    }
}
```



除了上面的双指针，其实还可以采用另外的方式。

1. 首先，分别遍历两条链表`A`，`B`，得到链表的长度`lA`和`lB`。
2. 获得两条链表的差值`c`，`c` = |`lA`  - `lB`|。
3. 让长链表移到`c`个长度，然后双指针遍历，双指针所指向的地址相同，那么说明相交



```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    //两条链表的差值法
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        int l1 = 0;
        int l2 = 0;
        int c = 0;
        ListNode pA = headA;
        ListNode pB = headB;
        while(pA != null)
        {
            pA = pA.next;
            l1++;
        }

        while(pB != null)
        {
            pB = pB.next;
            l2++;
        }

        //pA就是长的链表
        if(l1 > l2)
        {
            pA = headA;
            pB = headB;
            c = l1 - l2;
        }else
        {
            pA = headB;
            pB = headA;
            c = l2 -l1;
        }

        for(int i = 0; i < c; i++)
        {
            pA = pA.next;
        }

        while(null != pA && null != pB)
        {
            if(pA == pB)
            {
                return pA;
            }
            pA = pA.next;
            pB = pB.next;
        }

        return null;
    }
}
```

