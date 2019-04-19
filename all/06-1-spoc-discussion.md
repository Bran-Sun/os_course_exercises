# lec15: 调度算法概念 spoc 思考题


- 有"spoc"标记的题是要求拿清华学分的同学要在实体课上完成，并按时提交到学生对应的ucore_code和os_exercises的git repo上。


## 视频相关思考题

### 15.1 处理机调度概念

1. 处理机调度的功能是什么？
   * 选线程：从就绪队列中挑选下一个占用CPU运行的线程
   * 选CPU：从多个可用CPU中挑选就绪线程可使用的CPU资源

2. 在什么时候可以进行处理机调度？

   * 非抢占系统：占用CPU运行的线程的主动放弃（退出、等待）
   * 抢占系统：当前线程被抢先（时间片用完、高优先级线程就绪）

3. 当操作系统的处理机调度导致线程切换时，暂停线程的当前指令指针可能在什么位置？用户态代码或内核代码？给出理由。
   * 在线程切换时，暂停线程的指令指针一般在内核代码中。一般会在系统调用或中断时进入内核态，再在内核态进行切换。

### 15.2 调度准则

1. 处理机的使用模式有什么特征？
   * 进程在CPU计算与I/O操作之间来回交替。
   * 每次占用CPU执行指令的时间长度分布多数在10ms以内；

2. 处理机调度的目标是什么？
   * CPU利用率：CPU处于忙状态的时间百分比
   * 吞吐率：单位时间完成的进程数目
   * 周转时间：进程从初始化到结束的总时间
   * 等待时间：进程在就绪队列中的时间
   * 相应时间：从提交请求到相应的时间
   * 公平性（CPU时间分配公平性）
   * 带宽和延迟

3. 尝试在ucore上写一个外排序程序，然后分析它的执行时间分布统计（每次切换后开始执行时间和放弃CPU的时间、当前用户和内核栈信息）。

