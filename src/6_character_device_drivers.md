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

不过，`register_chrdev()`会根据给定的主设备号来占用一大片次设备号。为了在注册字符设备时减少浪费，推荐使用cdev接口。

这个新接口通过两个独立步骤完成字符设备的注册。首先，我们通过`register_chrdev_region`或`alloc_chrdev_region`函数注册一个设备号范围。

```c
int register_chrdev_region(dev_t from, unsigned count, const char *name); 
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```

选择哪个函数取决于你是否知道你设备的主设备号。如果你知道主设备号的话就用`register_chrdev_region`函数，而如果你想动态分配一个主设备号就用`alloc_chrdev_region`函数。

然后，我们针对我们的字符设备初始化`struct cdev`数据结构，并给它关联设备号。通过类似下面的代码来初始化`struct cdev`。

```c
struct cdev *my_dev = cdev_alloc(); 
my_cdev->ops = &my_fops;
```

不过，最常见的用法是，在你设备自己的数据结构中内嵌一个`struct dev`结构。这个时候我们用`cdev_init`来初始化。

```c
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```

完成初始化之后，用`cdev_add`来将字符设备添加到系统中。

```c
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

如果你想参考这个接口的用例，可以看看[第9章]()中提到的`ioctl.c`。

## 6.4 注销一个设备

我们不能让root想卸载模块就卸载模块（用`rmmod`命令）。如果设备文件还在被一个进程打开着然后我们移除了模块，使用这个设备文件就会导致调用到本来对应函数（read/write）所在的内存区域。运气好的话那片地方可能没有代码段加载进去，我们无非看到个奇怪的报错。运气不好的话，如果另一个内核模块也被加载到了相同的内存区域，内核就会调到别的函数中间的代码段，导致的结果就不好说了，但肯定不会是什么好事。

一般来说，如果你想禁止一些操作，你会让对应的函数返回错误码（一个负数）。`cleanup_module`做不到，因为它返回值是`void`。不过，有个计数器用来持续统计有多少进程在使用你模块。你可以`cat /proc/modules`或者`sudo lsmod`，输出的第三列即为对应的值。如果这个值不为0，`rmmod`就会失败。注意，你不需要在`cleanup_module`里面手动检查这个计数器，在调用定义于[include/linux/syscalls.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/syscalls.h)的`sys_delete_module`函数时，内核会自动帮你检查。你不应该直接用到这个计数器，不过[include/linux/module.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/module.h)里定义了几个函数，可以让你增加、减少和显示计数：

- `try_module_get(THIS_MODULE)`：增加当前模块的引用计数
- `module_put(THIS_MODULE)`：减少当前模块的引用计数
- `module_refcount(THIS_MODULE)`：返回当前模块的引用计数

保持计数准确非常重要。如果失去了正确的引用计数，你就再也没法卸载该模块了；这个时候还是重启吧，孩子们。在进行内核模块开发时，你或迟或早总得撞上这事儿。

## 6.5 chardev.c

接下来的例子是创建一个名为`chardev`的字符设备驱动。你可以打开它的设备文件

```bash
cat /proc/devices
```

（或者用其他程序打开它），驱动会把该设备已经被读取的次数写入该文件。我们还没有支持对文件的写入（像`echo "hi" > /dev/hello`），但我们会捕获用户的操作，然后告诉用户当前还不支持写入操作。如果你没看懂我们对读入缓冲区的数据做的操作也先别担心，我们这方面先不做过多处理。我们就单纯地读入数据，然后打印出一条消息告知用户我们已经收到了数据。

在多线程环境中，没有任何保护的情况下，对同一内存区域的并发访问可能会导致竞态，从而影响功能。内核模块中，在多个实例访问共享的资源时，就可能发生竞态。有一个解决方法是强制互斥访问。我们使用比较交换机制（CAS）维护两个状态`CDEV_NOT_USED`和`CDEV_EXCLUSIVE_OPEN`来确定当前文件有没有被人打开占用。CAS将特定内存区域的值与预期值做对比，如果二者一致的话，就将内存区域的值替换为一个指定值。在[第12章]()中有更多关于并发的细节可供参考。

```c
/* 
 * chardev.c: 创建一个只读的字符设备，
 * 告知设备读取的次数
 */ 
 
#include <linux/atomic.h> 
#include <linux/cdev.h> 
#include <linux/delay.h> 
#include <linux/device.h> 
#include <linux/fs.h> 
#include <linux/init.h> 
#include <linux/kernel.h> /* sprintf() 用到 */ 
#include <linux/module.h> 
#include <linux/printk.h> 
#include <linux/types.h> 
#include <linux/uaccess.h> /* get_user 和 put_user 用到 */ 
 
#include <asm/errno.h> 
 
/*  原型 - 一般会放在 .h 头文件里 */ 
static int device_open(struct inode *, struct file *); 
static int device_release(struct inode *, struct file *); 
static ssize_t device_read(struct file *, char __user *, size_t, loff_t *); 
static ssize_t device_write(struct file *, const char __user *, size_t, 
                            loff_t *); 
 
#define SUCCESS 0 
#define DEVICE_NAME "chardev" /* /proc/devices 中显示的设备名称   */ 
#define BUF_LEN 80 /* 设备返回消息最大长度 */ 
 
