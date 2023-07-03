# /proc 文件系统

Linux内核和内核模块有一种额外的机制用来给进程发送信息——`/proc`文件系统。起初，它被设计用来方便访问进程的相关信息（所以会叫这个名字），不过现在内核只要有点有意思的信息都能用它来报告，比如`/proc/modules`，提供了内核模块列表，而`/proc/meminfo`提供了内存使用统计相关信息。

proc文件系统的使用方式和设备驱动非常相似：创建一个结构体，里面包含一个`/proc`文件所需的所有信息，包括指向所有处理函数的指针（我们下面的例子里只有一个，在有人试图从`/proc`文件中读取时就会调用）。然后，用`init_module`在内核里注册这个结构体，用`cleanup_module`注销。

一般的文件系统位于硬盘上，而非内存中（`/proc`恰在此处），这种情况下索引节点（简称inode）号是一个指针，指向磁盘上对应文件inode的位置。inode包含这个文件相关的信息，比如文件权限信息和指向文件数据在硬盘上的具体位置的指针等。

因为在文件打开或关闭时我们不需要被调用，这个模块中我们就不放`try_module_get`和`module_put`了。这种情况下，在文件被打开而模块被移除时，我们无法回避可能带来的后果。

下面的例子展示了`/proc`文件的用法，给出了`/proc`文件系统的HelloWorld。源码包括三个部分：在`init_module`函数中创建`/proc/helloworld`文件，在`/proc/helloworld`被读取时调用回调函数`procfile_read`并返回一个值（和一个缓冲区），然后在`cleanup_module`中删除`/proc/helloworld`文件。

在模块加载时，通过`proc_create`函数创建`/proc/helloworld`文件，并返回`struct proc_dir_entry`。这个结构体会用来设置`/proc/helloworld`的相关项（比如文件的所有者）。空指针返回意味着文件创建失败。

每当`/proc/helloworld`被读取时，`procfile_read`函数就会被调用。该函数的参数中，有两个非常重要：缓冲区（第二个参数）和偏移量（第四个参数）。缓冲区的内容会被返回给读取该文件的程序（比如`cat`命令）。偏移量是该文件目前读取到的位置。如果该函数的返回值不为空，该函数就会再次被调用。所以使用该函数时要格外注意，如果函数永远不返回0，读取函数就会一直不停被循环调用。

```bash
$ cat /proc/helloworld
HelloWorld!
```

```c
/* 
 * procfs1.c 
 */ 
 
#include <linux/kernel.h> 
#include <linux/module.h> 
#include <linux/proc_fs.h> 
#include <linux/uaccess.h> 
#include <linux/version.h> 
 
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0) 
#define HAVE_PROC_OPS 
#endif 
 
#define procfs_name "helloworld" 
 
static struct proc_dir_entry *our_proc_file; 
 
static ssize_t procfile_read(struct file *filePointer, char __user *buffer, 
                             size_t buffer_length, loff_t *offset) 
{ 
    char s[13] = "HelloWorld!\n"; 
    int len = sizeof(s); 
    ssize_t ret = len; 
 
    if (*offset >= len || copy_to_user(buffer, s, len)) { 
        pr_info("copy_to_user failed\n"); 
        ret = 0; 
    } else { 
        pr_info("procfile read %s\n", filePointer->f_path.dentry->d_name.name); 
        *offset += len; 
    } 
 
    return ret; 
} 
 
#ifdef HAVE_PROC_OPS 
static const struct proc_ops proc_file_fops = { 
    .proc_read = procfile_read, 
}; 
#else 
static const struct file_operations proc_file_fops = { 
    .read = procfile_read, 
}; 
#endif 
 
static int __init procfs1_init(void) 
{ 
    our_proc_file = proc_create(procfs_name, 0644, NULL, &proc_file_fops); 
    if (NULL == our_proc_file) { 
        proc_remove(our_proc_file); 
        pr_alert("Error:Could not initialize /proc/%s\n", procfs_name); 
        return -ENOMEM; 
    } 
 
    pr_info("/proc/%s created\n", procfs_name); 
    return 0; 
} 
 
static void __exit procfs1_exit(void) 
{ 
    proc_remove(our_proc_file); 
    pr_info("/proc/%s removed\n", procfs_name); 
} 
 
module_init(procfs1_init); 
module_exit(procfs1_exit); 
 
MODULE_LICENSE("GPL");
```

