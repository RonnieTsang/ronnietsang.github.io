---
layout: post
title: "计算机基础(一): 操作系统篇 -- 进程与线程的比较"
date: 2014-10-06 07:13:41 +0800
comments: true
categories: "OperatingSystem"
---

## 一、概述

### （1）进程和线程的比较

**我们通常看到的说法是这样的**：

按照教科书上的定义，**进程是资源管理的最小单位，线程是程序执行的最小单位**。

**进程可以认为是程序执行时的一个实例**。进程是系统进行资源分配的独立实体，且每个进程拥有独立的地址空间。一个进程无法直接访问另一个进程的变量和数据结构，如果希望让一个进程访问另一个进程的资源，需要使用进程间通信，比如：管道，文件， 套接字等。从内核的观点看，进程的目的就是担当分配系统资源（CPU时间、内存等）的基本单位。

**一个进程可以拥有多个线程，每个线程使用其所属进程的栈空间。** 线程与进程的一个主要区别是，同一进程内的多个线程会共享部分状态，多个线程可以读写同一块内存(一个进程无法直接访问另一进程的内存)。同时，每个线程还拥有自己的寄存器和栈，其它线程可以读写这些栈内存。

**线程是进程的一个特定执行路径。**当一个线程修改了进程中的资源， 它的兄弟线程可以立即看到这种变化。线程是CPU调度和分派的基本单位，它是比进程更小的能独立运行的基本单位。

**总的来说，进程是资源分配的最小单位，线程则是程序执行的最小单位。**

<!-- more -->

更详细的分析，包括：

> * 进程是系统进行资源分配的基本单位，有独立的内存地址空间； 线程是CPU调度的基本单位，没有单独地址空间，有独立的栈，局部变量，寄存器，程序计数器等
> * 创建进程的开销大，包括创建虚拟地址空间等需要大量系统资源；创建线程开销小，基本上只有一个内核对象和一个堆栈
> * 一个进程无法直接访问另一个进程的资源；同一进程内的多个线程共享进程的资源
> * 进程切换开销大；线程切换开销小
> * 进程间通信开销大；线程间通信开销小
> * 线程属于进程，不能独立执行。每个进程至少要有一个线程，成为主线程

### （2）多进程和多线程的选择

> * 多进程的程序要比多线程的程序**健壮**，线程虽然有自己的堆栈和局部变量，但线程没有单独的地址空间，一个线程死掉就等于整个进程死掉。但在进程切换时，耗费资源较大，**效率**要差一些，因此：
> * 当我们需要一种相对比较**节俭**的多任务操作方式的情况下，多线程提供了很不错的解决方案，因为同一个进程的多个线程彼此之间使用相同的地址空间，共享大部分数据，启动一个线程所花费的空间远远小于启动一个进程所花费的**空间**，而且，线程间彼此切换所需的时间也远远小于进程间切换所需要的**时间**
> * 多线程的另一个明显的优点是线程间方便的**通信**机制。对不同进程来说，由于它们具有独立的数据空间，要进行数据的传递只能通过通信的方式进行，这不仅费时且很不方便。线程则不然，由于同一进程下的线程之间共享数据空间，所以一个线程的数据可以直接为其它线程所用，快捷又方便。（当然，数据的共享也带来其他一些问题，有的变量不能同时被两个线程所修改，有的子程序中声明为static的数据更有可能给多线程程序带来灾难性的打击，这些正是编写多线程程序时最需要注意的地方。）
> * 对于一些要求同时进行并且又要**共享**某些变量的**并发**操作，只能用多线程，不能用多进程

正如当年采用多进程模型代替单进程模型一样，多线程程序使设计更简洁、功能更完备，程序的执行效率也更高。除了以上所说的优点外，不和进程比较，多线程程序作为一种多任务、并发的工作方式，有以下的优点：

