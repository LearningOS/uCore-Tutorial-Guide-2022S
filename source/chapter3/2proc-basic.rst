进程基础结构
================================

本节导读
--------------------------

本节会介绍进程的调度方式。这是本章的重点之一。

进程的概念
---------------------------------

导语中提到了，进程就是运行的程序。既然是程序，那么它就需要程序执行的一切资源，包括栈、寄存器等等。不同于用户线程，用户进程有着自己独立的用户栈和内核栈。但是无论如何寄存器是只有一套的，因此进程切换时对于寄存器的保存以及恢复是我们需要关心的问题。

为了研究进程的切换，我们先来搞懂用户进程长啥样，是如何运行的。不妨从上一节的 run_all_app 函数开始研究：

.. code-block:: c

    int run_all_app()
    {
        for (int i = 0; i < app_num; ++i) {
            struct proc *p = allocproc();
            struct trapframe *trapframe = p->trapframe;
            load_app(i, app_info_ptr);
            uint64 entry = BASE_ADDRESS + i * MAX_APP_SIZE;
            trapframe->epc = entry;
            trapframe->sp = (uint64)p->ustack + USER_STACK_SIZE;
            p->state = RUNNABLE;
        }
        return 0;
    }

首先介绍 struct proc 的定义。本章中新增的proc.h定义了我们OS的进程的PCB（进程管理块，和进程一一对应。它包含了进程几乎所有的信息）结构体；

.. code-block:: C
    :linenos:

    // os/proc.h

    struct proc {
        enum procstate state;   // 进程状态
        int pid;                // 进程ID
        uint64 ustack;          // 进程用户栈虚拟地址(用户页表)
        uint64 kstack;          // 进程内核栈虚拟地址(内核页表)
        struct trapframe *trapframe;   // 进程中断帧
        struct context context; // 用于保存进程内核态的寄存器信息，进程切换时使用
    };

    enum procstate { 
        UNUSED,     // 未初始化
        USED,       // 基本初始化，未加载用户程序
        SLEEPING,   // 休眠状态(未使用，留待后续拓展)
        RUNNABLE,   // 可运行
        RUNNING,    // 当前正在运行
        ZOMBIE,     // 已经 exit
    };

可以看到每一个进程的PCB都保存了它当前的状态以及它的PID（每个进程的PID不同）。同时记录了其用户栈和内核栈的起始地址。trapframe和context在异常中断的切换以及进程之间的切换起到了保存的重要作用。

进程的状态是大家比较熟悉的问题了。OS课程上将进程的状态分为创建、就绪、执行、等待以及结束5大阶段（未来还会有挂起）。在我们的OS之中对状态的分类略有不同。我们一般用RUNNABLE代表就绪的进程，RUNNING代表正在执行的进程，UNUSED代表池中未分配或已经结束的进程，USED代表已经分配好但是还未加载完毕的进程。

进程的基本管理
---------------------------------

在我们的OS之中，我们采用了非常朴素的进程池方式来存放进程：

.. code-block:: C
    :linenos:
    
    // os/trap.c

    struct proc pool[NPROC];    // 全局进程池
    struct proc idle;           // boot 进程
    struct proc* current_proc;  // 指示当前进程

    // 由于还有没内存管理机制，静态分配一些进程资源
    char kstack[NPROC][PAGE_SIZE];
    __attribute__((aligned(4096))) char ustack[NPROC][PAGE_SIZE];
    __attribute__((aligned(4096))) char trapframe[NPROC][PAGE_SIZE];


可以看到我们最多同时有 NPROC 个进程，每一个进程的用户栈、内核栈以及trapframe所需的空间已经预先分配好了。当然缺点是进程池空间有限，不过直到lab8 之前大家都无需担心这个问题。

这里的 idle 进程是我们的 boot 进程，是我们执行初始化的进程，事实上，在引入用户进程前，idle 是唯一一个进程。比较重要的是 current_proc，它代表着当前正在执行的进程。因此这个变量在进程切换时也需要维护来保证其正确性。活用此变量能大大方便我们的编程。

进程模块初始化函数如下:

.. code-block:: C

    // kernel/trap.c

    void procinit()
    {
        struct proc *p;
        for(p = pool; p < &pool[NPROC]; p++) {
            p->state = UNUSED;
            p->kstack = (uint64)kstack[p - pool];
            p->ustack = (uint64)ustack[p - pool];
            p->trapframe = (struct trapframe*)trapframe[p - pool];
        }
        idle.kstack = (uint64)boot_stack_top;
        idle.pid = 0;
    }

进程的分配
---------------------------------

回到 run_all_app 函数，可以注意到首每个用户进程都被分配了一个 proc 结构，通过 alloc_proc 函数。进程的分配实际上本质就是从进程池中挑选一个还未使用（状态为UNUSED）的位置分配给进程。具体代码如下:

.. code-block:: C
    :linenos:
    
    // os/proc.c

    // Look in the process table for an UNUSED proc.
    // If found, initialize state required to run in the kernel.
    // If there are no free procs, or a memory allocation fails, return 0.
    struct proc *allocproc()
    {
        struct proc *p;
        for (p = pool; p < &pool[NPROC]; p++) {
            if (p->state == UNUSED) {
                goto found;
            }
        }
        return 0;

    found:
        p->pid = allocpid();
        p->state = USED;
        memset(&p->context, 0, sizeof(p->context));
        memset(p->trapframe, 0, PAGE_SIZE);
        memset((void *)p->kstack, 0, PAGE_SIZE);
        p->context.ra = (uint64)usertrapret;
        p->context.sp = p->kstack + PAGE_SIZE;
        return p;
    }

分配进程需要初始化其PID以及清空其栈空间，并设置 context 第一次运行的入口地址 usertrapret，使得进程能够从内核的S态返回U态并执行自己的代码。我们需要看看进程切换相关的东西了。