---
title: "ioctl通信方式之初出茅庐"
date: 2022-07-24
thumbnailImagePosition: left
thumbnailImage: ioctl/ioctl_thumb.jpg
coverImage: ioctl/ioctl_cover.jpg
metaAlignment: center
coverMeta: out
draft: true
categories:
- ioctl
- 2022
- July 
tags:
- kernel
- 软中断
- 系统调用表
- 用户与驱动之间的协议
- 驱动
- Android
- 源码
showSocial: false
---

我们通常用ioctl函数直接访问到内核，在内核执行操作，执行读写操作。那么这个函数指令的原理是什么，本文就展开介绍和使用一下ioctl。

<!--more-->
# 1ioctl简介

## 1.1什么是ioctl

> 从ioctl这个名称上看，它是设备驱动程序中对设备的I/O通道进行管理的函数。
>
> 所谓对I/O通道进行管理，就是对设备的一些特性进行控制。例如串口的传输波特率、马达的转速等等, 但实际上**ioctl所处理的对象并不限制是真正的I/O设备**，还可以是其它任何一个内核设备。

说白了，ioctl以系统调用的形式提供了一**条用户与内核交互的便捷途径**。ioctl 这种方式的特点是用户层为主动，当用户层通过ioctl下发指令，内核给出相应的操作，具体的细节由自己实现的代码决定。

## 1.2 基本原理

kernel3.0之前，叫ioctl，之后改名为unlocked_ioctl。功能和接口基本相同，名字发生了变化

ioctl既可以往内核读也可以写，read/write在执行大数据量读/写时比较有优势。

在应用层调用ioctl函数时，内核会调用对应驱动中的unlocked_ioctl函数，向内核读写数据。

笔者这里的kernel直接参考[最新](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/?h=v5.19-rc7)的kernel-v5.19源码来分析。

内核实现ioctl()函数的是sys_ioctl()。内核中主要调用框架图如下图1所示

{{< image classes="fancybox center fig-100" src="/ioctl/ioctl_4.png" thumbnail="/ioctl/ioctl_4.png" title="图1 内核中主要调用框架图">}}

> 至于为什么在驱动层中还会看到`compat_ioctl`
>
> 这是因为为了**相容性**而出现的`compat_ioctl`。为了让32-bit的进程可以在64-bit上的system来执行ioctl这个函数。

# 2通信流程

## 2.1调用用户态的ioctl函数

在用户态只有函数的声明，最终的实现是在内核态，也就是内核态封装了一个函数可以用于用户态的调用。

驱动程序中的ioctl属于内核空间，然而应用程序属于用户空间，没有权限访问内核空间，那么需要系统需要从用户态转为内核态，在系统调用中，是通过**`SWI`(Software Interrupt)的方式**陷入内核态的，这里实际上是一个靠**软中断**实现的。

```c
//development/ndk/platforms/android-30/include/sys/ioctl.h
int ioctl(int fd, int cmd, ...) ;
```



## 2.2在系统调用表中找到ioctl函数对应的函数名

### 2.2.1ioctl的驱动定义

首先找到定义的位置

```c
//kernel/include/uapi/asm-generic/unistd.h
#ifndef __SYSCALL
#define __SYSCALL(x, y)
#endif

#if __BITS_PER_LONG == 32 || defined(__SYSCALL_COMPAT)
#define __SC_3264(_nr, _32, _64) __SYSCALL(_nr, _32)
#else
#define __SC_3264(_nr, _32, _64) __SYSCALL(_nr, _64)
#endif

#ifdef __SYSCALL_COMPAT
#define __SC_COMP(_nr, _sys, _comp) __SYSCALL(_nr, _comp)
#define __SC_COMP_3264(_nr, _32, _64, _comp) __SYSCALL(_nr, _comp)
#else
#define __SC_COMP(_nr, _sys, _comp) __SYSCALL(_nr, _sys)
#define __SC_COMP_3264(_nr, _32, _64, _comp) __SC_3264(_nr, _32, _64)
#endif

/* fs/ioctl.c */
#define __NR_ioctl 29
__SC_COMP(__NR_ioctl, sys_ioctl, compat_sys_ioctl)
```

因为这里边的宏定义比较多，直接替换掉宏定义

```c
//kernel/include/uapi/asm-generic/unistd.h
__SYSCALL(29, sys_ioctl)
```

### 2.2.2系统调用表