> * 提高应用程序响应。这对图形界面的程序尤其有意义，当一个操作耗时很长时，整个系统都会等待这个操作，此时程序不会响应键盘、鼠标、菜单的操作，而使用多线程技术，将耗时长的操作（time consuming）置于一个新的线程，可以避免这种尴尬的情况。
> * 使多CPU系统更加有效。操作系统会保证当线程数不大于CPU数目时，不同的线程运行于不同的CPU上。
> * 改善程序结构。一个既长又复杂的进程可以考虑分为多个线程，成为几个独立或半独立的运行部分，这样的程序会利于理解和修改。

从函数调用上来说，进程创建使用`fork()`操作；线程创建使用`clone()`操作。 [Richard Stevens][1]大师这样说过：

>  **fork is expensive.** Memory is copied from the parent to the child， all descriptors are duplicated in the child， and so on. Current implementations use a technique called `copy-on-write`， which avoids a copy of the parent's data space to the child until the child needs its own copy. But， regardless of this optimization， fork is expensive.

> **IPC is required to pass information between the parent and child after the fork.** Passing information from the parent to the child before the fork is easy， since the child starts with a copy of the parent's data space and with a copy of all the parent's descriptors. But， returning information from the child to the parent takes more work.

> **Threads help with both problems.** Threads are sometimes called `lightweight processes` since a thread is "lighter weight" than a process. That is， thread creation can be 10–100 times faster than process creation.

> All threads within a process share the same global memory. This makes the sharing of information easy between the threads， but along with this `simplicity` comes the problem of `synchronization`.

### （3）从操作系统角度看

我们知道，操作系统有两个主要用途：

> * 防止硬件被失控的应用程序滥用
> * 在控制复杂而又通常广泛不同的低级硬件设备方面，为应用程序提供简单一致的方法。

操作系统通过以下几个不同层级的抽象概念实现这两个用途：

> * **文件：对I/O设备的抽象；**
> * **虚拟存储器：对I/O设备、主存储器的抽象**；
> * **进程：对I/O设备、主存储器和处理器的抽象。**

这几个概念是**相当基础而且重要**的，进程是操作系统对运行中的程序的一种抽象。在一个系统中可以同时运行多个进程，而每个进程都好像在独占地使用硬件。我们称之为`并发运行`，实际上是说一个进程的指令和另一个进程的指令是交错执行的，操作系统实现这种交错执行的机制称为`上下文切换(context switching)`。

而在现代系统中，一个进程实际上可以由多个称为线程的执行单元组成，每个线程都运行在进程的上下文中，并共享同样的代码和全局数据。这在前文已经有过说明了。

虚拟存储器，则为每个进程提供了一个假象，好像每个进程都在独占地使用主存。每个进程看到的存储器都是一致的，称之`为虚拟地址空间`。

`（关于虚拟存储器，我会找时间专门整理一篇详尽的文章进行总结，希望不会等太久）`

最后，文件只不过就是`字节序列`。每个I/O设备都可以看成是文件，系统中的所有输入输出都是通过使用称为Unix I/O的一小组系统函数调用读写文件来实现的。文件这个简单而精致的概念是非常强大的，因为它使得应用程序能够统一地看待系统中可能含有的各种I/O设备。

----------

## 二、线程的实现方式

在上一篇文章中，我们已经对进程的状态及其转换有所了解，这里我们会对线程做进一步的认识，关于`线程的实现方式`，或者说，`用户线程与内核线程的区别`。

线程的实现可以分为两类：`用户级线程` (User-Level Thread)和 `内核线线程` (Kernel-Level Thread)，后者又称为内核支持的线程或轻量级进程。**分类的标准主要是线程的调度者在核内还是在核外**。**前者更利于并发使用多处理器的资源，而后者则更多考虑的是上下文切换开销。**（在操作系统设计上，从进程演化出线程，最主要的目的就是**更好的支持 [SMP][2] 以及减小（进程/线程）上下文切换开销**。）

**用户线程**指不需要内核支持而在用户程序中实现的线程，其不依赖于操作系统核心，应用进程利用线程库提供创建、同步、调度和管理线程的函数来控制用户线程。不需要用户态/核心态切换，速度快，操作系统内核不知道多线程的存在，因此一个线程阻塞将使得整个进程（包括它的所有线程）阻塞。由于这里的处理器时间片分配是以进程为基本单位，所以每个线程执行的时间相对减少。

