---
title: "LeetCode 415字符串相加"
date: 2022-10-28
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_415/leetcode415_thumb.jpg
coverImage: leetcode/leetcode_415/leetcode415_cover.jpg
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
- ASCII
- C++
- Java
showSocial: false
---

leetcode第415题，字符串相加。采用字符串处理方式来完成。

<!--more-->
# LeetCode 415字符串相加

给定两个字符串形式的非负整数 `num1` 和`num2` ，计算它们的和并同样以字符串形式返回。

你不能使用任何內建的用于处理大整数的库（比如 `BigInteger`）， 也不能直接将输入的字符串转换为整数形式。

**示例 1：**

```text
输入：num1 = "11", num2 = "123"
输出："134"
```



**示例 2：**

```text
输入：num1 = "456", num2 = "77"
输出："533"
```



**示例 3：**

```text
输入：num1 = "0", num2 = "0"
输出："0"
```

**提示：**

- `1 <= num1.length, num2.length <= 104`
- `num1` 和`num2` 都只包含数字 `0-9`
- `num1` 和`num2` 都不包含任何前导零



# 思路

我们需要通过提出字符串中字符的数值，具体操作是减去`'0'`，

```c++
'0' - '0' = 0
'1' - '0' = 1
'2' - '0' = 2
'3' - '0' = 3
'4' - '0' = 4
'5' - '0' = 5
'6' - '0' = 6
'7' - '0' = 7
'8' - '0' = 8
'9' - '0' = 9
```

然后通过一个进位数来控制最终的值

```c++
//进位数的计算
//carryFlag开始的值为0
carryFlag = (num1[i] + num2[j] + carryFlag) / 10;
```

最终通过计算得出一个逆序值，再次转为正向然后转化为字符串的形式即可。

{{< image classes="fancybox center fig-100" src="/leetcode/leetcode_415/leetcode415_1.png" thumbnail="/leetcode/leetcode_415/leetcode415_1.png" title="">}}



# 解题

```java
class Solution {
    public String addStrings(String num1, String num2) {
        //java字符串拼接
        StringBuilder sb = new StringBuilder();
        //进位变量，初始值为0
        int carryFlag = 0;
        for(int i = num1.length() - 1, j = num2.length() - 1;
            i >= 0 || j >= 0 || carryFlag != 0;
            i--, j--)
        {
            int numChar1 = (i >= 0 ? num1.charAt(i) - '0' : 0);
            int numChar2 = (j >= 0 ? num2.charAt(j) - '0' : 0);
            //计算新的位数和数值
            sb.append((numChar1 + numChar2 + carryFlag) % 10);
            carryFlag = (numChar1 + numChar2 + carryFlag) / 10;
        }
        return sb.reverse().toString();
    }
}
```

