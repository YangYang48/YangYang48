---
title: "字符串匹配 BF(Brute Force)"
date: 2022-10-29
thumbnailImagePosition: left
thumbnailImage: leetcode/string/index/BF/BF_thumb.jpg
coverImage: leetcode/string/index/BF/BF_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- October
tags:
- 字符串匹配
- BF
- C++
- Java
showSocial: false
---

经典的字符串匹配算法，BF(Brute Force)算法。

<!--more-->
# 0准备

```c++
int main() {
    string s = "abababc";
    string p = "abc";
    string s1 = "abcdefgabcdee";
    string p1 = "abcdex";
    string s2 = "00000000000000000000000000000001";
    string p2 = "00000001";
    cout << BF(s, p) << endl;
    cout << BF(s1, p1) << endl;
    cout << BF(s2, p2) << endl;
}
```



# BF(Brute Force)算法

> BF(Brute Force)算法
>
> 又叫暴力匹配算法或者朴素匹配算法
>
> **思路很简单**：在主串中取前下标为[0,m-1]这m个字符的子串和模式串逐个字符逐个字符比较，如果完全一样就结束并返回下标；如果有不一样的，那么主串中的子串后移一位，主串中[1,m]这个子串和模式串继续比较，… ，主串中[n-m,n-1]这个子串和模式串继续比较。主串中长度为m的子串有n-m+1个。



# C的版本

```c++
int BF(string& s, string& pattern)
{
    int n = s.length(), m = pattern.length();
    for (int i = 0; i < n - m + 1; i++)
    {
        int j = 0;
        for (; j < m; j++)
        {
            if (i + j >= n || s[i + j] != pattern[j])
                break;
        }
        if(j == m)
        {
            //匹配到了，返回主串中的下标
            return i;
        }
    }
    //匹配不到
    return -1;
}
```



举例2，字符串`"abcdedgabcdee"`中匹配字符串`"abcdex"`。在`i = 0`会做一个匹配，但是最后一个字符匹配不上，只能继续遍历，在`i = 7`的时候会再一次匹配，但是最后一个字符匹配不上，最终返回`-1`。

`i = 0`

{{< image classes="fancybox center fig-100" src="/leetcode/string/index/BF/BF_4.png" thumbnail="/leetcode/string/index/BF/BF_4.png" title="">}}

`i = 1`

{{< image classes="fancybox center fig-100" src="/leetcode/string/index/BF/BF_5.png" thumbnail="/leetcode/string/index/BF/BF_5.png" title="">}}

`i = 7`

{{< image classes="fancybox center fig-100" src="/leetcode/string/index/BF/BF_6.png" thumbnail="/leetcode/string/index/BF/BF_6.png" title="">}}

举例3可以发现，当32位的`uint32`型数值的字符串中匹配`uint8`型数值的字符串就会比较繁琐，从`i = 0`开始，直到`i = 24`位置才能够完全匹配，这就是BF的暴力求解法。

`i = 0`

{{< image classes="fancybox center fig-100" src="/leetcode/string/index/BF/BF_1.png" thumbnail="/leetcode/string/index/BF/BF_1.png" title="">}}

`i = 1`

{{< image classes="fancybox center fig-100" src="/leetcode/string/index/BF/BF_2.png" thumbnail="/leetcode/string/index/BF/BF_2.png" title="">}}

`i = 24`

{{< image classes="fancybox center fig-100" src="/leetcode/string/index/BF/BF_3.png" thumbnail="/leetcode/string/index/BF/BF_3.png" title="">}}







# Java版本

```java
//String.java#indexOf
//jdk源码
static int indexOf(char[] source, int sourceOffset, int sourceCount,
                   char[] target, int targetOffset, int targetCount,
                   int fromIndex) {
    if (fromIndex >= sourceCount) {
        return (targetCount == 0 ? sourceCount : -1);
    }
    if (fromIndex < 0) {
        fromIndex = 0;
    }
    if (targetCount == 0) {
        return fromIndex;
    }

    char first = target[targetOffset];
    int max = sourceOffset + (sourceCount - targetCount);

    for (int i = sourceOffset + fromIndex; i <= max; i++) {
        /* Look for first character. */
        if (source[i] != first) {
            while (++i <= max && source[i] != first);
        }

        /* Found first character, now look at the rest of v2 */
        if (i <= max) {
            int j = i + 1;
            int end = j + targetCount - 1;
            for (int k = targetOffset + 1; j < end && source[j]
                 == target[k]; j++, k++);

            if (j == end) {
                /* Found whole string. */
                return i - sourceOffset;
            }
        }
    }
    return -1;
}
```



> **BF算法总结**
>
> 最坏的情况下，在第一个for循环里，i 从0到n-m走满共n-m+1次，第二个for循环里，j 从0到m-1走满共m次，因此最坏的情况下时间复杂度为O(n*m)，上述例子中`string a2 = "00000000000000000000000000000001"`，`string p2 = "00000001"`;所有n-m+1个子串都要走完，并且每次和模式串比较都要比较m次，总共比较n-m+1次，**最坏的时间复杂度为o((n - m + 1) * m)**
>
> BF算法最大的优点就是简单，代码不容易出错，在主串和模式串的长度都不大的时候还是比较实用的。
