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

下面是一系列的系统调用

- fork: 创建新进程（child process 复制 parent process 的内存内容，但用不同的空间）<br>
    两次返回? in parent, fork returns the child's pid; in the child, it returns zero （**看不懂**)
- exit: 结束当前进程，释放资源
- wait: 等待一个子进程结束并返回pid
- exec: 将进程的内存用 文件系统里的一个文件加载的 新内存映射代替，因此这个文件得符合特定的format操作系统才能识别（才executable）
