# Hello World

## 4.1 最简单的内核模块

大多数人都是从`Hello World`一类的例子开始学编程。有些人另辟蹊径，不知道这些人遭到了怎样的境遇，但我觉得还是不去打听为妙。我们将通过一系列的Hello World程序来学习编写内核模块涉及到的各方面基础知识。

以下是一个最简单的内核模块。

创建一个测试文件夹：

```bash
mkdir -p ~/develop/kernel/hello-1
cd ~/develop/kernel/hello-1
```

将以下内容复制粘贴到你最心爱的编辑器中，保存为`hello-1.c`。

```c
/* 
 * hello-1.c - 最简单的内核模块
 */ 
#include <linux/kernel.h> /* pr_info()用到 */ 
#include <linux/module.h> /* 所有内核模块都用到 */ 
 
int init_module(void) 
{ 
    pr_info("Hello world 1.\n"); 
 
    /* 非0返回值意味着init_module失败，模块无法加载 */ 
    return 0; 
} 
 
void cleanup_module(void) 
{ 
    pr_info("Goodbye world 1.\n"); 
} 
 
MODULE_LICENSE("GPL");
```

然后你需要一个`Makefile`。如果下面这段你也是复制粘贴过去，一定要把缩进里的空格改成tab。

```makefile
obj-m += hello-1.o 
 
PWD := $(CURDIR) 
 
all: 
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules 
 
clean: 
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

在`Makefile`中，`$(CURDIR)`会指到当前工作目录的绝对路径（不过如果你刻意指定，`-C`选项会优先处理）。详情可以参考[GNU make手册]()。

最后，直接执行`make`命令。

```bash
make
```

如果Makefile里没有指定`PWD := $(CURDIR)`，你执行`sudo make`时可能无法正确编译。因为一些环境变量可能受到安全策略的影响，无法被`sudo`继承。默认的安全策略在`sudoers`文件，该文件指定的安全策略中，`env_reset`默认开启，这会影响命令执行的环境变量，具体来说，`PATH`变量无法保留普通用户的设置，而是被设成默认值（具体参考[sudoers手册]()）。你可以以普通用户执行`sudo -s`或以root用户执行`sudo -V`查看环境变量设置。

下列Makefile可以直观感受到上面说的问题：

```makefile
all:
    echo $(PWD)
```

我们可以作为普通用户，用`-p`选项打印出这个Makefile涉及的环境变量。

```
$ make -p | grep PWD
PWD = /home/ubuntu/temp
OLDPWD = /home/ubuntu
    echo $(PWD)
```

`sudo`一下，可以看到`PWD`变量不会被继承。

```
$ sudo make -p | grep PWD
    echo $(PWD)
