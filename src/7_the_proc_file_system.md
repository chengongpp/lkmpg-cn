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

