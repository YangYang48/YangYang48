---
title: "字符串匹配 BM(Boyer-Moore)"
date: 2022-10-29
thumbnailImagePosition: left
thumbnailImage: leetcode/string/index/BM/BM_thumb.jpg
coverImage: leetcode/string/index/BM/BM_cover.jpg
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
- BM
- C++
- Java
showSocial: false
---

经典的字符串匹配算法，BM(Boyer-Moore)算法。

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
    cout << BM(s, p) << endl;
    cout << BM(s1, p1) << endl;
    cout << BM(s2, p2) << endl;
}
```



# BM(Boyer-Moore)算法

> 通常来说，文本工具中的搜索字符串用到了BM算法。
>
> BM算法的具体实现原理包含两个部分：坏字符规则(bad character rule)和好后缀规则(good suffix rule)。
>
> 1. **坏字符（Bad Character Heuristic）**：当文本 T 中的某个字符跟模式 P 的某个字符不匹配时，我们称文本 T 中的这个失配字符为坏字符。
> 2. **好后缀（Good Suffix Heuristic）**：当文本 T 中的某个字符跟模式 P 的某个字符不匹配时，我们称文本 T 中的已经匹配的字符串为好后缀。
>
> 
>
> 以 [J Strother Moore 提供的例子](http://www.cs.utexas.edu/users/moore/best-ideas/string-searching/fstrpos-example.html)作为示例。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051853550666243.png)
>
> 这样如果尾部的字符不匹配，则前面的字符也就无需比较了，直接跳过。我们看到，"S" 与 "E" 不匹配，我们称文本 T 中的失配字符 "S" 为**坏字符（Bad Character）**。
>
> 由于字符 "S" 在模式 "EXAMPLE" 中不存在，则可将搜索位置滑动到 "S" 的后面。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051904285979557.png)
>
> 仍然从尾部开始比较，发现 "P" 与 "E" 不匹配，所以 "P" 是坏字符。但此时，"P" 包含在模式 "EXAMPLE" 之中。所以，将模式后移两位，使两个 "P" 对齐。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051915506918007.png)
>
> **坏字符启发法的规则是**
>
> ```text
> 模式后移位数 = 坏字符在模式中失配的位置 - 坏字符在模式中最后一次出现的位置
> 
> 坏字符启发法规则中的特殊情况：
> 如果坏字符不存在于模式中，则最后一次出现的位置为 -1。
> 如果坏字符在模式中的位置位于失配位置的右侧，则此启发法不提供任何建议。
> ```
>
> 以上面示例中坏字符 "P" 为例，它的失配位置为 "E" 的位置 6 （位置从 0 开始），在模式中最后一次出现的位置是 4，则模式后移位数为 6 - 4 = 2 位。模式移动的结果就是使模式中最后出现的 "P" 与文本中的坏字符 "P" 进行对齐。
>
> 实际上，前面的坏字符 "S" 出现时，其失配位置为 6，最后一次出现位置为 -1，所以模式后移位数为 6 - (-1) = 7 位。也就是将模式整体移过坏字符。
>
> 我们继续上面的过程，仍然从尾部开始比较。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051942068005852.png)
>
> 仍然从尾部开始比较，发现 "E" 与 "E" 匹配，则继续倒序比较。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051943142064743.png)
>
> 发现 "L" 与 "L" 匹配，则继续倒序比较。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051944028941852.png)
>
> 发现 "P" 与 "P" 匹配，则继续倒序比较。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051944336445646.png)
>
> 发现 "M" 与 "M" 匹配，则继续倒序比较。
>
> ![img](https://images0.cnblogs.com/blog/175043/201410/051945148162991.png)
>
> 发现 "I" 与 "A" 不匹配，则 "I" 是坏字符。对于前面已经匹配的字符串 "MPLE"、"PLE"、"LE"、"E"，我们称它们为**好后缀（Good Suffix）**。
>
> **好后缀启发法的规则是**
>
> ```text
> 模式后移位数 = 好后缀在模式中的当前位置 - 好后缀在模式中最右出现且前缀字符不同的位置
> ```
>
> `Boyer–Moore` 算法的特点就在于此，选择上述两种启发法规则计算结果中最大的一个值来对模式 P 的比较位置进行滑动。



# C的版本

```c++
#define SIZE (256) //全局变量或成员变量
int moveByGs(int j,int m,int* suffix, const bool* prefix);
void generateBC(string& b,int m,int* bc);
void generateGS(string& b,int m,int* suffix,bool* prefix);
//a和b分别表示主串和模式串，n和m分别表示主串和模式串的长度
int BM(string& a, string& b){
    int n = a.length();
    int m = b.length();
    int* bc = new int[SIZE];//记录模式串中每个字符最后出现的位置
    generateBC(b, m, bc);//构建坏字符哈希表
    int* suffix = new int[m];
    bool* prefix = new bool[m];
    generateGS(b,m,suffix,prefix);
    int i = 0;//表示主串与模式串匹配的第一个字符
    while(i <= n - m){
        int j;
        for(j = m - 1;j >= 0;--j){
            //模式串从后往前匹配
            if(a[i+j] != b[j]) break; //坏字符对应模式串中的下标是j
        }
        if(j < 0){
            return i;   //匹配成功，返回主串与模式串第一个匹配的字符的位置
        }
        int x = j - bc[(int)a[i+j]];
        int y = 0;
        if(j < m-1){
            y = moveByGs(j,m,suffix,prefix);
        }
        i = i + max(x,y);
    }
    return -1;
}

//b为模式串，m为模式串长度，bc为哈希表
void generateBC(string& b,int m,int* bc)
{
    for(int i = 0;i < SIZE;++i){
        bc[i] = -1;   //初始化bc
    }

    for(int i = 0;i < m;++i){
        int ascii = (int)b[i];//计算b[i]的ASCII值
        bc[ascii] = i;
    }
}

void generateGS(string& b,int m,int* suffix,bool* prefix)
{
    for(int i = 0;i < m;++i){
        //初始化suffix,prefix数组
        suffix[i] = -1;
        prefix[i] = false;
    }
    for(int i = 0;i < m - 1;++i){
        //循环处理b[0,i]
        int j = i;
        int k = 0;//公共后缀子串的长度
        while(j >= 0 && b[j] == b[m-1-k]){
            //与b[0,m-1]求公共后缀子串
            --j;
            ++k;
            suffix[k] = j+1;//j+1表示公共后缀子串在b[0,i]中的起始下标
        }
        if(j == -1) prefix[k] = true;   //公共后缀子串也是模式串的前缀子串
    }
}

//j表示坏字符对应的模式串中的字符下标，m表示模式串长度
int moveByGs(int j,int m,int* suffix, const bool* prefix){
    int k = m - 1 - j;//好后缀的长度
    if(suffix[k] != -1) return j - suffix[k] + 1;
    for(int r = j+2;r <= m-1;++r){
        if(prefix[m - r]){
            return r;
        }
    }
    return m;
}
```



# Java版本





>Boyer-Moore 算法的主要特点有：
>
>1. 对模式字符的比较顺序时从右向左；
>2. 预处理需要 O(m + σ) 的时间和空间复杂度；
>3. 匹配阶段需要 O(m × n) 的时间复杂度；
>4. 匹配阶段在最坏情况下需要 3n 次字符比较；
>5. 最优复杂度 O(n/m)；

