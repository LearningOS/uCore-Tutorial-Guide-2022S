进程和进程切换
================================

本节导读
--------------------------

本节会介绍进程的调度方式。这是本章的重点之一。

进程的概念
---------------------------------

导语中提到了，进程就是运行的程序。既然是程序，那么它就需要程序执行的一切资源，包括栈、寄存器等等。不同于用户线程，用户进程有着自己独立的用户栈和内核栈。但是无论如何寄存器是只有一套的，因此进程切换时对于寄存器的保存以及恢复是我们需要关心的问题。

本章中新增的proc.h定义了我们OS的进程的PCB（进程管理块，和进程一一对应。它包含了进程几乎所有的信息。）结构体；

.. code-block:: C
    :linenos:

    // kernel/trap.h
    struct proc {
        enum procstate state;   // 进程状态
        int pid;                // 进程ID
        uint64 ustack;
        uint64 kstack;
        struct trapframe *trapframe; 
        struct context context; // 用于保存进程内核态的寄存器信息，进程切换时使用
    };

可以看到每一个进程的PCB都保存了它当前的状态以及它的PID（每个进程的PID不同）。同时记录了其用户栈和内核栈的起始地址。trapframe和context在异常中断的切换以及进程之间的切换起到了保存的重要作用。

进程的状态是大家比较熟悉的问题了。OS课程上将进程的状态分为创建、就绪、执行、等待以及结束5大阶段（未来还会有挂起）。在我们的OS之中对状态的分类略有不同。我们一般用RUNNABLE代表就绪的进程，RUNNING代表正在执行的进程，UNUSED代表池中未分配或已经结束的进程，USED代表已经分配好但是还未加载完毕可以就绪执行的进程。

OS的进程结构
---------------------------------

在我们的OS之中，我们采用了非常朴素的进程池方式来存放进程：

.. code-block:: C
    :linenos:
    
    // kernel/trap.c
    struct proc pool[NPROC];
    struct proc idle;           // boot proc，始终存在
    struct proc* current_proc;  // 指示当前进程

    char kstack[NPROC][PAGE_SIZE];
    char ustack[NPROC][PAGE_SIZE];
    char trapframe[NPROC][PAGE_SIZE];
    extern char boot_stack_top[];   // bootstack，用作 idle proc kernel stack

可以看到我们最多同时有NPROC个进程，并且进程池的下标对应着进程的PID。这样使得我们操作进程十分方便。每一个进程的用户栈、内核栈以及trapframe所需的空间已经预先分配好了。当然缺点是进程池空间有限，不过直到lab8之前大家都无需担心这个问题。

比较重要的是current_proc，它代表着当前正在执行的进程。因此这个变量在进程切换时也需要维护来保证其正确性。活用此变量能大大方便我们的编程。

进程的分配
---------------------------------

进程的分配实际上本质就是从进程池中挑选一个还未使用（状态为UNUSED）的位置分配给进程。具体代码如下:

.. code-block:: C
    :linenos:
    
    // kernel/proc.c
    struct proc* allocproc(void)
    {
        struct proc *p;
        for(p = pool; p < &pool[NPROC]; p++) {
            if(p->state == UNUSED) {
                goto found;
            }
        }
        return 0;

        found:
        p->pid = allocpid();    // 分配一个没有被使用过的 id
        p->state = USED;        // 标记该控制块被使用
        memset(&p->context, 0, sizeof(p->context));
        memset(p->trapframe, 0, PAGE_SIZE);
        memset((void*)p->kstack, 0, PAGE_SIZE);
        // 初始化第一次运行的上下文信息
        p->context.ra = (uint64)usertrapret;    
        p->context.sp = p->kstack + PAGE_SIZE;
        return p;
    }

分配进程需要初始化其PID以及清空其栈空间，并设置第一次运行的上下文信息为usertrapret，使得进程能够从内核的S态返回U态并执行自己的代码。还记得为什么第一次返回就能回到测例的开始处吗？可以回顾一下run_all_app函数是如何处理的。

