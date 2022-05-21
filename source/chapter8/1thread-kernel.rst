内核态的线程管理
=========================================

线程概念
---------------------------------------------

这里会结合与进程的比较来说明线程的概念。到本章之前，我们看到了进程这一抽象，操作系统让进程拥有相互隔离的虚拟的地址空间，
让进程感到在独占一个虚拟的处理器。其实这只是操作系统通过时分复用和空分复用技术来让每个进程复用有限的物理内存和物理CPU。
而线程是在进程内中的一个新的抽象。在没有线程之前，一个进程在一个时刻只有一个执行点（即程序计数器 (PC)
寄存器保存的要执行指令的指针）。但线程的引入把进程内的这个单一执行点给扩展为多个执行点，即在进程中存在多个线程，
每个线程都有一个执行点。而且这些线程共享进程的地址空间，所以可以不必采用相对比较复杂的 IPC 机制（一般需要内核的介入），
而可以很方便地直接访问进程内的数据。

在线程的具体运行过程中，需要有程序计数器寄存器来记录当前的执行位置，需要有一组通用寄存器记录当前的指令的操作数据，
需要有一个栈来保存线程执行过程的函数调用栈和局部变量等，这就形成了线程上下文的主体部分。
这样如果两个线程运行在一个处理器上，就需要采用类似两个进程运行在一个处理器上的调度/切换管理机制，
即需要在一定时刻进行线程切换，并进行线程上下文的保存与恢复。这样在一个进程中的多线程可以独立运行，
取代了进程，成为操作系统调度的基本单位。

由于把进程的结构进行了细化，通过线程来表示对处理器的虚拟化，使得进程成为了管理线程的容器。
在进程中的线程没有父子关系，大家都是兄弟，但还是有个老大。这个代表老大的线程其实就是创建进程（比如通过
``fork`` 系统调用创建进程）时，建立的第一个线程，它的线程标识符（TID）为 ``0`` 。


线程模型与重要系统调用
----------------------------------------------

目前，我们只介绍本章实现的内核中采用的一种非常简单的线程模型。这个线程模型有三个运行状态：
就绪态、运行态和等待态；共享所属进程的地址空间和其他共享资源（如文件等）；可被操作系统调度来分时占用CPU执行；
可以动态创建和退出；可通过系统调用获得操作系统的服务。我们实现的线程模型建立在进程的地址空间抽象之上：
每个线程都共享进程的代码段和和可共享的地址空间（如全局数据段、堆等），但有自己的独占的栈。
线程模型需要操作系统支持一些重要的系统调用：创建线程、等待子线程结束等，来支持灵活的多线程应用。
接下来会介绍这些系统调用的基本功能和设计思路。


线程创建系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在一个进程的运行过程中，进程可以创建多个属于这个进程的线程，每个线程有自己的线程标识符（TID，Thread Identifier）。
系统调用 ``thread_create`` 的原型如下：

.. code-block:: C
   :linenos:

   /// 功能：当前进程创建一个新的线程
   /// 参数：entry 表示线程的入口函数地址
   /// 参数：arg：表示线程的一个参数
   int sys_thread_create(uint64 entry, uint64 arg)

当进程调用 ``thread_create`` 系统调用后，内核会在这个进程内部创建一个新的线程，这个线程能够访问到进程所拥有的代码段，
堆和其他数据段。但内核会给这个新线程分配一个它专有的用户态栈，这样每个线程才能相对独立地被调度和执行。
另外，由于用户态进程与内核之间有各自独立的页表，所以二者需要有一个跳板页 ``TRAMPOLINE``
来处理用户态切换到内核态的地址空间平滑转换的事务。所以当出现线程后，在进程中的每个线程也需要有一个独立的跳板页
``TRAMPOLINE`` 来完成同样的事务。

相比于创建进程的 ``fork`` 系统调用，创建线程不需要要建立新的地址空间，这是二者之间最大的不同。
另外属于同一进程中的线程之间没有父子关系，这一点也与进程不一样。