**内核线程**由操作系统内核创建和撤销。内核维护进程及线程的上下文信息以及线程切换。一个内核线程由于I/O操作而阻塞，不会影响其它线程的运行。

以下是用户级线程和内核级线程的区别：
> * 内核支持线程是OS内核可感知的，而用户级线程是OS内核不可感知的。
> * 用户级线程的创建、撤消和调度不需要OS内核的支持，是在语言（如Java）这一级处理的；而内核支持线程的创建、撤消和调度都需OS内核提供支持，而且与进程的创建、撤消和调度大体是相同的。
> * 用户级线程执行系统调用指令时将导致其所属进程被中断，而内核支持线程执行系统调用指令时，只导致该线程被中断。
> * 在只有用户级线程的系统内，CPU调度还是以进程为单位，处于运行状态的进程中的多个线程，由用户程序控制线程的轮换运行；在有内核支持线程的系统内，CPU调度则以线程为单位，由OS的线程调度程序负责线程的调度。
> * 用户级线程的程序实体是运行在用户态下的程序，而内核支持线程的程序实体则是可以运行在任何状态下的程序。

在目前的商用系统中，通常都将两者结合起来使用，既提供**核心线程**以满足 SMP 系统的需要，也支持用**线程库**的方式在用户态实现另一套线程机制，此时一个核心线程同时成为多个用户态线程的调度者。正如很多技术一样，"混合"通常都能带来更高的效率，但同时也带来更大的实现难度，出于"简单"的设计思路，Linux从一开始就没有实现混合模型的计划，但它在实现上采用了另一种思路的"混合"。

在线程机制的具体实现上，可以在操作系统内核上实现线程，也可以在核外实现，后者显然要求核内至少实现了进程，而前者则一般要求在核内同时也支持进程。核心级线程模型显然要求前者的支持，而用户级线程模型则不一定基于后者实现。这种差异，正如前所述，是两种分类方式的标准不同带来的。

当核内既支持进程也支持线程时，就可以实现线程-进程的"多对多"模型，即一个进程的某个线程由核内调度，而同时它也可以作为用户级线程池的调度者，选择合适的用户级线程在其空间中运行。这就是前面提到的"混合"线程模型，既可满足多处理机系统的需要，也可以最大限度的减小调度开销。绝大多数商业操作系统（如Digital Unix、Solaris、Irix）都采用的这种能够完全实现 [POSIX1003.1c][3] 标准的线程模型。

在核外实现的线程又可以分为"一对一"、"多对一"两种模型，前者用一个核心进程（也许是轻量进程）对应一个线程，将线程调度等同于进程调度，交给核心完成，而后者则完全在核外实现多线程，调度也在用户态完成。后者就是前面提到的单纯的用户级线程模型的实现方式，显然，这种核外的线程调度器实际上只需要完成线程运行栈的切换，调度开销非常小，但同时因为核心信号（无论是同步的还是异步的）都是以进程为单位的，因而无法定位到线程，所以这种实现方式不能用于多处理器系统，而这个需求正变得越来越大，因此，在现实中，纯用户级线程的实现，除算法研究目的以外，几乎已经消失了。

Linux内核只提供了轻量进程的支持，限制了更高效的线程模型的实现，但Linux着重优化了进程的调度开销，一定程度上也弥补了这一缺陷。目前最流行的线程机制 [LinuxThreads][4] 所采用的就是线程-进程"一对一"模型，调度交给核心，而在用户级实现一个包括信号处理在内的线程管理机制。

在 Linux2.6 以前，pthread 线程库对应的实现就是这个名叫 LinuxThreads 的 lib，由 [Xavier Leroy][5] 负责开发完成，并已绑定在 GLIBC 中发行。它所实现的就是基于核心轻量级进程的"一对一"线程模型，一个线程实体对应一个核心轻量级进程，而线程之间的管理在核外函数库中实现。

### **LinuxThread的线程机制**