## 7.1 proc_ops结构体

在Linux内核版本5.6+中`proc_ops`结构体在[include/linux/proc_fs.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/proc_fs.h)中定义。在较老的内核版本中，这个结构体还用到`file_operations`来处理在`/proc`文件系统里的自定义钩子，但`file_operations`当中有些成员对VFS来说用不到，每次VFS展开`file_operations`集合时，`/proc`的代码段就变得特别臃肿。除了空间优化以外，这个结构还精简了一些操作流程来提高性能。比如`/proc`当中永远存在的文件可以将`proc_flag`设为`PROC_ENTRY_PERMANENT`，这样一来，在每个打开/读取/关闭流程中都可以省下2次原子操作、1次分配和1次释放。

## 7.2 对/proc文件的读写

刚才我们已经看了一个非常简单的`/proc`文件例子，例子中，我们仅对`/proc/helloworld`进行了读取。其实`/proc`文件也支持写入，工作方式和读取操一样，在`/proc`文件被写入时会调用一个函数。不过和读取操作不同的一点是，数据是从用户那写过来的，你必须把数据从用户空间导入到内核空间（用`copy_from_user`或者`get_user`实现）。

使用`copy_from_user`和`get_user`的原因在于Linux内存空间（在Intel架构上是这样的，在其他架构处理器上可能不同）是分段的。这意味着一个指针本身并不指向内存中具体的位置，而是指向内存段上的位置。你要使用这个指针，就必须知道指针用在哪个内存段上。内核本身只有一个内存段，而每个进程都有一个段。

每个进程都只能访问自己的内存段，所以你在写一般的作为进程跑起来的程序时，并不用考虑段的问题。你写内核模块时往往需要访问内核的段，这方面操作系统会帮你自动处理。然而，在内存缓冲区中的内容需要从当前进程传递到内核时，内核函数需要接收指向用户进程段上的缓冲区的指针。`put_user`和`get_user`宏可以让你访问对应的内存段。这两个函数每次只能处理一个字符，要处理多个字符时，可以使用`copy_from_user`和`copy_to_user`。`/proc`文件读写操作调到的函数中的缓冲区在内核空间，对于写操作函数，你必须从用户空间中导入数据；但对于读函数来说并不用，因为数据本身就在内核空间。

```c
/* 
 * procfs2.c -  在/proc中创建一个“文件”
 */ 
 
#include <linux/kernel.h> /* 处理内核相关的工作 */ 
#include <linux/module.h> /* 具体来说就是写内核模块 */ 
#include <linux/proc_fs.h> /* 操作 proc fs 要用到 */ 
#include <linux/uaccess.h> /* copy_from_user 要用到 */ 
#include <linux/version.h> 
 
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0) 
#define HAVE_PROC_OPS 
#endif 
 
#define PROCFS_MAX_SIZE 1024 
#define PROCFS_NAME "buffer1k" 
 
/* 这个结构体保存关于 /proc 文件的相关信息 */ 
static struct proc_dir_entry *our_proc_file; 
 
/* 这个缓冲区用来为该模块保存字符 */ 
static char procfs_buffer[PROCFS_MAX_SIZE]; 
 
/* 缓冲区大小 */ 
static unsigned long procfs_buffer_size = 0; 
 
/* 在 /proc 文件读取时该函数会被调用 */ 
static ssize_t procfile_read(struct file *file_pointer, char __user *buffer, 
                             size_t buffer_length, loff_t *offset) 
{ 
    char s[13] = "HelloWorld!\n"; 
    int len = sizeof(s); 
    ssize_t ret = len; 
 
    if (*offset >= len || copy_to_user(buffer, s, len)) { 
        pr_info("copy_to_user failed\n"); 
        ret = 0; 
    } else { 
        pr_info("procfile read %s\n", file_pointer->f_path.dentry->d_name.name); 
        *offset += len; 
    } 
 
    return ret; 
} 
 
/* 在写入到 /proc 文件时该函数会被调用 */ 
static ssize_t procfile_write(struct file *file, const char __user *buff, 
                              size_t len, loff_t *off) 
{ 
    procfs_buffer_size = len; 
    if (procfs_buffer_size > PROCFS_MAX_SIZE) 
        procfs_buffer_size = PROCFS_MAX_SIZE; 
 
    if (copy_from_user(procfs_buffer, buff, procfs_buffer_size)) 
        return -EFAULT; 
 
    procfs_buffer[procfs_buffer_size & (PROCFS_MAX_SIZE - 1)] = '\0'; 
    *off += procfs_buffer_size; 
    pr_info("procfile write %s\n", procfs_buffer); 
 
    return procfs_buffer_size; 
} 
 
#ifdef HAVE_PROC_OPS 
static const struct proc_ops proc_file_fops = { 
    .proc_read = procfile_read, 
    .proc_write = procfile_write, 
}; 
#else 
static const struct file_operations proc_file_fops = { 
    .read = procfile_read, 
    .write = procfile_write, 
}; 
#endif 
 
static int __init procfs2_init(void) 
{ 
    our_proc_file = proc_create(PROCFS_NAME, 0644, NULL, &proc_file_fops); 
    if (NULL == our_proc_file) { 
        proc_remove(our_proc_file); 
        pr_alert("Error:Could not initialize /proc/%s\n", PROCFS_NAME); 
        return -ENOMEM; 
    } 
 
    pr_info("/proc/%s created\n", PROCFS_NAME); 
    return 0; 
} 
 
static void __exit procfs2_exit(void) 
{ 
    proc_remove(our_proc_file); 
    pr_info("/proc/%s removed\n", PROCFS_NAME); 
} 
 
module_init(procfs2_init); 
module_exit(procfs2_exit); 
 
MODULE_LICENSE("GPL");
```

