---
title: "AES加密和解密流程"
date: 2023-10-03
thumbnailImagePosition: left
thumbnailImage: cryptography/aes/aes_thumb.jpg
coverImage: cryptography/aes/aes_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- AES
- 2023
- October
tags:
- 加密
- 解密
- BASE64
- AES128
- AES192
- AES256
- 轮密钥加
- 列混淆
- 字节代换
showSocial: false
---

AES (Advanced Encryption Standard)是一种区块加密标准算法，它的提出是为了升级替换原有的DES加密算法。因此它的安全强度高于DES算法。


<!--more-->
# 1简介



AES的加密模式

| AES的加密模式 | 说明                                                        |
| ------------- | ----------------------------------------------------------- |
| 密钥长度      | `AES128`、`AES192`、`AES256`                                |
| 加密模式      | `ECB`、`CBC`、`CFB`、`OFB`、`CTR`、`CCM`、`GCM`、`XTS`      |
| 填充模式      | `NONE`、`PKCS7`、`ZERO`、`ANS1923`、`ISO7816_4`、`ISO10126` |

现在的加密库一般会提供一些默认值，所以用户使用AES加密的时候，并没有去做这方面的配置。

密钥长度

这些密钥长度是以bit为单位的，一个字节为8个bit，所以128对应是16个字节，192对应24个字节，256对应是32个字节。

AES128

规定用于的秘钥是16个字节，超过16个字节，它不会被使用就当做没看见（截断处理）

然后，会把整个数据分成四行四列

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_2.png" thumbnail="/cryptography/aes/aes_2.png" title="">}}

AES192

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_4.png" thumbnail="/cryptography/aes/aes_4.png" title="">}}

AES256

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_5.png" thumbnail="/cryptography/aes/aes_5.png" title="">}}

以AES-256为例，假如现在我们想破解它，其难易程度如何呢？2^256^就是256位AES的密钥空间的组合数，远大于地球所有沙子的总数量（3×10^23）^。2^256^ > 2^(10*25)^ > 10^(3*25)^=10^75^>>3×10^23^。现在问题来了：假设你能够把每一粒沙子做出一个存储设备，存一个值。你只能存储3*10^23^*个不同的答案。而你没法全部试一遍，这样破解难度依靠现有技术资源和时间长度，几乎是无法进行计算破解，难度系数等同于不可能。

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_1.png" thumbnail="/cryptography/aes/aes_1.png" title="">}}

加密模式

AES有很多加密模式，其中ECB是最经典的加密模式，其他很多模式都是通过ECB改变过来的，本文主要以ECB为主，后续顺带说一下CBC的加密模式。

填充模式

AES加密数据是以16字节为一组进行分组加密。要求明文的长度一定要是**16个字节的倍数**。如果不够16个字节的倍数的话，需要填充至16个字节倍数。

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_6.png" thumbnail="/cryptography/aes/aes_6.png" title="">}}

