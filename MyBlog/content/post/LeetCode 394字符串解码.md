---
title: "LeetCode 394字符串解码"
date: 2022-08-25
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_394/leetcode394_thumb.jpg
coverImage: leetcode/leetcode_394/leetcode394_cover.jpg
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

leetcode第394题，字符串解码。采用栈的算法来完成。

<!--more-->
# LeetCode 394字符串解码



# 思路

利用栈数据结构

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_394/leetcode394_1.png" thumbnail="/leetcode/leetcode_394/leetcode394_1.png" title="">}}

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_394/leetcode394_2.png" thumbnail="/leetcode/leetcode_394/leetcode394_2.png" title="">}}

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_394/leetcode394_3.png" thumbnail="/leetcode/leetcode_394/leetcode394_3.png" title="">}}

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_394/leetcode394_4.png" thumbnail="/leetcode/leetcode_394/leetcode394_4.png" title="">}}

1. 将字符串元素逐个压入栈中，直到遇到右中括号`]`
2. 然后进行元素依次出栈，直到左括号`[`的前一个字符出栈
3. 拼接字符串，如果元素和栈不为空，重新压入栈中，继续重复1-2操作
4. 直到栈为空，并且字符串完成遍历



# 解题