```

不过，有三种方式解决这个问题。

1. 使用`-E`选项保留环境变量。
    ```
    $ sudo -E make -p | grep PWD
        PWD = /home/ubuntu/temp
        OLDPWD = /home/ubuntu
        echo $(PWD)
    ```
2. 作为root使用`visudo`命令编辑`/etc/sudoers`文件，禁用`env_reset`选项。
    ```sudoers
    ## sudoers 文件
    ## 
    ... 
    Defaults env_reset 
    ## 把这行的 env_reset 改成 !env_reset 来保留所有环境变量
    ```
    然后分别执行`env`和`sudo env`命令。
    ```bash
    # 禁用 env_reset 时 
    echo "user:" > non-env_reset.log; env >> non-env_reset.log 
    echo "root:" >> non-env_reset.log; sudo env >> non-env_reset.log 
    # 启用 env_reset 时 
    echo "user:" > env_reset.log; env >> env_reset.log 
    echo "root:" >> env_reset.log; sudo env >> env_reset.log
    ```
    对比两份日志，就能看到`env_reset`和`!env_reset`带来的差异。
3. 在`/etc/sudoers`文件里通过在`env_keep`变量后面追加来指定要保留的环境变量。
    ```sudoers
      Defaults env_keep += "PWD"
    ```
    保存后，再以普通用户执行`sudo -s`或以root用户执行`sudo -V`查看环境变量设置。

如果进展顺利，你会发现`hello-1.ko`模块已经被编出来了。可以使用下列命令查看模块相关信息：

```bash
modinfo hello-1.ko
```

这个时候，如果你执行下列命令，按说什么都不会返回：

```bash
sudo lsmod | grep hello
```

现在可以按以下命令把你崭新的模块加载到内核中了：

```bash
sudo insmod hello-1.ko
```

中横线（`-`）会被转成下划线（`_`）。你再次执行以下命令时，你会看到模块已经被加载了。

```bash
sudo lsmod | grep hello
```

如果你想卸载模块，可以执行以下命令：

```bash
sudo rmmod hello_1
```

注意到，中横线（`-`）会被转成下划线（`_`）。日志里可以看到具体发生的事情：

```bash
sudo journalctl --since "1 hour ago" | grep kernel
```

你现在已经了解了创建、编译、安装和移除内核模块的基本流程。现在向你详细介绍一下这个模块是如何运作的。

内核模块必须有至少两个函数：一个“启动”（初始化）函数`init_module()`，在你通过`insmod`将其插入内核后被调用；还有个“终止”（清理）函数`cleanup_module()`，在其被从内核中移除前调用。其实自从内核版本2.3.13开始，你已经可以给模块的启动和终止函数指定你心仪的名字了，在[4.2](#42-你好再见)中可以学到如何操作。这种新做法事实上已经成了推荐做法。不过，还是有很多人继续用`init_module()`和`cleanup_module()`来命名启动和终止函数。

一般来说，`init_module()`要么在内核里给某件东西注册一个句柄，要么把内核函数替换为自己的代码（一般来说会做点操作然后再调用原函数）。而`cleanup_module()`需要做的是把`init_module()`做过的操作全部还原，以实现安全卸下模块。

每个内核模块都必须包含`<linux/module.h>`。我们上面的例子包含`<linux/kernel.h>`只是因为`pr_alert()`日志等级的宏展开需要，你在下文*第2点*就会明白。

1. 谈几句编码风格。好些刚开始做内核编程的人没注意，代码缩进**必须使用tab而非空格**。这是内核代码的约定，你个人可能不喜欢，但如果你要给内核上游提交代码，你就必须适应。
2. 再谈谈输出宏。起初，我们有个`printk`函数，后面跟着日志优先级`KERN_INFO`或`KERN_DEBUG`。近来这种操作可以用一系列输出宏来表述，比如`pr_info`和`pr_debug`，这样做看起来更养眼，也更爱护键盘。你可以在[`include/linux/printk.h`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/printk.h)查看详细内容。建议你花点时间通读一遍优先级相关的宏。
3. 再谈谈编译。内核模块的编译方式和一般的用户态程序有些不一样。早先的内核版本需要我们格外留意相关配置，这些配置一般存在Makefile里。尽管经过详细组织整理，一些子层级的Makefile还是积累了许多多余的配置项，以至于又膨胀又难维护。还好，现在有了新的编译配置手段，叫`kbuild`，外部模块的构建流程，现在也被整合到标准的内核构建流程中了。如果想要深入了解如何编译不在主线内核中的模块（比如本书中你看到的所有例子），可以阅读内核仓库中的[`Documentation/kbuild/modules.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/kbuild/modules.rst)。
    内核模块Makefile的其他细节记录在[`Documentation/kbuild/makefiles.rst`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/kbuild/makefiles.rst)中。在开始魔改Makefile之前，你最好先通读一遍相关文档，能帮你少走不少弯路。

> 给读者一个小练习：看到init_module()函数返回语句前面的注释没？把返回值改成负数，重新编译并加载模块，看看会发生什么。

## 4.2 你好，再见

在早期内核版本中，你必须用`init_module`和`cleanup_module`来命名启动和终止函数，你在第一个hello world例子中就是这么做的。不过现在你可以用`module_init`宏和`module_exit`宏来给启动和终止函数指定名字了；这两个宏在[`include/linux/module.h`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/module.h)中。唯一的要求是你的启动和终止函数必须在调用宏之前声明，否则编译会出错。这种用法的例子如下：

```c
/* 
 * hello-2.c - 解释module_init()和module_exit()宏
 * 更建议使用这两个宏，而不是init_module()和cleanup_module()
 */ 
#include <linux/init.h> /* 这两个宏用到 */ 
#include <linux/kernel.h> /* pr_info()用到 */ 
#include <linux/module.h> /* 所有内核模块都用到 */ 
 
static int __init hello_2_init(void) 
{ 
    pr_info("Hello, world 2\n"); 
    return 0; 
} 
 
static void __exit hello_2_exit(void) 
{ 
    pr_info("Goodbye, world 2\n"); 
} 
 
module_init(hello_2_init); 
module_exit(hello_2_exit); 
 
MODULE_LICENSE("GPL");
```

现在我们名下已经有两个如假包换的内核模块了。把另一个模块加上去也很简单，操作如下：