/* 使用static声明全局变量，这样它就是在本文件内全局 */ 
 
static int major; /* 我们设备驱动绑定的主设备号 */ 
 
enum { 
    CDEV_NOT_USED = 0, 
    CDEV_EXCLUSIVE_OPEN = 1, 
}; 
 
/* 设备是否已被打开？用于防止该设备被并发访问 */ 
static atomic_t already_open = ATOMIC_INIT(CDEV_NOT_USED); 
 
static char msg[BUF_LEN + 1]; /* 被请求时，设备要返回的消息 */ 
 
static struct class *cls; 
 
static struct file_operations chardev_fops = { 
    .read = device_read, 
    .write = device_write, 
    .open = device_open, 
    .release = device_release, 
}; 
 
static int __init chardev_init(void) 
{ 
    major = register_chrdev(0, DEVICE_NAME, &chardev_fops); 
 
    if (major < 0) { 
        pr_alert("Registering char device failed with %d\n", major); 
        return major; 
    } 
 
    pr_info("I was assigned major number %d.\n", major); 
 
    cls = class_create(THIS_MODULE, DEVICE_NAME); 
    device_create(cls, NULL, MKDEV(major, 0), NULL, DEVICE_NAME); 
 
    pr_info("Device created on /dev/%s\n", DEVICE_NAME); 
 
    return SUCCESS; 
} 
 
static void __exit chardev_exit(void) 
{ 
    device_destroy(cls, MKDEV(major, 0)); 
    class_destroy(cls); 
 
    /* 注销该设备 */ 
    unregister_chrdev(major, DEVICE_NAME); 
} 
 
/* 方法 */ 
 
/* 在一个进程尝试打开设备文件时触发该方法，比如说 
 * "sudo cat /dev/chardev" 
 */ 
static int device_open(struct inode *inode, struct file *file) 
{ 
    static int counter = 0; 
 
    if (atomic_cmpxchg(&already_open, CDEV_NOT_USED, CDEV_EXCLUSIVE_OPEN)) 
        return -EBUSY; 
 
    sprintf(msg, "I already told you %d times Hello world!\n", counter++); 
    try_module_get(THIS_MODULE); 
 
    return SUCCESS; 
} 
 
/* 在一个进程关闭设备文件时触发该方法 */ 
static int device_release(struct inode *inode, struct file *file) 
{ 
    /* 现在设备已经可以给下一个调用者使用了 */ 
    atomic_set(&already_open, CDEV_NOT_USED); 
 
    /* 减少使用计数, 否则你一旦打开过设备文件，
     * 就没法再卸载该模块了
     */ 
    module_put(THIS_MODULE); 
 
    return SUCCESS; 
} 
 
/* 在已经打开了设备文件的进程试图从文件中读取数据时 
 * 触发该方法。
 */ 
static ssize_t device_read(struct file *filp, /* 见 include/linux/fs.h   */ 
                           char __user *buffer, /* 用来填充数据的缓冲区 */ 
                           size_t length, /* 缓冲区长度 */ 
                           loff_t *offset) 
{ 
    /* 已经写入缓冲区的字节数 */ 
    int bytes_read = 0; 
    const char *msg_ptr = msg; 
 
    if (!*(msg_ptr + *offset)) { /* 到达消息结尾 */ 
        *offset = 0; /* 重置偏移量 */ 
        return 0; /* 告知文件已到达结尾 */ 
    } 
 
    msg_ptr += *offset; 
 
    /* 将数据拷进缓冲区的部分 */ 
    while (length && *msg_ptr) { 
        /* 缓冲区位于用户空间的数据段中，而不在内核空间段中， 
         * 所以没法直接用 "*" 取值。我们必须使用 put_user，
         * 该函数将数据从内核数据段拷贝到
         * 用户空间数据段。
         */ 
        put_user(*(msg_ptr++), buffer++); 
        length--; 
        bytes_read++; 
    } 
 
    *offset += bytes_read; 
 
    /* 绝大多数读取函数都会将已放入缓冲区的字节数作为返回值 */ 
    return bytes_read; 
} 
 
/* 在进程写入设备文件时触发该方法，比如 echo "hi" > /dev/hello */ 
static ssize_t device_write(struct file *filp, const char __user *buff, 
                            size_t len, loff_t *off) 
{ 
    pr_alert("Sorry, this operation is not supported.\n"); 
    return -EINVAL; 
} 
 
module_init(chardev_init); 
module_exit(chardev_exit); 
 
MODULE_LICENSE("GPL");
```

## 6.6 为多个内核版本编写内核模块

系统调用是内核对进程提供的最主要的接口，一般来说在不同内核版本之间保持一致。可能会有新的系统调用加进来，但旧的调用会与之前的版本保持完全一致。对于后向兼容性来说这个设计至关重要——新的内核版本不应该影响正常的进程运行。多数情况下，设备文件也是一样保持一致的。不过，内核内部的接口在版本之间确实会发生变动。

不同内核版本间的内核接口存在差异，如果你想同时支持多个内核版本的话，你在代码里就必须写条件编译指示。做法就是比较`LINUX_VERSION_CODE`和`KERNEL_VERSION`两个宏。对于内核版本`a.b.c`而言，前者的值会表示为\\( 2^{16}a+2^{8}b+c \\)。
