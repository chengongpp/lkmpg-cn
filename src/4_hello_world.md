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
 * hello-1.c - The simplest kernel module. 
 */ 
#include <linux/kernel.h> /* Needed for pr_info() */ 
#include <linux/module.h> /* Needed by all modules */ 
 
int init_module(void) 
{ 
    pr_info("Hello world 1.\n"); 
 
    /* A non 0 return means init_module failed; module can't be loaded. */ 
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

内核模块必须有至少两个函数：一个“启动”（初始化）函数`init_module()`，在你通过`insmod`将其插入内核后被调用；还有个“终止”（清理）函数`cleanup_module()`，在其被从内核中移除前调用。其实自从内核版本2.3.13开始，你已经可以给模块的启动和终止函数指定你心仪的名字了，在[4.2]()中可以学到如何操作。这种新做法事实上已经成了推荐做法。不过，还是有很多人继续用`init_module()`和`cleanup_module()`来命名启动和终止函数。