系统调用表数组如下。如何解读系统调用表的用法，可以参考[这里](https://www.cnblogs.com/hazir/p/array_initialization.html)。

```c
//kernel/arch/arc/kernel/sys.c
#define __SYSCALL(nr, call) [nr] = (call),

void* sys_call_table[NR_syscalls] = {
    [0 ... NR_syscalls - 1] = sys_ni_syscall,
    #include<asm/unistd.h>
};
```

这里用的是c99的语法规则，其中数组初始化可以用[0 ... n]，这种写法。举例如下

```c
int my_array[6] = { [4] = 29, [2] = 15 };
//或者写成：
//省略到索引与值之间的=，GCC 2.5 之后该用法已经过时了，但 GCC 仍然支持
int my_array[6] = { [4] 29, [2] 15 };     
//两者均等价于：
int my_array[6] = {0, 0, 15, 0, 29, 0};
```

直到对应的语法之后，其中`#include<asm/unistd.h>`就是罗列了所有的系统调用函数名。

```c
//所以得到的ioctl函数就是下面这个宏定义
__SYSCALL(0, sys_io_setup)
__SYSCALL(1, sys_io_destroy)
__SYSCALL(2, sys_io_submit)
...
__SYSCALL(29, sys_ioctl)
...
//展开上述的宏定义
[0] = sys_io_setup,
[1] = sys_io_destroy,
[2] = sys_io_submit,
...
[29] = sys_ioctl,
...
```

1. 其中，`NR_syscalls`这个数是在`#include<asm/unistd.h>`中实际的系统调用函数名数量，比如435。那么`NR_syscalls`就会被435替换。

2. 这个`sys_call_table`数组中的[0 ... 434]是用于初始化的操作，其中这个`sys_ni_syscall`函数名返回的是`-ENOSYS`，说明如果没有找到`#include<asm/unistd.h>`中对应的函数，直接返回系统调用表初始化错误。

3. 后面的操作，是相当于对其重新赋值，让每一个系统调用表数组都能够对应一个函数名，那么在找这个函数名的过程，只需要记住这个函数名在表中对应的序号即可，这里的ioctl函数，对应的就是29序号。即在系统调用表中找到ioctl函数对应的函数名为sys_ioctl。

那么现在替换之后再来看系统调用表

```c
//kernel/arch/arc/kernel/sys.c
#define __SYSCALL(nr, call) [nr] = (call),

void* sys_call_table[435] = {
    [0 ... 434] = sys_ni_syscall,
    [0] = sys_restart_syscall,
    [1] = sys_exit,
    [2] = sys_fork,
    ...
    [29] = sys_ioctl,
    ...
};
```

> 关于`system_call`系统调用
>
> 因为上述得到的是系统调用表，而系统调用需要在这个系统调用表中查询。
>
> 通过软中断引发一个异常促使系统切换到内核去执行异常处理程序。该异常处理程序实际就是系统调用处理程序。这条指令会触发一个异常导致系统切换到内核态并执行第128号异常处理程序----系统调用处理程序。系统调用处理程序叫`system_call()`，它与硬件体系结构紧密相关，用汇编语言编写。
>
> 应用程序告诉内核自己需要执行一个系统调用时，通知内核的机制是靠`system_call`函数通过将给定的系统调用号（`eax`的值）与`NR_syscalls`做比较来检查其有效性（`eax`代表寄存器）。如果它大于或等于`NR_syscalls`，该函数返回`-ENOSYS`。否则执行相应的系统调用：`call *sys_call_table(,%eax,4)`。由于系统调用表中的表项是以32位（4字节）类型存放的，所以内核需要将给定的系统调用号乘以4。
>
>  除了系统调用号以外，大部分系统调用都还需要一些外部的参数输入，所以需要把**这些参数从用户空间传给内核**。最简单的方法就是像传递系统调用参数那样也存放在寄存器里。



对应的系统调用流程如下

{{< image classes="fancybox center fig-100" src="/ioctl/ioctl_1.png" thumbnail="/ioctl/ioctl_1.png" title="">}}

## 2.3找到内核中对应的函数指针

找到了函数名`sys_ioctl`，就要去找对应的定义和实现

### 2.3.1ioctl系统调用函数名定义

```c
//kernel/include/linux/syscalls.h
asmlinkage long sys_ioctl(unsigned int fd, unsigned int cmd, unsigned long long);
```

### 2.3.2ioctl系统调用函数名实现

找到了这个定义之后，去找到实现。不过多了一个file结构体指针参数，可以将用户传递的`fd`转换为系统调用到内核内部`file`结构指针。当当前文件描述符表仅用于单个任务时，`fget_light()` / `fput_light()`不会触及引用计数。有关更多信息，请访问[read this](http://www.mjmwired.net/kernel/Documentation/filesystems/files.txt)。

```c
//kernel/fs/ioctl.c
//实际上SYSCALL_DEFINE3就是sys_ioctl
SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
{
	struct file *filp;
	int error = -EBADF;
	int fput_needed;

	filp = fget_light(fd, &fput_needed);//由fd得带filp指针
	if (!filp)
		goto out;

	error = security_file_ioctl(filp, cmd, arg);
	if (error)
		goto out_fput;

	error = do_vfs_ioctl(filp, fd, cmd, arg);
 out_fput:
	fput_light(filp, fput_needed);
 out:
	return error;
}
```

> 补充说明:**为什么`SYSCALL_DEFINE3`就是`sys_ioctl`**。
>
> 实际上这里的3代表有三个参数的意思，具体拆解可以点击[这里](https://blog.csdn.net/weixin_43798887/article/details/118930476)，找到宏定义是如何一步步展开的，这里简单阐述。
>
> ```c
> //include/linux/syscalls.h
> #define SYSCALL_DEFINE3(name, ...)  SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
> #define SYSCALL_DEFINEx(x, sname, ...)	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
> #define __SYSCALL_DEFINEx(x, name, ...)	asmlinkage long sys##name(__SC_DECL##x(__VA_ARGS__))
> ```
>
> `__SYSCALL_DEFINEx`宏定义
>
> ```c
> //include/linux/syscalls.h
> #define __SYSCALL_DEFINEx(x, name, ...)					\
> 	asmlinkage long sys##name(__SC_DECL##x(__VA_ARGS__));		\
> 	static inline long SYSC##name(__SC_DECL##x(__VA_ARGS__));	\
> 	asmlinkage long SyS##name(__SC_LONG##x(__VA_ARGS__))		\
> 	{								\
> 		__SC_TEST##x(__VA_ARGS__);				\
> 		return (long) SYSC##name(__SC_CAST##x(__VA_ARGS__));	\
> 	}								\
> 	SYSCALL_ALIAS(sys##name, SyS##name);				\
> 	static inline long SYSC##name(__SC_DECL##x(__VA_ARGS__))
> ```
>
> `SYSCALL_ALIAS`定义
>
> ```c
> //include/linux/syscalls.h
> #define SYSCALL_ALIAS(alias, name)					\
> 	asm ("\t.globl " #alias "\n\t.set " #alias ", " #name "\n"	\
> 	     "\t.globl ." #alias "\n\t.set ." #alias ", ." #name)
> ```
>
> `__SC_DECL3`定义
>
> ```c
> //include/linux/syscalls.h
> #define __SC_DECL1(t1, a1)      t1 a1
> #define __SC_DECL2(t2, a2, ...) t2 a2, __SC_DECL1(__VA_ARGS__)
> #define __SC_DECL3(t3, a3, ...) t3 a3, __SC_DECL2(__VA_ARGS__)
> ```
>
> 展开所有的宏定义之后，得到结果
>
> ```c
> asmlinkage long sys_ioctl(unsigned int fd, unsigned int cmd, unsigned long arg);		\
> static inline long SYSC_ioctl(unsigned int fd,  unsigned int cmd, unsigned long arg);	\
> asmlinkage long SyS_ioctl(long fd, long cmd, long arg)		\
> {								\
>     BUILD_BUG_ON(sizeof(int) > sizeof(long)); BUILD_BUG_ON(sizeof(int) > sizeof(long));               BUILD_BUG_ON(sizeof(int) > sizeof(long));				\
>     return (long) SYSC_ioctl((unsigned int) fd, (unsigned int) cmd, (unsigned long) arg);	\
> }								\
>     SYSCALL_ALIAS(sys_ioctl, SyS_ioctl);				\
>     static inline long SYSC_ioctl(unsigned int fd,  unsigned int cmd, unsigned long arg)
> {
>     code...
> }
> ```
>
> 其实里面做的工作，就是将系统调用的参数统一变为了**使用long类型来接收，然后再强转为本来参数类型**。据说这样折腾是为了修复一个漏洞来的。



### 2.3.3struct file结构体

> **struct file**
>
> 这里的结构体相对比较复杂，我们只关注相关的结构体即可。
>
> ```c
> //kernel/include/linux/fs.h
> struct file {
> 	union {
> 		struct llist_node	fu_llist;
> 		struct rcu_head 	fu_rcuhead;
> 	} f_u;
> 	struct path		f_path;
> 	struct inode		*f_inode;	/* cached value */
> 	const struct file_operations	*f_op;
> 
> 	/*
> 	 * Protects f_ep, f_flags.
> 	 * Must not be taken from IRQ context.
> 	 */
> 	spinlock_t		f_lock;
> 	atomic_long_t		f_count;
> 	unsigned int 		f_flags;
> 	fmode_t			f_mode;
> 	struct mutex		f_pos_lock;
> 	loff_t			f_pos;
> 	struct fown_struct	f_owner;
> 	const struct cred	*f_cred;
> 	struct file_ra_state	f_ra;
> 
> 	u64			f_version;
> #ifdef CONFIG_SECURITY
> 	void			*f_security;
> #endif
> 	/* needed for tty driver, and maybe others */
> 	void			*private_data;
> 
> #ifdef CONFIG_EPOLL
> 	/* Used by fs/eventpoll.c to link all the hooks to this file */
> 	struct hlist_head	*f_ep;
> #endif /* #ifdef CONFIG_EPOLL */
> 	struct address_space	*f_mapping;
> 	errseq_t		f_wb_err;
> 	errseq_t		f_sb_err; /* for syncfs */
> } __randomize_layout
>   __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
> ```



## 2.4与自定义的ioctl函数对应

### 2.4.1do_vfs_ioctl

`sys_ioctl`函数调用之后，会走到最关键的函数`do_vfs_ioctl`。这里面会对传入的cmd进行拆解，默认的cmd有下面几种，这些都是**预定义的**，这些命令会被当作预定义命令被内核处理而不是被**设备驱动**处理。

```c
//kernel/fs/ioctl.c
int do_vfs_ioctl(struct file *filp, unsigned int fd, unsigned int cmd,
	     unsigned long arg)
{
	int error = 0;
	int __user *argp = (int __user *)arg;
	struct inode *inode = filp->f_path.dentry->d_inode;

	switch (cmd) {
	case FIOCLEX:
		set_close_on_exec(fd, 1);
		break;

	case FIONCLEX:
		set_close_on_exec(fd, 0);
		break;

	case FIONBIO:
		error = ioctl_fionbio(filp, argp);
		break;

	case FIOASYNC:
		error = ioctl_fioasync(fd, filp, argp);
		break;

	case FIOQSIZE:
		if (S_ISDIR(inode->i_mode) || S_ISREG(inode->i_mode) ||
		    S_ISLNK(inode->i_mode)) {
			loff_t res = inode_get_bytes(inode);
			error = copy_to_user(argp, &res, sizeof(res)) ?
					-EFAULT : 0;
		} else
			error = -ENOTTY;
		break;

	case FIFREEZE:
		error = ioctl_fsfreeze(filp);
		break;

	case FITHAW:
		error = ioctl_fsthaw(filp);
		break;

	case FS_IOC_FIEMAP:
		return ioctl_fiemap(filp, arg);

	case FIGETBSZ:
		return put_user(inode->i_sb->s_blocksize, argp);

	default:
		if (S_ISREG(inode->i_mode))//是否为常规文件若是常规文件
			error = file_ioctl(filp, cmd, arg);
		else
			error = vfs_ioctl(filp, cmd, arg);//调用vfs_ioctl
		break;
	}
	return error;
}
```

> 补充说明
>
> 以下的ioctl命令对任何文件都是预定义的(**属于内核系统级的预定义**)
>
> 前面的五个预定义根据下面的2.5的协议，可以知道，实际上是0x54代表type为**‘T’**
>
> -  **FIOCLEX**
>
>   设置执行时关闭标志(File IOctl CLose on Exec)。设置了这个标志之后，当调用进程执行一个新程序时，文件描述符将被关闭
>
> -  **FIONCLEX**
>
>   清除执行时关闭标志(File IOctl Not CLose on EXec)。该命令将恢复通常的文件行为，并撤销上述FIOCLEX命令所做的工作。
>
> -  **FIOASYNC**
>
>   设置或复位文件异步通知，注意知道Linux 2.2.4 内核都不正确使用这个命令修改O_SYNC标志。因为这两个动作可以通过fcntl完成，所以实际上没人使用这个命令。
>
> -  **FIOQSIZE**
>
>   该命令返回文件或者目录的大小。不过当用于设备文件时，会导致ENOTTY错误的返回
>
> -  **FIONBIO**
>
>   "File IOctl Non-Blocking I/O",文件ioctl非阻塞型I/O。改调用秀阿贵filp->f_flags中的O_NONBLOCK标志。传递系统调用的第三个参数指明了是设置还是清除该标准。修改该标志的常用方法是由fcntl系统调用使用F_SETFL命令来完成。
>
> ```c
> //kernel/include/uapi/asm-generic/ioctls.h
> #define FIONBIO		0x5421
> #define FIONCLEX	0x5450
> #define FIOCLEX		0x5451
> #define FIOASYNC	0x5452
> #ifndef FIOQSIZE
> # define FIOQSIZE	0x5460
> #endif
> ```
>
> 另外，还有几个预定义
>
> 其实内核cmd有一个格式，使用户cmd不与系统cmd冲突，解决办法就是用`_IO`、`_IOW`、`_IOR`和`_IOWR`产生cmd。
>
> ```c
> //kernel/include/uapi/linux/fs.h
> #define FIGETBSZ   _IO(0x00,2)	/* get the block size used for bmap */
> #define FIFREEZE	_IOWR('X', 119, int)	/* Freeze */
> #define FITHAW		_IOWR('X', 120, int)	/* Thaw */
> #define FS_IOC_FIEMAP			_IOWR('f', 11, struct fiemap)
> ```

#### 2.4.1.1inode结构体

这里的函数有一个inode结构体。整个结构是比较复杂的数据结构，我们这里只关注`i_mode`，这个实际上是用于查看是否是可读可写的标志位。

```c
//kernel/include/linux/fs.h
struct inode {
	umode_t			i_mode;
	unsigned short		i_opflags;
	kuid_t			i_uid;
	kgid_t			i_gid;
	unsigned int		i_flags;

	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;

	unsigned long		i_ino;
	union {
		const unsigned int i_nlink;
		unsigned int __i_nlink;
	};
	dev_t			i_rdev;
	loff_t			i_size;
	struct timespec64	i_atime;
	struct timespec64	i_mtime;
	struct timespec64	i_ctime;
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned short          i_bytes;
	u8			i_blkbits;
	u8			i_write_hint;
	blkcnt_t		i_blocks;

	/* Misc */
	unsigned long		i_state;
	struct rw_semaphore	i_rwsem;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when;

	struct hlist_node	i_hash;
	struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

	/* foreign inode detection, see wbc_detach_inode() */
	int			i_wb_frn_winner;
	u16			i_wb_frn_avg_time;
	u16			i_wb_frn_history;
#endif
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	struct list_head	i_wb_list;	/* backing dev writeback list */
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	atomic64_t		i_version;
	atomic64_t		i_sequence; /* see futex */
	atomic_t		i_count;
	atomic_t		i_dio_count;
	atomic_t		i_writecount;
	union {
		const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
		void (*free_inode)(struct inode *);
	};
	struct file_lock_context	*i_flctx;
	struct address_space	i_data;
	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct cdev		*i_cdev;
		char			*i_link;
		unsigned		i_dir_seq;
	};

	__u32			i_generation;

	void			*i_private; /* fs or device private pointer */
} __randomize_layout;
```



#### 2.4.1.2 常见的几个文件宏

关于几个宏定义，可以见下表所示

|      文件宏      |         含义         |
| :--------------: | :------------------: |
| S_ISLNK(st_mode) |    是否是一个连接    |
|     S_ISREG      |  是否是一个常规文件  |
|     S_ISDIR      |    是否是一个目录    |
|     S_ISCHR      |  是否是一个字符设备  |
|     S_ISBLK      |   是否是一个块设备   |
|     S_ISFIFO     |  是否是一个FIFO文件  |
|     S_ISSOCK     | 是否是一个SOCKET文件 |

下面所示的数据结构定义了stat节点，包含了linux的文件所有的属性

```c
//kernel/include/uapi/asm-generic/stat.h
struct stat {
	unsigned long	st_dev;		 //文件的设备编号
	unsigned long	st_ino;		 //节点
	unsigned int	st_mode;	//文件的类型和存取的权限
	unsigned int	st_nlink;	 //连到该文件的硬连接数目，刚建立的文件值为1
	unsigned int	st_uid;		 //用户ID
	unsigned int	st_gid;		  //组ID
	unsigned long	st_rdev;	 //(设备类型)若此文件为设备文件，则为其设备编号
	unsigned long	__pad1;
	long		st_size;	//文件字节数(文件大小)
	int		st_blksize;	 //块大小(文件系统的I/O 缓冲区大小)
	int		__pad2;
	long		st_blocks;	 //块数
	long		st_atime;	 //最后一次访问时间
	unsigned long	st_atime_nsec;
	long		st_mtime;	 //最后一次修改时间
	unsigned long	st_mtime_nsec;
	long		st_ctime;	//最后一次改变时间(指属性)
	unsigned long	st_ctime_nsec;
	unsigned int	__unused4;
	unsigned int	__unused5;
};
```



### 2.4.2vfs_ioctl

根据上述调用，来到了下面的`vfs_ioctl`，这里开始就没有`fd`这个句柄的参数，只需要`file`结构体指针

```c
//kernel/fs/ioctl.c
static long vfs_ioctl(struct file *filp, unsigned int cmd,
		      unsigned long arg)
{
	int error = -ENOTTY;

	if (!filp->f_op || !filp->f_op->unlocked_ioctl)
		goto out;
unlocked_ioctl
	error = filp->f_op->unlocked_ioctl(filp, cmd, arg);//调用unlocked_ioctl()
	if (error == -ENOIOCTLCMD)
		error = -EINVAL;
 out:
	return error;
}	
```

这里先关注以下filp->f_op->unlocked_ioctl，很明显这里有两个结构体，一个是struct file，另外一个是struct file_operations。对于file结构体，上面2.3.3就有阐述。这里主要是看file_operations结构体。



### 2.4.3struct file_operations

这里file_operations结构体中的unlocked_ioctl实际上就是函数指针

```c
//kernel/include/linux/fs.h
struct  file_operations { 
	struct module * owner;
	loff_t (* llseek ) (struct file *, loff_t , int ); 
	ssize_t (* read ) (struct file *, char __user *, size_t , loff_t *); 
	ssize_t (* write ) (struct file *, const char __user *, size_t , loff_t *); 
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *); 
	ssize_t(*write_iter) (struct kiocb *, struct iov_iter *); 
	int (*iterate) (struct file *, struct dir_context *); 
	unsigned int (*poll) (struct file *, struct poll_table_struct *) ; 
	long (*unlocked_ioctl) (struct file *, unsigned int , unsigned long ); 
	long (*compat_ioctl) (struct file *, unsigned int , unsigned long ); 
	int (*mmap) (struct file *, struct vm_area_struct *); 
	...
}
```

### 2.4.4自定义unlocked_ioctl的真实实现

可以自定义在文件操作集中，那么函数指针实际调用的就是`unlocked_ioctl`赋值的函数名。下面注册`my_ioctl`与函数指针`unlocked_ioctl`关联，直到这里才真正意义上的把ioctl的流程完整的对应起来了。也就是上述2.4.3中的filp->f_op->unlocked_ioctl(filp, cmd, arg)等于这里的my_ioctl(filp, cmd, arg)

```c
//文件操作集
struct file_operations misc_fops = {
    ...
    .unlocked_ioctl = my_ioctl
    ...
};
```



## 2.5 ioctl 用户与驱动之间的协议

可以看到下面这个宏定义，这个宏定义包含了协议

```c
//kernel/include/uapi/asm-generic/ioctl.h
#define _IOC(dir,type,nr,size) \
    (((dir)  << _IOC_DIRSHIFT) | \
     ((type) << _IOC_TYPESHIFT) | \
     ((nr)   << _IOC_NRSHIFT) | \
     ((size) << _IOC_SIZESHIFT))
```

这个协议就是 ioctl 方法第二个参数`cmd` 。理论上可以为任意 int 型数据，考虑到4个字节，正好是32位。在linux中，提供了一种 ioctl 命令的统一格式，将 32 位 int 型数据划分为四个位段，如下图2所示

{{< image classes="fancybox center fig-100" src="/ioctl/ioctl_3.png" thumbnail="/ioctl/ioctl_3.png" title="图2 用户与驱动之间的协议图">}}

| 分区                 | bit位 | 含义                        |
| -------------------- | :---: | --------------------------- |
| 第一个分区(**nr**)   |  0-7  | 命令的**编号**，范围是0-255 |
| 第二个分区(**type**) | 8-15  | 命令的**幻数**              |
| 第三个分区(**size**) | 16-29 | 表示传递的数据的**大小**    |
| 第四个分区(**dir**)  | 30-31 | 代表**读写的方向**          |

1. **nr（number）**

   命令编号/序数，占据 8 bit，可以为任意 unsigned char 型数据，取值范围 0~255，如果定义了多个 ioctl 命令，通常从 0 开始编号递增；

2. **type（device type）**

   设备类型，占据 8 bit，可以为任意 char 型字符，例如‘a’、’b’、’c’ 等等，其主要作用是使 ioctl 命令有唯一的设备标识

3. **size**

   涉及到 ioctl 函数第三个参数 arg ，占据14bit，指定了 arg 的数据类型及长度；

4. **dir（direction）**

   ioctl 命令访问模式（数据传输方向），占据 2 bit

   可以为 _IOC_NONE、_IOC_READ、_IOC_WRITE、_IOC_READ | _IOC_WRITE，分别指示了四种访问模式：**无数据0x00、读数据0x01、写数据0x11、读写数据0x11**



```c
//kernel/include/uapi/asm-generic/ioctl.h
/* used to create numbers */
//定义不带参数的 ioctl 命令
#define _IO(type,nr)        _IOC(_IOC_NONE,(type),(nr),0)
//定义带写参数的 ioctl 命令（copy_from_user）
#define _IOR(type,nr,size)  _IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
//定义带读参数的ioctl命令（copy_to_user）
#define _IOW(type,nr,size)  _IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
//定义带读写参数的 ioctl 命令
#define _IOWR(type,nr,size) _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))

//内核还提供了反向解析 ioctl 命令的宏接口：
#define _IOC_DIR(nr)        (((nr) >> _IOC_DIRSHIFT) & _IOC_DIRMASK)	//方向
#define _IOC_TYPE(nr)       (((nr) >> _IOC_TYPESHIFT) & _IOC_TYPEMASK)	//幻数
#define _IOC_NR(nr)     (((nr) >> _IOC_NRSHIFT) & _IOC_NRMASK)			//编号
#define _IOC_SIZE(nr)       (((nr) >> _IOC_SIZESHIFT) & _IOC_SIZEMASK)	//大小
```



# 3举例说明

## 3.1应用层部分

```c
//external/ioctl_test/ioctl_test.c
#include<stdio.h>
#include<stdlib.h>
#include<fcntl.h>
#include<string.h>
#include<unistd.h>
#include<sys/ioctl.h>

#define CMD_TEST_0 _IO('A', 0)
#define CMD_TEST_1 _IOR('A', 1, int)
#define CMD_TEST_2 _IOW('A', 2, int)
#define CMD_TEST_3 _IOWR('A', 3, int)

int main(int argc, char *argv[]){
    int fd = 0;
    int revData = 0;

    fd = open("/dev/ioctl_test", O_RDWR);
    if(fd < 0){
        printf("open failed\n");
        exit(1);
    }
    printf("open success\n");

    /*依次调用四个命令*/
    ioctl(fd, CMD_TEST_0);
    
    revData = ioctl(fd, CMD_TEST_1);
    printf("receive 1 data=%d\n", revData);

    ioctl(fd, CMD_TEST_2, 99);

    revData = ioctl(fd, CMD_TEST_3, 101);
    printf("receive 3 data=%d\n", revData);

    close(fd);
    return 0;
}
```



## 3.2驱动层部分

在 Linux 内核的include/linux目录下有miscdevice.h文件，要把自己定义的misc device从设备定义在这里。定义一个 MISC 设备(miscdevice 类型)以后我们需要设置 minor、name 和 fops 这三个成员变量。minor 表示子设备号，MISC 设备的主设备号为 10，这个是固定的，需要用户指定子设备号，Linux 系统已经预定义了一些 MISC 设备的子设备号。

name 就是此 MISC 设备名字，当此设备注册成功以后就会在/dev 目录下生成一个名为 name的设备文件。fops 就是[字符设备](https://so.csdn.net/so/search?q=字符设备&spm=1001.2101.3001.7020)的操作集合，MISC 设备驱动最终是需要使用用户提供的 fops操作集合。

 ### 3.2.1函数接口

```c
//完成misc设备注册。
int misc_register(struct miscdevice * misc);
//完成misc设备注销。
int misc_deregister(struct miscdevice *misc);
```

驱动部分中的`_init`，`_exit`，关键字实际上也是宏定义，可以点击[这里](https://blog.csdn.net/maopig/article/details/7409870)。

```c
//kernel/drivers/soc/xxx/sdmmc_ioctl_test.c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/io.h>

#define CMD_TEST_0 _IO('A', 0)       //不需要读写的命令
#define CMD_TEST_1 _IOR('A', 1, int) //从内核读取一个int的命令
#define CMD_TEST_2 _IOW('A', 2, int) //向内核写入一个int的命令
#define CMD_TEST_3 _IOWR('A', 3, int) //读写一个int的命令

int misc_open(struct inode *a,struct file *b){
    printk("misc open \n");
    return 0;
}

int misc_release (struct inode * a, struct file * b){
    printk("misc file release\n");
    return 0;
}

long my_ioctl(struct file *fd, unsigned int cmd, unsigned long b){
    /*将命令按内容分解，打印出来*/
    printk("cmd type=(%c)\t nr=(%d)\t dir=(%d)\t size=(%d)\n", _IOC_TYPE(cmd), _IOC_NR(cmd), _IOC_DIR(cmd), _IOC_SIZE(cmd));

    /* 检查设备类型 */
    if (_IOC_TYPE(cmd) != IOC_MAGIC) {
        pr_err("[%s] command type [%c] error!\n", \
               __func__, _IOC_TYPE(cmd));
        return -ENOTTY; 
    }

    /* 检查序数 */
    if (_IOC_NR(cmd) > IOC_MAXNR) { 
        pr_err("[%s] command numer [%d] exceeded!\n", 
               __func__, _IOC_NR(cmd));
        return -ENOTTY;
    }
    
    /* 检查访问模式 */
    if (_IOC_DIR(cmd) & _IOC_READ)
        ret= !access_ok(VERIFY_WRITE, (void __user *)arg, \
                        _IOC_SIZE(cmd));
    else if (_IOC_DIR(cmd) & _IOC_WRITE)
        ret= !access_ok(VERIFY_READ, (void __user *)arg, \
                        _IOC_SIZE(cmd));
    if (ret)
        return -EFAULT;

    switch(cmd){
        case CMD_TEST_0://不需要读写的命令
            printk("CMD_TEST_0\n");
            break;
        case CMD_TEST_1://从内核读取一个int的命令
            printk("CMD_TEST_1\n");
            return 1;
            break;
        case CMD_TEST_2://向内核写入一个int的命令
            printk("CMD_TEST_2 date=(%d)\n",b);
            break;
        case CMD_TEST_3://读写一个int的命令
            printk("CMD_TEST_3 date=(%d)\n",b);
            return b+1;
            break;
    }

    return 0;
}

//文件操作集
struct file_operations misc_fops = {
    .owner = THIS_MODULE,
    .open = misc_open,
    .release = misc_release,
    .unlocked_ioctl = my_ioctl
    .compat_ioctl = my_ioctl
};

//ioctl_test，设备节点名,添加了这个语句可以在设备中ls找到这个设备节点，并且是c开头的
struct miscdevice misc_dev = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "ioctl_test",   
    .fops = &misc_fops
};

//这里的__exit,这是一个宏定义,同下面的__init类似
static __exit int ioctl_init(void){
    int ret;

    ret = misc_register(&misc_dev);  //注册杂项设备
    if(ret < 0){
        printk("misc regist failed\n");
        return -1;
    }

    printk("misc regist succeed\n");
    return 0;
}

//这里的__init，这是一个宏定义
//最常用的地方是驱动模块初始化函数的定义处，其目的是将驱动模块的初始化函数放入名叫.init.text的输入段
static __init void ioctl_exit(void){
    misc_deregister(&misc_dev);
}

//下面这两个宏，可以展开
MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("yangyang48");

//这里可以改成module_init
device_initcall_sync(ioctl_init); 
module_exit(ioctl_exit);  
```

> MODULE_AUTHOR和MODULE_LICENSE宏定义
>
> 这部分其实就是许多宏定义的嵌套，具体可以点击[这里](https://blog.csdn.net/qq_36412526/article/details/82885631)，可以详细的宏定义的展开说明
>
> ```c
> //kernel/include/linux/module.h
> #define MODULE_AUTHOR(_author) MODULE_INFO(author, _author)
> #define MODULE_INFO(tag, info) __MODULE_INFO(tag, tag, info)
> 
> // 连接形参的宏
> #define ___module_cat(a,b) __mod_ ## a ## b 
> #define __module_cat(a,b) ___module_cat(a,b)
> 
> #ifdef MODULE
> #define __MODULE_INFO(tag, name, info)					  \
> static const char __module_cat(name,__LINE__)[]				  \
>   __used __attribute__((section(".modinfo"), unused, aligned(1)))	  \
>   = __stringify(tag) "=" info
> #else  /* !MODULE */
> /* This struct is here for syntactic coherency, it is not used */
> #define __MODULE_INFO(tag, name, info)					  \
>   struct __module_cat(name,__LINE__) {}
> #endif
> ```
>
> MODULE_AUTHOR这部分其实可以直接展开
>
> ```c
>  static const char __mod_authorXX[]  \
>  __used  __attribute__((section(".modinfo"),unused,aligned(1)))  \
>  = "author = yangyang48"
> ```
>
> 同理，MODULE_LICENSE也可以展开
>
> ```c
> static const char __mod_licenseXX[]  \
>  __used  __attribute__((section(".modinfo"),unused,aligned(1)))  \
>  = "license= Dual BSD/GPL"
> ```

### 3.2.2驱动的校验部分

上面有一段内容是检验的部分，主要用于对用户自定义的协议进行校验。

- 如果是cmd中的dir为读取模式，那么是用户空间向内核空间读取数据，那么需要检查用户空间的可写空间地址。
- 如果是cmd中的dir为可写模式，那么是用户空间向内核空间写数据，那么需要检查用户空间的可写空间地址。

```c
/* 检查设备类型 */
//这里定义的type是'A'
if (_IOC_TYPE(cmd) != IOC_MAGIC) {
    pr_err("[%s] command type [%c] error!\n", \
           __func__, _IOC_TYPE(cmd));
    return -ENOTTY; 
}

/* 检查序数 */
//这里定义的序数为4
if (_IOC_NR(cmd) > IOC_MAXNR) { 
    pr_err("[%s] command numer [%d] exceeded!\n", 
           __func__, _IOC_NR(cmd));
    return -ENOTTY;
}    

/* 检查访问模式 */
if (_IOC_DIR(cmd) & _IOC_READ)
    ret= !access_ok(VERIFY_WRITE, (void __user *)arg, \
                    _IOC_SIZE(cmd));
else if (_IOC_DIR(cmd) & _IOC_WRITE)
    ret= !access_ok(VERIFY_READ, (void __user *)arg, \
                    _IOC_SIZE(cmd));
if (ret)
    return -EFAULT;
```
> 补充
>
> 这里的access_ok函数，主要是防止内核被外部攻击
>
> ```c
> //返回值：布尔值，1表示成功，0表示失败
> //type:检查用户空间地址的权限；VERIFY_READ或者VERIFY_WRITE；
> //<1>VERIFY_READ：驱动是否可以读取用户空间的指定地址；
> //<2>VERIFY_WRITE：驱动是否可以读取用户空间的指定地址；
> //<3>VERIFY_WRITE：驱动在指定用户空间地址既要读取也要写入，也是填这个；
> //addr：用户空间地址；
> //size:要操作的字节数；例如驱动要从指定用户空间地址读取一个int型整数，则size就是sizeof(int)；
> int access_ok(int type, const void __user *addr, unsigned long size);
> ```



### 3.2.3调用流程总结

最终得到的调用流程如下所示，通过用户态调用的ioctl函数设备节点/dev/ioctl_test最终在内核态的my_ioctl实现这个流程。

{{< image classes="fancybox center fig-100" src="/ioctl/ioctl_2.png" thumbnail="/ioctl/ioctl_2.png" title="">}}

## 3.3编译

kernel在驱动层的编译，需要依赖于Kconfig和Makefile文件；在应用层的编译，需要依赖于Android.mk。

### 3.3.1驱动层的编译

- Kconfig
  每个源码目录下提供选项
- .config
  源码顶层目录下保存选择结果
- Makefile
  每个源码目录下根据.config中的内容来告知编译系统如何编译

从这里可以看到，这里也是有一定的语法规则。其中Kconfig用来配置内核，它就是各种配置界面的源文件，内核的配置工具读取各个Kconfig文件，生成配置界面供开发人员配置内核，最后**生成配置文件.config**。



`config`是关键字，表示一个配置选项的开始；紧跟着的`IOCTL_TEST`是配置选项的名称，省略了前缀`CONFIG_`
bool表示变量类型，即`CONFIG_IOCTL_TEST `的类型。

**字符串“This is IOCTL_TEST”是提示信息**，在配置界面中上下移动光标选中它时，就可以通过按空格或回车键来设置`CONFIG_IOCTL_TEST `的值

default代表tristate变量的值，一共有三个值y、n和m。如果是n代表所需要的文件不需要被编译进去，这里笔者需要编译进入所有使得，`CONFIG_IOCTL_TEST `的值为y。

```kconfig
config IOCTL_TEST
    bool "This is IOCTL_TEST"
    default y
    help
      IOCTL TEST support permissions for users and groups beyond the owner/group/world scheme.
      If you don't know what Access Control Lists are, say N.
```

然后在Makefile文件中添加这个需要编译的二进制文件

```makefile
obj-y += sdmmc_ioctl_test.o
//可以写成上述的样子
obj-$(CONFIG_IOCTL_TEST) += sdmmc_ioctl_test.o
```

### 3.3.2应用层编译

这个比较简单，就是单纯的Android.mk文件即可

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := ioctl_test
LOCAL_SRC_FILES += ioctl_test.c
LOCAL_C_INCLUDE += $(LOCAL_PATH)/
LOCAL_C_FLAG := -Wreturn-local-addr -Wwriting_strings -fpermissive -fexceptions -Wall -Wno-unused-variable -lgcc_s -std=c99
#编译生成动态文件so
include $(BUILD_SHARE_LIBRARY)
include $(call all_makefiles_under, $(LOCAL_PATH))
```

# 4总结

ioctl 是一种面向硬件驱动，是内核比较早的一种用户态内核态的交互方式，侧重于文件系统，方便添加对硬件驱动的处理。

ioctl机制可以在驱动中扩展特定的ioctl消息，用于将一些状态从内核反应到用户态。ioctl有很好的数据同步保护机制，不用担心内核和用户层的数据访问冲突，**但是ioctl不适合传输大量的数据**。

本文通过介绍ioctl函数的流程，涉及到**软中断**、**系统调用表**、**file结构指针**、**用户与驱动之间的协议**，通过一个demo把整个流程贯穿一遍，更好的熟悉了解到整个ioctl的通信流程。

# 两个可以参考的源码网站

[aospxref.com](http://aospxref.com/android-11.0.0_r21/)

[kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/?h=v5.19-rc7)



# 5下载

本文源码下载，点击[这里](https://github.com/YangYang48/project/tree/master/ioctl)



# 参考

[[1]酷派技术团队, Linux用户态与内核态通信方式介绍及选型参考, 2022.](https://mp.weixin.qq.com/s/8zoa7Ywftm4TmLdf8PAPFg)

[[2] WuYunTaXue, ioctl使用方法, 2021.](https://www.cnblogs.com/TaXueWuYun/p/15315594.html)

[[3] pingandezhufu, Linux内核system_call中断处理过程, 2015.](https://www.cnblogs.com/pingandezhufu/p/4392642.html)

[[4] WuYunTaXue, ioctl使用方法, 2022.](https://www.cnblogs.com/TaXueWuYun/p/15315594.html)

[[5] 鸿鹄实验室, 浅析syscall, 2021.](https://cloud.tencent.com/developer/article/1944012)

[[6] 彼 方, Linux系统调用SYSCALL_DEFINE详解, 2021.](https://blog.csdn.net/weixin_43798887/article/details/118930476)

[[7] maopig, 内核中_init，_exit中的作用, 2012.](https://blog.csdn.net/maopig/article/details/7409870)

[[8] Android系统攻城狮, GFP_KERNEL的作用, 2018.](https://unbroken.blog.csdn.net/article/details/84789220)

[[9] 雪舞飞影, ioctl函数详解（Linux内核 ）, 2021.](https://blog.csdn.net/dongxianfei/article/details/118256267)

[[10] ora___, ioctl系统调用过程(深入Linux(ARM)内核源码), 2019.](https://blog.csdn.net/weixin_38815998/article/details/103510385)

[[11] 内核笔记, RK3399平台开发系列讲解（内核入门篇）1.8、IOCTL的用法详解, 2022.](https://xuesong.blog.csdn.net/article/details/81489268)

[[12] monkea123, 杂项设备（misc device）, 2020.](https://blog.csdn.net/monkea123/article/details/109137398)

[[13] 菜鸟程序员的成长历程,Linux Kernel代码艺术——数组初始化 , 2013.](https://www.cnblogs.com/hazir/p/array_initialization.html)

[[14] Felix_8_, Kconfig文件的用途及解析, 2020.](https://blog.csdn.net/wojiaowoen/article/details/107938455)

[[15] 无棱, Kconfig, 2022.](https://blog.csdn.net/weixin_42817735/article/details/122982569)

[[16] qq_36412526, MODULE_AUTHOR宏(一), 2018.](https://blog.csdn.net/qq_36412526/article/details/82885631)

[[17] c枫_撸码的日子, [读书笔记]高级字符驱动程序（第六章）, 2018.](https://www.jianshu.com/p/2f092e81cbc1?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[[18] bytxl, S_ISDIR S_ISREG等常见的几个宏, 2012.](https://blog.csdn.net/bytxl/article/details/8215116)

[[19] 正在起飞的蜗牛, access_ok()函数介绍, 2022.](https://blog.csdn.net/weixin_42817735/article/details/122982569)

[[20] 宋宝华, 为什么内核访问用户数据之前，要做access_ok?, 2019.](https://zhuanlan.zhihu.com/p/94020714)