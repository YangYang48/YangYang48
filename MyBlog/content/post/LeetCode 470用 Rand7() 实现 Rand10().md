---
title: "LeetCode 470用 Rand7() 实现 Rand10()"
date: 2022-10-30T10:00:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_470/leetcode470_thumb.jpg
coverImage: leetcode/leetcode_470/leetcode470_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 数据结构
- 算法
- 2022
- October
tags:
- 拒绝采样
- C++
- Java
showSocial: false
---

leetcode第470题，用 Rand7() 实现 Rand10()。采用拒绝采样来完成。

<!--more-->
# LeetCode 470用 Rand7() 实现 Rand10()

给定方法 `rand7` 可生成 `[1,7]` 范围内的均匀随机整数，试写一个方法 `rand10` 生成 `[1,10]` 范围内的均匀随机整数。

你只能调用 `rand7()` 且不能调用其他方法。请不要使用系统的 `Math.random()` 方法。

每个测试用例将有一个内部参数 `n`，即你实现的函数 `rand10()` 在测试时将被调用的次数。请注意，这不是传递给 `rand10()` 的参数。

**示例 1:**

```text
输入: 1
输出: [2]
```

**示例 2:**

```text
输入: 2
输出: [2,8]
```

**示例 3:**

```text
输入: 3
输出: [3,8,10]
```

**提示:**

- `1 <= n <= 105`

 **进阶:**

- `rand7()`调用次数的 [期望值](https://en.wikipedia.org/wiki/Expected_value) 是多少 ?
- 你能否尽量少调用 `rand7()` ?

# 思路

这题的思路就是拒绝采样的思路

##### 从 `rand10()` 到 `rand7()`

如果题目是给你 `rand10()`，让你生成 `1～7` 之间的某个数，那非常好办，我们只要不断调用 `rand10()` 即可，直到得到我们要的数，但是为什么可以呢？你可能会怀疑这个是不是等概率的，我们来计算一下

- 如果第一次就 `rand`到 `1～7`之间的数，那就是直接命中了，概率为 1/10
- 如果第二次命中,那么第一次必定没命中，没命中的概率为3/10,再乘命中的概率1/10,所以第二次命中的概率是(3/10) * (1/10)

![image.png](https://pic.leetcode-cn.com/13662225e7f9704ff4475d2a539c7228028ec61d3762f94fb833d29fb237c808-image.png)

1. `ran7()` (1~7)之间的随机数
2. `ran7()-1` (0~6)之间的随机数
3. `(rand7()-1)*7` (0,7,14,21,28,35,42)之间的随机数
4. `(rand7()-1)*7 + rand7()-1` 实际上就是(0~48)之间的随机数
5. 判断第四步的值，去除大于等于40的值
6. 第五步的值`mod 10 + 1` 就能求的`rand10()`

>这类题有通用的解题思路
>
>`randN` --> `randM`
>
>`randN` --> `randX`  （`X>=M`，X是N的整数倍N=7，X=49，M=10）
>
>`randX` --> `randY` （`randY` Y又是M的整数倍 Y=40，M=10）
>
>`randY` `mod M + 1` --> `randM`
>
>举例说明`rand3` --> `rand11`
>
>`rand3` --> `rand27` -->` rand22` --> `rand11`
>
>1. `rand3()` (1~3)之间的随机数
>2. `rand3()-1` (0~2)之间的随机数
>3. `(rand3()-1)*3` (0, 3, 6)之间的随机数
>4. `(rand3()-1)*9` (0, 9, 18)之间的随机数
>5. `(rand3()-1)*9 +  (rand3()-1)*3  + rand3()-1` (0~26)之间的随机数
>6. 第五步产生的结果 >=22,重复第五步
>7. `mod 11 + 1` --> (1~11)之间的随机数

# 解题

```java
class Solution extends SolBase {
    public int rand10() {
        int tmp = 40;
        while(tmp >= 40)
        {
            tmp = (rand7() - 1) * 7 + rand7() - 1;
        }
        return tmp % 10 + 1;
    }
}
```

