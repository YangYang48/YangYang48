---
title: "LeetCode 20有效的括号"
date: 2022-10-27
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_20/leetcode20_thumb.jpg
coverImage: leetcode/leetcode_20/leetcode20_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- October
tags:
- 字符串
- 栈
- C++
- Java
showSocial: false
---

leetcode第20题，有效的括号。采用栈来完成。

<!--more-->
# LeetCode 20有效的括号

给定一个只包括 `'('`，`')'`，`'{'`，`'}'`，`'['`，`']'` 的字符串 `s` ，判断字符串是否有效。

有效字符串需满足：

1. 左括号必须用相同类型的右括号闭合。
2. 左括号必须以正确的顺序闭合。
3. 每个右括号都有一个对应的相同类型的左括号。



**示例 1：**

```text
输入：s = "()"
输出：true
```



**示例 2：**

```text
输入：s = "()[]{}"
输出：true
```



**示例 3：**

```text
输入：s = "(]"
输出：false
```



**提示：**

- `1 <= s.length <= 104`
- `s` 仅由括号 `'()[]{}'` 组成



# 思路

这种先进后出的顺序，需要用到栈的数据结构



# 解题

```java
class Solution {
    public boolean isValid(String s) {
        //LinkedList同时实现了List接口和Deque对口
        //也就是收它既可以看作一个顺序容器
        //又可以看作一个队列（Queue）
        //同时又可以看作一个栈（stack）
        Deque<Character> stack = new LinkedList<Character>();
        //java String中的单个字符的操作
        //String.toCharArray()把整个String转成 char[] 数组，然后就可以按着数组的方式处理
        //使用String.charAt(i)函数，也可以实现字符串的处理。
        for(char i : s.toCharArray())
        {
            if('(' == i)
            {
                stack.push(')');
            }else if('[' == i)
            {
                stack.push(']');
            }else if('{' == i)
            {
                stack.push('}');
            }else if(stack.isEmpty() || stack.pop() != i)
            {
                return false;
            }
        }
        return stack.isEmpty();
        
    }
}
```

