---
layout: post
title: "计算机基础(一): 操作系统篇 -- 理解 Linux 进程状态及其转换"
date: 2014-10-06 06:45:59 +0800
comments: true
categories: OperatingSystem
---

## 一、进程基本状态

不同的操作系统对进程的状态解释不同，但是最基本的状态都是一样的。包括以下三种：

> * **执行态**（Running）：进程占用CPU，并在CPU上运行；
> * **就绪态**（Ready）：进程已经具备运行条件，但是CPU还没有分配过来；
> * **阻塞态**（Blocked）：进程因等待某件事发生而暂时不能运行

<!-- more -->

进程在一生中，都处于上述3中状态之一。

理论上上述三种状态之间转换分为六种情况：

> * **执行->就绪**：这是由调度引起的，主要是由于分配给进程CPU时间片用完
> * **就绪->执行**：运行中的进程的时间片用完，调度就转到就绪队列中选择合适的进程（一般是优先级最高的进程）分配CPU
> * **执行->阻塞**：正在执行的进程，由于等待某个事件发生而无法执行时，便放弃CPU而处于阻塞状态，例如等待I/O完成、申请缓冲区不能满足、等待信号等
> * **阻塞->就绪**：处于阻塞状态的进程，若其等待的事件已经发生，于是进程由阻塞状态转变为就绪状态。

还有另外两种不会出现的状态转换：

> * **阻塞->执行**：即使给阻塞进程分配CPU，也无法执行，操作系统在进行调度时不会从阻塞队列进行挑选，其调度的选择对象为就绪队列
> * **就绪->阻塞**：因为就绪态根本就没有执行，何来进入阻塞态？

----------

## 二、Linux进程状态

Linux是一个多用户，多任务的系统，可以同时运行多个用户的多个程序，就必然会产生很多的进程，为了对进程从产生到消亡的整个过程进行跟踪和描述，就需要定义进程的各种状态并制定相应的状态转换策略，以此来控制进程的运行。

Linux进程状态包括以下几种：

### 1、**可执行态** R (TASK_RUNNING)

这是执行态和就绪态的合并，表示进程正在运行或准备运行，Linux 中使用 `TASK_RUNNING` 宏表示此状态。

只有在该状态的进程才可能在CPU上运行。而同一时刻可能有多个进程处于可执行状态，这些进程的 task_struct 结构（进程控制块）被放入对应CPU的可执行队列中（一个进程最多只能出现在一个CPU的可执行队列中）。进程调度器的任务就是从各个CPU的可执行队列中分别选择一个进程在该CPU上运行。

很多操作系统教科书将正在CPU上执行的进程定义为RUNNING状态、而将可执行但是尚未被调度执行的进程定义为READY状态，这两种状态在linux下统一为 `TASK_RUNNING` 状态。

### 2、**浅度睡眠态** S (TASK_INTERRUPTIBLE)

又称 `可中断的睡眠状态`，进程正在睡眠（被阻塞），等待资源到来是唤醒，也可以通过其他进程信号或时钟中断唤醒，进入运行队列。Linux 使用 `TASK_INTERRUPTIBLE` 宏表示此状态。

处于这个状态的进程因为等待某某事件的发生（比如等待socket连接、等待信号量），而被挂起。这些进程的 task_struct 结构被放入对应事件的等待队列中。当这些事件发生时（由外部中断触发、或由其他进程触发），对应的等待队列中的一个或多个进程将被唤醒。

通过 `ps` 命令我们会看到，一般情况下，进程列表中的绝大多数进程都处于 TASK_INTERRUPTIBLE 状态（除非机器的负载很高）。毕竟CPU就这么几个，进程动辄几十上百个，如果不是绝大多数进程都在睡眠，CPU又怎么响应得过来。

### 3、**深度睡眠态** D (TASK_UNINTERRUPTIBLE)

