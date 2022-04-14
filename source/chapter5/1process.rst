与进程有关的重要系统调用
================================================

进程复习
-------------------------

本章添加了一系列的系统调用，主要修改的是进程的结构体以及针对系统调用的支持，以及部分关于进程调度相关的数据结构。

我们看一看我们进程支持的状态::

    UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE

其中的ZOMBIE（僵尸）状态在本章开始我们就可能遇到了。ZOMBIE在我们的OS中可能会在如下情景出现:一个进程存在父进程且在父进程未结束时就结束，在等待父进程释放其资源时，我们设定其处于ZOMBIE态。

对其他部分有点忘的同学可以复习一下ch3的实验~。

.. note::
    
    **进程，线程和协程**

    进程，线程和协程是操作系统中经常出现的名词，它们都是操作系统中的抽象概念，有联系和共同的地方，但也有区别。计算机的核心是CPU，它承担了基本上所有的计算任务；而操作系统是计算机的管理者，它可以以进程，线程和协程为基本的管理和调度单位来使用CPU执行具体的程序逻辑。

    从历史角度上看，它们依次出现的顺序是进程、线程和协程。在还没有进程抽象的早期操作系统中，计算机科学家把程序在计算机上的一次执行过程称为一个任务（task）或一个工作（job），其特点是任务和工作在其整个的执行过程中，不会被切换。这样其他任务必须等待一个任务结束后，才能执行，这样系统的效率会比较低。
    
    在引入面向CPU的分时切换机制和面向内存的虚拟内存机制后，进程的概念就被提出了，进程成为CPU（也称处理器）调度（scheduling）和分派（switch）的对象，各个进程间以时间片为单位轮流使用CPU，且每个进程有各自独立的一块内存，使得各个进程之间内存地址相互隔离。这时，操作系统通过进程这个抽象来完成对应用程序在CPU和内存使用上的管理。

    随着计算机的发展，对计算机系统性能的要求越来越高，而进程之间的切换开销相对较大，于是计算机科学家就提出了线程。线程是程序执行中一个单一的顺序控制流程，线程是进程的一部分，一个进程可以包含一个或多个线程。各个线程之间共享进程的地址空间，但线程要有自己独立的栈（用于函数访问，局部变量等）和独立的控制流。且线程是处理器调度和分派的基本单位。对于线程的调度和管理，可以在操作系统层面完成，也可以在用户态的线程库中完成。用户态线程也称为绿色线程（GreenThread）。如果是在用户态的线程库中完成，操作系统是“看不到”这样的线程的，也就谈不上对这样线程的管理了。

    协程（coroutines，也称纤程（Fiber）），也是程序执行中一个单一的顺序控制流程，建立在线程之上（即一个线程上可以有多个协程），但又比线程更加轻量级的处理器调度对象。协程一般是由用户态的协程管理库来进行管理和调度，这样操作系统是看不到协程的。而且多个协程共享同一线程的栈，这样协程在时间和空间的管理开销上，相对于线程又有很大的改善。在具体实现上，协程可以在用户态运行时库这一层面通过函数调用来实现；也可在语言级支持协程，比如Rust语言引入的 ``async`` 、 ``wait`` 关键字等，通过编译器和运行时库二者配合来简化程序员编程的负担并提高整体的性能。

重要系统调用
------------------------------------------------------------

fork 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. _term-pid:
.. _term-initial-process:

系统中同一时间存在的每个进程都被一个不同的 **进程标识符** (PID, Process Identifier) 所标识。在内核初始化完毕之后会创建一个进程——即 **用户初始进程** (Initial Process) ，它是目前在内核中以硬编码方式创建的唯一一个进程。其他所有的进程都是通过一个名为 ``fork`` 的系统调用来创建的。

首先，创建一个进程，就意味着我们要完成对应进程的PCB结构体以及其页表和栈的初始化等等。

.. code-block:: c

    // os/proc.c
    struct proc {
        enum procstate state;     
        int pid;                   
        pagetable_t pagetable;      
        uint64 ustack;
        uint64 kstack;       
        struct trapframe *trapframe; 
        struct context context;   
        uint64 sz;                   // Memory size
        struct proc *parent;         // Parent process
        uint64 exit_code;
    };


进程A调用 ``fork`` 系统调用之后，内核会创建一个新进程B，我们设定B是成为A的子进程。也就会设定其parent指向A的地址。我们再来看一下fork是如何进行新进程的初始化的:

.. code-block:: c

    int fork()
    {
        struct proc *p = curr_proc();
        struct proc *np = allocproc();
        // Copy user memory from parent to child.
        uvmcopy(p->pagetable, np->pagetable, p->max_page);
        np->max_page = p->max_page;
        // copy saved user registers.
        *(np->trapframe) = *(p->trapframe);
        // Cause fork to return 0 in the child.
        np->trapframe->a0 = 0;
        np->parent = p;
        np->state = RUNNABLE;
        return np->pid;
    }

首先，fork调用allocproc分配一个新的进程PCB（具体内容请见前几个lab，注意页表的初始化也在alloc时完成了）。之后，根据fork的规定，我们需要把进程A的内存拷贝至B的进程使得二者一样。我们不能仅仅拷贝一份一模一样的页表，那么父子进程就会修改同样的物理内存，发生数据冲突，不符合进程隔离的要求。需要把页表对应的页先拷贝一份，然后建立一个对这些新页有同样映射的页表。这一工作由一个 uvmcopy 的函数去做。uvmcopy函数会遍历A进程的页表，以页为单位将对应的内存复制到B进程页表中新kalloc的空闲地址之中。

