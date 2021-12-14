---
title: "Android数据存储"
date: 2021-10-17
thumbnailImagePosition: left
thumbnailImage: datastorage/datastorage_thumb.jpg
coverImage: datastorage/datastorage_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- ShraePreference
- SQLite
- 2021
- August
tags:
- Android
- datastorage
showSocial: false
---


如果我需要在音视频app中的登录界面记住账号密码，并且在音视频的传输过程中，将编码后的mp4文件和aac文件保存，这个就需要用到Android数据存储。<!--more-->
# Android数据存储

目前主流的关于android的数据存储有五种：ShraePreference、文件SQLite数据库、文件存储数据、ContentProvider、网络存储数据。

| 类型名称          | 位置                                                         | 优点                                                         | 缺点                                                         |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| SharedPreferences | /data/data/appid/shared_prefs                                | xml文件形式存储在本地，程序卸载后会也会一并被清除            | 不适合大量的数据存储，多线程和跨进程无法保证                 |
| SQLite数据库      | /data/data/appid/databases                                   | 可以轻量级的操作增删改查                                     | 读写操作容易阻塞或者出错                                     |
| 文件存储          | 外部存储/sdcard <br />内部存储/data/data/appid/files 和 /data/data/appid/cache | 前者：大量储存图片，音频，视频，文档<br />后者：app卸载后一并删除 | 前者：app卸载需要手动删除<br />后者：内存有限不能储存太多数据 |
| ContentProvider   | 没有具体位置                                                 | 用于跨进程之间数据交互，数据交互提供了一个安全的增删改查环境 | 一般出于安全原因，不会把数据提供给第三方App使用，用得不多    |
| 网络存储          | 没有具体位置                                                 | 不用担心内存问题                                             | 完全依赖于网络                                               |

本文主要针对前两种进行说明，文件存储这个类似于c中写文件不再展开，ContentProvider作为四大组件之一，后续会单独详解，网络存储也会在接下来的缓冲篇中梳理。

# ShraePreference

SharedPreferences，它是一个轻量级的xml文件，用于保存k-v信息，通常包括一些username，passwd等信息的本地存储。这里边可以通过连接adb，查看到shared_prefs目录，当然可以在不需要root的情况下进行映射

> 1.工程目录下AndroidManifest.xml中添加android:debuggable="true"
>
> 2.用命令run-as com.example.mydatastorage 通过映射方式来打开/data/data/包名/ 目录下文件
>
> 注：这里虽然列出的是包名，但实际上只需要appid即可，类似debug和release版本虽然都是在同一个包名下，但是我的appid可以是不同的。上述操作只是为了查看不代表原来没有进行映射就没有这些文件，类似磁场电场。

{{< image classes="fancybox center fig-100" src="/datastorage/datastorage_1.png" thumbnail="/datastorage/datastorage_1.png" title=" com.example.mydatastorage目录图">}}

## 常用操作

在Android中的操作比较简单

> 1.初始化sp，生成一个my_shared_preferences.xml文件
>
> ```java
> //这里边的第一个参数，指的是只能被当前应用访问，文件不存在创建，存在覆盖
> //MODE_APPEND 只能被当前应用访问，文件不存在创建，存在追加
> //像MODE_WORLD_READABLE和MODE_WORLD_WRITEABLE都被废弃，即跨应用已经不推荐使用
> SharedPreferences sp = context.getSharedPreferences("my_shared_preferences", Context.MODE_PRIVATE)
> ```
>
> {{< image classes="fancybox center fig-100" src="/datastorage/datastorage_7.png" thumbnail="/datastorage/datastorage_7.png" title=" Context参数图">}}
>
> 2.插入k-v键值
>
> ```java
> SharedPreferences.Editor edit = sp.edit();
> //通过editor对象写入数据
> edit.putString("username","passwd");
> //提交数据存入到xml文件中
> edit.commit();
> ```
>
> 3.读取k-v键值
>
> ```java
> //这里只考虑k-v都为String类
> String value = sp.getString("name","");
> ```



因为考虑到这里用到了账号密码，密码不允许明文保存，一般可以通到一些加密手段，这里使用的是多次md5值得加密手法，解密用的是输入的数做同样的加密去对比原先保存加密的字符串是否相同，相同则认为是相同的密码。

{{< image classes="fancybox center fig-100" src="/datastorage/datastorage_5.png" thumbnail="/datastorage/datastorage_5.png" title="MD5在线加密图">}}

{{< image classes="fancybox center fig-100" src="/datastorage/datastorage_6.png" thumbnail="/datastorage/datastorage_6.png" title="xml中的k-v对应值">}}



## 常见的封装

这里边主要是用到两个封装，一个是来自讯飞的封装ConfigUtils.kt，一个是来自鸿洋的封装SPUtils.kt，前者还加入了xml键值对的监听。具体可以看一下源码是如何封装的，这里不展开描述，原理大同小异。



```kotlin
//ConfigUtils.kt
//1.初始化调用 初始化配置文件
ConfigUtils.init(this)
//2.插入k-v键值
ConfigUtils.putString(strname, strpasswd)
//3.读取k-v键值
ConfigUtils.getString(strname, "")
//4.关闭xml读写
ConfigUtils.destroy()
```

