---
title: "LeetCode 232用栈实现队列"
date: 2022-08-26
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_232/leetcode232_thumb.jpg
coverImage: leetcode/leetcode_232/leetcode232_cover.jpg
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
- 数组
showSocial: false
---

leetcode第232题，用栈实现队列。采用栈的算法来完成。

<!--more-->
# LeetCode 232用栈实现队列

请你仅使用两个栈实现先入先出队列。队列应当支持一般队列支持的所有操作（`push`、`pop`、`peek`、`empty`）：

实现 `MyQueue` 类：

- `void push(int x)` 将元素 x 推到队列的末尾
- `int pop()` 从队列的开头移除并返回元素
- `int peek()` 返回队列开头的元素
- `boolean empty()` 如果队列为空，返回 `true` ；否则，返回 `false`



**说明：**

- 你 **只能** 使用标准的栈操作 —— 也就是只有 `push to top`, `peek/pop from top`, `size`, 和 `is empty` 操作是合法的。
- 你所使用的语言也许不支持栈。你可以使用 `list` 或者 `deque`（双端队列）来模拟一个栈，只要是标准的栈操作即可。



**示例 1：**

```
输入：
["MyQueue", "push", "push", "peek", "pop", "empty"]
[[], [1], [2], [], [], []]
输出：
[null, null, null, 1, 1, false]

解释：
MyQueue myQueue = new MyQueue();
myQueue.push(1); // queue is: [1]
myQueue.push(2); // queue is: [1, 2] (leftmost is front of the queue)
myQueue.peek(); // return 1
myQueue.pop(); // return 1, queue is [2]
myQueue.empty(); // return false
```



**提示：**

- `1 <= x <= 9`
- 最多调用 `100` 次 `push`、`pop`、`peek` 和 `empty`
- 假设所有操作都是有效的 （例如，一个空的队列不会调用 `pop` 或者 `peek` 操作）



**进阶：**

- 你能否实现每个操作均摊时间复杂度为 `O(1)` 的队列？换句话说，执行 `n` 个操作的总时间复杂度为 `O(n)` ，即使其中一个操作可能花费较长时间。



# 思路

解题思路是**双堆栈**

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_232/leetcode232_1.png" thumbnail="/leetcode/leetcode_232/leetcode232_1.png" title="">}}

1. 有两个栈，其中一个是A栈，另外一个是B栈。A栈用于队列元素的进入，B栈用于队列元素的出去。
2. 不断的往A栈压入元素，然后我们需要从输出B栈拿取元素时。如果现B栈为空栈，那么需要把输入A栈的元素一次性压入B栈中。等到输入栈为空且输出栈不为空的时候，逐个去除输出栈元素。



# 解题

```java
class MyQueue {
    private Stack<Integer> in;
    private Stack<Integer> out;

    public MyQueue() {
        in = new Stack<Integer>();
        out = new Stack<Integer>();
    }
    
    public void push(int x) {
        in.push(x);
    }
    
    public int pop() {
        if(out.isEmpty())
        {
            in2out();
        }
        return out.pop();
    }
    
    public int peek() {
        if(out.isEmpty())
        {
            in2out();
        }
        return out.peek();
    }
    
    public boolean empty() {
        return in.isEmpty() && out.isEmpty();
    }

    private void in2out()
    {
        while(!in.isEmpty())
        {
            out.push(in.pop());
        }
    }
}

/**
 * Your MyQueue object will be instantiated and called as such:
 * MyQueue obj = new MyQueue();
 * obj.push(x);
 * int param_2 = obj.pop();
 * int param_3 = obj.peek();
 * boolean param_4 = obj.empty();
 */
```

