---
title: "linux2 权限和管理"
date: 2022-04-05
thumbnailImagePosition: left
thumbnailImage: linux/linux4_thumb.jpg
coverImage: linux/linux4_cover.png
metaAlignment: center
coverMeta: out
draft: true
categories:
- 2022
- April 
tags:
- linux
- 手册
- 系统
- 权限和管理
showSocial: false
---

Linux中万物皆可文件，但是文件的管理，用户的管理，文件的权限，用户的权限如果没有搞清楚的话，那么就不是一名合格的程序员了。

<!--more-->
# 群组管理和文件权限

## 用户管理

默认状态下只有root，普通用户都在/home目录下，可以创建普通用户，然后就可以登录普通用户账号。

普通用户切回root用户方式

### sudo

暂时成为 root

 ### sudo su

一直成为 root

### sudo su -/sudo -i

切换为 root，并且直接定位到 root 的目录

{{< image classes="fancybox center fig-100" src="/linux/linux1_44.png" thumbnail="/linux/linux1_44.png" title="">}}

### adduser

添加新用户，需要root权限

**adduser -d -m产生一个主目录/home/newuser，并选择为登录用户**

**adduser -s /bin/bash 指定用户所使用的shell**

```shell
sudo useradd -d /home/newuser -m -s /bin/bash newuser
```

> useradd --help
>
> | 符号标识 | 说明                                                     | 解释                                 |
> | -------- | -------------------------------------------------------- | ------------------------------------ |
> | -b       | base directory for the home directory of the new account | 设置基本路径作为用户的登录目录       |
> | -c       | GECOS field of the new account                           | 对用户的注释                         |
> | -d       | home directory of the new account                        | 设置用户的登录目录                   |
> | -D       | print or change default useradd configuration            | 改变设置                             |
> | -e       | expiration date of the new account                       | 设置用户的有效期                     |
> | -f       | password inactivity period of the new account            | 用户过期后，让密码无效               |
> | -g       | name or ID of the primary group of the new account       | 使用户只属于某个组                   |
> | -G       | list of supplementary groups of the new account          | 使用户加入某个组                     |
> | -h       | display this help message and exit                       | 帮助                                 |
> | -k       | use this alternative skeleton directory                  | 指定其他的skel目录                   |
> | -K       | override /etc/login.defs defaults                        | 覆盖 /etc/login.defs 配置文件        |
> | -m       | create the user's home directory                         | 自动创建登录目录                     |
> | -M       | do not create the user's home directory                  | 不自动创建登录目录                   |
> | -l       | do not add the user to the lastlog and faillog databases | 不把用户加入到lastlog文件中          |
> | -r       | create a system account                                  | 建立系统账号                         |
> | -o       | allow to create users with duplicate (non-unique) UID    | 允许用户拥有相同的UID                |
> | -p       | encrypted password of the new account                    | 为新用户使用加密密码                 |
> | -s       | login shell of the new account                           | 指定登录时候的shell                  |
> | -u       | user ID of the new account                               | 为新用户指定一个UID                  |
> | -Z       | use a specific SEUSER for the SELinux user mapping       | 为 SELinux 用户映射使用特定的 SEUSER |

{{< image classes="fancybox center fig-100" src="/linux/linux1_45.png" thumbnail="/linux/linux1_45.png" title="">}}

### passwd

```shell
sudo passwd newuser
```

修改密码，会输入两次密码，需要管理员权限

{{< image classes="fancybox center fig-100" src="/linux/linux1_46.png" thumbnail="/linux/linux1_46.png" title="">}}

### 赋予root权限

修改 /etc/sudoers 文件，找到下面一行，在root下面添加一行，可以使得yangyang账号可以sudo的时候获取root权限