```makefile
obj-m += hello-1.o 
obj-m += hello-2.o 
 
PWD := $(CURDIR) 
 
all: 
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules 
 
clean: 
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

现在看个真实世界的例子，[`drivers/char/Makefile`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/char/Makefile)。你可以看到，有些玩意与内核紧紧连在一起（`obj-y`）。但`obj-m`哪去了？熟悉shell脚本的人稍动脑筋就能明白。随处可见的`obj-$(CONFIG_FOO)`会展开为obj-y或obj-m，这取决于`CONFIG_FOO`被赋值为`y`还是`m`。顺带一提，这些变量定义在Linux内核源码树顶层目录里的`.config`文件，你执行`make menuconfig`之类的命令时就会设好。

## 4.3 `__init`和`__exit`宏

`__init`宏负责在内建驱动程序的初始化函数执行完毕时，把初始化函数扔掉并释放内存；不过在可加载的模块中不起作用。考虑到二者init函数调用的时机不同，就能想通。

还有一个`__initdata`宏，和`__init`机制类似，不过是作用在初始化变量而非函数上。

`__exit`宏在模块被集成到内核中时，会导致相应函数失活不执行。不过和`__init`宏一样，这个宏对可加载模块不起作用。同理，思考清理函数执行的时机，不难想通，集成到内核里的驱动不需要被清理，而动态加载的内核模块需要。

上述的宏都在[`include/linux/init.h`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/init.h)中定义，用于清理内核内存区域。在你启动内核时看到“Freeing unused kernel memory: 236k freed”之类的东西时，被清理的就是上述相应内容。

```c
/* 
 * hello-3.c - 解释 __init, __initdata 和 __exit 宏
 */ 
#include <linux/init.h> /* 这几个宏用到 */ 
#include <linux/kernel.h> /* pr_info()用到 */ 
#include <linux/module.h> /* 所有模块都用到 */ 
 
static int hello3_data __initdata = 3; 
 
static int __init hello_3_init(void) 
{ 
    pr_info("Hello, world %d\n", hello3_data); 
    return 0; 
} 
 
static void __exit hello_3_exit(void) 
{ 
    pr_info("Goodbye, world 3\n"); 
} 
 
module_init(hello_3_init); 
module_exit(hello_3_exit); 
 
MODULE_LICENSE("GPL");
```

## 4.4 许可证和内核模块文档

有一说一，真的有人加载甚至哪怕关心专有版权的内核模块吗？你可以试试加载专有版权的模块，可能会得到以下结果；

```
$ sudo insmod xxxxxx.ko
loading out-of-tree module taints kernel.
module license 'unspecified' taints kernel.
```

你可以通过几个宏来为你的模块声明许可证。比如说"GPL", "GPL v2", "GPL and additional rights", "Dual BSD/GPL", "Dual MIT/GPL", "Dual MPL/GPL" 和 "Proprietary"。相关声明在[`include/linux/module.h`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/module.h)中定义。

可以使用`MODULE_LICENSE`宏来引用你想用的许可证。此外还有几个宏用于说明内核模块的相关信息，请参考下面的例子。

```c
/* 
 * hello-4.c - 解释内核模块文档 
 */ 
#include <linux/init.h> /* 这几个宏用到 */ 
#include <linux/kernel.h> /* pr_info()用到 */ 
#include <linux/module.h> /* 所有模块都用到 */ 
 
MODULE_LICENSE("GPL"); 
MODULE_AUTHOR("LKMPG"); 
MODULE_DESCRIPTION("A sample driver"); 
 
static int __init init_hello_4(void) 
{ 
    pr_info("Hello, world 4\n"); 
    return 0; 
} 
 
static void __exit cleanup_hello_4(void) 
{ 
    pr_info("Goodbye, world 4\n"); 
} 
 
module_init(init_hello_4); 
module_exit(cleanup_hello_4);
```
## 4.5 向内核模块传递命令行参数

可以给内核模块传入命令行参数，不过并不是以大家熟悉的argc/argv方式。

为了允许向模块传参，你需要声明全局变量来接收命令行参数，然后使用`module_param()`宏（在[`include/linux/moduleparam.h`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/moduleparam.h)中定义）进行相关设置。在运行时，`insmod`命令会把接收的命令行参数放到相关变量中，比如`insmod mymodule.ko myvariable=5`。为了保持代码整洁，变量声明和宏调用应该放在模块的开头。后面的示例代码比我含糊的解释清晰得多。

`module_param()`宏接收3个参数：变量名、变量类型和与之相对应的sysfs文件。整型可以带符号，也可以不带。如果你想用整数数组或者字符串，请参考`module_param_array()`和`module_param_string()`。

```c
int myint = 3; 
module_param(myint, int, 0);
```

补充说明，数组的支持方式已经和稍早的内核版本有所不同了。为了保证你输入的参数个数可控，你得在第三个参数传入一个指针，指针指向一个变量用于计数。不过你也可以选择无视计数，传入`NULL`。两种情况参考以下例子：

```c
int myintarray[2]; 
module_param_array(myintarray, int, NULL, 0); /* 不关心计数 */ 
 
