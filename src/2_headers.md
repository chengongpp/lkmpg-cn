# 头文件

构建程序之前，你首先需要安装与你用的内核相匹配的内核头文件。

在Ubuntu和Debian上，执行：

```shell
sudo apt-get update 
apt-cache search linux-headers-`uname -r`
```

这条命令会告诉你有哪些内核头文件包可以安装。然后参考你的内核版本，比如：

```shell
# 注意这里的版本号可能跟你用的不一样
sudo apt-get install kmod linux-headers-5.4.0-80-generic
```

在Arch Linux上，执行：

```shell
sudo pacman -S linux-headers
```