```shell
## Allow root to run any commands anywhere
root       ALL=(ALL)     ALL
yangyang   ALL=(ALL)     ALL
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_47.png" thumbnail="/linux/linux1_47.png" title="">}}

查看/etc/passwd的登录权限，属于1001的用户和同名的群组

具体解释：

```shell
#登录的用户名:密码:用户id:群组id::主目录:指定用户所使用的shell
newuser:x:1001:1001::/home/newuser:bin/bash
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_48.png" thumbnail="/linux/linux1_48.png" title="">}}

### su username

```shell
su newuser
```

切换用户

{{< image classes="fancybox center fig-100" src="/linux/linux1_49.png" thumbnail="/linux/linux1_49.png" title="">}}

### userdel

```shell
userdel newuser
```

删除用户，需要root权限

**sudo deluser --remove-home newuser**

删除用户并且删除用户所在的目录

到这一步，仅仅删除了目录，newuser用户依然存在

{{< image classes="fancybox center fig-100" src="/linux/linux1_64.png" thumbnail="/linux/linux1_64.png" title="">}}

这里继续查看14090进程，可以发现是shell没结束，需要手动删除/etc/passwd和/etc/sudoers里面关于newuser信息。**最后还需要删除newuser群组**，才算真正删除用户。

{{< image classes="fancybox center fig-100" src="/linux/linux1_65.png" thumbnail="/linux/linux1_65.png" title="">}}

## 群组管理

#### addgroup

创建群组，需要root权限

{{< image classes="fancybox center fig-100" src="/linux/linux1_50.png" thumbnail="/linux/linux1_50.png" title="">}}

### usermod 

修改用户账户，需要root权限

- -l：对用户重命名，但是 /home 目录中的用户家目录名不会改变，需要手动修改；
- -g：修改用户所在群组。此用户的家目录里的所有文件的所在群组会相应改变。
- -G：修改用户添加到多个群组中。

> -g 或 -G 参数时，它会把用户从原先的群组里剔除，加入到新的群组。如果需要不想离开原先的群组，又想加入新的群组，可以增加-a参数
>
> ```shell
> usermod -aG good thomas
> ```

```shell
sudo usermod -g happy newuser
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_54.png" thumbnail="/linux/linux1_54.png" title="">}}

```shell
sudo usermod -aG linux,android newuser
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_55.png" thumbnail="/linux/linux1_55.png" title="">}}

### gpasswd 

群组中添加/删除用户，需要root权限

gpasswd -a user groups 

将用户user加入到groups组

{{< image classes="fancybox center fig-100" src="/linux/linux1_56.png" thumbnail="/linux/linux1_56.png" title="">}}

gpasswd -d user groups

将用户test从test2组中移出

{{< image classes="fancybox center fig-100" src="/linux/linux1_57.png" thumbnail="/linux/linux1_57.png" title="">}}

### groups

查询一个用户在哪些群组中，这里用户可以被添加到多个群组中

{{< image classes="fancybox center fig-100" src="/linux/linux1_53.png" thumbnail="/linux/linux1_53.png" title="">}}

### delgroup

删除群组，需要root权限，如果删除的是用户第一个添加的群组是无法删除，必须先移除用户

```shell
delgroup happy
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_58.png" thumbnail="/linux/linux1_58.png" title="">}}

### chown

改变文件的所有者，需要root权限

**chown newuser file.txt**

将文件的所有者移交给newuser

{{< image classes="fancybox center fig-100" src="/linux/linux1_59.png" thumbnail="/linux/linux1_59.png" title="">}}

**chown newuser:newgroup file.txt**

同时修改文件所有者和群组。将文件的所有者移交给newuser，将文件所有者归到newgroup中

{{< image classes="fancybox center fig-100" src="/linux/linux1_60.png" thumbnail="/linux/linux1_60.png" title="">}}



### chgrp

改变文件的群组，需要root权限

chgrp newgroup file.txt

{{< image classes="fancybox center fig-100" src="/linux/linux1_61.png" thumbnail="/linux/linux1_61.png" title="">}}

## 访问权限

### ls -l

```shell
# 文件/文件夹访问权限
newuser@iZbp1788g5aoi5p9z8e0ugZ:~$ ls -al file.txt
-rw-r--r-- 1 newuser newuser 0 Mar 13 20:27 file.txt
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_62.png" thumbnail="/linux/linux1_62.png" title="">}}