"一对一"模型的好处之一是线程的调度由核心完成了，而其他诸如线程取消、线程间的同步等工作，都是在核外线程库中完成的。在LinuxThreads中，专门为每一个进程构造了一个管理线程，负责处理线程相关的管理工作。当进程第一次调用pthread_create()创建一个线程的时候就会创建（__clone()）并启动管理线程。

在一个进程空间内，管理线程与其他线程之间通过一对"管理管道（manager_pipe[2]）"来通讯，该管道在创建管理线程之前创建，在成功启动了管理线程之后，管理管道的读端和写端分别赋给两个全局变量__pthread_manager_reader和__pthread_manager_request，之后，每个用户线程都通过__pthread_manager_request向管理线程发请求，但管理线程本身并没有直接使用__pthread_manager_reader，管道的读端（manager_pipe[0]）是作为__clone()的参数之一传给管理线程的，管理线程的工作主要就是监听管道读端，并对从中取出的请求作出反应。

使用 pthread 以后，在用户看来，每一个 task_struct 就对应一个线程，而一组线程以及它们所共同引用的一组资源就是一个进程。

但是，一组线程并不仅仅是引用同一组资源就够了，它们还必须被视为一个整体。对此，POSIX 标准提出了如下要求：

 1. 查看进程列表的时候，相关的一组 task_struct 应当被展现为列表中的一个节点
 2. 发送给这个"进程"的信号(对应 kill 系统调用)，将被对应的这一组 task_struct 所共享，并且被其中的任意一个"线程"处理
 3. 发送给某个"线程"的信号(对应 pthread_kill)，将只被对应的一个 task_struct 接收，并且由它自己来处理
 4. 当"进程"被停止或继续时(对应 SIGSTOP/SIGCONT 信号)，对应的这一组 task_struct 状态将改变 
 5. 当"进程"收到一个致命信号(比如由于段错误收到 SIGSEGV 信号)，对应的这一组 task_struct 将全部退出
 6. 等等(以上可能不够全)

LinuxThreads 利用前面提到的轻量级进程来实现线程，但是对于 POSIX 提出的那些要求，LinuxThreads 除了第5点以外，都没有实现(实际上是无能为力，至于原因，后文谈到 LinuxThreads 的不足时会解释)： 

> * 如果运行了A程序，A程序创建了10个线程，那么在shell下执行ps命令时将看到11个A进程，而不是1个(注意，也不是10个，下面会解释)
> * 不管是kill还是pthread_kill，信号只能被一个对应的线程所接收
> * SIGSTOP/SIGCONT信号只对一个线程起作用

还好 LinuxThreads 实现了第5点，我认为这一点是最重要的。如果某个线程"挂"了，整个进程还在若无其事地运行着，可能会出现很多的不一致状态。进程将不是一个整体，而线程也不能称为线程。

或许这也是为什么 LinuxThreads 虽然与 POSIX 的要求差距甚远，却能够存在，并且还被使用了好几年的原因吧。

但是， LinuxThreads 为了实现这个"第5点"，还是付出了很多代价，并且创造了 LinuxThreads 本身的一大性能瓶颈。

接下来要说说，为什么A程序创建了10个线程，但是ps时却会出现11个A进程了。因为 LinuxThreads 自动创建了一个管理线程，上面提到的"第5点"就是靠管理线程来实现的。

当程序开始运行时，并没有管理线程存在(因为尽管程序已经链接了 pthread 库，但是未必会使用多线程)。程序第一次调用 pthread_create 时， LinuxThreads 发现管理线程不存在，于是创建这个管理线程。这个管理线程是进程中的第一个线程(主线程)的儿子。

然后在 pthread_create 中，会通过前面提到的管道向管理线程发送一个命令，告诉它创建线程。即是说，除主线程外，所有的线程都是由管理线程来创建的，管理线程是它们的父亲。

于是，当任何一个子线程退出时，管理线程将收到 SIGUSER1 信号(这是在通过 clone 创建子线程时指定的)。管理线程在对应的 sig_handler 中会判断子线程是否正常退出，如果不是，则杀死所有线程，然后自杀。

