# 字符设备驱动

## 6.1 `file_operations` 结构体

[include/linux/fs.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/fs.h)中定义了`file_operations`结构体，该结构体用于存放指针，指向由驱动定义的、用来对硬件做各种操作的函数。该结构体中的每个变量都对应驱动定义的某个函数，用于处理请求。

比如说，每个字符驱动都需要定义一个从设备读取内容的函数。`file_operations`结构体保存了内核模块中该函数的地址。以下是在内核版本5.4中，一个定义的示例：

```c
struct file_operations { 
    struct module *owner; 
    loff_t (*llseek) (struct file *, loff_t, int); 
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *); 
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *); 
    ssize_t (*read_iter) (struct kiocb *, struct iov_iter *); 
    ssize_t (*write_iter) (struct kiocb *, struct iov_iter *); 
    int (*iopoll)(struct kiocb *kiocb, bool spin); 
    int (*iterate) (struct file *, struct dir_context *); 
    int (*iterate_shared) (struct file *, struct dir_context *); 
    __poll_t (*poll) (struct file *, struct poll_table_struct *); 
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long); 
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long); 
    int (*mmap) (struct file *, struct vm_area_struct *); 
    unsigned long mmap_supported_flags; 
    int (*open) (struct inode *, struct file *); 
    int (*flush) (struct file *, fl_owner_t id); 
    int (*release) (struct inode *, struct file *); 
    int (*fsync) (struct file *, loff_t, loff_t, int datasync); 
    int (*fasync) (int, struct file *, int); 
    int (*lock) (struct file *, int, struct file_lock *); 
    ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int); 
    unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long); 
    int (*check_flags)(int); 
    int (*flock) (struct file *, int, struct file_lock *); 
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int); 
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int); 
    int (*setlease)(struct file *, long, struct file_lock **, void **); 
    long (*fallocate)(struct file *file, int mode, loff_t offset, 
        loff_t len); 
    void (*show_fdinfo)(struct seq_file *m, struct file *f); 
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *, 
        loff_t, size_t, unsigned int); 
    loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in, 
             struct file *file_out, loff_t pos_out, 
             loff_t len, unsigned int remap_flags); 
    int (*fadvise)(struct file *, loff_t, loff_t, int); 
} __randomize_layout;
```

有些操作并不是由驱动实现的。举个例子，显卡驱动不需要从目录结构中进行读取。对应项在`file_operations`结构体中应被置为`NULL`。

有一个gcc扩展语法可以方便你给这个结构赋值。这个写法在现代的驱动代码里能看到，说不定还能吸引你的眼球。以下是给该结构赋值的新写法：

```c
struct file_operations fops = { 
    read: device_read, 
    write: device_write, 
    open: device_open, 
    release: device_release 
};
```

不过，还有一个C99语法用于给结构体中的元素赋值，叫做[指定初始化](https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html)，比起上述的GNU扩展语法，更推荐使用这个语法。说不定有人会想移植你写的驱动，所以你应该这么写来提供更好的兼容性：

```c
struct file_operations fops = { 
    .read = device_read, 
    .write = device_write, 
    .open = device_open, 
    .release = device_release 
};
```

语义非常清晰，不过你还需要注意，结构体中任何你没有显式赋值的成员都会被gcc默认初始化为`NULL`。

包含指向用于实现读取、写入、打开等系统调用的函数的指针的 `file_operations`结构体实例通常被称做`fops`。

自从Linux内核版本v3.14开始，读、写和查询操作都使用特定的`f_pos`锁来保证线程安全，文件位置更新变成了互斥操作，所以我们现在不需要锁也能安全实现上述操作了。

此外，Linux内核版本v5.6开始引入`proc_ops`结构体，用于在注册进程句柄时取代原先的`file_operations`结构体。详情参考[7.1]()章节。

## 6.2 文件结构体

每个设备在内核中都表示为一个文件结构体，在[include/linux/fs.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/fs.h)中定义。注意，文件是内核层面的结构体，用户态程序中是不可能见到的。它和`FILE`结构体不一样，`FILE`是由glibc定义的，且绝不可能在内核空间的函数里出现。此外，这个叫法其实有点误导，它表示的是一个抽象的打开的“文件”，不是磁盘里的文件；磁盘里的文件表示成名叫`inode`的结构体。

文件结构体的实例通常被称作`filp`。你也会看到这个名字用来指代文件结构体对象。先别生气。

接下来看看文件的定义。你看到的绝大多数entry,比如dentry结构体，并不会在设备驱动中用到，可以直接无视；这是因为驱动并不直接填充文件，它们只是使用别的地方创建的文件中包含的结构体。

## 注册一个设备

前面说过，字符设备通过一般位于`/dev`下的设备文件访问，这是惯例做法。在写驱动的时候，你可以在当前文件夹放设备文件。只要确保在生产环境的驱动中放到`/dev`即可。主设备号告诉你哪个驱动处理哪个设备文件，次设备号在驱动需要处理不止一个设备时，给驱动自己用来区分它在操作的设备。

把驱动加到你的系统里意味着在内核中注册它。这和在内核模块初始化时给它分配个主设备号是一个意思。这个操作通过在[include/linux/fs.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/fs.h)中定义的`register_chrdev`函数来实现。

```c
int register_chrdev(unsigned int major, const char *name, struct file_operations *fops);
```

`unsigned int major`就是你想申请的主设备号；`const char *name`是设备的名称，在`/proc/devices`中会列出；`struct file_operations *fops`是指向你的驱动的`file_operations`表的指针。返回值如果为负，意味着注册失败。注意，次设备号并不传入`register_chrdev`函数里，因为内核不关心次设备号，只有我们的驱动要用到。

问题来了，你怎么避免和已经在用的主设备号起冲突而重新分一个呢？最简单的办法是看过一遍[Documentation/admin-guide/devices.txt](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/admin-guide/devices.txt)然后挑个没用过的号。但这是个坏习惯，因为你不能保证你挑的号后面会不会有别人也分去用。一个解法是，你可以问内核要一个动态的主设备号。

如果你向`register_chrdev`传的主设备号是`0`，返回值就是动态分配的主设备号。缺点就是你没法提前创建设备文件，因为你不知道主设备号会是啥。有许多解决办法。一是让驱动自己打印新分配到的设备号，我们自己手动创建设备文件；二是新注册的设备会在`/proc/devices`留下一条记录，我们可以自己手动创建设备文件，也可以写个脚本文件读取这个文件，并创建设备文件；三是可以让驱动自己使用`device_create`函数，在注册成功后创建设备文件，然后在`cleanup_module`调用中用`device_destroy`函数清理。