具体含义说明：

- d：英语 directory 的缩写，表示“目录”。就是说这是一个目录
- l：英语 link 的缩写，表示“链接”。就是说这是一个链接
- r：英语 read 的缩写，表示“读”。就是说可以读这个文件
- w：英语 write 的缩写，表示“写”。就是说可以写这个文件，也就是可以修改
- x：英语 execute 的缩写，表示“执行，运行”。就是说可以运行这个文件

- 第一组 rwx 表示文件的**所有者**对于此文件的访问权限
- 第二组 rwx 表示文件所属**群组**的其他用户对于此文件的访问权限
- 第三组 rwx 表示除前两组之外的**其他用户**对于此文件的访问权限

### chmod

权限说明表

| 权限 | 数字 |
| ---- | ---- |
| r    | 4    |
| w    | 2    |
| x    | 1    |

权限所有计算表

| 权限 | 数字 | 计算  |
| ---- | ---- | ----- |
| rwx  | 7    | 4+2+1 |
| rw-  | 6    | 4+2+0 |
| r-x  | 5    | 4+0+1 |
| r–   | 4    | 4+0+0 |
| -wx  | 3    | 0+2+1 |
| -w-  | 2    | 0+2+0 |
| –x   | 1    | 0+0+1 |
| —    | 0    | 0+0+0 |

### 字母分配权限

字母符号含义

> - u：user 的缩写，是英语“用户”的意思。表示所有者
> - g：group 的缩写，是英语“群组”的意思。表示群组用户
> - o：other 的缩写，是英语“其他”的意思。表示其他用户
> - a：all 的缩写，是英语“所有”的意思。表示所有用户
>
> - +：加号，表示添加权限
> - -：减号，表示去除权限
> - =：等号，表示分配权限

```shell
#文件 file.txt 的所有者增加读和运行的权限。
chmod u+rx file.txt

#文件 file.txt 的群组其他用户增加读的权限。
chmod g+r file.txt 

#文件 file.txt 的其他用户移除读的权限。
chmod o-r file.txt 

#文件 file.txt 的群组其他用户增加读的权限，其他用户移除读的权限。
chmod g+r,o-r file.txt 

#文件 file.txt 的群组其他用户和其他用户均移除读的权限。
chmod go-r file.txt 

#文件 file.txt 的所有用户增加运行的权限。
chmod +x file.txt 

#文件 file.txt 的所有者分配读，写和执行的权限；
#群组其他用户分配读的权限，不能写或执行；
#其他用户没有任何权限。
chmod u=rwx,g=r,o=- file.txt
```

{{< image classes="fancybox center fig-100" src="/linux/linux1_63.png" thumbnail="/linux/linux1_63.png" title="">}}




# 猜你喜欢

[linux1 基础学习](https://yangyang48.github.io/2022/03/linux1-%E5%9F%BA%E7%A1%80%E5%AD%A6%E4%B9%A0/)

[linux3 Vim和shell脚本](https://yangyang48.github.io/2022/04/linux3-vim%E5%92%8Cshell%E8%84%9A%E6%9C%AC/)

[linux4 NDK交叉编译](https://yangyang48.github.io/2022/03/linux4-ndk%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91/)



# 参考

[[1] 伏特加的滋味, linux下创建用户和添加用户权限,2017.](https://blog.csdn.net/yuanyuan214365/article/details/75153928)

[[2]panqidong95, Linux添加用户及用户权限管理,2019.](https://blog.csdn.net/panqidong95/article/details/95517278)

[[3]panqidong95, linux adduser,2016.](https://blog.csdn.net/gaoqiao1988)