进程的切换
---------------------------------

下面我们介绍本章的最最最重要的进程切换（调度）问题。

进程的切换？

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

说到切换，大家肯定对第二章中异常产生后，从U态切换到S态的流程历历在目。实际上，进程的切换和它十分类似。大家可以类比一下，第二章我们是从进程1--->内核态处理异常--->进程1。那我们完全可以把整个流程转换为进程1-->内核切换-->进程2-->切换-->进程1来实现执行流的切换，并且保证中途的保存和恢复不出错呀！当然这么做会比较复杂，我们的处理方式更加复古一点，但是思路基本是一样的。回顾一下进程的PCB结构体中两个用于切换的结构体成员：

.. code-block:: C
    :linenos:

    struct trapframe *trapframe; 
    struct context context; 

trapframe大家在第二章已经和它打过交到了。那么context这个结构体又记录了什么呢?

.. code-block:: C
    :linenos:

    // kernel/trap.h

    // Saved registers for kernel context switches.
    struct context {
        uint64 ra;
        uint64 sp;
        // callee-saved
        uint64 s0;
        uint64 s1;
        uint64 s2;
        uint64 s3;
        uint64 s4;
        uint64 s5;
        uint64 s6;
        uint64 s7;
        uint64 s8;
        uint64 s9;
        uint64 s10;
        uint64 s11;
    };

它相比trapframe，只记录了寄存器的信息。聪明的你可能已经发现它们都是被调用者保存的寄存器。在切换的核心函数swtch（注意拼写）之中，就是对这个结构体进行了操作:

.. code-block:: riscv
    :linenos:

    # Context switch
    #
    #   void swtch(struct context *old, struct context *new);
    #
    # Save current registers in old. Load from new.

    .globl swtch

    # a0 = &old_context, a1 = &new_context

    swtch:
        sd ra, 0(a0)        # save `ra`
        sd sp, 8(a0)        # save `sp`
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)        # restore `ra`
        ld sp, 8(a1)        # restore `sp`
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)

        ret                 # return to new `ra`

为什么只切换这些寄存器就能实现一个切换的效果呢？这是因为执行了swtch切换状态之后，切换的目标进程恢复了保存在context之中的寄存器，并且sp寄存器也指向了它自己栈的位置，ra指向自己测例代码的位置而不是之前函数的位置，这已经足够其从切换出去的位置继续执行了（切换的过程可以视为一次函数调用）。因为真正切换swtch都在内核态发生，也无需记录更多的数据。

总结一下，swtch函数干了这些事情：
    - 执行流：通过切换 ra
    - 堆栈：通过切换 sp
    - 寄存器： 通过保存和恢复被调用者保存寄存器。调用者保存寄存器由编译器生成的代码负责保存和恢复。

一旦你理解了上述的过程，那么本章剩余内容就会十分简单~~

下面介绍进程切换的具体细节。

idle进程与scheduler

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

大家可能注意到proc.c文件中除了current_proc还记录了一个idle_proc。这个进程是干什么的呢？实际上，idle 进程是第一个进程(boot进程)，也是唯一一个永远会存在的进程，它还有一个大家更熟悉的面孔，它就是 os 的 main 函数。

.. code-block:: C
    :linenos:
    
    void main() {
        clean_bss();    // 清空 bss 段
        trapinit();     // 开启中断
        batchinit();    // 初始化 app_info_ptr 指针
        procinit();     // 初始化线程池
        // timerinit();    // 开启时钟中断，现在还没有
        run_all_app();  // 加载所有用户程序
        scheduler();    // 开始调度
    }

可以看到，在main函数完成了一系列的初始化，并且执行了run_all_app加载完了所有测例之后。它就进入了scheduler调度函数。这个函数就完成了一系列的调度工作：

.. code-block:: C
    :linenos:
    
    void
    scheduler(void)
    {
        struct proc *p;

        for(;;){
            for(p = pool; p < &pool[NPROC]; p++) {
                if(p->state == RUNNABLE) {
                    p->state = RUNNING;
                    current_proc = p;
                    swtch(&idle.context, &p->context);
                }
            }
        }
    }