等待子线程系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当一个线程执行完代表它的功能后，会通过 ``exit`` 系统调用退出。内核在收到线程发出的 ``exit`` 系统调用后，
会回收线程占用的部分资源，即用户态用到的资源，比如用户态的栈，用于系统调用和异常处理的跳板页等。
而该线程的内核态用到的资源，比如内核栈等，需要通过进程/主线程调用 ``waittid`` 来回收了，
这样整个线程才能被彻底销毁。系统调用 ``waittid`` 的原型如下：

.. code-block:: C
   :linenos:

   /// 参数：tid表示线程id
   /// 返回值：如果线程不存在，返回-1；如果线程还没退出，返回-2；其他情况下，返回结束线程的退出码
   int sys_waittid(int tid);


一般情况下进程/主线程要负责通过 ``waittid`` 来等待它创建出来的线程（不是主线程）结束并回收它们在内核中的资源
（如线程的内核栈、线程控制块等）。如果进程/主线程先调用了 ``exit`` 系统调用来退出，那么整个进程
（包括所属的所有线程）都会退出，而对应父进程会通过 ``waitpid`` 回收子进程剩余还没被回收的资源。


进程相关的系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在引入了线程机制后，进程相关的重要系统调用： ``fork`` 、 ``exec`` 、 ``waitpid`` 虽然在接口上没有变化，
但在它要完成的功能上需要有一定的扩展。首先，需要注意到把以前进程中与处理器执行相关的部分拆分到线程中。这样，在通过
``fork`` 创建进程其实也意味着要单独建立一个主线程来使用处理器，并为以后创建新的线程建立相应的线程控制块向量。
相对而言， ``exec`` 和 ``waitpid`` 这两个系统调用要做的改动比较小，还是按照与之前进程的处理方式来进行。总体上看，
进程相关的这三个系统调用还是保持了已有的进程操作的语义，并没有由于引入了线程，而带来大的变化。


应用程序示例
----------------------------------------------

我们刚刚介绍了 thread_create/waittid 两个重要系统调用，我们可以借助它们和之前实现的系统调用，
开发出功能更为灵活的应用程序。下面我们通过描述一个多线程应用程序 ``threads`` 的开发过程来展示这些系统调用的使用方法。


系统调用封装
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

同学可以在 user/lib/syscall.c 中看到系统调用的封装 ：

.. code-block:: C
   :linenos:

   int thread_create(void *entry, void *arg)
   {
      // on first thread create enable, here must be single thread
      if (!buffer_lock_enabled) {
         enable_thread_io_buffer();
      }
      return syscall(SYS_thread_create, (uint64)entry, (uint64)arg);
   }

   int waittid(int tid)
   {
      int ret = syscall(SYS_waittid, tid);
      while (ret == -2) {
         sched_yield();
         ret = syscall(SYS_waittid, tid);
      }
      return ret;
   }

``waittid`` 等待一个线程标识符的值为 tid 的线程结束。在具体实现方面，我们看到当 sys_waittid 返回值为 -2 ，即要等待的线程存在但它却尚未退出的时候，主线程调用 yield_ 主动交出 CPU 使用权，待下次 CPU 使用权被内核交还给它的时候再次调用 sys_waittid 查看要等待的线程是否退出。这样做是为了减小 CPU 资源的浪费。这种方法是为了尽可能简化内核的实现。

特别的，注意这里 ``thread_create`` 中有一个全局变量 ``buffer_lock_enabled`` 的判断，这是因为我们的输出使用了输出缓冲区，在旧版本里它没有考虑线程安全性，为了在多线程模式下正常使用缓冲区，我们新增了一个互斥锁用于输出，这一模式会在主线程首次调用 ``thread_create`` 时被启用。


多线程应用程序 -- threads
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

多线程应用程序 -- threads 开始执行后，先调用 ``thread_create`` 创建了三个线程，加上进程自带的主线程，其实一共有四个线程。每个线程在打印了1000个字符后，会执行 ``exit`` 退出。进程通过 ``waittid`` 等待这三个线程结束后，最终结束进程的执行。下面是多线程应用程序 -- threads 的源代码：

