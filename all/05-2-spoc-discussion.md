#lec12: 进程／线程控制spoc练习

## 视频相关思考题
### 12.1 进程切换

1. 进程切换的可能时机有哪些?
   * 时间片使用完
   * 进程结束
   * 被高优先级进程抢占
   * 进入等待状态
2. 分析ucore的进程切换代码，说明ucore的进程切换触发时机和进程切换的判断时机都有哪些。
   * ucore的进程切换函数switch_to函数是由schedule()来调用的
   * 当一个进程结束时，在do_exit()函数中，会出现进程切换
   * 当父进程调用wait()函数时，如果还有子进程在运行，就会进程切换
   * 在idle_proc中，也会判断是否有可执行的进程，并切换进程
   * 在trap函数中，应该是和时钟中断配合进行进程切换
   * 在加锁的时候，也会有进程切换
3. ucore的进程控制块数据结构是如何组织的？主要字段分别表示什么？
   * mm_struct: 内存管理模块
   * need_schedule: 是否需要被调度
   * cr3: 页目录表地址
   * kstack：内核堆栈地址
   * parent: 父进程
   * cptr, yptr, optr: 进程间的联系
   * list_link, hash_link: 进程间的链表

### 12.2 进程创建

1. fork()的返回值是唯一的吗？父进程和子进程的返回值是不同的。请找到相应的赋值代码。
   * 在fork函数中，ret = proc->pid; 父进程的返回值为子进程的pid
   * 在fork函数的拷贝中，proc->tf->tf_regs.reg_eax = 0; 设置了子进程返回时，返回值为0
2. 新进程创建时的进程标识是如何设置的？请指明相关代码。
   * 在get_pid函数中，找到一个没被使用的pid返回
3. 请通过fork()的例子中进程标识的赋值顺序说明进程的执行顺序。
   * 在课件ppt中，主进程fork()完第一个进程后，继续循环fork()完第二个和第三个，然后切换到第一个子进程，进行相同操作，依次进行。
4. 请在ucore启动时显示空闲进程（idleproc）和初始进程（initproc）的进程标识。
   * idleproc的pid为0，initproc的pid为1
5. 请在ucore启动时显示空闲线程（idleproc）和初始进程(initproc)的进程控制块中的“pde_t *pgdir”的内容。它们是否一致？为什么？
   * 都是boot_cr3，是相同的。
   * 因为这两个线程都是内核线程，在ucore中内核线程共享一个地址空间。

### 12.3 进程加载

1. 加载进程后，新进程进入就绪状态，它开始执行时的第一条指令的位置，在elf中保存在什么地方？在加载后，保存在什么地方？
   * 在elf的entry中
   * 加载后，放在PCB的trapframe中
2. 第一个用户进程执行的代码在哪里？它是什么时候加载到内存中的？
   * 在lab5中，把用户进程放在了ucore后面，在加载os时一起加载到了内存中。
   * 后面运行用户程序时，在do_execve函数中调用lode_icode时加载到相应位置

### 12.4 进程等待与退出

1. 试分析wait()和exit()的结果放在什么地方？exit()是在什么时候放进去的？wait()在什么地方取到出的？
   * wait()的结果放在sys_wait()的参数中，exit()的结果放在current->exit\_code中，在do_exit函数中放进去。wait()在遍历子进程时，会得到子进程的exit_code()。
2. 试分析ucore操作系统内核是如何把子进程exit()的返回值传递给父进程wait()的？
   * 子进程在结束时，将返回值放在exit\_code中，并唤醒父进程。父进程在wait()函数中，取出子进程的exit\_code
3. 什么是僵尸进程和孤儿进程？
   * 僵尸进程：子进程退出，但还未被回收
   * 孤儿进程：子进程还在运行，但父进程已经结束
4. 试分析sleep()系统调用的实现。在什么地方设置的定时器？它对应的等待队列是哪个？它的唤醒操作在什么地方？
   * 在系统调用中，在外设中设置定时器。
   * 对应的等待队列应该是某个IO设备的等待队列
   * 唤醒操作应该在该外设的中断处
5. 通常的函数调用和函数返回都是一一对应的。有不是一一对应的例外情况？如果有，请举例说明。
   * 比如调用switch_to函数时，可能就切换到另一个进程中，就不一定对应了

## 小组思考题

(1) (spoc)设计一个简化的进程管理子系统，可以管理并调度支持“就绪”和“等待”状态的简化进程。给出了[参考代码](https://github.com/chyyuu/ucore_lab/blob/master/related_info/lab5/process-cpuio-homework.py)，请理解代码，并完成＂YOUR CODE"部分的内容．　可２个人一组

### 进程的状态 
```
 - RUNNING - 进程正在使用CPU
 - READY   - 进程可使用CPU
 - WAIT    - 进程等待I/O完成
 - DONE    - 进程结束
```

### 进程的行为
```
 - 使用CPU, 
 - 发出YIELD请求,放弃使用CPU
 - 发出I/O操作请求,放弃使用CPU
```

### 进程调度
 - 使用FIFO/FCFS：先来先服务, 只有进程done, yield, io时才会执行切换
   - 先查找位于proc_info队列的curr_proc元素(当前进程)之后的进程(curr_proc+1..end)是否处于READY态，
   - 再查找位于proc_info队列的curr_proc元素(当前进程)之前的进程(begin..curr_proc-1)是否处于READY态
   - 如都没有，继续执行curr_proc直到结束

### 关键模拟变量
 - io_length : IO操作的执行时间
 - 进程控制块
```
PROC_CODE = 'code_'
PROC_PC = 'pc_'
PROC_ID = 'pid_'
PROC_STATE = 'proc_state_'
```
 - 当前进程 curr_proc 
 - 进程列表：proc_info是就绪进程的队列（list），
 - 在命令行（如下所示）需要说明每进程的行为特征：（１）使用CPU ;(2)等待I/O
```
   -l PROCESS_LIST, --processlist= X1:Y1,X2:Y2,...
   X 是进程的执行指令数; 
   Ｙ是执行yield指令（进程放弃CPU,进入READY状态）的比例(0..100) 
   Ｚ是执行I/O请求指令（进程放弃CPU,进入WAIT状态）的比例(0..100)
```
 - 进程切换行为：系统决定何时(when)切换进程:进程结束或进程发出yield请求

### 进程执行
```
instruction_to_execute = self.proc_info[self.curr_proc][PROC_CODE].pop(0)
```

### 关键函数
 - 系统执行过程：run
 - 执行状态切换函数:　move_to_ready/running/done　
 - 调度函数：next_proc

### 执行实例

#### 例1
```
$./process-simulation.py  -l 5:30:30,5:40:30 -c
Produce a trace of what would happen when you run these processes:
Process 0
  io
  io
  yld
  cpu
  yld

Process 1
  yld
  io
  yld
  yld
  yld

Important behaviors:
  System will switch when the current process is FINISHED or ISSUES AN YIELD or IO
Time     PID: 0     PID: 1        CPU        IOs 
  1      RUN:io      READY          1            
  2     WAITING    RUN:yld          1          1 
  3     WAITING     RUN:io          1          1 
  4     WAITING    WAITING                     2 
  5     WAITING    WAITING                     2 
  6*     RUN:io    WAITING          1          1 
  7     WAITING    WAITING                     2 
  8*    WAITING    RUN:yld          1          1 
  9     WAITING    RUN:yld          1          1 
 10     WAITING    RUN:yld          1          1 
 11*    RUN:yld       DONE          1            
 12     RUN:cpu       DONE          1            
 13     RUN:yld       DONE          1            
```