## 7.3 使用标准文件系统管理/proc文件

上文中，我们学会了如何通过`/proc`接口来对`/proc`文件进行读写。此外，我们也可以通过inode来管理`/proc`。我们主要需要注意高级函数的使用，比如权限相关的操作。

在Linux中，文件系统注册有一套标准的机制。因为每个文件系统都必须得有自己的用来处理inode和文件操作的函数，内核中有一个特殊的结构`struct inode_operations`来保存指向这些函数的指针，这个结构也包括一个指向`struct proc_ops`的指针。

文件操作和inode操作的区别在于，文件操作处理文件本身，而inode操作处理对该文件的引用，比如创建一个到文件的链接。

在`/proc`中，每当我们注册一个新文件的时候，我们都可以指定使用具体的某个`struct inode_operations`来访问该文件；这就是我们用到的机制，一个`struct inode_operations`包含一个指针指向`struct proc_ops`，而后者又包括指向`procfs_read`和`procfs_write`函数的指针。

还有一个有趣之处在于`module_permission`函数。当一个进程试图对`/proc`文件进行操作时，该函数就会被调用，并决定是否允许该进程访问。现在这个函数只是基于目前用户的UID和操作来做判断（目前而言，是一个指针，指向了包含当前运行进程信息的结构），不过它也可以按我们想要的方式来做判断，比如说别的进程在对相同文件做的操作，日期时间，或者我们最后收到的输入。

需要注意，标准的读和写的角色在内核中已经预留好了，读函数用于输出，写函数用于输入。这样做的原因在于，读和写操作都是从用户的视角来看的，如果进程需要从内核中读取数据，内核就需要把数据输出来，而如果进程需要把数据写入内核，内核就需要接受数据输入。

