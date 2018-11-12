# Lab 1: Booting a PC

[链接](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/)


本实验分为三个部分：
1. 熟悉x86, QEMU x86模拟器, PC开机引导流程
2. 研究 JOS 的 boot loader(引导加载程序), 位于实验文件夹的```boot```目录
3. 研究 JOS 的 template, 位于实验文件夹的```kernel```目录 

# Part 0: 实验环境搭建

> 工欲善其事，必先利其器

Lab所需环境由三部分组成 - JOS模板仓库，gcc编译器， qemu模拟器；gcc编译器是许多Linux Distribution自带的（我用的是Ubuntu 18.04 LTS），因此只介绍 JOS模板仓库 和 qemu 的安装过程即可

## JOS Git

很简单: ```git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab```

## QEMU

[官网](https://www.qemu.org/)<br>QEMU是一个PC模拟器（可以在上面搭建虚拟机）；但QEMU的调试功能不足以进行实验，因此 6.828 为它加入了调试功能，下面是安装步骤：
1. ```git clone https://github.com/mit-pdos/6.828-qemu.git qemu```
2. 在Linux环境下，可能得安装一些libraries:  libsdl1.2-dev, libtool-bin, libglib2.0-dev, libz-dev, and libpixman-1-dev. ```sudo apt-get install xxx```
3. 然后进到仓库目录，进行一些配置<br>
Linux: ```./configure --disable-kvm --disable-werror [--prefix=PFX] [--target-list="i386-softmmu x86_64-softmmu"]```<br>其中 ```prefix```声明安装qemu的位置，如果不声明会安装在 ```/usr/local```; ```target-list``` 缩减了qemu所支持的架构（为qemu瘦身）
4. 进行编译和安装 ```make```, ```make install```
# Part 1: PC Bootstrap

## x86学习

首先需要学习 x86 汇编 - [PC Assembly Language Book](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)<br>
**注意**：上面这本书里是 NASM assembler(Intel syntax)，而JOS用的是GNU assembler(AT&T syntax)；[这里](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)阐述了他们的区别，读一下这里面的 "The Syntax" 章节; 另外，详细的工具书保存到了主目录 ```资料.md```

## x86模拟器 - QEMU

虽然实验用的是模拟器，但要**代码能在模拟器上跑，就能在真机上跑**；使用模拟器更好**调试**

6.828使用 [QEMU 模拟器](https://www.qemu.org/)，并配合 [GDB](http://www.gnu.org/software/gdb/) 进行Debug（QEMU内置调试功能不完善），下面的实验就马上用到GDB来研究开机流程

首先来编译前面git clone下来的jos（现在还只是模板）。输入```make```会编译产生最简版本的jos引导加载程序 以及 jos内核。如果出现错误```undefined reference to `__udivdi3'```, 是因为没有 32位的gcc multilib, ```sudo apt-get install gcc-multilib```可破

> 回忆一下开机流程：BIOS启动后，从MBR里抽取出引导加载程序(boot loader) 然后boot loader载入内核程序开始运行操作系统；boot loader 也可以放在其他存储设备的 boot section 内，由MBR转向它来实现多系统启动的功能

make生成的```kernel.img```就是一块虚拟硬盘，包含了boot loader和kernel(可以假设boot loader放在kernel.img的boot section内)

接下来就可以启动JOS了（\\(≧▽≦)/）; 实验里介绍的方式是从命令行里运行，而我研究出了通过直接运行qemu的方法。实验方法是```make qemu```；我的方法（假设刚才qemu安装在了/usr/local）：```/usr/local/bin/qemu-system-i386 obj/kern/kernel.img```. 

**注意**：使用make qemu会导致VGA和命令行同时输出；同样的，make qemu也会同时接受VGA和命令行的输入；使用 ```make qemu-nox```甚至不会有VGA输出，便于ssh连接。退出qemu使用 ```ctrl+a x```

然后jos就跑起来了(有时快乐就是这么简单)，注意点击屏幕会失去鼠标控制，```Ctrl+Alt```可以恢复。截图留念：
![](jos_boot.png)

'Booting from Hard Disk...' 之后的内容就是 JOS内核 (前面可看作BIOS)；K> 是个提示符，由kernel的```monitor```模块产生。现在只有两个指令：```help``` 和 ```kerninfo```

## PC的物理地址空间

此处来研究一下PC启动和物理地址空间的关系。该图是32位的

> 和物理地址空间相对应的是逻辑地址空间，现在概念还比较朦胧

![](physical_address_space.png)

- 8088 处理器只能处理 1MB 物理地址，因此早期PC只能使用 Low Memory 部分作为 RAM；
- 早期PC 将 BIOS 存放在 BIOS ROM 中，而现在是存放在 updatable flash memory 中；BIOS负责设备初始化并从存储设备中导入boot loader进行开机程序
- 80286-16MB; 80386-4GB; 但**仍保留低地址的架构以向下兼容**。因此现在PC的物理地址空间被分割为两部分：low memory (头640KB) 和 extended memory (>1MB部分) 
- 32-bit 机器在RAM之后（也即地址的最末端）还由BIOS分配了一些空间给 PCI设备（百度了一下类似于扩展内存槽的东西）
- x86 处理器现在又能支持高于4GB的物理地址空间，因此RAM会被分割为三块
- **JOS只使用PC物理地址空间的 头256MB**，因此实验只讨论32位地址空间

## 探究 ROM BIOS

在这一部分，我们将使用 QEMU 的调试设备探究 兼容 ```IA-32``` 的PC是如何启动的。

启动两个终端并cd到jos目录。在一个终端输入```make qemu-gdb```(或者```make qemu-nox-gdb```)，于是 QEMU 启动并开始等待GDB的连接；在另一个终端，输入```make gdb```

> 在make gdb这一步骤可能会有小错误，但我没遇到就没记录下来，可以在[原文件](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/)对应部分查看

首先可以看到第一条输出:
```
[f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b
```

> 由于对x86汇编知识的缺乏，这一部分暂且跳过。后面的内容就是使用gdb的 ```si```(step) 指令一步一步地追踪BIOS的运作过程；这一篇文章说的比较详细，日后可以参考：[博客](http://www.cnblogs.com/fatsheep9146/p/5078179.html)

# Part2：The Boot Loader 引导加载程序