> **填充模式注意事项**
>
> 除了**NONE模式**填充以外的任何填充模式下，就算原始明文长度已经是16个字节的倍数，也一定要进行填充
>
> {{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_7.png" thumbnail="/cryptography/aes/aes_7.png" title="">}}

AES排列方式

在AES当中，它的排列方式是以列从上往下进行排列的，如下图所示

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_13.png" thumbnail="/cryptography/aes/aes_13.png" title="">}}

# 1加密

## 1.1加密流程

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_8.png" thumbnail="/cryptography/aes/aes_8.png" title="">}}

## 1.2加密器内部流程

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_9.png" thumbnail="/cryptography/aes/aes_9.png" title="">}}

AES128加密器（10轮）

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_10.png" thumbnail="/cryptography/aes/aes_10.png" title="">}}

AES192加密器（12轮）

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_11.png" thumbnail="/cryptography/aes/aes_11.png" title="">}}

AES256加密器（14轮）

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_12.png" thumbnail="/cryptography/aes/aes_12.png" title="">}}

## 1.3秘钥扩展

秘钥扩展就是把用户输入的（16/24/32）字节的秘钥扩展到（176,208,240）字节的轮秘钥。通俗的来讲，将用户输入的秘钥，扩展成比较长的秘钥。

> 176/208/240这三个数是怎么来的
>
> （`11*16 = 176`），（` 13*16 = 208`），（` 15*16 = 240`）

nk值（秘钥的列数）

AES128的nk值为4（16/4=4）

AES192的nk值为6（24/4=6）

AES256的nk值为8（32/4=8）

## 1.4秘钥扩展公式

这里以秘钥是0102030405060708090A0B0C0D0E0F10为例子

### 1.4.1情况1

情况1：[当前列] % nk值 == 0

```
公式：[当前列-nk值] ⊕ 轮常量异或(字节代换(列位移([当前列-1])))
```

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_14.png" thumbnail="/cryptography/aes/aes_14.png" title="">}}

其中涉及到了**字节代换表**

|      |  0   |  1   |  2   |  3   |  4   |  5   |  6   |  7   |  8   |  9   |  A   |  B   |  C   |  D   |  E   |  F   |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|  0   |  63  |  7C  |  77  |  7B  |  F2  |  6B  |  6F  |  C5  |  30  |  01  |  67  |  2B  |  FE  |  D7  |  AB  |  76  |
|  1   |  CA  |  82  |  C9  |  7D  |  FA  |  59  |  47  |  F0  |  AD  |  D4  |  A2  |  AF  |  9C  |  A4  |  72  |  C0  |
|  2   |  B7  |  FD  |  93  |  26  |  36  |  3F  |  F7  |  CC  |  34  |  A5  |  E5  |  F1  |  71  |  D8  |  31  |  15  |
|  3   |  04  |  C7  |  23  |  C3  |  1   |  96  |  05  |  9A  |  07  |  12  |  80  |  E2  |  EB  |  27  |  B2  |  75  |
|  4   |  09  |  83  |  2C  |  1A  |  1B  |  6E  |  5A  |  A0  |  52  |  3B  |  D6  |  B3  |  29  |  E3  |  2F  |  84  |
|  5   |  53  |  D1  |  00  |  ED  |  20  |  FC  |  B1  |  5B  |  6A  |  CB  |  BE  |  39  |  4A  |  4C  |  58  |  CF  |
|  6   |  D0  |  EF  |  AA  |  FB  |  43  |  4D  |  33  |  85  |  45  |  F9  |  02  |  7F  |  50  |  3C  |  9F  |  A8  |
|  7   |  51  |  A3  |  40  |  8F  |  92  |  9D  |  38  |  F5  |  BC  |  B6  |  DA  |  21  |  10  |  FF  |  F3  |  D2  |
|  8   |  CD  |  0C  |  13  |  EC  |  5F  |  97  |  44  |  17  |  C4  |  A7  |  7E  |  3D  |  64  |  5D  |  19  |  73  |
|  9   |  60  |  81  |  4F  |  DC  |  22  |  2A  |  90  |  88  |  46  |  EE  |  B8  |  14  |  DE  |  5E  |  0B  |  DB  |
|  A   |  E0  |  32  |  3A  |  0A  |  49  |  06  |  24  |  5C  |  C2  |  D3  |  AC  |  62  |  91  |  95  |  E4  |  79  |
|  B   |  E7  |  C8  |  37  |  6D  |  8D  |  D5  |  4E  |  A9  |  6C  |  56  |  F4  |  EA  |  65  |  7A  |  AE  |  08  |
|  C   |  BA  |  78  |  25  |  2E  |  1C  |  A6  |  B4  |  C6  |  E8  |  DD  |  74  |  1F  |  4B  |  BD  |  8B  |  8A  |
|  D   |  70  |  3E  |  B5  |  66  |  48  |  03  |  F6  |  0E  |  61  |  35  |  57  |  B9  |  86  |  C1  |  1D  |  9E  |
|  E   |  E1  |  F8  |  98  |  11  |  69  |  D9  |  8E  |  94  |  9B  |  1E  |  87  |  E9  |  CE  |  55  |  28  |  DF  |
|  F   |  8C  |  A1  |  89  |  0D  |  BF  |  E6  |  42  |  68  |  41  |  99  |  2D  |  0F  |  B0  |  54  |  BB  |  16  |

另外，还有常量轮值表

|  01  |  02  |  04  |  08  |  10  |  20  |  40  |  80  |  1B  |  36  |  6C  |  db  |  ab  |  4d  |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |

#### 1.4.1.1AES128

对于AES128而言，10轮循环，每轮循环正好对应一次常量轮值表中的元素

[当前列] % nk值 == 0，这里的nk值为4，所以就是04的倍数，总共扩展`16*16*10`的大小

```
04,08,0C,10,14,18,1C,20,24,28
```

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_15.png" thumbnail="/cryptography/aes/aes_15.png" title="">}}

```
第04列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
0D			0E			AB			AB ⊕ 01 =AA	       01 = AB
0E			0F			76			76				 	02 = 74
0F			10			CA			CA					03 = C9
10			0D			D7			D7					04 = D3
第08列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
AA			76			38			38 ⊕ 02 = 3A	   AB = 91
76			CA			74			74					74 = 0
CA			C7			C6			C6					C9 = 0F
C7			AA			AC			AC					D3 = 7F
第0C列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
32			7C			10			10 ⊕ 04 = 14	   91 = 85
7C			CE			8B			8B					0 = 8B
CE			B4			8D			8D					0F = 82
B4			32			23			23					7F = 5C
第10列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
10			8F			73			73 ⊕ 08 = 7B	   85 = FE
8F			89			A7			A7					8B = 2C
89			3F			75			75					82 = F7
3F			10			CA			CA					5C = 96
第14列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
76			A9			D3			D3 ⊕ 10 = C3	   FE = 3D
A9			7A			DA			DA					2C = F6
7A			DA			57			57					F7 = A0
DA			76			38			38					96 = AE
第18列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
69			AC			91			91 ⊕ 20 = B1	   3D = 8C
AC			9D			5E			5E					F6 = A8
9D			FF			16			16					A0 = B6
FF			69			F9			F9					AE = 57
第1C列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
83			22			93			93 ⊕ 40 = D3	   8C = 5F
22			D8			61			61					A8 = C9
D8			4D			E3			E3					B6 = 55
4D			83			EC			EC					57 = BB
第20列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
C3			EE			28			28 ⊕ 80 = A8	   5F = F7
EE			6A			02			02					C9 = CB
6A			D3			66			66					55 = 33
D3			C3			2E			2E					BB = 95
第24列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
DE			AB			62			62 ⊕ 1B = 79	   F7 = 8E
AB			1C			9C			9C					CB = 57
1C			F4			BF			BF					33 = 8C
F4			DE			1D			1D					95 = 88
第28列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
10			30			04			04 ⊕ 36 = 32	   8E = BC
30			22			93			93					57 = C4
22			E2			98			98					8C = 14
E2			10			CA			CA					88 = 42
```

#### 1.4.1.2AES256

同理，对于对于AES256而言，14轮循环，每轮循环正好对应一次常量轮值表中的元素

因为本身AES256就有`16*16*2`的大小，所以需要拓展`16*16*13`的大小

[当前列] % nk值 == 0，这里的nk值为8，所以就是08的倍数，这里扩展`16*16*7`的大小

```
08,10,18,20,28,30,38
```

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_16.png" thumbnail="/cryptography/aes/aes_16.png" title="">}}

```
第08列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
1D			1E			72			72 ⊕ 01 = 73	   01 = 72
1E			1F			C0			C0				 	02 = C2
1F			20			B7			B7					03 = B4
20			1D			A4			A4					04 = A0
第10列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
8F			BA			F4			F4 ⊕ 02 = F6	   72 = 84
BA			A9			D3			D3					C2 = 11
A9			BD			7A			7A					B4 = CE
BD			8F			73			73					A0 = D3
第18列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
B3			B1			C8			C8 ⊕ 04 = CC	   84 = 48
B1			48			52			52					11 = 43
48			47			A0			A0					CE = 6E
47			B3			6D			6D					D3 = BE
第20列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
EC			B5			D5			D5 ⊕ 08 = DD	   48 = 95
B5			4D			E3			E3					43 = A0
4D			9F			DB			DB					6E = B5
9F			EC			CE			CE					BE = 70
第28列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
6D			4B			B3			B3 ⊕ 10 = A3	   95 = 36
4B			57			5B			5B					A0 = FB
57			3D			27			27					B5 = 92
3D			6D			3C			3C					70 = 4C
第30列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
FF			AE			E4			E4 ⊕ 20 = C4	   36 = F2
AE			C9			DD			DD					FB = 26
C9			79			B6			B6					92 = 24
79			FF			16			16					4C = 5A
第38列计算
当前列-1	列位移			字节代换	轮常量异或			异或当前列-nk
33			DE			1D			1D ⊕ 40 = 5D	   F2 = AF
DE			54			20			20					26 = 06
54			B8			6C			6C					24 = 48
B8			33			C3			C3					5A = 99
```

### 1.4.2情况2

情况2比较特殊，只有在AES256存在。

情况2：在AES256中，[当前列]%nk值 != 0 && [当前列]%4 == 0

```
公式：[当前列-nk值] ⊕ 字节代换([当前列-1])
```

[当前列]%nk值 != 0 && [当前列]%4 == 0，这里的nk值为8，所以就是08的倍数，这里扩展`16*16*6`的大小

```
0C,14,1C,24,2C,34
```

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_17.png" thumbnail="/cryptography/aes/aes_17.png" title="">}}

```
第0C列计算
当前列-1	字节代换	异或当前列-nk
73			8F	   		11 = 9E
C0			BA			12 = A8
B7			A9			13 = BA
B4			8D			14 = 99
第14列计算
当前列-1	字节代换	异或当前列-nk
FE			BB	   		9E = 25
DB			B9			A8 = 11
72			40			BA = FA
6B			7F			99 = E6
第1C列计算
当前列-1	字节代换	异或当前列-nk
C8			E8	   		25 = CD
56			B1			11 = A0
A4			49			FA = B3
71			A3			E6 = 45
第24列计算
当前列-1	字节代换	异或当前列-nk
D0			70	   		CD = BD
ED			55			A0 = F5
D4			48			B3 = FB
DE			1D			45 = 58
第2C列计算
当前列-1	字节代换	异或当前列-nk
D0			70	   		BD = CD
9B			14			F5 = E1
90			60			FB = 9B
88			C4			58 = 9C
第34列计算
当前列-1	字节代换	异或当前列-nk
3A			80	   		CD = 4D
06			6F			E1 = 8E
C4			1C			9B = 87
7D			FF			9C = 63
```

### 1.4.3情况3

情况3，不属于情况1和情况2的其余情况

```
公式：[当前列-nk值] ⊕ [当前列-1]
```

#### 1.4.3.1AES128

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_18.png" thumbnail="/cryptography/aes/aes_18.png" title="">}}

```
第05列计算
当前列-1	异或当前列-nk
AB			05 = AE
74			06 = 72
C9			07 = CE
D3			08 = DB
第06列计算
当前列-1	异或当前列-nk
AE			09 = A7
72			0A = 78
CE			0B = C5
DB			0C = D7
第07列计算
当前列-1	异或当前列-nk
A7			0D = AA
78			0E = 76
C5			0F = CA
D7			10 = C7

第09列计算
当前列-1	异或当前列-nk
91			AE = 3F
00			72 = 72
0F			CE = C1
7F			DB = A4
第0A列计算
当前列-1	异或当前列-nk
3F			A7 = 98
72			78 = 0A
C1			C5 = 04
A4			D7 = 73
第0B列计算
当前列-1	异或当前列-nk
98			AA = 32
0A			76 = 7C
04			CA = CE
73			C7 = 84

第0D列计算
当前列-1	异或当前列-nk
85			3F = BA
8B			72 = F9
82			C1 = 43
5C			A4 = F8
第0E列计算
当前列-1	异或当前列-nk
BA			98 = 22
F9			0A = F3
43			04 = 47
F8			73 = 8B
第0F列计算
当前列-1	异或当前列-nk
22			32 = 10
F3			7C = 8F
47			CE = 89
8B			B4 = 3F

第11列计算
当前列-1	异或当前列-nk
FE			BA = 44
2C			F9 = D5
F7			43 = B4
96			F8 = 6E
第12列计算
当前列-1	异或当前列-nk
44			22 = 66
D5			F3 = 26
B4			47 = F3
6E			8B = E5
第13列计算
当前列-1	异或当前列-nk
66			10 = 76
26			8F = A9
F3			89 = 7A
E5			3F = DA

第15列计算
当前列-1	异或当前列-nk
3D			44 = 79
F6			D5 = 23
A0			B4 = 14
AE			6E = C0
第16列计算
当前列-1	异或当前列-nk
79			66 = 1F
23			26 = 05
14			F3 = E7
C0			E5 = 25
第17列计算
当前列-1	异或当前列-nk
1F			76 = 69
05			A9 = AC
E7			7A = 9D
25			DA = FF

第19列计算
当前列-1	异或当前列-nk
8C			79 = F5
A8			23 = 8B
B6			14 = A2
57			C0 = 97
第1A列计算
当前列-1	异或当前列-nk
F5			1F = EA
8B			05 = 8E
A2			E7 = 45
97			25 = B2
第1B列计算
当前列-1	异或当前列-nk
EA			69 = 83
8E			AC = 22
45			9D = D8
B2			FF = 4D

第1D列计算
当前列-1	异或当前列-nk
5F			F5 = AA
C9			8B = 42
55			A2 = F7
BB			97 = 2C
第1E列计算
当前列-1	异或当前列-nk
AA			EA = 40
42			8E = CC
F7			45 = B2
2C			B2 = 9E
第1F列计算
当前列-1	异或当前列-nk
40			83 = C3
CC			22 = EE
B2			D8 = 6A
9E			4D = D3

第21列计算
当前列-1	异或当前列-nk
F7			AA = 5D
CB			42 = 89
33			F7 = C4
95			2C = B9
第22列计算
当前列-1	异或当前列-nk
5D			40 = 1D
89			CC = 45
C4			B2 = 76
B9			9E = 27
第23列计算
当前列-1	异或当前列-nk
1D			C3 = DE
45			EE = AB
76			6A = 1C
27			D3 = F4

第25列计算
当前列-1	异或当前列-nk
8E			5D = D3
57			89 = DE
8C			C4 = 48
88			B9 = 31
第26列计算
当前列-1	异或当前列-nk
D3			1D = CE
DE			45 = 9B
48			76 = 3E
31			27 = 16
第27列计算
当前列-1	异或当前列-nk
CE			DE = 10
9B			AB = 30
3E			1C = 22
16			F4 = E2

第29列计算
当前列-1	异或当前列-nk
BC			D3 = 6F
C4			DE = 1A
14			48 = 5C
42			31 = 73
第2A列计算
当前列-1	异或当前列-nk
6F			CE = A1
1A			9B = 81
5C			3E = 62
73			16 = 65
第2B列计算
当前列-1	异或当前列-nk
A1			10 = B1
81			30 = B1
62			22 = 40
65			E2 = 87
```



#### 1.4.3.2AES256

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_19.png" thumbnail="/cryptography/aes/aes_19.png" title="">}}

```
第09列计算
当前列-1	异或当前列-nk
72			05 = 77
C2			06 = C4
B4			07 = B3
A0			08 = A8
第0A列计算
当前列-1	异或当前列-nk
77			09 = 7E
C4			0A = CE
B3			0B = B8
A8			0C = A4
第0B列计算
当前列-1	异或当前列-nk
7E			0D = 73
CE			0E = C0
B8			0F = B7
A4			10 = B4

第0D列计算
当前列-1	异或当前列-nk
9E			15 = 8B
A8			16 = 8E
BA			17 = AD
99			18 = 81
第0E列计算
当前列-1	异或当前列-nk
8B			19 = 92
8E			1A = A4
AD			1B = B6
81			1C = 9D
第0F列计算
当前列-1	异或当前列-nk
92			1D = 8F
A4			1E = BA
B6			1F = A9
9D			20 = BD

第11列计算
当前列-1	异或当前列-nk
84			77 = F3
11			C4 = D5
CE			B3 = 7D
D3			A8 = 7B
第12列计算
当前列-1	异或当前列-nk
F3			7E = 8D
D5			CE = 1B
7D			B8 = C5
7B			A4 = DF
第13列计算
当前列-1	异或当前列-nk
8D			73 = FE
1B			70 = DB
C5			B7 = 72
DF			B4 = 6B

第15列计算
当前列-1	异或当前列-nk
25			8B = AE
11			8E = AF
FA			AD = 57
E6			81 = 67
第16列计算
当前列-1	异或当前列-nk
AE			92 = 3C
AF			A4 = 0B
57			B6 = E1
67			9D = FA
第17列计算
当前列-1	异或当前列-nk
3C			8F = B3
0B			BA = B1
E1			A9 = 48
FA			BD = 47

第19列计算
当前列-1	异或当前列-nk
48			F3 = BB
43			D5 = 96
6E			7D = 13
BE			7B = C5
第1A列计算
当前列-1	异或当前列-nk
BB			8D = 36
96			1B = 8D
13			C5 = D6
C5			DF = 1A
第1B列计算
当前列-1	异或当前列-nk
36			FE = C8
8D			DB = 56
D6			72 = A4
1A			6B = 71

第1D列计算
当前列-1	异或当前列-nk
CD			AE = 63
A0			AF = 0F
B3			57 = E4
45			67 = 22
第1E列计算
当前列-1	异或当前列-nk
63			3C = 5F
0F			0B = 04
E4			E1 = 05
22			FA = D8
第1F列计算
当前列-1	异或当前列-nk
5F			B3 = EC
04			B1 = B5
05			48 = 4D
D8			47 = 9F

第21列计算
当前列-1	异或当前列-nk
95			BB = 2E
A0			96 = 36
B5			13 = A6
70			C5 = B5
第22列计算
当前列-1	异或当前列-nk
2E			36 = 18
36			8D = BB
A6			D6 = 70
B5			1A = AF
第23列计算
当前列-1	异或当前列-nk
18			C8 = D0
BB			56 = ED
70			A4 = D4
AF			71 = DE

第25列计算
当前列-1	异或当前列-nk
BD			63 = DE
F5			0F = FA
FB			E4 = 1F
58			22 = 7A
第26列计算
当前列-1	异或当前列-nk
DE			5F = 81
FA			04 = FE
1F			05 = 1A
7A			DB = A2
第27列计算
当前列-1	异或当前列-nk
81			EC = 6D
FE			B5 = 4B
1A			4D = 57
A2			9F = 3D

第29列计算
当前列-1	异或当前列-nk
36			2E = 18
FB			36 = CD
92			A6 = 34
4C			B5 = F9
第2A列计算
当前列-1	异或当前列-nk
18			18 = 00
CD			BB = 76
34			70 = 44
F9			AF = 56
第2B列计算
当前列-1	异或当前列-nk
00			D0 = D0
76			ED = 9B
44			D4 = 90
56			DE = 88

第2D列计算
当前列-1	异或当前列-nk
CD			DE = 13
E1			FA = 1B
9B			1F = 84
9C			7A = E6
第2E列计算
当前列-1	异或当前列-nk
13			81 = 92
1B			FE = E5
84			1A = 9E
E6			A2 = 44
第2F列计算
当前列-1	异或当前列-nk
92			6D = FF
E5			4B = AE
9E			57 = C9
44			3D = 79

第31列计算
当前列-1	异或当前列-nk
F2			18 = EA
26			CD = EB
24			34 = 10
5A			F9 = A3
第32列计算
当前列-1	异或当前列-nk
EA			00 = EA
EB			76 = 9D
10			44 = 54
A3			56 = F5
第33列计算
当前列-1	异或当前列-nk
EA			D0 = 3A
9D			9B = 06
54			90 = C4
F5			88 = 7D

第35列计算
当前列-1	异或当前列-nk
4D			13 = 5E
8E			1B = 95
87			84 = 03
63			E6 = 85
第36列计算
当前列-1	异或当前列-nk
5E			92 = CC
95			E5 = 70
03			9E = 9D
85			44 = C1
第37列计算
当前列-1	异或当前列-nk
CC			FF = 33
70			AE = DE
9D			C9 = 54
C1			79 = B8

第39列计算
当前列-1	异或当前列-nk
AF			EA = 45
06			EB = ED
48			10 = 58
99			A3 = 3A
第3A列计算
当前列-1	异或当前列-nk
45			EA = AF
ED			9D = 70
58			54 = 0C
3A			F5 = CF
第3B列计算
当前列-1	异或当前列-nk
AF			3A = 95
70			06 = 76
0C			C4 = C8
CF			7D = B2
```



## 1.5轮密钥加

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_9.png" thumbnail="/cryptography/aes/aes_9.png" title="">}}

轮秘钥加有两个参数，分别为**明文**和**轮秘钥**

对输入数组（16个字节）和轮秘钥的其中16个字节进行`异或运算`。

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_20.png" thumbnail="/cryptography/aes/aes_20.png" title="">}}

### 1.5.1AES128

对于AES128而言，正好需要11次轮密钥加跟上述的秘钥拓展对应上了。

对于轮秘钥的选择

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_21.png" thumbnail="/cryptography/aes/aes_21.png" title="">}}

### 1.5.2AES256

对于AES256而言，正好需要15次轮密钥加跟上述的秘钥拓展对应上了。

对于轮秘钥的选择

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_22.png" thumbnail="/cryptography/aes/aes_22.png" title="">}}

# 1.6字节代换

字节代换就是通过字节代换表来替换其中的字符数字

下面为字节代换表

|      |  0   |  1   |  2   |  3   |  4   |  5   |  6   |  7   |  8   |  9   |  A   |  B   |  C   |  D   |  E   |  F   |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|  0   |  63  |  7C  |  77  |  7B  |  F2  |  6B  |  6F  |  C5  |  30  |  01  |  67  |  2B  |  FE  |  D7  |  AB  |  76  |
|  1   |  CA  |  82  |  C9  |  7D  |  FA  |  59  |  47  |  F0  |  AD  |  D4  |  A2  |  AF  |  9C  |  A4  |  72  |  C0  |
|  2   |  B7  |  FD  |  93  |  26  |  36  |  3F  |  F7  |  CC  |  34  |  A5  |  E5  |  F1  |  71  |  D8  |  31  |  15  |
|  3   |  04  |  C7  |  23  |  C3  |  01  |  96  |  05  |  9A  |  07  |  12  |  80  |  E2  |  EB  |  27  |  B2  |  75  |
|  4   |  09  |  83  |  2C  |  1A  |  1B  |  6E  |  5A  |  A0  |  52  |  3B  |  D6  |  B3  |  29  |  E3  |  2F  |  84  |
|  5   |  53  |  D1  |  00  |  ED  |  20  |  FC  |  B1  |  5B  |  6A  |  CB  |  BE  |  39  |  4A  |  4C  |  58  |  CF  |
|  6   |  D0  |  EF  |  AA  |  FB  |  43  |  4D  |  33  |  85  |  45  |  F9  |  02  |  7F  |  50  |  3C  |  9F  |  A8  |
|  7   |  51  |  A3  |  40  |  8F  |  92  |  9D  |  38  |  F5  |  BC  |  B6  |  DA  |  21  |  10  |  FF  |  F3  |  D2  |
|  8   |  CD  |  0C  |  13  |  EC  |  5F  |  97  |  44  |  17  |  C4  |  A7  |  7E  |  3D  |  64  |  5D  |  19  |  73  |
|  9   |  60  |  81  |  4F  |  DC  |  22  |  2A  |  90  |  88  |  46  |  EE  |  B8  |  14  |  DE  |  5E  |  0B  |  DB  |
|  A   |  E0  |  32  |  3A  |  0A  |  49  |  06  |  24  |  5C  |  C2  |  D3  |  AC  |  62  |  91  |  95  |  E4  |  79  |
|  B   |  E7  |  C8  |  37  |  6D  |  8D  |  D5  |  4E  |  A9  |  6C  |  56  |  F4  |  EA  |  65  |  7A  |  AE  |  08  |
|  C   |  BA  |  78  |  25  |  2E  |  1C  |  A6  |  B4  |  C6  |  E8  |  DD  |  74  |  1F  |  4B  |  BD  |  8B  |  8A  |
|  D   |  70  |  3E  |  B5  |  66  |  48  |  03  |  F6  |  0E  |  61  |  35  |  57  |  B9  |  86  |  C1  |  1D  |  9E  |
|  E   |  E1  |  F8  |  98  |  11  |  69  |  D9  |  8E  |  94  |  9B  |  1E  |  87  |  E9  |  CE  |  55  |  28  |  DF  |
|  F   |  8C  |  A1  |  89  |  0D  |  BF  |  E6  |  42  |  68  |  41  |  99  |  2D  |  0F  |  B0  |  54  |  BB  |  16  |

举例说明

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_23.png" thumbnail="/cryptography/aes/aes_23.png" title="">}}

## 1.7行位移

行位移规则

```
第一行不变
第二行向左循环移动1字节
第三行向做循环移动2字节
第四行向左循环移动3字节
```

举例说明

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_24.png" thumbnail="/cryptography/aes/aes_24.png" title="">}}

## 1.8列混淆

列混淆是加密过程中最复杂的一块

```
公式：使用GF(2^8)对[列混淆左乘矩阵的行]与[当前字节的列]进行矩阵乘法
```

举例说明

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_25.png" thumbnail="/cryptography/aes/aes_25.png" title="">}}

```
上述计算
[x1,y1] = 0x02 × [x1,y1] ⊕ 0x03 × [x2,y1] ⊕ 0x01 × [x3,y1] ⊕ 0x01 × [x4,y1]
[x2,y1] = 0x01 × [x1,y1] ⊕ 0x02 × [x2,y1] ⊕ 0x03 × [x3,y1] ⊕ 0x01 × [x4,y1]
[x3,y3] = 0x01 × [x1,y3] ⊕ 0x01 × [x2,y3] ⊕ 0x02 × [x3,y3] ⊕ 0x03 × [x4,y3]
[x1,y1] = 0x01 × [x1,y4] ⊕ 0x01 × [x2,y4] ⊕ 0x02 × [x3,y4] ⊕ 0x03 × [x4,y4]
```

> 在GF(2^8)中**加法和乘法**是不一样的
>
> 1.加法操作
>
> 直接改成**异或操作**，a+b 等效为 a ⊕ b
>
> 2.乘法操作
>
> 2.1.定义一个函数f(a)
>
> ​	如果a小于0x80，则返回2*a
>
> ​	否则返货((2*a)mod 0x100) ⊕ 0x1B
>
> 2.2乘法操作中的其中一个数值是2的N次方，则可以套用以下公式
>
> ```c
> 0x01 × a = a
> 0x02 × a = f(a)
> 0x04 × a = f(0x02 × a)
> 0x08 × a = f(0x04 × a)
> 0x10 × a = f(0x08 × a)
> ...
> ```
>
> 2.3.非2的次方可以拆分后相加（这里的相加操作依然为异或操作）
>
> 例子
>
> ```c
> 0x0D × a = (0x08 × a) ⊕ (0x04 × a) ⊕ (0x01 × a) = f(0x04 × a)⊕ f(0x02 × a)⊕ a
> 0x0D × a = (0x08 × a) ⊕ (0x04 × a) ⊕ (0x02 × a) = f(0x04 × a)⊕ f(0x02 × a)⊕ f(a)
> ```

直接计算实际例子

```
以前面例子中的[x3,y1]为例
[x3,y1] = 0x01 × [x1,y1] ⊕ 0x01 × [x2,y1] ⊕ 0x02 × [x3,y1] ⊕ 0x03 × [x4,y1]
= 0x01 × 0x3B ⊕ 0x01 × 0xF7 ⊕ 0x02 × 0xA8 ⊕ 0x03 × 0xB7
0x01 × 0x3B = 0x3B
0x01 × 0xF7 = 0xF7
0x02 × 0xA8 = f(0xA8) = ((2*0xA8)mod 0x100) ⊕ 0x1B = 0x50 ⊕ 0x1B = 0x4B
0x03 × 0xB7 = f(0xB7) ⊕ 0xB7 = ((2*0xB7)mod 0x100) ⊕ 0x1B ⊕ 0xB7 = 0x75 ⊕ 0xB7 = 0xC2
[x3,y1] = 0x3B ⊕ 0xF7 ⊕ 0x4B ⊕ 0xC2 = 0x45
再举例[x2,y4]
[x2,y4] = 0x01 × [x1,y4] ⊕ 0x02 × [x2,y4] ⊕ 0x03 × [x3,y4] ⊕ 0x01 × [x4,y4]
= 0x01 × 0x27 ⊕ 0x02 × 0x85 ⊕ 0x03 × 0x53 ⊕ 0x01 × 0xD8
0x01 × 0x27 = 0x27
0x02 × 0x85 = f(0x85) = ((2*0x85)mod 0x100) ⊕ 0x1B = A ⊕ 0x1B = 0x11
0x03 × 0x53 = f(0x53) ⊕ 0x53 = (2*0x53) ⊕ 0x53 = A6 ⊕ 0x53 = 0xF5
0x01 × 0xD8 = 0xD8
[x2,y4] = 0x27 ⊕ 0x11 ⊕ 0xF5 ⊕ 0xD8 = 1B
```

## 1.9加密演示

在线AES加减密链接(https://the-x.cn/cryptography/Aes.aspx)

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_3.png" thumbnail="/cryptography/aes/aes_3.png" title="">}}

这里的明文是Hello World!，补充为0

密文是Password，补充为0

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_2.png" thumbnail="/cryptography/aes/aes_2.png" title="">}}

### 1.9.1秘钥扩展

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_26.png" thumbnail="/cryptography/aes/aes_26.png" title="">}}

### 1.9.2加密

#### 1.9.2.1第0轮

第0轮只有轮密钥加

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_32.png" thumbnail="/cryptography/aes/aes_32.png" title="">}}

第一轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_33.png" thumbnail="/cryptography/aes/aes_33.png" title="">}}

第二轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_34.png" thumbnail="/cryptography/aes/aes_34.png" title="">}}

第三轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_35.png" thumbnail="/cryptography/aes/aes_35.png" title="">}}

第四轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_36.png" thumbnail="/cryptography/aes/aes_36.png" title="">}}

第五轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_37.png" thumbnail="/cryptography/aes/aes_37.png" title="">}}

第六轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_38.png" thumbnail="/cryptography/aes/aes_38.png" title="">}}

第七轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_39.png" thumbnail="/cryptography/aes/aes_39.png" title="">}}

第八轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_40.png" thumbnail="/cryptography/aes/aes_40.png" title="">}}

第九轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_41.png" thumbnail="/cryptography/aes/aes_41.png" title="">}}

第十轮（没有列混淆）

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_42.png" thumbnail="/cryptography/aes/aes_42.png" title="">}}

# 2解密

解密和加密的区别只在于

- 执行顺序相反
- 字节代换的表不一样
- 行位移由加密的向左移动改成解密的向右移动
- 列混淆的左乘数组不一样

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_27.png" thumbnail="/cryptography/aes/aes_27.png" title="">}}

里面的解密器流程如下

和加密的流程正好相反

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_28.png" thumbnail="/cryptography/aes/aes_28.png" title="">}}

## 2.1解密的字节代换表

|      |  0   |  1   |  2   |  3   |  4   |  5   |  6   |  7   |  8   |  9   |  A   |  B   |  C   |  D   |  E   |  F   |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
|  0   |  52  |  09  |  6A  |  D5  |  30  |  36  |  A5  |  38  |  BF  |  40  |  A3  |  9E  |  81  |  F3  |  D7  |  FB  |
|  1   |  7C  |  E3  |  39  |  82  |  9B  |  2F  |  FF  |  87  |  34  |  8E  |  43  |  44  |  C4  |  DE  |  E9  |  CB  |
|  2   |  54  |  7B  |  94  |  32  |  A6  |  C2  |  23  |  3D  |  EE  |  4C  |  95  |  0B  |  42  |  FA  |  C3  |  4E  |
|  3   |  08  |  2E  |  A1  |  66  |  28  |  D9  |  24  |  B2  |  76  |  5B  |  A2  |  49  |  6D  |  8B  |  D1  |  25  |
|  4   |  72  |  F8  |  F6  |  64  |  86  |  68  |  98  |  16  |  D4  |  A4  |  5C  |  CC  |  5D  |  65  |  B6  |  92  |
|  5   |  6C  |  70  |  48  |  50  |  FD  |  ED  |  B9  |  DA  |  5E  |  15  |  46  |  57  |  A7  |  8D  |  9D  |  84  |
|  6   |  90  |  D8  |  AB  |  00  |  8C  |  BC  |  D3  |  0A  |  F7  |  E4  |  58  |  05  |  B8  |  B3  |  45  |  06  |
|  7   |  D0  |  2C  |  1E  |  8F  |  CA  |  3F  |  0F  |  02  |  C1  |  AF  |  BD  |  03  |  01  |  13  |  8A  |  6B  |
|  8   |  3A  |  91  |  11  |  41  |  4F  |  67  |  DC  |  EA  |  97  |  F2  |  CF  |  CE  |  F0  |  B4  |  E6  |  73  |
|  9   |  96  |  AC  |  74  |  22  |  E7  |  AD  |  35  |  85  |  E2  |  F9  |  37  |  E8  |  1C  |  75  |  DF  |  6E  |
|  A   |  47  |  F1  |  1A  |  71  |  1D  |  29  |  C5  |  89  |  6F  |  B7  |  62  |  0E  |  AA  |  18  |  BE  |  1B  |
|  B   |  FC  |  56  |  3E  |  4B  |  C6  |  D2  |  79  |  20  |  9A  |  DB  |  C0  |  FE  |  78  |  CD  |  5A  |  F4  |
|  C   |  1F  |  DD  |  A8  |  33  |  88  |  07  |  C7  |  31  |  B1  |  12  |  10  |  59  |  27  |  80  |  EC  |  5F  |
|  D   |  60  |  51  |  7F  |  A9  |  19  |  B5  |  4A  |  0D  |  2D  |  E5  |  7A  |  9F  |  93  |  C9  |  9C  |  EF  |
|  E   |  A0  |  E0  |  3B  |  4D  |  AE  |  2A  |  F5  |  B0  |  C8  |  EB  |  BB  |  3C  |  83  |  53  |  99  |  61  |
|  F   |  17  |  2B  |  04  |  7E  |  BA  |  77  |  D6  |  26  |  E1  |  69  |  14  |  63  |  55  |  21  |  0C  |  7D  |

## 2.2解密逆行位移

```
流程说明
第一行不变
第二行向右循环移动1字节
第三行向右循环移动2字节
第四行向右循环移动3字节
```

举例说明

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_29.png" thumbnail="/cryptography/aes/aes_29.png" title="">}}

## 2.3解密列混淆左乘矩阵

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_30.png" thumbnail="/cryptography/aes/aes_30.png" title="">}}

## 2.4具体步骤

这里的轮秘钥的顺序跟解密是相反的，解密第一轮的轮秘钥正好对应加密过程的最后一次的轮秘钥。

第一轮（**没有逆列混淆**）

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_43.png" thumbnail="/cryptography/aes/aes_43.png" title="">}}

第二轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_44.png" thumbnail="/cryptography/aes/aes_44.png" title="">}}

第三轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_45.png" thumbnail="/cryptography/aes/aes_45.png" title="">}}

第四轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_46.png" thumbnail="/cryptography/aes/aes_46.png" title="">}}

第五轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_47.png" thumbnail="/cryptography/aes/aes_47.png" title="">}}

第六轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_48.png" thumbnail="/cryptography/aes/aes_48.png" title="">}}

第七轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_49.png" thumbnail="/cryptography/aes/aes_49.png" title="">}}

第八轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_50.png" thumbnail="/cryptography/aes/aes_50.png" title="">}}

第九轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_51.png" thumbnail="/cryptography/aes/aes_51.png" title="">}}

第十轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_52.png" thumbnail="/cryptography/aes/aes_52.png" title="">}}

第0轮

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_53.png" thumbnail="/cryptography/aes/aes_53.png" title="">}}

# 3补充说明

总的来说，上面的例子说加密模式为ecb、填充模式为数字0（0x30）的AES128，对16字节的明文字符串通过秘钥是16个字节进行加密和解密操作流程。

除了上述的ecb模式，还有cbc加密模式。

## 3.1CBC加密模式

cbc模式多一个16字节**初始向量**(`iv`)参数

加密流程

- 第一次进入加密器之前，先试用iv对明文分组1进行异或运算
- 字后进入加密器之前，使用上一次加密的密文分组对明文分组进行异或运算

举例说明

```
iv   ⊕ 明文分组1 -> 加密器 -> 密文1
密文1 ⊕ 明文分组2 -> 加密器 -> 密文2
密文2 ⊕ 明文分组3 -> 加密器 -> 密文3
```

解密流程

- 第一次进入解密器之后，先试用当前密文组的前面那块密文组队当前密文组进行异或运算
- 最后一次进入解密器之后用iv对当前密文组进行异或运算

举例说明

```
密文3 -> 解密器 -> ⊕ 密文2 -> 明文3
密文2 -> 解密器 -> ⊕ 密文1 -> 明文2
密文1 -> 解密器 -> ⊕ iv   -> 明文1
```

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_31.png" thumbnail="/cryptography/aes/aes_31.png" title="">}}

## 3.2base64编码解码

Base64算法生成的字符串可以安全地在任何实现了Base64算法的计算机之间传输。Base64典型的用途是给二进制数据（例如：图片文件）进行编码的，我们为了简化，就不用二进制，而是使用一段ASCII文本字符串来作为例子进行讲解。Base64编码出来的数据只会包含最多64种不同的ASCII字符，因此，使用Base64算法，我们就能避免数据在部分系统传输过程中发生改变。

### 3.2.1生成Base64索引表

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_54.png" thumbnail="/cryptography/aes/aes_54.png" title="">}}

### 3.2.2加密方式

首先，我们发现字符串`“Man”`的Base64编码是`“TWFu”`，那么这是怎么转换过来的呢？不急，我们一个一个字符来分析，首先对于`“M”`来说，`"M"`对应的ASCII编码是`77`，二进制形式即`01001101`；同理，字符`“a”`对应的ASCII编码是`97`，二进制表现形式为`01100001`；`“n”`的ASCII编码为`110`，二进制形式为：`01101110`。（ASCII编码表可参考：ASCII码对照表）这三个字符的二进制位组合在一起就变成了一个24位的字符串`“010011010110000101101110”`，接下来，我们从左至右，每次抽取6位作为1组（因为6位一共有2^6=64种不同的组合），因此每一组的6位又代表一个数字（0~63），接下来，我们查看索引表，找到这个数字对应的字符，就是我们最后的结果，是不是很简单呢？

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_57.png" thumbnail="/cryptography/aes/aes_57.png" title="">}}

### 3.2.3举例说明

上面的AES128的时候，有BASE64的字符串`62uCg0QGx/SVxr1iML8iig==`，使用BASE64解码

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_55.png" thumbnail="/cryptography/aes/aes_55.png" title="">}}

发现这些解码出来的正好拼接成4*4的AES128加密后的原始的矩阵

{{< image classes="fancybox center fig-100" src="/cryptography/aes/aes_56.png" thumbnail="/cryptography/aes/aes_56.png" title="">}}

# 4程序demo

另外对于AES而言，代码层面也有对应的写法，这里不具体列出只列出对应的算法demo

```c
//aes.h
#ifndef AES_H
#define AES_H

/**
 * 参数 p: 明文的字符串数组。
 * 参数 plen: 明文的长度,长度必须为16的倍数。
 * 参数 key: 密钥的字符串数组。
 */
void aes(char *p, int plen, char *key);

/**
 * 参数 c: 密文的字符串数组。
 * 参数 clen: 密文的长度,长度必须为16的倍数。
 * 参数 key: 密钥的字符串数组。
 */
void deAes(char *c, int clen, char *key);

#endif
```

具体实现

```c
//aes.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "aes.h"

/**
 * S盒
 */
static const int S[16][16] = { 
    0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5, 0x30, 0x01, 0x67, 0x2b, 0xfe, 0xd7, 0xab, 0x76,
	0xca, 0x82, 0xc9, 0x7d, 0xfa, 0x59, 0x47, 0xf0, 0xad, 0xd4, 0xa2, 0xaf, 0x9c, 0xa4, 0x72, 0xc0,
	0xb7, 0xfd, 0x93, 0x26, 0x36, 0x3f, 0xf7, 0xcc, 0x34, 0xa5, 0xe5, 0xf1, 0x71, 0xd8, 0x31, 0x15,
	0x04, 0xc7, 0x23, 0xc3, 0x18, 0x96, 0x05, 0x9a, 0x07, 0x12, 0x80, 0xe2, 0xeb, 0x27, 0xb2, 0x75,
	0x09, 0x83, 0x2c, 0x1a, 0x1b, 0x6e, 0x5a, 0xa0, 0x52, 0x3b, 0xd6, 0xb3, 0x29, 0xe3, 0x2f, 0x84,
	0x53, 0xd1, 0x00, 0xed, 0x20, 0xfc, 0xb1, 0x5b, 0x6a, 0xcb, 0xbe, 0x39, 0x4a, 0x4c, 0x58, 0xcf,
	0xd0, 0xef, 0xaa, 0xfb, 0x43, 0x4d, 0x33, 0x85, 0x45, 0xf9, 0x02, 0x7f, 0x50, 0x3c, 0x9f, 0xa8,
	0x51, 0xa3, 0x40, 0x8f, 0x92, 0x9d, 0x38, 0xf5, 0xbc, 0xb6, 0xda, 0x21, 0x10, 0xff, 0xf3, 0xd2,
	0xcd, 0x0c, 0x13, 0xec, 0x5f, 0x97, 0x44, 0x17, 0xc4, 0xa7, 0x7e, 0x3d, 0x64, 0x5d, 0x19, 0x73,
	0x60, 0x81, 0x4f, 0xdc, 0x22, 0x2a, 0x90, 0x88, 0x46, 0xee, 0xb8, 0x14, 0xde, 0x5e, 0x0b, 0xdb,
	0xe0, 0x32, 0x3a, 0x0a, 0x49, 0x06, 0x24, 0x5c, 0xc2, 0xd3, 0xac, 0x62, 0x91, 0x95, 0xe4, 0x79,
	0xe7, 0xc8, 0x37, 0x6d, 0x8d, 0xd5, 0x4e, 0xa9, 0x6c, 0x56, 0xf4, 0xea, 0x65, 0x7a, 0xae, 0x08,
	0xba, 0x78, 0x25, 0x2e, 0x1c, 0xa6, 0xb4, 0xc6, 0xe8, 0xdd, 0x74, 0x1f, 0x4b, 0xbd, 0x8b, 0x8a,
	0x70, 0x3e, 0xb5, 0x66, 0x48, 0x03, 0xf6, 0x0e, 0x61, 0x35, 0x57, 0xb9, 0x86, 0xc1, 0x1d, 0x9e,
	0xe1, 0xf8, 0x98, 0x11, 0x69, 0xd9, 0x8e, 0x94, 0x9b, 0x1e, 0x87, 0xe9, 0xce, 0x55, 0x28, 0xdf,
	0x8c, 0xa1, 0x89, 0x0d, 0xbf, 0xe6, 0x42, 0x68, 0x41, 0x99, 0x2d, 0x0f, 0xb0, 0x54, 0xbb, 0x16 
};

/**
 * 逆S盒
 */
static const int S2[16][16] = { 
    0x52, 0x09, 0x6a, 0xd5, 0x30, 0x36, 0xa5, 0x38, 0xbf, 0x40, 0xa3, 0x9e, 0x81, 0xf3, 0xd7, 0xfb,
	0x7c, 0xe3, 0x39, 0x82, 0x9b, 0x2f, 0xff, 0x87, 0x34, 0x8e, 0x43, 0x44, 0xc4, 0xde, 0xe9, 0xcb,
	0x54, 0x7b, 0x94, 0x32, 0xa6, 0xc2, 0x23, 0x3d, 0xee, 0x4c, 0x95, 0x0b, 0x42, 0xfa, 0xc3, 0x4e,
	0x08, 0x2e, 0xa1, 0x66, 0x28, 0xd9, 0x24, 0xb2, 0x76, 0x5b, 0xa2, 0x49, 0x6d, 0x8b, 0xd1, 0x25,
	0x72, 0xf8, 0xf6, 0x64, 0x86, 0x68, 0x98, 0x16, 0xd4, 0xa4, 0x5c, 0xcc, 0x5d, 0x65, 0xb6, 0x92,
	0x6c, 0x70, 0x48, 0x50, 0xfd, 0xed, 0xb9, 0xda, 0x5e, 0x15, 0x46, 0x57, 0xa7, 0x8d, 0x9d, 0x84,
	0x90, 0xd8, 0xab, 0x00, 0x8c, 0xbc, 0xd3, 0x0a, 0xf7, 0xe4, 0x58, 0x05, 0xb8, 0xb3, 0x45, 0x06,
	0xd0, 0x2c, 0x1e, 0x8f, 0xca, 0x3f, 0x0f, 0x02, 0xc1, 0xaf, 0xbd, 0x03, 0x01, 0x13, 0x8a, 0x6b,
	0x3a, 0x91, 0x11, 0x41, 0x4f, 0x67, 0xdc, 0xea, 0x97, 0xf2, 0xcf, 0xce, 0xf0, 0xb4, 0xe6, 0x73,
	0x96, 0xac, 0x74, 0x22, 0xe7, 0xad, 0x35, 0x85, 0xe2, 0xf9, 0x37, 0xe8, 0x1c, 0x75, 0xdf, 0x6e,
	0x47, 0xf1, 0x1a, 0x71, 0x1d, 0x29, 0xc5, 0x89, 0x6f, 0xb7, 0x62, 0x0e, 0xaa, 0x18, 0xbe, 0x1b,
	0xfc, 0x56, 0x3e, 0x4b, 0xc6, 0xd2, 0x79, 0x20, 0x9a, 0xdb, 0xc0, 0xfe, 0x78, 0xcd, 0x5a, 0xf4,
	0x1f, 0xdd, 0xa8, 0x33, 0x88, 0x07, 0xc7, 0x31, 0xb1, 0x12, 0x10, 0x59, 0x27, 0x80, 0xec, 0x5f,
	0x60, 0x51, 0x7f, 0xa9, 0x19, 0xb5, 0x4a, 0x0d, 0x2d, 0xe5, 0x7a, 0x9f, 0x93, 0xc9, 0x9c, 0xef,
	0xa0, 0xe0, 0x3b, 0x4d, 0xae, 0x2a, 0xf5, 0xb0, 0xc8, 0xeb, 0xbb, 0x3c, 0x83, 0x53, 0x99, 0x61,
	0x17, 0x2b, 0x04, 0x7e, 0xba, 0x77, 0xd6, 0x26, 0xe1, 0x69, 0x14, 0x63, 0x55, 0x21, 0x0c, 0x7d 
};

/**
 * 获取整形数据的低8位的左4个位
 */
static int getLeft4Bit(int num) {
	int left = num & 0x000000f0;
	return left >> 4;
}

/**
 * 获取整形数据的低8位的右4个位
 */
static int getRight4Bit(int num) {
	return num & 0x0000000f;
}
/**
 * 根据索引，从S盒中获得元素
 */
static int getNumFromSBox(int index) {
	int row = getLeft4Bit(index);
	int col = getRight4Bit(index);
	return S[row][col];
}

/**
 * 把一个字符转变成整型
 */
static int getIntFromChar(char c) {
	int result = (int) c;
	return result & 0x000000ff;
}

/**
 * 把16个字符转变成4X4的数组，
 * 该矩阵中字节的排列顺序为从上到下，
 * 从左到右依次排列。
 */
static void convertToIntArray(char *str, int pa[4][4]) {
	int k = 0;
	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++) {
			pa[j][i] = getIntFromChar(str[k]);
			k++;
		}
}

/**
 * 打印4X4的数组
 */
static void printArray(int a[4][4]) {
	for(int i = 0; i < 4; i++){
		for(int j = 0; j < 4; j++)
			printf("a[%d][%d] = 0x%x ", i, j, a[i][j]);
		printf("\n");
	}
	printf("\n");
}

/**
 * 打印字符串的ASSCI，
 * 以十六进制显示。
 */
static void printASSCI(char *str, int len) {
	for(int i = 0; i < len; i++)
		printf("0x%x ", getIntFromChar(str[i]));
	printf("\n");
}

/**
 * 把连续的4个字符合并成一个4字节的整型
 */
static int getWordFromStr(char *str) {
	int one = getIntFromChar(str[0]);
	one = one << 24;
	int two = getIntFromChar(str[1]);
	two = two << 16;
	int three = getIntFromChar(str[2]);
	three = three << 8;
	int four = getIntFromChar(str[3]);
	return one | two | three | four;
}

/**
 * 把一个4字节的数的第一、二、三、四个字节取出，
 * 入进一个4个元素的整型数组里面。
 */
static void splitIntToArray(int num, int array[4]) {
	int one = num >> 24;
	array[0] = one & 0x000000ff;
	int two = num >> 16;
	array[1] = two & 0x000000ff;
	int three = num >> 8;
	array[2] = three & 0x000000ff;
	array[3] = num & 0x000000ff;
}

/**
 * 将数组中的元素循环左移step位
 */
static void leftLoop4int(int array[4], int step) {
	int temp[4];
	for(int i = 0; i < 4; i++)
		temp[i] = array[i];

	int index = step % 4 == 0 ? 0 : step % 4;
	for(int i = 0; i < 4; i++){
		array[i] = temp[index];
		index++;
		index = index % 4;
	}
}

/**
 * 把数组中的第一、二、三和四元素分别作为
 * 4字节整型的第一、二、三和四字节，合并成一个4字节整型
 */
static int mergeArrayToInt(int array[4]) {
	int one = array[0] << 24;
	int two = array[1] << 16;
	int three = array[2] << 8;
	int four = array[3];
	return one | two | three | four;
}

/**
 * 常量轮值表
 */
static const int Rcon[10] = { 
    0x01000000, 0x02000000,
	0x04000000, 0x08000000,
	0x10000000, 0x20000000,
	0x40000000, 0x80000000,
	0x1b000000, 0x36000000 
};
/**
 * 密钥扩展中的T函数
 */
static int T(int num, int round) {
	int numArray[4];
	splitIntToArray(num, numArray);
	leftLoop4int(numArray, 1);//字循环

	//字节代换
	for(int i = 0; i < 4; i++)
		numArray[i] = getNumFromSBox(numArray[i]);

	int result = mergeArrayToInt(numArray);
	return result ^ Rcon[round];
}

//密钥对应的扩展数组
static int w[44];

/**
 * 扩展密钥，结果是把w[44]中的每个元素初始化
 */
static void extendKey(char *key) {
	for(int i = 0; i < 4; i++)
		w[i] = getWordFromStr(key + i * 4);

	for(int i = 4, j = 0; i < 44; i++) {
		if( i % 4 == 0) {
			w[i] = w[i - 4] ^ T(w[i - 1], j);
			j++;//下一轮
		}else {
			w[i] = w[i - 4] ^ w[i - 1];
		}
	}

}

/**
 * 轮密钥加
 */
static void addRoundKey(int array[4][4], int round) {
	int warray[4];
	for(int i = 0; i < 4; i++) {

		splitIntToArray(w[ round * 4 + i], warray);

		for(int j = 0; j < 4; j++) {
			array[j][i] = array[j][i] ^ warray[j];
		}
	}
}

/**
 * 字节代换
 */
static void subBytes(int array[4][4]){
	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++)
			array[i][j] = getNumFromSBox(array[i][j]);
}

/**
 * 行移位
 */
static void shiftRows(int array[4][4]) {
	int rowTwo[4], rowThree[4], rowFour[4];
	//复制状态矩阵的第2,3,4行
	for(int i = 0; i < 4; i++) {
		rowTwo[i] = array[1][i];
		rowThree[i] = array[2][i];
		rowFour[i] = array[3][i];
	}
	//循环左移相应的位数
	leftLoop4int(rowTwo, 1);
	leftLoop4int(rowThree, 2);
	leftLoop4int(rowFour, 3);

	//把左移后的行复制回状态矩阵中
	for(int i = 0; i < 4; i++) {
		array[1][i] = rowTwo[i];
		array[2][i] = rowThree[i];
		array[3][i] = rowFour[i];
	}
}

/**
 * 列混合要用到的矩阵
 */
static const int colM[4][4] = { 2, 3, 1, 1,
	1, 2, 3, 1,
	1, 1, 2, 3,
	3, 1, 1, 2 };

static int GFMul2(int s) {
	int result = s << 1;
	int a7 = result & 0x00000100;

	if(a7 != 0) {
		result = result & 0x000000ff;
		result = result ^ 0x1b;
	}

	return result;
}

static int GFMul3(int s) {
	return GFMul2(s) ^ s;
}

static int GFMul4(int s) {
	return GFMul2(GFMul2(s));
}

static int GFMul8(int s) {
	return GFMul2(GFMul4(s));
}

static int GFMul9(int s) {
	return GFMul8(s) ^ s;
}

static int GFMul11(int s) {
	return GFMul9(s) ^ GFMul2(s);
}

static int GFMul12(int s) {
	return GFMul8(s) ^ GFMul4(s);
}

static int GFMul13(int s) {
	return GFMul12(s) ^ s;
}

static int GFMul14(int s) {
	return GFMul12(s) ^ GFMul2(s);
}

/**
 * GF上的二元运算
 */
static int GFMul(int n, int s) {
	int result;

	if(n == 1)
		result = s;
	else if(n == 2)
		result = GFMul2(s);
	else if(n == 3)
		result = GFMul3(s);
	else if(n == 0x9)
		result = GFMul9(s);
	else if(n == 0xb)//11
		result = GFMul11(s);
	else if(n == 0xd)//13
		result = GFMul13(s);
	else if(n == 0xe)//14
		result = GFMul14(s);

	return result;
}
/**
 * 列混合
 */
static void mixColumns(int array[4][4]) {

	int tempArray[4][4];

	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++)
			tempArray[i][j] = array[i][j];

	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++){
			array[i][j] = GFMul(colM[i][0],tempArray[0][j]) ^ GFMul(colM[i][1],tempArray[1][j]) 
				^ GFMul(colM[i][2],tempArray[2][j]) ^ GFMul(colM[i][3], tempArray[3][j]);
		}
}
/**
 * 把4X4数组转回字符串
 */
static void convertArrayToStr(int array[4][4], char *str) {
	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++)
			*str++ = (char)array[j][i];	
}
/**
 * 检查密钥长度
 */
static int checkKeyLen(int len) {
	if(len == 16)
		return 1;
	else
		return 0;
}

/**
 * 参数 p: 明文的字符串数组。
 * 参数 plen: 明文的长度。
 * 参数 key: 密钥的字符串数组。
 */
void aes(char *p, int plen, char *key){

	int keylen = strlen(key);
	if(plen == 0 || plen % 16 != 0) {
		printf("明文字符长度必须为16的倍数！\n");
		exit(0);
	}

	if(!checkKeyLen(keylen)) {
		printf("密钥字符长度错误！长度必须为16、24和32。当前长度为%d\n",keylen);
		exit(0);
	}

	extendKey(key);//扩展密钥
	int pArray[4][4];

	for(int k = 0; k < plen; k += 16) {	
		convertToIntArray(p + k, pArray);

		addRoundKey(pArray, 0);//一开始的轮密钥加

		for(int i = 1; i < 10; i++){//前9轮

			subBytes(pArray);//字节代换

			shiftRows(pArray);//行移位

			mixColumns(pArray);//列混合

			addRoundKey(pArray, i);

		}

		//第10轮
		subBytes(pArray);//字节代换

		shiftRows(pArray);//行移位

		addRoundKey(pArray, 10);

		convertArrayToStr(pArray, p + k);
	}
}
/**
 * 根据索引从逆S盒中获取值
 */
static int getNumFromS1Box(int index) {
	int row = getLeft4Bit(index);
	int col = getRight4Bit(index);
	return S2[row][col];
}
/**
 * 逆字节变换
 */
static void deSubBytes(int array[4][4]) {
	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++)
			array[i][j] = getNumFromS1Box(array[i][j]);
}
/**
 * 把4个元素的数组循环右移step位
 */
static void rightLoop4int(int array[4], int step) {
	int temp[4];
	for(int i = 0; i < 4; i++)
		temp[i] = array[i];

	int index = step % 4 == 0 ? 0 : step % 4;
	index = 3 - index;
	for(int i = 3; i >= 0; i--) {
		array[i] = temp[index];
		index--;
		index = index == -1 ? 3 : index;
	}
}

/**
 * 逆行移位
 */
static void deShiftRows(int array[4][4]) {
	int rowTwo[4], rowThree[4], rowFour[4];
	for(int i = 0; i < 4; i++) {
		rowTwo[i] = array[1][i];
		rowThree[i] = array[2][i];
		rowFour[i] = array[3][i];
	}

	rightLoop4int(rowTwo, 1);
	rightLoop4int(rowThree, 2);
	rightLoop4int(rowFour, 3);

	for(int i = 0; i < 4; i++) {
		array[1][i] = rowTwo[i];
		array[2][i] = rowThree[i];
		array[3][i] = rowFour[i];
	}
}
/**
 * 逆列混合用到的矩阵
 */
static const int deColM[4][4] = { 0xe, 0xb, 0xd, 0x9,
	0x9, 0xe, 0xb, 0xd,
	0xd, 0x9, 0xe, 0xb,
	0xb, 0xd, 0x9, 0xe };

/**
 * 逆列混合
 */
static void deMixColumns(int array[4][4]) {
	int tempArray[4][4];

	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++)
			tempArray[i][j] = array[i][j];

	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++){
			array[i][j] = GFMul(deColM[i][0],tempArray[0][j]) ^ GFMul(deColM[i][1],tempArray[1][j]) 
				^ GFMul(deColM[i][2],tempArray[2][j]) ^ GFMul(deColM[i][3], tempArray[3][j]);
		}
}
/**
 * 把两个4X4数组进行异或
 */
static void addRoundTowArray(int aArray[4][4],int bArray[4][4]) {
	for(int i = 0; i < 4; i++)
		for(int j = 0; j < 4; j++)
			aArray[i][j] = aArray[i][j] ^ bArray[i][j];
}
/**
 * 从4个32位的密钥字中获得4X4数组，
 * 用于进行逆列混合
 */
static void getArrayFrom4W(int i, int array[4][4]) {
	int index = i * 4;
	int colOne[4], colTwo[4], colThree[4], colFour[4];
	splitIntToArray(w[index], colOne);
	splitIntToArray(w[index + 1], colTwo);
	splitIntToArray(w[index + 2], colThree);
	splitIntToArray(w[index + 3], colFour);

	for(int i = 0; i < 4; i++) {
		array[i][0] = colOne[i];
		array[i][1] = colTwo[i];
		array[i][2] = colThree[i];
		array[i][3] = colFour[i];
	}

}

/**
 * 参数 c: 密文的字符串数组。
 * 参数 clen: 密文的长度。
 * 参数 key: 密钥的字符串数组。
 */
void deAes(char *c, int clen, char *key) {

	int keylen = strlen(key);
	if(clen == 0 || clen % 16 != 0) {
		printf("密文字符长度必须为16的倍数！现在的长度为%d\n",clen);
		exit(0);
	}

	if(!checkKeyLen(keylen)) {
		printf("密钥字符长度错误！长度必须为16、24和32。当前长度为%d\n",keylen);
		exit(0);
	}

	extendKey(key);//扩展密钥
	int cArray[4][4];
	for(int k = 0; k < clen; k += 16) {
		convertToIntArray(c + k, cArray);


		addRoundKey(cArray, 10);

		int wArray[4][4];
		for(int i = 9; i >= 1; i--) {
			deSubBytes(cArray);

			deShiftRows(cArray);

			deMixColumns(cArray);
			getArrayFrom4W(i, wArray);
			deMixColumns(wArray);

			addRoundTowArray(cArray, wArray);
		}

		deSubBytes(cArray);

		deShiftRows(cArray);

		addRoundKey(cArray, 0);

		convertArrayToStr(cArray, c + k);

	}
}
```

`main.c`

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>

#include "aes.h"

#define MAXLEN 1024

void getString(char *str, int len){

	int slen = read(0, str, len);
	for(int i = 0; i < slen; i++,str++){
		if(*str == '\n'){
			*str = '\0';
			break;
		}
	}
}

void printASCCI(char *str, int len) {
	int c;
	for(int i = 0; i < len; i++) {
		c = (int)*str++;
		c = c & 0x000000ff;
		printf("0x%x ", c);
	}
	printf("\n");
}

/**
 * 从标准输入中读取用户输入的字符串
 */
void readPlainText(char *str, int *len) {
	int plen;
	while(1) {
		getString(str, MAXLEN);
		plen = strlen(str);
		if(plen != 0 && plen % 16 == 0) {
			printf("你输入的明文为：%s\n", str);
			break;
		}else{
			printf("明文字符长度必须为16的倍数,现在的长度为%d\n", plen);
		}
	}
	*len = plen;
}
/**
 * 把字符串写进文件
 */
void writeStrToFile(char *str, int len, char *fileName) {
	FILE *fp;
	fp = fopen(fileName, "wb");
	for(int i = 0; i < len; i++)
		putc(str[i], fp);
	fclose(fp);
}


void aesStrToFile(char *key) {

	char p[MAXLEN];
	int plen;
	printf("请输入你的明文，明文字符长度必须为16的倍数\n");
	readPlainText(p,&plen);
	printf("进行AES加密..................\n");

	aes(p, plen, key);//AES加密

	printf("加密完后的明文的ASCCI为：\n");
	printASCCI(p, plen);
	char fileName[64];
	printf("请输入你想要写进的文件名，比如'test.txt':\n");
	if(scanf("%s", fileName) == 1) {	
		writeStrToFile(p, plen, fileName);
		printf("已经将密文写进%s中了,可以在运行该程序的当前目录中找到它。\n", fileName);
	}
}
/**
 * 从文件中读取字符串
 */
int readStrFromFile(char *fileName, char *str) {
	FILE *fp = fopen(fileName, "rb");
	if(fp == NULL) {
		printf("打开文件出错，请确认文件存在当前目录下！\n");
		exit(0);
	}

	int i;
	for(i = 0; i < MAXLEN && (str[i] = getc(fp)) != EOF; i++);

	if(i >= MAXLEN) {
		printf("解密文件过大！\n");
		exit(0);
	}

	str[i] = '\0';
	fclose(fp);
	return i;
}


void deAesFile(char *key) {
	char fileName[64];
	char c[MAXLEN];//密文字符串
	printf("请输入要解密的文件名，该文件必须和本程序在同一个目录\n");
	if(scanf("%s", fileName) == 1) {
		int clen = readStrFromFile(fileName, c);
		printf("开始解密.........\n");
		deAes(c, clen, key);
		printf("解密后的明文ASCII为：\n");
		printASCCI(c, clen);
		printf("明文为：%s\n", c);
		writeStrToFile(c,clen,fileName);
		printf("现在可以打开%s来查看解密后的密文了！\n",fileName);
	}
}

void aesFile(char *key) {
	char fileName[64];
	char fileP[MAXLEN];

	printf("请输入要加密的文件名，该文件必须和本程序在同一个目录\n");
	if(scanf("%s", fileName) == 1) {
		readStrFromFile(fileName, fileP);
		int plen = strlen(fileP);
		printf("开始加密.........\n");
		printf("加密前文件中字符的ASCII为：\n");
		printASCCI(fileP, plen);

		aes(fileP, plen, key);//开始加密

		printf("加密后的密文ASCII为：\n");
		printASCCI(fileP, plen);
		writeStrToFile(fileP,plen,fileName);
		printf("已经将加密后的密文写进%s中了\n",fileName);
	}
}

int main(int argc, char const *argv[]) {

	char key[17];
	printf("请输入16个字符的密钥：\n");
	int klen;
	while(1){
		getString(key,17);
		klen = strlen(key);
		if(klen != 16){
			printf("请输入16个字符的密钥,当前密钥的长度为%d\n",klen);
		}else{
			printf("你输入的密钥为：%s\n",key);
			break;
		}
	}

	printf("输入's'表示要加密输入的字符串,并将加密后的内容写入到文件\n");
	printf("请输入要功能选项并按回车，输入'f'表示要加密文件\n");
	printf("输入'p'表示要解密文件\n");
	char c;
	if(scanf("%s",&c) == 1) {
		if(c == 's')
			aesStrToFile(key);//用AES加密字符串，并将字符串写进文件中
		else if(c == 'p')
			deAesFile(key);//把文件中的密文解密，并写回文件中
		else if(c == 'f')//用AES加密文件
			aesFile(key);
	}
	return 0;
}
```

通过下面的gcc命令来编译运行：

```shell
gcc -o aes aes.c main.c
```

# 5总结

[高级加密标准](https://so.csdn.net/so/search?q=高级加密标准&spm=1001.2101.3001.7020)(AES,Advanced Encryption Standard)为最常见的对称加密算法(微信小程序加密传输就是用这个加密算法的)。对称加密算法也就是加密和解密用相同的密钥。总的来说介绍了最基本的AES的ecb的加密模式，另外还有具体的demo可以作为学习参考，不过除此之外AES还会跟`base64`相结合。



# 参考

[[1] TimeShatter, AES加密算法原理的详细介绍与实现, 2023.](https://blog.csdn.net/qq_28205153/article/details/55798628)

[[2] 刘扬俊, 什么是Base64算法？——全网最详细讲解, 2023.](https://blog.csdn.net/qq_19782019/article/details/88117150)