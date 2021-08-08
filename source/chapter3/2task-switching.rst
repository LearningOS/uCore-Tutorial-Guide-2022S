进程和进程切换
================================

本节导读
--------------------------

本节会介绍进程的调度方式。这是本章的重点之一。

进程的概念
---------------------------------

导语中提到了，进程就是运行的程序。既然是程序，那么它就需要程序执行的一切资源，包括栈、寄存器等等。不同于用户线程，用户进程有着自己独立的用户栈和内核栈。但是无论如何寄存器是只有一套的，因此进程切换时对于寄存器的保存以及恢复是我们需要关心的问题。

本章中新增的proc.h定义了我们OS的进程的PCB（进程管理块，和进程一一对应）结构体；

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

分配进程需要初始化其PID以及清空其栈空间，并设置第一次运行的上下文信息为usertrapret，使得进程能够从内核的S态返回U态并执行自己的代码。

接下来我们同样从栈上内容的角度来看 ``__switch`` 的整体流程：

.. image:: switch-1.png

.. image:: switch-2.png

Trap 执行流在调用 ``__switch`` 之前就需要明确知道即将切换到哪一条目前正处于暂停状态的 Trap 执行流，因此 ``__switch`` 有两个参数，第一个参数代表它自己，第二个参数则代表即将切换到的那条 Trap 执行流。这里我们用上面提到过的 ``task_cx_ptr2`` 作为代表。在上图中我们假设某次 ``__switch`` 调用要从 Trap 执行流 A 切换到 B，一共可以分为五个阶段，在每个阶段中我们都给出了 A 和 B 内核栈上的内容。

- 阶段 [1]：在 Trap 执行流 A 调用 ``__switch`` 之前，A 的内核栈上只有 Trap 上下文和 Trap 处理的调用栈信息，而 B 是之前被切换出去的，它的栈顶还有额外的一个任务上下文；
- 阶段 [2]：A 在自身的内核栈上分配一块任务上下文的空间在里面保存 CPU 当前的寄存器快照。随后，我们更新 A 的 ``task_cx_ptr``，只需写入指向它的指针 ``task_cx_ptr2`` 指向的内存即可；
- 阶段 [3]：这一步极为关键。这里读取 B 的 ``task_cx_ptr`` 或者说 ``task_cx_ptr2`` 指向的那块内存获取到 B 的内核栈栈顶位置，并复制给 ``sp`` 寄存器来换到 B 的内核栈。由于内核栈保存着它迄今为止的执行历史记录，可以说 **换栈也就实现了执行流的切换** 。正是因为这一步， ``__switch`` 才能做到一个函数跨两条执行流执行。
- 阶段 [4]：CPU 从 B 的内核栈栈顶取出任务上下文并恢复寄存器状态，在这之后还要进行退栈操作。
- 阶段 [5]：对于 B 而言， ``__switch`` 函数返回，可以从调用 ``__switch`` 的位置继续向下执行。

从结果来看，我们看到 A 执行流 和 B 执行流的状态发生了互换， A 在保存任务上下文之后进入暂停状态，而 B 则恢复了上下文并在 CPU 上执行。

下面我们给出 ``__switch`` 的实现：

.. code-block:: riscv
    :linenos:

    # os/src/task/switch.S

    .altmacro
    .macro SAVE_SN n
        sd s\n, (\n+1)*8(sp)
    .endm
    .macro LOAD_SN n
        ld s\n, (\n+1)*8(sp)
    .endm
        .section .text
        .globl __switch
    __switch:
        # __switch(
        #     current_task_cx_ptr2: &*const TaskContext,
        #     next_task_cx_ptr2: &*const TaskContext
        # )
        # push TaskContext to current sp and save its address to where a0 points to
        addi sp, sp, -13*8
        sd sp, 0(a0)
        # fill TaskContext with ra & s0-s11
        sd ra, 0(sp)
        .set n, 0
        .rept 12
            SAVE_SN %n
            .set n, n + 1
        .endr
        # ready for loading TaskContext a1 points to
        ld sp, 0(a1)
        # load registers in the TaskContext
        ld ra, 0(sp)
        .set n, 0
        .rept 12
            LOAD_SN %n
            .set n, n + 1
        .endr
        # pop TaskContext
        addi sp, sp, 13*8
        ret

我们手写汇编代码来实现 ``__switch`` 。可以看到它的函数原型中的两个参数分别是当前 Trap 执行流和即将被切换到的 Trap 执行流的 ``task_cx_ptr2`` ，从 :ref:`RISC-V 调用规范 <term-calling-convention>` 可以知道它们分别通过寄存器 ``a0/a1`` 传入。

阶段 [2] 体现在第 18~26 行。第 18 行在 A 的内核栈上预留任务上下文的空间，然后将当前的栈顶位置保存下来。接下来就是逐个对寄存器进行保存，从中我们也能够看出 ``TaskContext`` 里面究竟包含哪些寄存器：

.. code-block:: rust
    :linenos:

    // os/src/task/context.rs

    #[repr(C)]
    pub struct TaskContext {
        ra: usize,
        s: [usize; 12],
    }

这里面只保存了 ``ra`` 和被调用者保存的 ``s0~s11`` 。``ra`` 的保存很重要，它记录了 ``__switch`` 返回之后应该到哪里继续执行，从而在切换回来并 ``ret`` 之后能到正确的位置。而保存调用者保存的寄存器是因为，调用者保存的寄存器可以由编译器帮我们自动保存。我们会将这段汇编代码中的全局符号 ``__switch`` 解释为一个 Rust 函数：

.. code-block:: rust
    :linenos:

    // os/src/task/switch.rs

    global_asm!(include_str!("switch.S"));

    extern "C" {
        pub fn __switch(
            current_task_cx_ptr2: *const usize,
            next_task_cx_ptr2: *const usize
        );
    }

我们会调用该函数来完成切换功能而不是直接跳转到符号 ``__switch`` 的地址。因此在调用前后 Rust 编译器会自动帮助我们插入保存/恢复调用者保存寄存器的汇编代码。

仔细观察的话可以发现 ``TaskContext`` 很像一个普通函数栈帧中的内容。正如之前所说， ``__switch`` 的实现除了换栈之外几乎就是一个普通函数，也能在这里得到体现。尽管如此，二者的内涵却有着很大的不同。

剩下的汇编代码就比较简单了。读者可以自行对照注释看看图示中的后面几个阶段各是如何实现的。另外，后面会出现传给 ``__switch`` 的两个参数相同，也就是某个 Trap 执行流自己切换到自己的情形，请读者对照图示思考目前的实现能否对它进行正确处理。

..
  chyyuu：有一个内核态切换的例子。

  