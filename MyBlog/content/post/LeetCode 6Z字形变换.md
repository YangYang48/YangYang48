---
title: "LeetCode 6Z字形变换"
date: 2021-07-11T17:12:24+08:00
thumbnailImagePosition: left
thumbnailImage: leetcode/leetcode_6/leetcode6_thumb.jpg
coverImage: leetcode/leetcode_6/leetcode6_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- 2021
- July 
tags:
- leetcode
- C++
- Java
- 动态规划
showSocial: false
---

leetcode第6题，采用动态规划的算法来完成。

<!--more-->
# Z字形变换


## 问题描述

将一个给定字符串根据给定的行数，以从上往下、从左到右进行 Z 字形排列。

比如输入字符串为 "LEETCODEISHIRING" 行数为 3 时，排列如下：**(这里用#表示空格加以区分)**

> L # C #  I # R #
E T O E S I I G
E # D # H # N #

之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如："LCIRETOESIIGEDHN"。

请你实现这个将字符串进行指定行数变换的函数：

> string convert(string s, int numRows);



## 示例 1:

> 输入: s = "LEETCODEISHIRING", numRows = 3
输出: "LCIRETOESIIGEDHN"

## 示例 2:

> 输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。

## 示例 3：

> 输入: s = "LEETCODEISHIRING", numRows = 4
输出: "LDREOEIIECIHNTSG"
解释:
> L #  # D # # R
E # O E # I I
E C # I  H # N
T # #  S # # G





# 思路1：遍历法
**这里以c/c++语言来具体化**



 **具体思路开始**（可以参考下面几幅图）
 1. 创建一个字符串的动态数组vector < string  >  str(min(int(s.length()),numRows))，用来存放numRows个元素

 
2. 循环时，动态数组的行数增加或减小，设置一个flag值用于转换方向，设置一个i值用于累加行数，判断如果i为0或者i为numRows-1，flag反值，继续叠加（叠加不一定是正叠加，可能是负叠加）
3. 遍历字符串的动态数组

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102118596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102127513.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102136370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102144446.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102153385.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019083010220362.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102210178.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102217590.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102224615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102232207.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102239759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102249900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102301165.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102311546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102319725.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190830102327137.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmdqdTE0NzUzMjg5Ng==,size_16,color_FFFFFF,t_70)
## 代码

```c
/*
直接遍历,遍历的过程中输出添加到vector<string> str
*/
class Solution {
public:
    string convert(string s, int numRows) {
      //边界条件
      if(numRows==1) return s;
      int i=0;
      bool flag=false;
      string tmp;
      vector<string> str(min(int(s.length()),numRows));//强制转换s.length()为int  
      for(char c:s)//c为当前元素，遍历整个s
      {
          str[i]+=c;
          if(i==0||i==numRows-1) flag=!flag;
          i+=flag?1:-1;   
      }
      for(string ss:str)
          tmp+=ss; 
      return tmp;                   
    }
};
```

# 思路2：找规律

 1. 很明显可以查找到规律，第一行和最后一行的间距都是2*numRows-2
 2. 其他的行，都会出现两次辗转累加的过程，而两次累加和为2*numRows-2，例如第i行，第二个数比第一个数多2*numRows-2-2*i，第三个数则比第二个数多2*i
 3. 设置一个flag用来判断辗转



## 代码

```c
/*
找规律
*/
class Solution {
public:
    string convert(string s, int numRows) {
      //临时变量,用于存放新数组
        string tmp;
        int i,j;
        bool flag=true;//除第一行最后一行都会辗转相加
      //判断边界条件
       if(s.length() < numRows) return s;
       else if(numRows == 1) return s;
       else if(numRows == 2)//奇偶输出
       {
           for(i = 0;i < s.length();i += 2)
           {
               tmp.push_back(s[i]);
           }
           for(i = 1;i <s.length();i += 2)
           {
               tmp.push_back(s[i]);
           }
           return tmp;
       }
       else //一般情况
       {
           for( i=0;i<numRows;i++)
           {
               
               if(i == 0)
               {
                   for(j=i;j<s.length();j+= numRows*2-2)
                       tmp.push_back(s[j]);
               }
               else if(i == numRows-1)
               {
                   for(j=i;j<s.length();j+= numRows*2-2)
                       tmp.push_back(s[j]);
               }
               else//其他的位置
               {
                   flag=true;//下一行自动为true
                   tmp.push_back(s[i]);//第一列第一个数
                   j=i;
                   while(j<s.length())
                   {
                       if(flag)
                       {
                       flag=false;
                      
                           j+= numRows*2-2-2*i;
                           if(j>=s.length()) break;
                            tmp.push_back(s[j]);
                              
                       }
                       if(!flag)
                       {
                       flag=true;
                      
                           j+= 2*i;
                           if(j>=s.length()) break;
                            tmp.push_back(s[j]);
                         
                       }
                   }
               }    
           }
           
       }
        return tmp;
    }
};
```

**如果本文对宝宝打开思路有帮助，可以点个赞哦~**


# 参考

[[1]最忆是南山](https://leetcode-cn.com/problems/zigzag-conversion/solution/tong-guo-zhao-chu-shu-ju-zhi-jian-de-nei-zai-gui-l/)

[[2]扁扁熊](https://leetcode-cn.com/problems/zigzag-conversion/solution/6-z-zi-xing-bian-huan-c-c-by-bian-bian-xiong/)

[[3]力扣官方](https://leetcode-cn.com/problems/zigzag-conversion/solution/z-zi-xing-bian-huan-by-leetcode/)