那么，主线程怎么办呢？主线程是管理线程的父亲，其退出时并不会给管理线程发信号。于是，在管理线程的主循环中通过 getppid 检查父进程的ID号，如果ID号是1，说明父亲已经退出，并把自己托管给了 init 进程(1号进程)。这时候，管理线程也会杀掉所有子线程，然后自杀。

可见，线程的创建与销毁都是通过管理线程来完成的，于是管理线程就成了 LinuxThreads 的一个性能瓶颈。创建与销毁需要一次进程间通信，一次上下文切换之后才能被管理线程执行，并且多个请求会被管理线程串行地执行。

#### LinuxThreads的不足

由于Linux内核的限制以及实现难度等等原因，LinuxThreads并不是完全POSIX兼容的，在它的发行README中有说明。

（1）**进程id问题**

这个不足是最关键的不足，引起的原因牵涉到 LinuxThreads 的"一对一"模型。Linux内核并不支持真正意义上的线程，LinuxThreads 是用与普通进程具有同样内核调度视图的轻量级进程来实现线程支持的。这些轻量级进程拥有独立的进程id，在进程调度、信号处理、IO等方面享有与普通进程一样的能力。

在源码阅读者看来，就是Linux内核的 `clone()` 没有实现对 `CLONE_PID` 参数的支持。在内核 `do_fork()` 中对 `CLONE_PID` 的处理是这样的：

```c
          if (clone_flags & CLONE_PID) {
                if (current->pid)
                        goto fork_out;
        }
```

这段代码表明，目前的Linux内核仅在pid为0的时候认可 CLONE_PID 参数，实际上，仅在 SMP 初始化，手工创建进程的时候才会使用 CLONE_PID 参数。

按照 POSIX 定义，同一进程的所有线程应该共享一个进程id和父进程id，这在目前的"一对一"模型下是无法实现的。

（2）**信号处理问题**

由于异步信号是内核以进程为单位分发的，而 LinuxThreads 的每个线程对内核来说都是一个进程，且没有实现"线程组"，因此，某些语义不符合 POSIX 标准，比如没有实现向进程中所有线程发送信号，README对此作了说明。

如果核心不提供实时信号，LinuxThreads 将使用 SIGUSR1 和 SIGUSR2 作为内部使用的 restart 和 cancel 信号，这样应用程序就不能使用这两个原本为用户保留的信号了。在Linux kernel 2.1.60以后的版本都支持扩展的实时信号（从 _SIGRTMIN 到 _SIGRTMAX），因此不存在这个问题。

某些信号的缺省动作难以在现行体系上实现，比如 SIGSTOP 和 SIGCONT，LinuxThreads 只能将一个线程挂起，而无法挂起整个进程。

（3）**线程总数问题**

LinuxThreads 将每个进程的线程最大数目定义为1024，但实际上这个数值还受到整个系统的总进程数限制，这又是由于线程其实是核心进程。

在kernel 2.4.x中，采用一套全新的总进程数计算方法，使得总进程数基本上仅受限于物理内存的大小，计算公式在kernel/fork.c的fork_init()函数中：

```c
    max_threads = mempages / (THREAD_SIZE/PAGE_SIZE) / 8
```

在i386上，THREAD_SIZE=2*PAGE_SIZE，PAGE_SIZE=2^12（4KB），mempages=物理内存大小/PAGE_SIZE，对于256M的内存的机器，mempages=256*2^20/2^12=256*2^8，此时最大线程数为4096。

但为了保证每个用户（除了root）的进程总数不至于占用一半以上物理内存，fork_init()中继续指定：

```c
    init_task.rlim[RLIMIT_NPROC].rlim_cur = max_threads/2;
    init_task.rlim[RLIMIT_NPROC].rlim_max = max_threads/2;
```

这些进程数目的检查都在do_fork()中进行，因此，对于LinuxThreads来说，线程总数同时受这三个因素的限制。

（4）**管理线程问题**

管理线程容易成为瓶颈，这是这种结构的通病；同时，管理线程又负责用户线程的清理工作，因此，尽管管理线程已经屏蔽了大部分的信号，但一旦管理线程死亡，用户线程就不得不手工清理了，而且用户线程并不知道管理线程的状态，之后的线程创建等请求将无人处理。