```c
/* 
 * procfs3.c 
 */ 
 
#include <linux/kernel.h> 
#include <linux/module.h> 
#include <linux/proc_fs.h> 
#include <linux/sched.h> 
#include <linux/uaccess.h> 
#include <linux/version.h> 
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 10, 0) 
#include <linux/minmax.h> 
#endif 
 
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0) 
#define HAVE_PROC_OPS 
#endif 
 
#define PROCFS_MAX_SIZE 2048UL 
#define PROCFS_ENTRY_FILENAME "buffer2k" 
 
static struct proc_dir_entry *our_proc_file; 
static char procfs_buffer[PROCFS_MAX_SIZE]; 
static unsigned long procfs_buffer_size = 0; 
 
static ssize_t procfs_read(struct file *filp, char __user *buffer, 
                           size_t length, loff_t *offset) 
{ 
    if (*offset || procfs_buffer_size == 0) { 
        pr_debug("procfs_read: END\n"); 
        *offset = 0; 
        return 0; 
    } 
    procfs_buffer_size = min(procfs_buffer_size, length); 
    if (copy_to_user(buffer, procfs_buffer, procfs_buffer_size)) 
        return -EFAULT; 
    *offset += procfs_buffer_size; 
 
    pr_debug("procfs_read: read %lu bytes\n", procfs_buffer_size); 
    return procfs_buffer_size; 
} 
static ssize_t procfs_write(struct file *file, const char __user *buffer, 
                            size_t len, loff_t *off) 
{ 
    procfs_buffer_size = min(PROCFS_MAX_SIZE, len); 
    if (copy_from_user(procfs_buffer, buffer, procfs_buffer_size)) 
        return -EFAULT; 
    *off += procfs_buffer_size; 
 
    pr_debug("procfs_write: write %lu bytes\n", procfs_buffer_size); 
    return procfs_buffer_size; 
} 
static int procfs_open(struct inode *inode, struct file *file) 
{ 
    try_module_get(THIS_MODULE); 
    return 0; 
} 
static int procfs_close(struct inode *inode, struct file *file) 
{ 
    module_put(THIS_MODULE); 
    return 0; 
} 
 
#ifdef HAVE_PROC_OPS 
static struct proc_ops file_ops_4_our_proc_file = { 
    .proc_read = procfs_read, 
    .proc_write = procfs_write, 
    .proc_open = procfs_open, 
    .proc_release = procfs_close, 
}; 
#else 
static const struct file_operations file_ops_4_our_proc_file = { 
    .read = procfs_read, 
    .write = procfs_write, 
    .open = procfs_open, 
    .release = procfs_close, 
}; 
#endif 
 
static int __init procfs3_init(void) 
{ 
    our_proc_file = proc_create(PROCFS_ENTRY_FILENAME, 0644, NULL, 
                                &file_ops_4_our_proc_file); 
    if (our_proc_file == NULL) { 
        remove_proc_entry(PROCFS_ENTRY_FILENAME, NULL); 
        pr_debug("Error: Could not initialize /proc/%s\n", 
                 PROCFS_ENTRY_FILENAME); 
        return -ENOMEM; 
    } 
    proc_set_size(our_proc_file, 80); 
    proc_set_user(our_proc_file, GLOBAL_ROOT_UID, GLOBAL_ROOT_GID); 
 
    pr_debug("/proc/%s created\n", PROCFS_ENTRY_FILENAME); 
    return 0; 
} 
 
static void __exit procfs3_exit(void) 
{ 
    remove_proc_entry(PROCFS_ENTRY_FILENAME, NULL); 
    pr_debug("/proc/%s removed\n", PROCFS_ENTRY_FILENAME); 
} 
 
module_init(procfs3_init); 
module_exit(procfs3_exit); 
 
MODULE_LICENSE("GPL");
```

还想看更多的关于procfs的例子吗？首先提醒一下你，有些流言说procfs已经过时了，要用`sysfs`替代。如果你自己想要学习内核相关的知识，还是可以考虑用这个机制的。

## 7.4 使用seq_file管理/proc文件

在前面的章节中，我们能感受到，创建一个`/proc`文件的流程可能相当“复杂”。为了帮助人们创建`/proc`文件，Linux提供了一个名叫`seq_file`的API，用于帮助规范`/proc`文件以便输出。这个API囊括了由`start()`、`next()`和`stop()`三个函数组成的序列操作。每当用户读取`/proc`文件时，`seq_file`就会调这一序列操作。

序列在最开头调用`start()`函数。如果返回值不为`NULL`，则接下来`next()`函数就会被调用。这函数是个迭代器，用来遍历一遍所有的数据。每次`next()`函数被调用时，`show()`函数也会被调用。

**注意：**一个序列操作完成后，马上又会调另一个序列操作，也就是说，`stop()`函数在末尾会再次调用`start()`函数。直到`start()`函数返回`NULL`，循环才会停止。具体机制参考下图。


TODO: seq_file的图