.. warning::

    注意 mmap 对于进程 max_page 的影响。在 ch4 中，即便实现错误导致了内存泄漏也不会有直接致命的影响，但在 lab5 就不是这样了！修复你的 mmap 实现！

之后，我们把A的trapframe也复制给B，确保了B能继续A的执行流。但是我们设定a0寄存器的值为a，这是因为fork要求子进程的fork返回值是0。之后就是对于PCB的状态设定。

全部处理完之后，我们就得到了fork的新进程，并且父进程此时的返回值就是子进程的pid。

这里大家要仔细思考一下，当调度的我们新生成的子进程B的时候，它的执行流具体是什么样子的？这个问题对于理解OS框架十分重要。提示：新进程的 context 是怎样的？allocproc 会在进程池中新增一个进程，那么调度到的这个进程会从哪里开始执行？

wait 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在 fork 设 定好父子关系之后，wait 的实现就很简单了。我们通过直接遍历进程池数组来获得当前进程的所有子进程。我们来看一下具体系统调用的要求.

.. code-block:: c

    /// pid 表示要等待结束的子进程的进程 ID，如果为 0或者-1 的话表示等待任意一个子进程结束；
    /// status 表示保存子进程返回值的地址，如果这个地址为 0 的话表示不必保存。
    /// 返回值：如果出现了错误则返回 -1；否则返回结束的子进程的进程 ID。
    /// 如果子进程存在且尚未完成，该系统调用阻塞等待。
    /// pid 非法或者指定的不是该进程的子进程或传入的地址 status 不为 0 但是不合法均会导致错误。
    int waitpid(int pid, int *status);

来看一下具体waitpid的实现.

.. code-block:: c

    int
    wait(int pid, int* code)
    {
        struct proc *np;
        int havekids;
        struct proc *p = curr_proc();

        for(;;){
            // Scan through table looking for exited children.
            havekids = 0;
            for(np = pool; np < &pool[NPROC]; np++){
                if(np->state != UNUSED && np->parent == p && (pid <= 0 || np->pid == pid)){
                    havekids = 1;
                    if(np->state == ZOMBIE){
                        // Found one.
                        np->state = UNUSED;
                        pid = np->pid;
                        *code = np->exit_code;
                        return pid;
                    }
                }
            }
            if(!havekids){
                return -1;
            }
            p->state = RUNNABLE;
            sched();
        }
    }

wait 的思路就是遍历进程数组，看有没有和 pid 匹配的进程。如果有且已经结束(ZOMBIE态），按要求返回。如果指定进程不存在或者不是当前进程子进程，返回错误。如果子进程存在但未结束，调用 sched 切换到其他进程来等待子进程结束。

exec 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果仅有 ``fork`` 的话，那么所有的进程都只能和用户初始进程一样执行同样的代码段，这显然是远远不够的。于是我们还需要引入 ``exec`` 系统调用来执行不同的可执行文件。exec要干的事情和 bin_loader 是很相似的。事实上，不同点在于，exec 需要先清理并回收掉当前进程占用的资源，目前只有内存。

.. code-block:: c

    int exec(char *name)
    {
        int id = get_id_by_name(name);
        if (id < 0)
            return -1;
        struct proc *p = curr_proc();
        uvmunmap(p->pagetable, 0, p->max_page, 1);
        p->max_page = 0;
        loader(id, p);
        return 0;
    }

我们exec的设计是传入待执行测例的文件名。之后会找到文件名对应的id。如果存在对应文件，就会执行内存的释放。

由于 trapframe 和 trampoline 是可以复用的（每个进程都一样），所以我们并不会把他们 unmap。而对于用户真正的数据，就会删掉映射的同时把物理页面也 free 掉。

之后就是执行 loader 函数，这个loader函数相较前面的章节有比较大的修改，我们会在下一节说明。

支持了fork和exec之后，我们就用拥有了支持shell的基本能力。

.. _term-redirection:

.. note::

    **为何创建进程要通过两个系统调用而不是一个？**

    读者可能会有疑问，对于要达成执行不同应用的目标，我们为什么不设计一个系统调用接口同时实现创建一个新进程并加载给定的可执行文件两种功能？
    因为如果使用 ``fork`` 和 ``exec`` 的组合，那么 ``fork`` 出来的进程仅仅是为了 ``exec`` 一个新应用提供空间。而执行 ``fork`` 中对父进程的地址空间拷贝没有用处，还浪费了时间，且在后续清空地址空间的时候还会产生一些资源回收的额外开销。
    然而这样做是经过实践考验的——事实上 ``fork`` 和 ``exec`` 是一种灵活的系统调用组合。上述的这些开销能够通过一些技术方法（如 ``copy on write`` 等）大幅降低，且拆分为两个系统调用后，可以灵活地支持 **重定向** (Redirection) 等功能。
    上述方法是UNIX类操作系统的典型做法，这一点与Windows操作系统不一样。在Windows中， ``CreateProcess`` 函数用来创建一个新的进程和它的主线程，通过这个新进程运行指定的可执行文件。虽然是一个函数，但这个函数的参数十个之多，使得这个函数很复杂，且没有 ``fork`` 和 ``exec`` 的组合的灵活性。