可以看到一旦main进入调度状态就进入一个死循环再也回不去了。。但它也没必要回去，它现在活着的意义就是为了进行进程的调度。在循环中每一次idle进程都会遍历整个进程池来寻找RUNNABLE（就绪）状态的进程并执行swtch函数切换到它。我们这里的scheduler函数就是最普通的调度函数，完全没有考虑优先度以及复杂度。

这里大家要思考一下，这个函数写对了吗？它真的满足我们每次执行遍历一次的要求，而不是写成了每次都从第0个进程开始遍历查找吗？

yield，时钟中断与抢占式调度

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们的框架也支持协助式调度的yield函数。它的实现十分简单，本质就是调用了下面这个函数

.. code-block:: C
    :linenos:

    // kernel/trap.c
    void sched(void)
    {
        struct proc *p = curr_proc();
        swtch(&p->context, &idle.context);
    }

它本质就是主动放弃执行，并把context移交给负责scheduler进程的idle进程。这样下一个执行的就不会是此进程了。

我们思考一个问题，如果一个程序不放弃执行，那它直到结束之前就会一直执行。为了避免这个情况我们引入了抢占式调度的进程调度方式。这个方式使得有某种机制能够让当前进程的运行被打断。这个过程不能是应用程序自己导致的（不可控）。我们采用的策略就是时钟中断。

timer.c 中包含了相关函数，功能分别为：打开了时钟中断使能，设置下一次中断间隔，读取当前的机器 cycle 数：

.. code-block:: C
    :linenos:

    // kernel/timer.c

    /// Enable timer interrupt
    void timerinit() {
        // Enable supervisor timer interrupt
        w_sie(r_sie() | SIE_STIE);
        set_next_timer();
    }
    /// Set the next timer interrupt
    void set_next_timer() {
        uint64 timebase = 125000;
        set_timer(get_cycle() + timebase);
    }

    uint64 get_cycle() {
        return r_time();
    }

RISC-V 架构要求处理器要有一个内置时钟，其频率一般低于 CPU 主频。此外，还有一个计数器统计处理器自上电以来经过了多少个内置时钟的时钟周期。在 RV64 架构上，该计数器保存在一个 64 位的 CSR ``mtime`` 中，我们无需担心它的溢出问题，在内核运行全程可以认为它是一直递增的。

另外一个 64 位的 CSR ``mtimecmp`` 的作用是：一旦计数器 ``mtime`` 的值超过了 ``mtimecmp``，就会触发一次时钟中断。这使得我们可以方便的通过设置 ``mtimecmp`` 的值来决定下一次时钟中断何时触发。

可惜的是，它们都是 M 特权级的 CSR ，而我们的内核处在 S 特权级，是不被硬件允许直接访问它们的。好在运行在 M 特权级的 SEE 已经预留了相应的接口，我们可以调用它们来间接实现计时器的控制。

我们通过设置时钟中断相关的寄存器开启了时钟中断。这样用户程序在执行的时候经过一个指定的时间片（我们这里是10ms)就会产生一个中断，从U态回到内核的S态。此时我们就可以执行内存的切换了:

.. code-block:: C
    :linenos:

    void usertrap() {
        // ...
        uint64 cause = r_scause();
        if(cause & (1ULL << 63)) {
            cause &= ~(1ULL << 63);
            switch(cause) {
            case SupervisorTimer:
                set_next_timer();
                yield();
                break;
            default:
                unknown_trap();
                break;
            }
        } else {
            // ....
        }
        usertrapret();
    }

可以看到如果应用程序进程是因为时间中断而陷入trap的话，OS就会调用yield让进程切换到scheduler的idle进程，达到进程调度的效果。当下一次回到此进程时，该进程会继续完成usertrap函数，并通过usertrapret函数回到U态继续执行。

注意，为了避免混乱，在内核态我们屏蔽了所有的异常和中断。一旦内核出现问题OS会直接panic退出。这个直到lab7才会改变。