`seq_file`提供了操作`proc_ops`的基础函数，如`seq_read`、`seq_lseek`等，但不提供对`/proc`文件的写入操作。当然，你仍然可以像上一个例子一样操作

```c
/* 
 * procfs4.c -  在/proc中创建一个“文件”
 * 这个程序使用 seq_file 管理 /proc 文件 
 */ 
 
#include <linux/kernel.h> /* 处理内核相关的工作 */ 
#include <linux/module.h> /* 具体来说就是写内核模块 */ 
#include <linux/proc_fs.h> /* 操作 proc fs 要用到 */ 
#include <linux/seq_file.h> /* seq_file 要用到 */ 
#include <linux/version.h> 
 
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 6, 0) 
#define HAVE_PROC_OPS 
#endif 
 
#define PROC_NAME "iter" 
 
/* 这个函数在每次序列操作开始时被调用
 * 比如说：
 *   - 第一次阅读 /proc 文件的时候 
 *   - 函数终止的时候(序列结尾) 
 */ 
static void *my_seq_start(struct seq_file *s, loff_t *pos) 
{ 
    static unsigned long counter = 0; 
 
    /* 是新的序列吗？ */ 
    if (*pos == 0) { 
        /* 是 => 返回非空值，序列操作开始 */ 
        return &counter; 
    } 
 
    /* 不是 => 序列结束，返回空停止操作 */ 
    *pos = 0; 
    return NULL; 
} 
 
/* 这个函数在每次序列操作开始后被调用
 * 一直调用直到返回NULL为止 (从而停止序列读取操作). 
 */ 
static void *my_seq_next(struct seq_file *s, void *v, loff_t *pos) 
{ 
    unsigned long *tmp_v = (unsigned long *)v; 
    (*tmp_v)++; 
    (*pos)++; 
    return NULL; 
} 
 
/* 在序列结束时被调用 */ 
static void my_seq_stop(struct seq_file *s, void *v) 
{ 
    /* 我们在start()中使用了静态值，这儿什么都不需要做 */ 
} 
 
/* This function is called for each "step" of a sequence. */ 
static int my_seq_show(struct seq_file *s, void *v) 
{ 
    loff_t *spos = (loff_t *)v; 
 
    seq_printf(s, "%Ld\n", *spos); 
    return 0; 
} 
 
/* This structure gather "function" to manage the sequence */ 
static struct seq_operations my_seq_ops = { 
    .start = my_seq_start, 
    .next = my_seq_next, 
    .stop = my_seq_stop, 
    .show = my_seq_show, 
}; 
 
/* This function is called when the /proc file is open. */ 
static int my_open(struct inode *inode, struct file *file) 
{ 
    return seq_open(file, &my_seq_ops); 
}; 
 
/* This structure gather "function" that manage the /proc file */ 
#ifdef HAVE_PROC_OPS 
static const struct proc_ops my_file_ops = { 
    .proc_open = my_open, 
    .proc_read = seq_read, 
    .proc_lseek = seq_lseek, 
    .proc_release = seq_release, 
}; 
#else 
static const struct file_operations my_file_ops = { 
    .open = my_open, 
    .read = seq_read, 
    .llseek = seq_lseek, 
    .release = seq_release, 
}; 
#endif 
 
static int __init procfs4_init(void) 
{ 
    struct proc_dir_entry *entry; 
 
    entry = proc_create(PROC_NAME, 0, NULL, &my_file_ops); 
    if (entry == NULL) { 
        remove_proc_entry(PROC_NAME, NULL); 
        pr_debug("Error: Could not initialize /proc/%s\n", PROC_NAME); 
        return -ENOMEM; 
    } 
 
    return 0; 
} 
 
static void __exit procfs4_exit(void) 
{ 
    remove_proc_entry(PROC_NAME, NULL); 
    pr_debug("/proc/%s removed\n", PROC_NAME); 
} 
 
module_init(procfs4_init); 
module_exit(procfs4_exit); 
 
MODULE_LICENSE("GPL");
```

如果你想了解更多信息，可以参考以下网页：

- <https://lwn.net/Articles/22355/>
- <https://kernelnewbies.org/Documents/SeqFileHowTo>

你也可以直接阅读Linux内核中的[`fs/seq_file.c`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/fs/seq_file.c)。
