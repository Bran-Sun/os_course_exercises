# lec13: lab4 spoc 思考题

- 有"spoc"标记的题是要求拿清华学分的同学要在实体课上完成，并按时提交到学生对应的ucore_code和os_exercises的git repo上。

## 视频相关思考题

### 13.1 总体介绍

1. 为什么讲基本原理时先讲进程后讲线程，而在做实验时先做线程后做进程？

   * 因为在实验中先做线程会更加简单一些。不需要考虑PCB中的资源管理部分。
2. ucore的线程控制块数据结构是什么？
   * 是PCB

### 13.2 关键数据结构

1. 分析proc_struct数据结构，说明每个字段的用途，是线程控制块或进程控制块的，会在哪些函数中修改。
   * 寄存器状态、堆栈、context等信息是线程控制块的。这些信息会在切换线程(switch\_to)等函数中进行修改。
   * mm, vma等内存管理字段是进程控制块的。这些字段会在缺页中断(do\_pgfault)等函数中进行修改。

1. 如何知道ucore的两个线程同在一个进程？
   * 查看页目录起始地址是否相同。

1. context和trapframe分别在什么时候用到？

   * trapframe在中断处理时使用到
   * context在线程切换时用到；
2. 用户态或内核态下的中断处理有什么区别？在trapframe中有什么体现？
   * 在用户态中断响应时，要切换到内核态；而在内核态中断响应时，没有这种切换；
   * 当发生了特权级转换时，需要记录用户堆栈对应的地址，即esp和ss寄存器的内容。

1. 分析trapframe数据结构，说明每个字段的用途，是由硬件或软件保存的，在内核态中断响应时是否会保存。

   * eip, cs等由硬件保存；
   * esp, ss等在用户态响应时由硬件保存；在内核态的中断响应时不会被保存。

   * 通用寄存器由软件保存；

### 13.3 执行流程

1. kernel_thread创建的内核线程执行的第一条指令是什么？它是如何过渡到内核线程对应的函数的？

   * 第一条指令是kernel_thread_entry中的第一条指令， pushl %edx
   * 在中断返回时，进入kernel_thread_entry函数，在函数中将对应函数的参数起始地址压栈，并跳转到函数入口处运行。

2. 内核线程的堆栈初始化在哪？

   * setup_kstack：初始化内核堆栈
   * tf和context中的esp：初始化线程切换和中断相应后的堆栈地址

3. fork()父子进程的返回值是不同的。这在源代码中的体现中哪？
   * do_fork()函数的返回值是子进程的pid
   * 在do_fork()函数中，将子进程tf中的eax设置成0
4. 内核线程initproc的第一次执行流程是什么样的？能跟踪出来吗？
   * 在idleproc中，运行schedule函数，找到就绪的initproc时，调用proc_run函数
   * 切换页表，进入switch\_to函数
   * 回复现场后，跳转到forkret函数中
   * 调用forkrets函数
   * 通过iret返回到kernel\_thread\_entry函数
   * 将参数压栈，跳转到函数入口执行
5. 分析线程切换流程，找到内核堆栈、页表、寄存器切换的代码位置。
   * 内核堆栈在switch\_to函数中的movl 4(%eax), %esp
   * 页表切换在proc_run的lcr3(next->cr3);
   * 寄存器切换在switch\_to函数中
6. 分析C语言中调用汇编函数switch_to()的参数传递位置。
   * esp指向返回地址
   * esp上面分别指向from和to两个参数
7. 分析内核线程idleproc的创建流程，说明线程切换后执行的第一条是什么。
   * 切换回idleproc时，首先执行前一次保存的eip地址。

8. 分析内核线程initproc的创建流程，说明线程切换后执行的第一条是什么。

   * 线程切换后首先执行 proc->context.eip = forkret;
   * 在iret后，执行函数入口地址 tf.tf_eip = (uint32_t) kernel_thread_entry;

## 小组练习与思考题

(1)(spoc) 理解内核线程的生命周期。

> 需写练习报告和简单编码，完成后放到git server 对应的git repo中

### 掌握知识点
1. 内核线程的启动、运行、就绪、等待、退出
2. 内核线程的管理与简单调度
3. 内核线程的切换过程

### 练习用的[lab4 spoc exercise project source code](https://github.com/chyyuu/ucore_lab/tree/master/related_info/lab4/lab4-spoc-discuss)


请完成如下练习，完成代码填写，并形成spoc练习报告

### 1. 分析并描述创建分配进程的过程

> 注意 state、pid、cr3，context，trapframe的含义

### 练习2：分析并描述新创建的内核线程是如何分配资源的

> 注意 理解对kstack, trapframe, context等的初始化


当前进程中唯一，操作系统的整个生命周期不唯一，在get_pid中会循环使用pid，耗尽会等待

### 练习3：

阅读代码，在现有基础上再增加一个内核线程，并通过增加cprintf函数到ucore代码中
能够把内核线程的生命周期和调度动态执行过程完整地展现出来

### 练习4 （非必须，有空就做）：

增加可以睡眠的内核线程，睡眠的条件和唤醒的条件可自行设计，并给出测试用例，并在spoc练习报告中给出设计实现说明

### 扩展练习1: 

进一步裁剪本练习中的代码，比如去掉页表的管理，只保留段机制，中断，内核线程切换，print功能。看看代码规模会小到什么程度。

### 思考：

switch_to函数的汇编码与编译器生成的C函数的汇编码区别是什么？能否用C语言实现switch_to函数？为什么？
