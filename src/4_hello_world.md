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

现在看个真实世界的例子，[`drivers/char/Makefile`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/char/Makefile)。你可以看到，