.. code-block:: c
   :linenos:

   //user/src/ch8b_threads.c
   ...
   #define LOOP 1000
   #define NTHREAD 3

   void thread_a()
   {
      for (int i = 0; i < LOOP; ++i) {
         putchar('a');
      }
      exit(1);
   }

   void thread_b()
   {
      for (int i = 0; i < LOOP; ++i) {
         putchar('b');
      }
      exit(2);
   }

   void thread_c()
   {
      for (int i = 0; i < LOOP; ++i) {
         putchar('c');
      }
      exit(3);
   }

   int main(void)
   {
      int tids[NTHREAD];
      tids[0] = thread_create(thread_a, 0);
      tids[1] = thread_create(thread_b, 0);
      tids[2] = thread_create(thread_c, 0);
      for (int i = 0; i < NTHREAD; ++i) {
         int tid = tids[i];
         int exit_code = waittid(tid);
         printf("thread %d exited with code %d\n", tid, exit_code);
         assert_eq(tid, exit_code);
      }
      puts("threads test passed!");
      return 0;
   }

线程管理的核心数据结构
-----------------------------------------------

为了在现有进程管理的基础上实现线程管理，我们需要改进一些数据结构包含的内容及接口。
基本思路就是把进程中与处理器相关的部分分拆出来，形成线程相关的部分。

本节将按照如下顺序来进行介绍：

- 线程控制块 thread ：表示线程的核心数据结构。
- 调度器 scheduler ：用于线程调度。

线程控制块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在内核中，每个线程的执行状态和线程上下文等均保存在一个被称为线程控制块
的结构中，它是内核对线程进行管理的核心数据结构。在内核看来，它就等价于一个线程。

.. code-block:: c
   :linenos:

   struct thread {
      enum threadstate state; // Thread state
      int tid; // Thread ID
      struct proc *process;
      uint64 ustack; // Virtual address of user stack
      uint64 kstack; // Virtual address of kernel stack
      struct trapframe *trapframe; // data page for trampoline.S
      struct context context; // swtch() here to run process
      uint64 exit_code;
   };

之前进程中的定义不存在的：

- tid：线程标识符
- process：线程所属的进程

与之前进程中的定义相同/类似的部分：

- ``trapframe`` 线程的 Trap 上下文在内核中的地址。
- ``context`` 保存暂停线程的线程上下文，用于线程切换。
- ``state`` 维护当前线程的执行状态。
- ``exit_code`` 线程退出码。


包含线程的进程控制块
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

把线程相关数据单独组织成数据结构后，进程的结构也需要进行一定的调整：

.. code-block:: c
   :linenos:

   // Per-process state
   struct proc {
      ...
      uint64 ustack_base; // Virtual address of user stack base
      struct thread threads[NTHREAD];
   };

从中可以看出，进程把与处理器执行相关的部分都移到了 ``thread`` 中，并组织为一个线程控制块数组，
这就自然对应到多个线程的管理上了。此外由于多个线程拥有各自不同的用户栈空间，
需要在进程控制块中记录用户栈的基地址，每个线程的用户栈基地址为 ustack_base + tid * USTACK_SIZE 。

线程管理机制的设计与实现
-----------------------------------------------

在上述线程模型和内核数据结构的基础上，我们还需完成线程管理的基本实现，从而构造出一个完整的“达科塔盗龙”操作系统。
本节将分析如何实现线程管理：

- 线程创建、线程退出与等待线程结束
- 线程执行中的特权级切换


线程创建、线程退出与等待线程结束
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


线程创建
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当一个进程执行中发出了创建线程的系统调用 ``sys_thread_create`` 后，操作系统就需要在当前进程的基础上创建一个线程了，
这里重点是需要了解初始化线程控制块的各个成员变量，建立好进程和线程的关系等。只有建立好这些成员变量，
才能给线程建立一个灵活方便的执行环境。这里列出支持线程正确运行所需的重要的执行环境要素：

- 线程的用户态栈：确保在用户态的线程能正常执行函数调用；
- 线程的内核态栈：确保线程陷入内核后能正常执行函数调用；
- 线程的跳板页：确保线程能正确的进行用户态<-->内核态切换；
- 线程上下文：即线程用到的寄存器信息，用于线程切换。