short myshortarray[4]; 
int count; 
module_param_array(myshortarray, short, &count, 0); /* 计数会被放在count变量 */
```

推荐的做法是给内核模块的变量设置默认值（比如端口或者IO地址），当变量是默认值时，进行自动探测（后面会讲），否则保持现在的值。后面我们会详细讲解。

还有个宏函数`MODULE_PARM_DESC()`，用于说明模块接收参数的作用。该函数接收两个参数：变量名和说明字符串。

```c
/* 
 * hello-5.c - 解释如何向内核模块传递命令行参数
 */ 
#include <linux/init.h> 
#include <linux/kernel.h> 
#include <linux/module.h> 
#include <linux/moduleparam.h> 
#include <linux/stat.h> 
 
MODULE_LICENSE("GPL"); 
 
static short int myshort = 1; 
static int myint = 420; 
static long int mylong = 9999; 
static char *mystring = "blah"; 
static int myintarray[2] = { 420, 420 }; 
static int arr_argc = 0; 
 
/* module_param(foo, int, 0000) 
 * 第一个参数是参数名称
 * 第二个参数是参数类型 
 * 最后一个参数是权限位 
 * 在稍后的阶段用于将该参数暴露在sysfs下（如果设置成非0）
 */ 
module_param(myshort, short, S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP); 
MODULE_PARM_DESC(myshort, "A short integer"); 
module_param(myint, int, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH); 
MODULE_PARM_DESC(myint, "An integer"); 
module_param(mylong, long, S_IRUSR); 
MODULE_PARM_DESC(mylong, "A long integer"); 
module_param(mystring, charp, 0000); 
MODULE_PARM_DESC(mystring, "A character string"); 
 
/* module_param_array(name, type, num, perm); 
 * 第一个参数是参数名称（这里的情况是数组名称）
 * 第二个参数是数组类型名称 
 * 第三个参数是一个指针，指向的变量用于存储模块加载时
 * 被用户初始化的数组中元素的个数
 * 第四个参数是权限位
 */ 
module_param_array(myintarray, int, &arr_argc, 0000); 
MODULE_PARM_DESC(myintarray, "An array of integers"); 
 
static int __init hello_5_init(void) 
{ 
    int i; 
 
    pr_info("Hello, world 5\n=============\n"); 
    pr_info("myshort is a short integer: %hd\n", myshort); 
    pr_info("myint is an integer: %d\n", myint); 
    pr_info("mylong is a long integer: %ld\n", mylong); 
    pr_info("mystring is a string: %s\n", mystring); 
 
    for (i = 0; i < ARRAY_SIZE(myintarray); i++) 
        pr_info("myintarray[%d] = %d\n", i, myintarray[i]); 
 
    pr_info("got %d arguments for myintarray.\n", arr_argc); 
    return 0; 
} 
 
static void __exit hello_5_exit(void) 
{ 
    pr_info("Goodbye, world 5\n"); 
} 
 
module_init(hello_5_init); 
module_exit(hello_5_exit);
```

个人推荐按下面的操作折腾一番：

```
$ sudo insmod hello-5.ko mystring="bebop" myintarray=-1
$ sudo dmesg -t | tail -7
myshort is a short integer: 1
myint is an integer: 420
mylong is a long integer: 9999
mystring is a string: bebop
myintarray[0] = -1
myintarray[1] = 420
got 1 arguments for myintarray.

$ sudo rmmod hello-5
$ sudo dmesg -t | tail -1
Goodbye, world 5

$ sudo insmod hello-5.ko mystring="supercalifragilisticexpialidocious" myintarray=-1,-1
$ sudo dmesg -t | tail -7
myshort is a short integer: 1
myint is an integer: 420
mylong is a long integer: 9999
mystring is a string: supercalifragilisticexpialidocious
myintarray[0] = -1
myintarray[1] = -1
got 2 arguments for myintarray.

$ sudo rmmod hello-5
$ sudo dmesg -t | tail -1
Goodbye, world 5

