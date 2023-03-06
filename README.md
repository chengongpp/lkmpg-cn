# Linux内核模块编程指南（内核版本5.x）中文翻译

本项目为基于内核版本5.x编写的新版 *The Linux Kernel Module Programming Guide*《Linux内核模块编程指南》的中文翻译，目前尚在建设中。原项目地址为[lkmpg](https://github.com/sysprog21/lkmpg)，遵循OSL-3.0协议释出，如有转发请标注原书作者，如有二次加工编辑请遵循原书OSL-3.0协议保持开源等特性。

原书作者：`Peter Jay Salzman`, `Michael Burian`, `Ori Pomerantz`, `Bob Mottram`, `Jim Huang`

参与翻译工作：[`Predmet Chen`](mailto:chengongpp@bupt.cn)

## 使用方式

本书使用[mdBook](https://github.com/rust-lang/mdBook)作为构建系统，可以使用以下命令进行构建：

```bash
mdbook build
```

如欲对本书进行实时修改调试，可以使用以下命令：

```bash
mdbook serve
```

## 内容说明

本书相较于英文原书有以下更改：

- 进行了中文翻译，但较少参考旧版本的中文翻译；
- 在尽量不影响描述准确性的前提下，小幅度修改语句，照顾中文读者阅读习惯；
- 采用mdBook编写，方便阅读和维护；

## 翻译进度

- [x] 1 Introduction
- [x] 2 Headers
- [x] 3 Examples
- [ ] 4 Hello World
- [ ] 5 Preliminaries
- [ ] 6 Character Device drivers
- [ ] 7 The /proc File System
- [ ] 8 sysfs: Interacting with your module
- [ ] 9 Talking To Device Files
- [ ] 10 System Calls
- [ ] 11 Blocking Processes and threads
- [ ] 12 Avoiding Collisions and Deadlocks
- [ ] 13 Replacing Print Macros
- [ ] 14 Scheduling Tasks
- [ ] 15 Interrupt Handlers
- [ ] 16 Crypto
- [ ] 17 Virtual Input Device Driver
- [ ] 18 Standardizing the interfaces: The Device Model
- [ ] 19 Optimizations
- [ ] 20 Common Pitfalls
- [ ] 21 Where To Go From Here?