又称 `不可中断的睡眠状态`，其和浅度睡眠基本类似，但有一点就是不可被其他进程信号或时钟中断唤醒。Linux 使用 `TASK_UNINTERRUPTIBLE` 宏表示此状态。

与 `TASK_INTERRUPTIBLE` 状态类似，进程处于睡眠状态，但是此刻进程是不可中断的。`不可中断，指的并不是CPU不响应外部硬件的中断，而是指进程不响应异步信号`。

> * 绝大多数情况下，进程处在睡眠状态时，总是应该能够响应异步信号的。否则你将惊奇的发现，kill -9 竟然杀不死一个正在睡眠的进程了！ 于是我们也很好理解，为什么 ps 命令看到的进程几乎不会出现 `TASK_UNINTERRUPTIBLE` 状态，而总是 `TASK_INTERRUPTIBLE` 状态。
> * 而 `TASK_UNINTERRUPTIBLE` 状态存在的意义就在于，内核的某些处理流程是不能被打断的。如果响应异步信号，程序的执行流程中就会被插入一段用于处理异步信号的流程（这个插入的流程可能只存在于内核态，也可能延伸到用户态），于是原有的流程就被中断了。（参见[《Linux内核中断内幕][1]》）
> * 在进程对某些硬件进行操作时（比如进程调用 read 系统调用对某个设备文件进行读操作，而 read 系统调用最终执行到对应设备驱动的代码，并与对应的物理设备进行交互），可能需要使用 `TASK_UNINTERRUPTIBLE` 状态对进程进行保护，以避免进程与设备交互的过程被打断，造成设备陷入不可控的状态。这种情况下的 `TASK_UNINTERRUPTIBLE` 状态总是非常短暂的，通过 ps 命令基本上不可能捕捉到。

Linux系统中也存在容易捕捉的 TASK_UNINTERRUPTIBLE 状态。执行 vfork 系统调用后，父进程将进入 TASK_UNINTERRUPTIBLE 状态，直到子进程调用 exit 或 exec （参见[《神奇的vfork》][2]）。

通过下面的代码就能得到处于 `TASK_UNINTERRUPTIBLE` 状态的进程：

```c
#include<unistd.h>
int main() {
    if ( !vfork() )
        sleep(100);
}
```

编译运行，然后 ps 一下：

```sh
$ ps -ax | grep a.out
4371 pts/0    D+     0:00 ./a.out
4372 pts/0    S+     0:00 ./a.out
4374 pts/1    S+     0:00 grep a.out
```

然后我们可以试验一下 `TASK_UNINTERRUPTIBLE` 状态的威力。不管kill还是kill -9，这个 `TASK_UNINTERRUPTIBLE` 状态的父进程依然屹立不倒。

### 4、**暂停态** T (TASK_STOPPED or TASK_TRACED)

又称暂停状态或跟踪状态，进程暂停执行接受某种处理。如正在接受调试的进程处于这种状态，Linux 使用 `TASK_STOPPED` 宏表示此状态。

向进程发送一个 `SIGSTOP` 信号，它就会因响应该信号而进入 TASK_STOPPED 状态（除非该进程本身处于 TASK_UNINTERRUPTIBLE 状态而不响应信号）。（ SIGSTOP 与 SIGKILL 信号一样，是非常强制的。不允许用户进程通 过signal 系列的系统调用重新设置对应的信号处理函数。）

向进程发送一个 `SIGCONT` 信号，可以让其从 TASK_STOPPED 状态恢复到 TASK_RUNNING 状态。

当进程正在被跟踪时，它处于 TASK_TRACED 这个特殊的状态。“正在被跟踪”指的是进程暂停下来，等待跟踪它的进程对它进行操作。比如在 gdb 中对被跟踪的进程下一个断点，进程在断点处停下来的时候就处于 TASK_TRACED 状态。而在其他时候，被跟踪的进程还是处于前面提到的那些状态。