```kotlin
//SPUtils.kt
//1.插入k-v键值
SPUtils.put(this, strname, strpasswd)
//2.读取k-v键值
SPUtils.get(this, strname, "")
```



# SQLite

这个就相当于Android端的数据库,目前内置的SQLite是SQLite3

## 常用操作

> 1.创建一个类，用来继承SQLiteOpenHelper
>
> ```kotlin
> //SQLite支持五种数据类型:NULL,INTEGER,REAL(浮点数),TEXT(字符串文本)和BLOB(二进制对象)
> class SQLiteUtils(
>     context: Context?,
>     name: String?,
>     factory: SQLiteDatabase.CursorFactory?,
>     version: Int
> ) : SQLiteOpenHelper(context, name, factory, version) {
>     //用于创建一个数据库，创建之后有personid和name两个字段，前者类型为INTEGER，后者类型是VARCHAR(20)
>     override fun onCreate(db: SQLiteDatabase?) {
>         db?.execSQL("CREATE TABLE person(personid INTEGER PRIMARY KEY AUTOINCREMENT,name VARCHAR(20))")
>     }
> 	//一般软件版本号改变的时候会调用
>     override fun onUpgrade(db: SQLiteDatabase?, oldVersion: Int, newVersion: Int) {
>         db?.execSQL("ALTER TABLE person ADD phone VARCHAR(12) " + NULL)
>     }
> }
> ```
>
> 2.对数据库操作，增删改查（有两种形式Android自带，和sql语句都可以使用）
>
> ```kotlin
> //数据库操作-数据插入
> val values1 = ContentValues()
> values1.put("name", "yangyang48,NO $i SQLIte data")
> //参数依次是：表名，强行插入null值得数据列的列名，一行记录的数据
> db?.insert("person", null, values1)
> //数据库操作-数据查询，通过轮询的方式将他们字符串拼接打印
>  sb = StringBuilder()
> //参数依次是:表名，列名，where约束条件，where中占位符提供具体的值，指定group by的列，进一步约束
> //指定查询结果的排序方式,通过遍历的方式将所有的data罗列出来
> var cursor: Cursor? = db?.query("person", null, null, null, null, null, null)
> if (cursor?.moveToFirst() == true) {
>     do {
>         var pid: Int? = cursor?.getInt(cursor.getColumnIndex("personid"))
>         var name: String? = cursor?.getString(cursor.getColumnIndex("name"))
>         sb?.append("\nid：$pid：$name")
>     } while (cursor?.moveToNext())
> }
> cursor?.close()
> //数据库操作-数据修改
> val values2 = ContentValues()
> values2.put("name", "yangyang48 has modified $j")
> //参数依次是表名，修改后的值，where条件，以及约束，如果不指定三四两个参数，会更改所有行
> //约束主要是对两个字段的约束，personid和name
> db?.update("person", values2, "personid = ?", arrayOf("${i - 1}"))
> //数据库操作-数据删除
> //参数依次是表名，以及where条件与约束,约束主要是对两个字段的约束，personid和name
> db?.delete("person", "personid = ?", arrayOf("$i"))
> ```
>
> 

## 常见的封装

很多时候，SQLite也是直接用现成的框架，比如[GreenDao](https://blog.csdn.net/qq_28643195/article/details/107780106)



直接用工具很难再非root的机子上操作数据库，这里导出数据来看，导出文件my.db，通过SQLite Expert可以查看到具体的内容

非root用户可以如下操作

> 1.cp将/data/data/com.example.mydatastorage/databases数据库文件复制到/sdcard下面
>
> 2.adb pull从/sdcard导出db文件
>
> 3.SQLite Expert可以查看到具体的内容


{{< image classes="fancybox center fig-100" src="/datastorage/datastorage_3.png" thumbnail="/datastorage/datastorage_3.png" title="SQLite Expert 查看对应data数据图">}}

# demo

{{< image classes="fancybox center fig-100" src="/datastorage/datastorage_2.jpg" thumbnail="/datastorage/datastorage_2.jpg" title="demo示例图">}}

[本文所有实例代码下载](https://github.com/YangYang48/project/tree/master/MyDataStorage)

# 参考

[[1] 明昕ztoy, Android学习之五种数据存储方式（一）, 2019.](https://blog.csdn.net/weixin_43468667/article/details/90243311?spm=1001.2014.3001.5501)

[[2] FamilyYan, Android五大数据存储, 2019.](https://blog.csdn.net/qq_37982823/article/details/86482154)

[[3] 菜鸟教程, 6.2 数据存储与访问之——SharedPreferences保存用户偏好参数, 2015.](https://www.runoob.com/w3cnote/android-tutorial-sharedpreferences.html)

[[4] 明昕ztoy, Android学习之五种数据存储方式（二）, 2019.](https://blog.csdn.net/weixin_43468667/article/details/91889829?spm=1001.2014.3001.5501)

[[5] 森森先生666, Android 五大数据存储 (最实用的开发详解) 一 五种存储方式区别, 2020.](https://blog.csdn.net/qq_28643195/article/details/107556187)

