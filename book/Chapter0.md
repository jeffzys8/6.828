# Chapter 0 - OS interfaces

- xv6: 一个简单的 Unix OS
- kernel: 一个为运行程序提供服务的特殊程序
- running program = progress : has memory(instructions, data, stack)
- 进程进行系统调用，指示kernel进行操作（由user space跳转到了kernel space）
- 在user space跑的进程只能访问自己的内存 (kernel privilege)
- shell is a user program, not part of the kernel

下图显示xv6实现的unix系统调用：<br>
![](chap0-1.png)

## processes and memory

- Xv6 can time-share processes 分时系统
- pid: process identifier 进程标识符，kernel分配

**下面是一系列的系统调用**

- ```fork```: 创建新进程（child process 复制 parent process 的内存内容，但用不同的空间）
    - 两次返回? in parent, fork returns the child's pid; in the child, it returns zero （**看不懂**)
- ```exit```: 结束当前进程，释放资源
- ```wait```: 等待一个子进程结束并返回pid
- ```exec```: 将进程的内存用 文件系统里的一个文件加载的 新内存映射代替，因此这个文件得符合特定的format操作系统才能识别（executable）<br>
    - xv6 识别 ELF格式, Chapter2讨论
    - exec(filename, args[])
    - 通常第一个参数是文件名，会被忽略

**shell整体的流程**：

1. 循环读取指令```getcmd```
2. 读取到指令后```fork```，获得一份shell process的拷贝
3. parent(也即shell process) 调用```wait```，等待child(刚才fork得的拷贝)执行指令```runcmd```
4. 对于类似```echo hello```的指令，需要读取可执行文件echo，此时会调用```exec```执行
5. exec调用```exit```，parent(shell process)结束wait等待；

- fork 和 exec 不同时作为一个 SysCall的原因会在之后说
- user-space 内存是隐式分配的：
    - fork 分配的空间刚好够复制shell process
    - exec 分配的空间刚好够承载executable file
    - 运行时需更多空间? ```sbrk(n)```, n表示增长n bytes
- xv6 没有多用户的概念，无需设计沙箱，都作为root

## I/O and File descriptors