（5）**同步问题**

LinuxThreads 中的线程同步很大程度上是建立在信号基础上的，这种通过内核复杂的信号处理机制的同步方式，效率一直是个问题。

（6）**其他POSIX兼容性问题**

Linux中很多系统调用，按照语义都是与进程相关的，比如nice、setuid、setrlimit等，在目前的 LinuxThreads 中，这些调用都仅仅影响调用者线程。

（7）**实时性问题**

线程的引入有一定的实时性考虑，但 LinuxThreads 暂时不支持，比如调度选项，目前还没有实现。不仅 LinuxThreads 如此，标准的Linux在实时性上考虑都很少。


### **其他的线程机制**

LinuxThreads 的问题，特别是兼容性上的问题，严重阻碍了Linux上的跨平台应用（如Apache）采用多线程设计，从而使得Linux上的线程应用一直保持在比较低的水平。在Linux社区中，已经有很多人在为改进线程性能而努力，其中既包括用户级线程库，也包括核心级和用户级配合改进的线程库。

很显然，为了改进 LinuxThread 必须得到内核的支持，并且需要重写线程库。为了实现这个需求，起先最为人看好的有两个项目，一个是 RedHat 公司牵头研发的 **[NPTL][6]**（Native Posix Thread Library），另一个则是 IBM 投资开发的 **NGPT**（Next Generation Posix Threading），二者都是围绕完全兼容 POSIX 1003.1c，同时在核内和核外做工作以而实现多对多线程模型。这两种模型都在一定程度上弥补了 LinuxThreads 的缺点，且都是重起炉灶全新设计的。

在2003年的年中，IBM放弃了NGTP，也就是大约那时，Redhat发布了最初的NPTL。

NPTL最开始在redhat linux 9里发布，现在从RHEL3起内核2.6起都支持NPTL，并且完全成了GNU C库的一部分。

#### NPTL

NPTL实现了前面提到的POSIX的全部5点要求。但是，实际上，与其说是NPTL实现了，不如说是Linux内核实现了。

在Linux2.6中，内核有了线程组的概念，`task_struct` 结构中增加了一个 `tgid`(`threadgroupid`) 字段。如果这个task是一个"主线程"，则它的 `tgid` 等于 `pid`，否则 `tgid` 等于进程的 `pid` (即主线程的 `pid`)。在 `clone` 系统调用中，传递 `CLONE_THREAD` 参数就可以把新进程的 `tgid` 设置为父进程的 `tgid` (否则新进程的 `tgid` 会设为其自身的 `pid`)。

类似的 XXid 在 `task_struct` 中还有两个：`task->signal->pgid` 保存进程组的打头进程的 pid、`task->signal->session` 保存会话打头进程的 pid。通过这两个 id 来关联进程组和会话。

有了 tgid，内核或相关的 shell 程序就知道某个 tast_struct 是代表一个进程还是代表一个线程，也就知道在什么时候该展现它们，什么时候不该展现(比如在 ps 的时候，线程就不要展现了)。而 getpid(获取进程ID)系统调用返回的也是 tast_struct 中的 tgid，而 tast_struct 中的 pid 则由 gettid 系统调用来返回。

在执行ps命令的时候不展现子线程，也是有一些问题的。比如程序a.out运行时，创建了一个线程。假设主线程的pid是10001、子线程是10002（它们的tgid都是10001）。这时如果你kill10002，是可以把10001和10002这两个线程一起杀死的，尽管执行ps命令的时候根本看不到10002这个进程。如果你不知道Linux线程背后的故事，肯定会觉得遇到灵异事件了。

为了应付"发送给进程的信号"和"发送给线程的信号"，task_struc t里面维护了两套 `signal_pending`，一套是线程组共享的，一套是线程独有的。通过kill发送的信号被放在线程组共享的 signal_pending 中，可以由任意一个线程来处理；通过 `pthread_kill` 发送的信号( pthread_kill 是 pthread 库的接口，对应的系统调用 中tkill )被放在线程独有的 signal_pending 中，只能由本线程来处理。当线程停止/继续，或者是收到一个致命信号时，内核会将处理动作施加到整个线程组中。