$ sudo insmod hello-5.ko mylong=hello
insmod: ERROR: could not insert module hello-5.ko: Invalid parameters
```

## 4.6 将内核模块源码分成多个文件

有的时候你会需要把内核模块源码分成多个文件。

以下是一个例子。

```c
/* 
 * start.c - 演示多文件的内核模块
 */ 
 
#include <linux/kernel.h> /* 整内核相关的活 */ 
#include <linux/module.h> /* 模块要用 */ 
 
int init_module(void) 
{ 
    pr_info("Hello, world - this is the kernel speaking\n"); 
    return 0; 
} 
 
MODULE_LICENSE("GPL");
```

另一个文件：

```c
/* 
 * stop.c - 演示多文件的内核模块
 */ 
 
#include <linux/kernel.h> /* 整内核相关的活 */ 
#include <linux/module.h> /* 模块要用 */ 
 
void cleanup_module(void) 
{ 
    pr_info("Short is the life of a kernel module\n"); 
} 
 
MODULE_LICENSE("GPL");
```

最后是Makefile：

```makefile
obj-m += hello-1.o 
obj-m += hello-2.o 
obj-m += hello-3.o 
obj-m += hello-4.o 
obj-m += hello-5.o 
obj-m += startstop.o 
startstop-objs := start.o stop.o 
 
PWD := $(CURDIR) 
 
all: 
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules 
 
clean: 
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

这份Makefile包括了我们之前提到的所有例子。前五行普普通通，而后面两行就是刚才的例子要用到的。我们先给组合起来的模块取个名，然后我们告诉`make`这个模块包含哪些目标文件。

## 4.7 为预编译好的内核构建模块

我们强烈推荐你重新编译你自己的内核，从而开启一些非常方便你调试的特性。比如说强制模块卸载（`MODULE_FORCE_UNLOAD`）：开启这个选项时，你可以用`sudo rmmod -f 模块名`命令，在内核认为操作不安全时依然强制卸下模块。这个选项可以在开发模块时帮你省下许多时间，避免许多不必要的重启。如果你不想重新编译内核，推荐你在虚拟机上用测试发行版来跑相关的示例。万一不小心操作失误了，重启或者恢复虚拟机都很方便。

很多时候你可能会想把模块加载到已经编译好的内核里，比如说适配市面常见Linux发行版，或者你要适配你之前编译过的内核。还有些特殊场景，比如你可能想现场编译并把模块插入到正在运行的内核，这个时候你可能没法重新编译内核或者不方便重启机器。如果你觉得你遇不到一定要在预编译好的内核上加载模块的场景，你可以跳过本小节，把本章剩余部分当做一个大脚注。

如果你直接安装内核源码树，用它来编译你的内核模块，然后试着把这个模块插入你现在的内核，你很可能会得到以下错误：

```
insmod: ERROR: could not insert module poet.ko: Invalid module format
```

比较易懂的信息在systemd日志里有记录：

```
kernel: poet: disagrees about version of symbol module_layout
```

也就是说，你的内核拒绝接受你的模块，因为版本字符串（确切来说是*版本魔数*，参见[include/linux/vermagic.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/vermagic.h)）不匹配。版本魔数以静态字符串的形式存在模块的目标文件中，开头是`vermagic:`。版本信息在链接到`kernel/module.o`文件时写入你的模块中。要查看一个模块的版本魔数等其他字符串信息，使用`modinfo module.ko`命令：

```
$ modinfo hello-4.ko
description:    A sample driver
author:         LKMPG
license:        GPL
srcversion:     B2AA7FBFCC2C39AED665382
depends:
retpoline:      Y
name:           hello_4
vermagic:       5.4.0-70-generic SMP mod_unload modversions
```

为了解决上述问题，我们可以使用`--force-vermagic`选项，但这个做法有潜在风险，在生产环境中，毫无疑问不可接受。于是，我们最好还是要在与预编译好的内核一致的环境中编译我们的内科模块。本章剩余部分就会说到具体做法。

首先，确保你已经有了与你当前内核版本完全一致的内核源码树。然后，找到你当前预编译好的内核对应的编译配置。一般来说，你可以在你的`/boot`目录下找到它，名字类似`config-5.14.x`。你只需要把它复制到你的内核源码树中：```cp /boot/config-`uname -r` .config```。

再回头看看之前的报错：进一步检查版本魔数发现，即使使用完全相同的两份配置文件，版本魔数中还是可能会产生微小的差异，导致你无法向内核中插入模块。这个微小的差异叫做自定义字符串，出现在模块的版本魔数而不是内核的版本魔数中。