对于进程本身来说，TASK_STOPPED 和 TASK_TRACED 状态很类似，都是表示进程暂停下来。而 TASK_TRACED 状态相当于在 TASK_STOPPED 之上多了一层保护，处于 TASK_TRACED 状态的进程不能响应 SIGCONT 信号而被唤醒。只能等到调试进程通过 ptrace 系统调用执行 PTRACE_CONT、PTRACE_DETACH 等操作（通过 ptrace 系统调用的参数指定操作），或调试进程退出，被调试的进程才能恢复 TASK_RUNNING 状态。

### 5、**退出状态-僵死态** Z (TASK_DEAD – EXIT_ZOMBIE)

又称退出状态，进程已经结束但未释放PCB，进程成为僵尸进程，Linux 使用 `EXIT_ZOMBIE` 宏表示此状态。

进程在退出的过程中，处于 TASK_DEAD 状态。

在这个退出过程中，进程占有的所有资源将被回收，除了 task_struct 结构（以及少数资源）以外。于是进程就只剩下 task_struct 这么个空壳，故称为僵尸。

之所以保留 task_struct，是 **因为** task_struct 里面保存了进程的退出码、以及一些统计信息。而其父进程很可能会关心这些信息。比如在 shell 中，`$?` 变量就保存了最后一个退出的前台进程的退出码，而这个退出码往往被作为 if 语句的判断条件。当然，内核也可以将这些信息保存在别的地方，而将 task_struct 结构释放掉，以节省一些空间。但是使用 task_struct 结构更为方便，因为在内核中已经建立了从 pid 到 task_struct 的查找关系，还有进程间的父子关系。释放掉 task_struct，则需要建立一些新的数据结构，以便让父进程找到它的子进程的退出信息。

父进程可以通过 wait 系列的系统调用（如 wait4、waitid ）来等待某个或某些子进程的退出，并获取它的退出信息。然后 wait 系列的系统调用会顺便将子进程的尸体（task_struct）也释放掉。子进程在退出的过程中，内核会给其父进程发送一个信号，通知父进程来“收尸”。这个信号默认是 SIGCHLD ，但是在通过 clone 系统调用创建子进程时，可以设置这个信号。

通过下面的代码能够制造一个 EXIT_ZOMBIE 状态的进程：

```c
#include<unistd.h>
int main() {
    if ( fork() )
        while ( 1 ) sleep(100);
}
```

编译运行，然后 ps 一下：

```sh
$ ps -ax | grep a.out
10410 pts/0    S+     0:00 ./a.out
10411 pts/0    Z+     0:00 [a.out]
10413 pts/1    S+     0:00 grep a.out
```

只要父进程不退出，这个僵尸状态的子进程就一直存在。那么如果父进程退出了呢，谁又来给子进程“收尸”？

当进程退出的时候，会将它的所有子进程都托管给别的进程（使之成为别的进程的子进程）。托管给谁呢？可能是退出进程所在进程组的下一个进程（如果存在的话），或者是1号进程。所以每个进程、每时每刻都有父进程存在。除非它是1号进程。

1号进程，pid为1的进程，又称 `init 进程`：

Linux 系统启动后，第一个被创建的用户态进程就是 init 进程。它有两项使命：
（1）执行系统初始化脚本，创建一系列的进程（它们都是 init 进程的子孙）；
（2）在一个死循环中等待其子进程的退出事件，并调用 waitid 系统调用来完成“收尸”工作；

init 进程不会被暂停、也不会被杀死（这是由内核来保证的）。它在等待子进程退出的过程中处于 TASK_INTERRUPTIBLE 状态，“收尸”过程中则处于 TASK_RUNNING 状态。

### 6、**退出状态** X (TASK_DEAD – EXIT_DEAD)