线程创建的具体实现如下：

.. code-block:: c
   :linenos:

   // os/syscall.c
   int sys_thread_create(uint64 entry, uint64 arg)
   {
      struct proc *p = curr_proc();
      int tid = allocthread(p, entry, 1);
      if (tid < 0) {
         errorf("fail to create thread");
         return -1;
      }
      struct thread *t = &p->threads[tid];
      t->trapframe->a0 = arg;
      t->state = RUNNABLE;
      add_task(t);
      return tid;
   }

上述代码主要完成了如下事务：

- 第 4 行，找到当前正在执行的线程所属的进程 ``p`` 。
- 第 5 行，调用 ``allocthread`` 函数，创建一个新的线程，线程编号 ``tid`` 即为该函数的返回值。在创建过程中，
  建立与进程 ``p`` 的所属关系，分配了线程用户态栈、内核态栈、用于异常/中断的跳板页，设置线程的函数入口点、
  任务上下文、内核栈和用户栈地址。
- 第 11-12 行，设置线程的传入参数，并将其状态设为可被调度运行。
- 第 13 行，把线程挂到调度队列中。

线程退出
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当一个非主线程的其他线程发出 ``sys_exit`` 系统调用时，内核会调用 ``exit``
函数退出当前线程并切换到下一个线程，但不会导致其所属进程的退出。当 **主线程** 即进程发出这个系统调用，
内核会回收整个进程（这包括了其管理的所有线程）资源，并退出。具体实现如下：

.. code-block:: c
   :linenos:

   // os/proc.c
   void exit(int code)
   {
      struct proc *p = curr_proc();
      struct thread *t = curr_thread();
      t->exit_code = code;
      t->state = EXITED;
      int tid = t->tid;
      debugf("thread exit with %d", code);
      freethread(t);
      if (tid == 0) {
         p->exit_code = code;
         freeproc(p);
         debugf("proc exit");
         if (p->parent != NULL) {
               // Parent should `wait`
               p->state = ZOMBIE;
         }
         // Set the `parent` of all children to NULL
         struct proc *np;
         for (np = pool; np < &pool[NPROC]; np++) {
               if (np->parent == p) {
                  np->parent = NULL;
               }
         }
      }
      sched();
   }

上述代码主要完成了如下事务：

- 第 10 行，调用 ``freethread`` 回收线程的各种资源。
- 第 11-26 行，如果是主线程发出的退出请求，则回收整个进程的部分资源，并退出进程。第 20-25
  行所做的事情是将当前进程的所有子进程的父进程置为 NULL。
- 第 27 行，进行线程调度切换。

上述实现中很大一部分与第五章讲解的 **进程的退出** 的功能实现大致相同。


等待线程结束
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

主线程通过系统调用 ``sys_waittid`` 来等待其他线程的结束。具体实现如下：

.. code-block:: c
   :linenos:

   // os/syscall.c
   int sys_waittid(int tid)
   {
      if (tid < 0 || tid >= NTHREAD) {
         errorf("unexpected tid %d", tid);
         return -1;
      }
      struct thread *t = &curr_proc()->threads[tid];
      if (t->state == T_UNUSED || tid == curr_thread()->tid) {
         return -1;
      }
      if (t->state != EXITED) {
         return -2;
      }
      memset((void *)t->kstack, 7, KSTACK_SIZE);
      t->tid = -1;
      t->state = T_UNUSED;
      return t->exit_code;
   }

上述代码主要完成了如下事务：

- 第 4-11 行，如果是线程等待一个不存在的 tid 或自己，则返回错误.
- 第 12-14 行，如果 ``tid`` 对应的线程还未退出，则返回错误。
- 第 16-18 行，对应的线程已经退出，则将该线程的线程控制块标记为已释放，并返回该线程的退出码。


线程执行中的特权级切换和调度切换
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

线程执行中的特权级切换与第三章中 **多道程序与协作式调度** 小节中讲解的过程是一致的，
而线程执行中的调度切换过程与第五章的 **进程管理的核心数据结构** 小节中讲解的过程是一致的。
这里就不用再赘述一遍了。
