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

7.2 对/proc文件的读写

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