而进程在退出过程中也可能 `不会保留` 它的 task_struct 。比如这个进程是多线程程序中被 detach 过的进程（进程？线程？参见[《linux线程浅析》][3]）。或者父进程通过设置 SIGCHLD 信号的 handler 为 SIG_IGN，显式的忽略了 SIGCHLD 信号。（这是 posix 的规定，尽管子进程的退出信号可以被设置为 SIGCHLD 以外的其他信号。）此时，进程将被置于 EXIT_DEAD 退出状态，这意味着接下来的代码立即就会将该进程彻底释放。所以 EXIT_DEAD 状态是非常短暂的，几乎不可能通过 ps 命令捕捉到。

----------

## 三、**Linux 进程的初始状态**

进程是通过 fork 系列的系统调用（fork、clone、vfork）来创建的，内核（或内核模块）也可以通过 kernel_thread 函数创建内核进程。

这些创建子进程的函数本质上都完成了相同的功能——将调用进程复制一份，得到子进程。（可以通过选项参数来决定各种资源是共享、还是私有。）

那么既然调用进程处于 TASK_RUNNING 状态（否则，它若不是正在运行，又怎么进行调用？），则子进程默认也处于 TASK_RUNNING 状态。

另外，在系统调用调用 clone 和内核函数 kernel_thread 也接受 CLONE_STOPPED 选项，从而将子进程的初始状态置为 TASK_STOPPED 。

----------

## 四、**Linux 进程状态变迁**

进程自创建以后，状态可能发生一系列的变化，直到进程退出。而尽管进程状态有好几种，但是进程状态的变迁却只有两个方向：从 `TASK_RUNNING状态` 变为 `非TASK_RUNNING状态` 、或者从 `非TASK_RUNNING状态` 变为 `TASK_RUNNING状态`。

也就是说，如果给一个 TASK_INTERRUPTIBLE 状态的进程发送 SIGKILL 信号，这个进程将先被唤醒（进入TASK_RUNNING状态），然后再响应 SIGKILL 信号而退出（变为TASK_DEAD状态）。并不会从 TASK_INTERRUPTIBLE 状态直接退出。

进程从非 TASK_RUNNING 状态变为 TASK_RUNNING 状态，是由别的进程（也可能是中断处理程序）执行唤醒操作来实现的。执行唤醒的进程设置被唤醒进程的状态为 TASK_RUNNING ，然后将其 task_struct 结构加入到某个CPU的可执行队列中。于是被唤醒的进程将有机会被调度执行。

而进程从 TASK_RUNNING 状态变为非 TASK_RUNNING 状态，则有两种途径：
> * 响应信号而进入 TASK_STOPED 状态、或 TASK_DEAD 状态；
> * 执行系统调用主动进入 TASK_INTERRUPTIBLE 状态（如 nanosleep 系统调用）、或 TASK_DEAD 状态（如 exit 系统调用）；或由于执行系统调用需要的资源得不到满足，而进入 TASK_INTERRUPTIBLE 状态或 TASK_UNINTERRUPTIBLE 状态（如 select 系统调用）。

显然，这两种情况都只能发生在进程正在CPU上执行的情况下。

以上关于Linux进程及其状态转换的描述，可参考下图的总结：

{% imgcap /images/computer-science-basic-1-operating-system-process-state-1.jpg Linux进程状态及其转换 %}

----------

Reference:

  1、http://blog.csdn.net/huzia/article/details/18946491
  2、http://blog.chinaunix.net/uid-26126915-id-2948970.html

  [1]: http://www.ibm.com/developerworks/cn/linux/l-cn-linuxkernelint/
  [2]: http://blog.sina.com.cn/s/blog_5eb8ebcb0100rp56.html
  [3]: http://wenku.baidu.com/link?url=fMmb8NKSH9tolCwBh50f4ocE-bmlMuo0_J5_-V9_djN1sdm0n2XzQcf8zkUF0IBNeGHMwM9KUHBCXOs0qldPOqce9ycaYFuiCLFdSP4B7vi