4. 在Linux上有一个应用程序time，可以统计应用程序的执行时间信息。请分析它是如何统计进程执行时间信息的。如可能，请在ucore上实现相同功能的应用程序。

   * time执行时，先进行计时，再通过fork来执行应用程序，执行完后再用times函数获得子进程的用户态时间和内核态时间，并将其输出。

   * 下面是可能的参考。

   - [Linux用户态程序计时方式详解](http://www.cnblogs.com/clover-toeic/p/3845210.html)
   - [Get Source Code for any Linux Command](http://www.thegeekstuff.com/2010/02/get-source-code-for-any-linux-command/)
   - [How does time command work](http://unix.stackexchange.com/questions/29800/how-does-time-command-work)
   - https://github.com/illumos/illumos-gate/blob/master/usr/src/cmd/time/time.c

5. 尝试获取一个操作系统的调度算法的性能统计数据（CPU使用率、进程执行）。

### 15.3 先来先服务、短进程优先和最高响应比优先调度算法

1. 尝试描述下列处理机调度算法的工作原理和算法优缺点。
   - 先来先服务算法（FCFC、FIFO）
     - 按照进程进入就绪队列的顺序排列
     - 优点：简单
     - 缺点：平均等待时间长；I/O资源和CPU资源利用效率低，当I/O密集型程序运行时，I/O被闲置
   - 短进程优先算法（SPN、SJF）
     - 就绪队列中选择执行时间最短的进程执行
     - 优点：平均周转时间最短
     - 缺点：可能导致饥饿，即长时间进程难以运行；需要预估未来。
   - 最高响应比优先算法（HRRN）
     - 选择响应比最高的进程运行，响应比R = (w + s) / s
     - 优点：基于短进程优先算法的改进，避免长时间进程一直等待

2. 为什么短进程优先算法的平均周转时间最优？

   * 可以证明知进程优先算法的调度顺序的平均周转时间是最短的。
3. 如何调试你的调度算法？
   * 监控调度算法的执行状态、调度结果等信息。

### 15.4 时间片轮转、多级反馈队列、公平共享调度算法和ucore调度框架

1. 尝试描述下列处理机调度算法的工作原理和算法优缺点。
   - 时间片轮转算法（RR）
     - 时间片结束时，按照FCFS算法选择下一个进程
     - 缺点：有额外的上下文切换开销，一般控制在1%
     - 优点：平均等待时间介于最好和最坏之间，公平，交互性比较好
   - 多级反馈队列算法（MLFQ）
     * 就绪队列被分成许多子队列，每个队列拥有自己的调度策略
     * 队列之间：时间片轮转，前台80，后台20；或者固定优先级
     * 进程可以在不同队列中移动
     * 时间片可以随优先级而变化
     * 优点：CPU密集型优先级下降很快，时间片较长；I/O密集型相反
     * 缺点：比较复杂
   - 公平共享调度算法（FSS）
     - 对用户进行分组，不重要的组无法垄断资源
     - 对于用户，如果没有使用到应该有的份额，就提高优先级
     - 优点：公平

2. RR算法选择时间片长度的依据有哪些？
   * 进程切换开销
   * 进程当前操作占用的CPU时间
   * 切换开销与响应时间的折中权衡

3. 请描述在MLFQ中，如何提升或降低进程的优先级？
   * 如果进程在当前时间片没有完成，就降到下一个优先级
   * 如果在时间片没用完就停止，就升到上一个优先级

4. 尝试跟踪ucore中进程切换和调度算法的就绪进程选择过程。

   * 中断响应、进程的中断现场保存、中断处理、调度判断、进程切换、新进程的中断现场恢复、新进程的继续执行
5. 为什么要引入调度框架？定义调度算法接口需要考虑哪些因素？
   * 引入调度框架的目的：分离调度操作和调度策略，以支持多种调度算法；
   * 定义调度算法接口的考虑因素：调度算法需要的调度操作类型、调度算法在各调度操作中的体现方式；

6. 通过观察FIFO、SJF和RR调度算法的[模拟程序](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep8-scheduler.py)运行结果及其[描述文档](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep8-scheduler.md)，理解其工作原理和算法特征。

7. 通过观察MLFQ调度算法的[模拟程序](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep9-mlfq.py)运行结果及其[描述文档](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep9-mlfq.md)，理解其工作原理和算法特征。

8. 通过观察彩票调度算法([Lottery scheduling](https://en.wikipedia.org/wiki/Lottery_scheduling))的[模拟程序](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep10-lottery.py)运行结果及其[描述文档](https://github.com/chyyuu/os_tutorial_lab/blob/master/ostep/ostep10-lottery.md)，理解其工作原理和算法特征。

### 15.5 实时调度和多处理器调度

1. 什么是实时操作系统？

   * 实时操作系统是保证在一定时间限制内完成特定功能的操作系统。
   * 正确性依赖于时间和功能两方面的操作系统
   * 指标：时间约束的及时性
   * 实时任务：资源，执行时间，截止时间

2. 什么是可调度性？

   * 可调度表示一个实时操作系统能够满足任务时限要求
   * 需要确定任务的执行时序

3. 尝试描述下列处理机调度算法的工作原理和算法优缺点。
   - 速率单调调度算法(RM, Rate Monotonic) 
     - 静态算法：通过周期安排优先级，周期越短优先级越高
   - 最早截止时间优先算法 (EDF, Earliest Deadline First) 
     - 截止时间最早的先执行。

   任务优先级定义：按周期大小排队、按截止时间先后排队

   [RM](https://en.wikipedia.org/wiki/Rate-monotonic_scheduling)的可调度条件：CPU利用率小于ln2时，是可调度的。

   [EDF](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling)的可调度条件：CPU利用率小于100%。

4. 有兴趣的同学，请阅读下面论文，然后说明实时调度面临的主要困难是什么？
   - [Buttazzo, “Rate monotonic vs. EDF: Judgement Day”, EMSOFT 2003.](http://www.eecs.umich.edu/courses/eecs571/reading/rm-vs-edf.pdf)
   - [单调速率及其扩展算法的可调度性判定](http://www.jos.org.cn/ch/reader/create_pdf.aspx?file_no=20040602)

5. 多处理机调度中每个处理机一个就绪队列与整个系统一个就绪队列有什么不同？

   * 静态进程分配：进程从开始到结束都在一个固定的处理机上执行，调度开销小，但各处理机忙闲不均
   * 动态进程分派：公共的就绪队列，调度开销大，但比较均衡

### 15.6 优先级反置

1. 什么是优先级反置现象？

   * 操作系统中出现高优先级进程长时间等待低优先级进程所占用资源的现象
   * 导致高优先级进程长时间等待
2. 什么是优先级继承(Priority Inheritance)和优先级天花板协议(priority ceiling protocol)？它们的区别是什么？
   * 优先级继承：占用资源的低优先级进程继承申请资源的高优先级进程的优先级
   * 优先级天花板协议：占用资源进程的优先级和所有可能申请该资源的进程的最高优先级相同
   * 区别：提升占用资源的低优先级进程的优先级的时机不同

3. 在单CPU情况下，基于以前的OS知识，能否设计一个更简单的方法（也许执行效率会低一些）解决优先级反置现象？
   * 结合优先级继承和优先级天花板策略：当低优先级任务C占有资源s，高优先级任务A也申请资源s时，将任务C的优先级提升到天花板，释放后再调回来。
4. ppt中“优先级继承”页里的图示有误，你能指出来吗？
   * T1转换到T3时缺少了临界区阶段。

## 小组练习与思考题

参考往届同学的处理机调度算法实现练习，从下列8个算法中选择一个你感兴趣的调度算法，对其实现进行完善，并分析算法特征。
 - [2016春季-第十五讲 课堂思考题回答-向勇班](https://piazza.com/class/i5j09fnsl7k5x0?cid=803)
 - [陈渝班-2016春季-第15讲 课堂思考题回答](https://piazza.com/class/i5j09fnsl7k5x0?cid=806)

### (1)(spoc) 理解并实现FIFO调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

请参考[scheduler-homework.py](https://github.com/chyyuu/ucore_lab/blob/master/related_info/lab6/scheduler-homework.py)代码或独自实现。
最后统计采用不同调度算法的每个任务的相关时间和总体的平均时间：

　- turnaround time　周转时间
　- response time 响应时间
　- wait time　等待时间

#### 对模拟环境的抽象
- 任务/进程，及其执行时间
  Job 0 (length = 1)
  Job 1 (length = 4)
  Job 2 (length = 7)

 - 何时切换？
 - 如何统计？

#### 执行结果

采用FIFO调度算法

```
 ./scheduler-homework.py  -p FIFO
ARG policy FIFO
ARG jobs 3
ARG maxlen 10
ARG seed 0

Here is the job list, with the run time of each job: 
  Job 0 ( length = 9 )
  Job 1 ( length = 8 )
  Job 2 ( length = 5 )


** Solutions **

Execution trace:
  [ time   0 ] Run job 0 for 9.00 secs ( DONE at 9.00 )
  [ time   9 ] Run job 1 for 8.00 secs ( DONE at 17.00 )
  [ time  17 ] Run job 2 for 5.00 secs ( DONE at 22.00 )

Final statistics:
  Job   0 -- Response: 0.00  Turnaround 9.00  Wait 0.00
  Job   1 -- Response: 9.00  Turnaround 17.00  Wait 9.00
  Job   2 -- Response: 17.00  Turnaround 22.00  Wait 17.00

  Average -- Response: 8.67  Turnaround 16.00  Wait 8.67

```
### (2)(spoc) 理解并实现SJF调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

#### 执行结果

采用SJF调度算法
```
 ./scheduler-homework.py  -p SJF
ARG policy SJF
ARG jobs 3
ARG maxlen 10
ARG seed 0

Here is the job list, with the run time of each job: 
  Job 0 ( length = 9 )
  Job 1 ( length = 8 )
  Job 2 ( length = 5 )


** Solutions **

Execution trace:
  [ time   0 ] Run job 2 for 5.00 secs ( DONE at 5.00 )
  [ time   5 ] Run job 1 for 8.00 secs ( DONE at 13.00 )
  [ time  13 ] Run job 0 for 9.00 secs ( DONE at 22.00 )

Final statistics:
  Job   2 -- Response: 0.00  Turnaround 5.00  Wait 0.00
  Job   1 -- Response: 5.00  Turnaround 13.00  Wait 5.00
  Job   0 -- Response: 13.00  Turnaround 22.00  Wait 13.00

  Average -- Response: 6.00  Turnaround 13.33  Wait 6.00
```
### (3)(spoc) 理解并实现RR调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

#### 执行结果


采用RR调度算法
```
 ./scheduler-homework.py  -p RR
ARG policy RR
ARG jobs 3
ARG maxlen 10
ARG seed 0

Here is the job list, with the run time of each job: 
  Job 0 ( length = 9 )
  Job 1 ( length = 8 )
  Job 2 ( length = 5 )


** Solutions **

Execution trace:
  [ time   0 ] Run job   0 for 1.00 secs
  [ time   1 ] Run job   1 for 1.00 secs
  [ time   2 ] Run job   2 for 1.00 secs
  [ time   3 ] Run job   0 for 1.00 secs
  [ time   4 ] Run job   1 for 1.00 secs
  [ time   5 ] Run job   2 for 1.00 secs
  [ time   6 ] Run job   0 for 1.00 secs
  [ time   7 ] Run job   1 for 1.00 secs
  [ time   8 ] Run job   2 for 1.00 secs
  [ time   9 ] Run job   0 for 1.00 secs
  [ time  10 ] Run job   1 for 1.00 secs
  [ time  11 ] Run job   2 for 1.00 secs
  [ time  12 ] Run job   0 for 1.00 secs
  [ time  13 ] Run job   1 for 1.00 secs
  [ time  14 ] Run job   2 for 1.00 secs ( DONE at 15.00 )
  [ time  15 ] Run job   0 for 1.00 secs
  [ time  16 ] Run job   1 for 1.00 secs
  [ time  17 ] Run job   0 for 1.00 secs
  [ time  18 ] Run job   1 for 1.00 secs
  [ time  19 ] Run job   0 for 1.00 secs
  [ time  20 ] Run job   1 for 1.00 secs ( DONE at 21.00 )
  [ time  21 ] Run job   0 for 1.00 secs ( DONE at 22.00 )

Final statistics:
  Job   0 -- Response: 0.00  Turnaround 22.00  Wait 13.00
  Job   1 -- Response: 1.00  Turnaround 21.00  Wait 13.00
  Job   2 -- Response: 2.00  Turnaround 15.00  Wait 10.00

  Average -- Response: 1.00  Turnaround 19.33  Wait 12.00

```

### (4)理解并实现MLFQ调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

### (5)理解并实现stride调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

### (6)理解并实现EDF实时调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

### (7)理解并实现RM实时调度算法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。

### (8)理解并实现优先级反置方法

可基于“python, ruby, C, C++，LISP、JavaScript”等语言模拟实现，并给出测试，在试验报告写出设计思路和测试结果分析。