NPTL的设计目标归纳可归纳为以下几点：

> * POSIX兼容性
> * SMP结构的利用
> * 低启动开销
> * 低链接开销（即不使用线程的程序不应当受线程库的影响）
> * 与LinuxThreads应用的二进制兼容性
> * 软硬件的可扩展能力
> * 多体系结构支持
> * NUMA支持
> * 与C++集成

在技术实现上，NPTL仍然采用1:1的线程模型，并配合glibc和最新的Linux Kernel2.5.x开发版在信号处理、线程同步、存储管理等多方面进行了优化。和LinuxThreads不同，NPTL没有使用管理线程，核心线程的管理直接放在核内进行，这也带了性能的优化。
主要是因为核心的问题，NPTL仍然不是100%POSIX兼容的，但就性能而言相对LinuxThreads已经有很大程度上的改进了。

除NPTL的1*1模型外还有一个m*n模型（NGPT就是），通常这种模型的用户线程数会比内核的调度实体多。在这种实现里，线程库本身必须去处理可能存在的调度，这样在线程库内部的上下文切换通常都会相当的快，因为它避免了系统调用转到内核态。然而这种模型增加了线程实现的复杂性,并可能出现诸如优先级反转的问题，此外，用户态的调度如何跟内核态的调度进行协调也是很难让人满意。

----------

## 写在后面

这篇文章东拼西凑了很多资料，其中有很多地方并不是很理解，权当做一个粗浅的概览。通过这篇文章的整理，才发现自己以前对进程和线程的理解是多么的丢人，现在呢也只能算对其基本概念能有初步的认识，以后有需要的时候再作深入的学习。

其他一些参考，有一些还没看的，也先一起记录下来了：

 - [Linux线程浅析][7]
 - [Linux多线程编程][8]
 - [Linux线程实现机制分析][9]
 - [通用线程：POSIX线程详解，第1部分][10]
 - [通用线程：POSIX线程详解，第2部分][11]
 - [通用线程：POSIX线程详解，第3部分][12]
 - [Linux 线程模型的比较：LinuxThreads 和 NPTL][13]
 - [Linux 多线程应用中如何编写安全的信号处理函数][14]
 - [Posix线程编程指南(1)][15]
 - [Posix线程编程指南(2)][16]
 - [Posix线程编程指南(3)][17]
 - [Posix线程编程指南(4)][18]
 - [Posix线程编程指南(5)][19]


  [1]: http://www.kohala.com/start/
  [2]: http://en.wikipedia.org/wiki/Symmetric_multiprocessing
  [3]: http://en.wikipedia.org/wiki/POSIX_Threads
  [4]: http://en.wikipedia.org/wiki/LinuxThreads
  [5]: http://en.wikipedia.org/wiki/Xavier_Leroy
  [6]: http://en.wikipedia.org/wiki/Native_POSIX_Thread_Library
  [7]: http://wenku.baidu.com/link?url=MFJwqO1I6UdonzUo4OAFiIU4YYjJkXFrQRH3l4w0HIzjk_s1idNts4Cwhbj10T5UF_pAescBWXB9pQmsG9oSpLGpJWHvdB0f-UATkviQebmwiki/LinuxThreads
  [8]: http://blog.sina.com.cn/s/blog_69708ebe0100sqm0.html
  [9]: http://www.ibm.com/developerworks/cn/linux/kernel/l-thread/
  [10]: http://www.ibm.com/developerworks/cn/linux/thread/posix_thread1/
  [11]: http://www.ibm.com/developerworks/cn/linux/thread/posix_thread2/
  [12]: http://www.ibm.com/developerworks/cn/linux/thread/posix_thread3/
  [13]: http://www.ibm.com/developerworks/cn/linux/l-threading.html
  [14]: http://www.ibm.com/developerworks/cn/linux/l-cn-signalsec/
  [15]: http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part1/
  [16]: http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part2/
  [17]: http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part3/
  [18]: http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part4/
  [19]: http://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part5/
