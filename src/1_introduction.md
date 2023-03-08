# 引言

Linux内核模块编程指南是一本自由的书籍；你可以在[开放软件协议](https://opensource.org/licenses/OSL-3.0)3.0的许可下复制或改动这本书。

本书的在不提供任何保障的情况下，希望能够帮助大家，本书甚至不刻意声明在销售、健康等方面能提供任何保障。

在上述版权说明完整、且方式遵循[开放软件协议](https://opensource.org/licenses/OSL-3.0)的前提下，作者鼓励这本书以个人使用或商用的模式广泛传播。简单来说，你可以免费或收费复制和分发这本书，在实体或电子书等任何媒介复制这本书都不需要作者的额外授权。

由本书衍生的工作以及对本书的翻译必须遵循开放软件协议，且原始的版权声明必须保持完整。如果你对本书贡献了新的素材，你必须有能力订正你的素材和源码。请直接向文档维护者Jim Huang <jserv@ccns.ncku.edu.tw>提交维护和更新，以容许对更新内容进行合并梳理，并向Linux社区提供长久的校订。

如果你以商业的形式出版或分发了该书，作者和[Linux文档项目（LDP）](https://tldp.org/)将高度赞扬你的版费或印本等捐赠。这种方式的建设体现了你对自由软件和LDP的支持。如果你有相关疑问或建议，请联系上述地址。

## 1.1 作者

Linux内核模块编程指南最初是由Ori Pomerantz为内核版本2.2编写的。后来，Ori没空维护本书了，毕竟Linux内核发展得很快。Peter Jay Salzman接过了维护工作，并将本书更新到适配内核版本2.4。后来，到了内核版本2.6，Peter也没空跟上最新的开发进度了，于是Michael Burian成为了共同维护者，将本书更新到内核版本2.6。Bob Mottram将例子更新到了内核版本3.8+。Jim Huang将本书更新到了适配最近的内核版本（v5.x）并修订了LaTeX文档。

## 1.2 鸣谢

以下人士贡献了对本书的修正和不错的建议。

2011eric, 25077667, Arush Sharma, asas1asas200, Benno Bielmeier, Bob Lee, Brad Baker, ccs100203, Chih-Yu Chen, Ching-Hua (Vivian) Lin, ChinYikMing, Cyril Brulebois, Daniele Paolo Scarpazza, David Porter, demonsome, Dimo Velev, Ekang Monyet, fennecJ, Francois Audeon, gagachang, Gilad Reti, Horst Schirmeier, Hsin-Hsiang Peng, Ignacio Martin, JianXing Wu, linD026, lyctw, manbing, Marconi Jiang, mengxinayan, RinHizakura, Roman Lakeev, Stacy Prowell, Steven Lung, Tristan Lelong, Tucker Polomik, VxTeemo, Wei-Lun Tsai, xatier, Ylowy.

## 1.3 内核模块是什么

假设你现在想写一个内核模块。你懂C,你已经写了好些作为进程跑得起来的普通程序，而现在你想要触摸到真实运行着的部分，在这里，一个野指针就可以干掉你的文件系统、一个吐核就能导致重启。

内核模块到底是什么？模块是可以根据需要在内核中安装或卸载的代码段。它们在不需要重启的情况下扩展内核的功能。比如，有一类模块是设备驱动，允许内核访问连接到系统的硬件设备。没有内核模块的话，我们就不得不反复构建宏内核，把新的功能直接塞到内核镜像中。带来的缺点除了导致内核体积膨胀外，还有就是每次要添加点新功能，我们都不得不重新构建内核并重启。

## 1.4 内核模块软件包

Linux发行版提供了包括`modprobe`、`insmod`和`depmod`等命令的软件包。命令参考如下。

在Ubuntu和Debian上，执行

```shell
sudo apt-get install build-essential kmod
```

在Arch Linux上，执行

```shell
sudo pacman -S gcc kmod
```

## 1.5 我的内核里有哪些模块？

要查看你的内核里已经加载的模块，可以使用`lsmod`命令。

```shell
sudo lsmod
```

内核模块记录在`/proc/modules`文件中，所以你也可以这样查看内核模块：

```shell
sudo cat /proc/modules
```

输出内容可能很长，有时你只想从中找到一个特定模块。比如如果你想找`fat`模块，可以这样：

```shell
sudo lsmod | grep fat
```

## 1.6 我需要下载内核源码重新编译一遍内核吗？

如果只是跟着本指南学习，你并不需要这么做。但是，建议你在虚拟机上跑一个测试用的实例，在测试的实例上跑本书中的例子，以免稍有不慎把你自己的系统搅得一团糟。

## 1.7 开始动手之前

敲代码之前，我们先来学习一些基本知识。每个人的系统和调校习惯都不一样，所以编译跑通你的第一个“Hello World”程序有时也会出岔子。放松心情，只要你克服了开头的阻碍，后面的路途一马平川。

1. 模块版本（modversioning）问题。为一个版本的内核编译的模块，到了另一个内核上没法加载，除非编译内核时开启了`CONFIG_MODVERSIONS`选项。我们直到本书后面才会讲到模块版本问题。在开启了该选项的内核上，本书前面的例子可能没法正常跑通，然而多数流行发行版的内核都开启了这个选项。如果你因为版本错误的问题跑不通本书的例子，请将模块版本选项关掉，重新编译内核。
2. 使用桌面环境（X Window）。极其建议你在命令行终端上解压、编译和加载本书中涉及的模块，最好不要在桌面环境整。模块不像`printf`函数，没法向屏幕输出内容；不过模块可以以日志形式记录相关信息和警告，你可以通过终端开来看。如果你在xterm等图形化模拟终端用`insmod`命令加载模块，相关信息和警告是会被记录，但只会直接输出到你的systemd日志里，这样一来你只能用`journalctl`命令看到，具体细节参考[4](4_hello_world.md)。为了即时获取相关信息，请直接在字符终端上操作。
3. 安全启动（Secure Boot）。当代很多电脑出厂预设开启了UEFI安全启动特性。安全启动是一个安全标准，确保设备只能从原厂商信任的软件引导启动。一些Linux发行版的内核支持安全启动，这些发行版上，内核模块都得用密钥进行签名，否则你在跑第一个helloworld模块时就会报错`ERROR: could not insert module`。

    ```shell
    insmod ./hello-1.ko
    ```
    
    之后你用`dmesg`命令深入探究，会看到`Lockdown: insmod: unsigned module loading is restricted; see man kernel lockdown.7`

    如果你看到了这条信息，最简单的方法就是在BIOS里关闭安全启动，然后再试试加载模块。你也可以通过一通复杂的操作生成密钥，然后把密钥安装到你的系统上，再然后用密钥签名模块，最后再加载模块跑起来。然而这个操作对新手来说不太适合。如果你感兴趣的话可以参考[安全启动](https://wiki.debian.org/SecureBoot)中的步骤。
