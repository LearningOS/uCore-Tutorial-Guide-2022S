引言
=========================================

本章导读
-----------------------------------------

到本章开始之前，我们完成了组成应用程序执行环境的操作系统的三个重要抽象：进程、地址空间和文件，
操作系统基于处理器的时间片不断地切换进程可以实现宏观的多应用并发，但仅限于进程间。
下面我们引入线程（Thread）：提高一个进程内的并发性。

为什么有了进程还需要线程呢？因为对于很多单进程应用，逻辑上存在多个可并行执行的任务，
如果没有多线程，其中一个任务被阻塞，将会引起不依赖该任务的其他任务也被阻塞。
譬如我们用 Word 时，会有一个定时自动保存功能，如果你电脑突然崩溃关机或者停电，已有的文档内容或许已被提前保存。
假设没有多线程，自动保存时由于磁盘性能导致写入较慢，可能导致整个进程被操作系统挂起，
对我们来说便是 Word 过一阵子就卡一会儿，严重影响体验。


.. _term-thread-define:

线程定义
~~~~~~~~~~~~~~~~~~~~

简单地说，线程是进程的组成部分，进程可包含1 -- n个线程，属于同一个进程的线程共享进程的资源，
比如地址空间、打开的文件等。基本的线程由线程ID、执行状态、当前指令指针 (PC)、寄存器集合和栈组成。
线程是可以被操作系统或用户态调度器独立调度（Scheduling）和分派（Dispatch）的基本单位。

在本章之前，进程是程序的基本执行实体，是程序关于某数据集合上的一次运行活动，是系统进行资源（处理器、
地址空间和文件等）分配和调度的基本单位。在有了线程后，对进程的定义也要调整了，进程是线程的资源容器，
线程成为了程序的基本执行实体。


同步互斥
~~~~~~~~~~~~~~~~~~~~~~

在上面提到了同步互斥和数据一致性，它们的含义是什么呢？当多个线程共享同一进程的地址空间时，
每个线程都可以访问属于这个进程的数据（全局变量）。如果每个线程使用到的变量都是其他线程不会读取或者修改的话，
那么就不存在一致性问题。如果变量是只读的，多个线程读取该变量也不会有一致性问题。但是，当一个线程修改变量时，
其他线程在读取这个变量时，可能会看到一个不一致的值，这就是数据不一致性的问题。

.. note::

    **并发相关术语**

    - 共享资源（shared resource）：不同的线程/进程都能访问的变量或数据结构。
    - 临界区（critical section）：访问共享资源的一段代码。
    - 竞态条件（race condition）：多个线程/进程都进入临界区时，都试图更新共享的数据结构，导致产生了不期望的结果。
    - 不确定性（indeterminate）： 多个线程/进程在执行过程中出现了竞态条件，导致执行结果取决于哪些线程在何时运行，
      即执行结果不确定，而开发者期望得到的是确定的结果。
    - 互斥（mutual exclusion）：一种操作原语，能保证只有一个线程进入临界区，从而避免出现竞态，并产生确定的执行结果。
    - 原子性（atomic）：一系列操作要么全部完成，要么一个都没执行，不会看到中间状态。在数据库领域，
      具有原子性的一系列操作称为事务（transaction）。
    - 同步（synchronization）：多个并发执行的进程/线程在一些关键点上需要互相等待，这种相互制约的等待称为进程/线程同步。
    - 死锁（dead lock）：一个线程/进程集合里面的每个线程/进程都在等待只能由这个集合中的其他一个线程/进程
      （包括他自身）才能引发的事件，这种情况就是死锁。
    - 饥饿（hungry）：指一个可运行的线程/进程尽管能继续执行，但由于操作系统的调度而被无限期地忽视，导致不能执行的情况。

在后续的章节中，会大量使用上述术语，如果现在还不够理解，没关系，随着后续的一步一步的分析和实验，
相信大家能够掌握上述术语的实际含义。



实践体验
-----------------------------------------

获取本章代码：

.. code-block:: console

   $ git clone https://github.com/LearningOS/uCore-Tutorial-Code-2022S.git
   $ cd uCore-Tutorial-Code-2022S
   $ git checkout ch8
   $ git clone https://github.com/LearningOS/uCore-Tutorial-Test-2022S.git user

或者你也可以在自己原来的仓库里 fetch 它，记得更新测例仓库的代码。

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ make BASE=1 test

内核初始化完成之后就会进入 shell 程序，我们可以体会一下线程的创建和执行过程。在这里我们运行一下本章的测例 ``ch8b_threads`` ：

.. code-block::

   >> ch8b_threads
   aaa....bbb...ccc...
   thread#1 exited with code 1
   thread#2 exited with code 2
   thread#3 exited with code 3
   threads test passed!
   Shell: Process 2 exited with code 0
   >>

它会有4个线程在执行，等前3个线程执行完毕并输出大量 a/b/c 后，主线程退出，导致整个进程退出。

此外，在本章的操作系统支持通过互斥来执行“哲学家就餐问题”这个应用程序：

.. code-block::

   >> ch8b_mut_phi_din
   Here comes 5 philosophers!
   Phil threads created
   time cost = 720 ms
   '-' -> THINKING; 'x' -> EATING; ' ' -> WAITING 
   #0:--------                xxxxxxxx-----------      xxxx------ xxxxxx---xxx
   #1:----xxxxx---     xxxxxxx-----------   x----xxxxxx                       
   #2:------         xx----------x-----xxxxx-------------       xxxxx         
   #3:------xxxxxxxxx-------xxxx---------   xxxxxx---  xxxxxxxxxx             
   #4:-------        x-------         xxxxxx---   xxxxx-------  xxx           
   #0:--------                xxxxxxxx-----------      xxxx------ xxxxxx---xxx
   Shell: Process 2 exited with code 0
   >>

我们可以看到5个代表“哲学家”的线程通过操作系统的 **信号量** 互斥机制在进行 “THINKING”、“EATING”、“WAITING” 的日常生活。
没有哲学家由于拿不到筷子而饥饿，也没有两个哲学家同时拿到一个筷子。

.. note::

    **哲学家就餐问题**

    计算机科学家 Dijkstra 提出并解决的哲学家就餐问题是经典的进程同步互斥问题。哲学家就餐问题描述如下：

    有5个哲学家共用一张圆桌，分别坐在周围的5张椅子上，在圆桌上有5个碗和5只筷子，他们的生活方式是交替地进行思考和进餐。
    平时，每个哲学家进行思考，饥饿时便试图拿起其左右最靠近他的筷子，只有在他拿到两只筷子时才能进餐。进餐完毕，放下筷子继续思考。


本章代码树
-----------------------------------------

.. code-block::
   :linenos:

   .
   ├── bootloader
   │   └── rustsbi-qemu.bin
   ├── LICENSE
   ├── Makefile
   ├── nfs
   │   ├── fs.c
   │   ├── fs.h
   │   ├── Makefile
   │   └── types.h
   ├── os
   │   ├── bio.c
   │   ├── bio.h
   │   ├── console.c
   │   ├── console.h
   │   ├── const.h
   │   ├── defs.h
   │   ├── entry.S
   │   ├── fcntl.h
   │   ├── file.c
   │   ├── file.h
   │   ├── fs.c
   │   ├── fs.h
   │   ├── kalloc.c
   │   ├── kalloc.h
   │   ├── kernel.ld
   │   ├── kernelld.py
   │   ├── kernelvec.S
   │   ├── loader.c（修改：更改了加载用户程序的逻辑，此时不再为进程分配用户栈）
   │   ├── loader.h
   │   ├── log.h（修改：log头中新增了线程号打印）
   │   ├── main.c
   │   ├── pipe.c
   │   ├── plic.c
   │   ├── plic.h
   │   ├── printf.c
   │   ├── printf.h
   │   ├── proc.c（修改：为每个线程而非进程分配栈空间和trapframe；更改进程初始化逻辑，为其分配主线程；任务调度粒度从进程改为线程；新增线程id与线程指针转换的辅助函数；新增线程分配、释放逻辑；exit由退出进程改为退出线程）
   │   ├── proc.h（修改：新增线程相关结构体和状态枚举；在PCB中新增线程相关变量；增改部分函数签名）
   │   ├── queue.c（修改：由进程专用队列改为通用队列，初始化时需指定数组地址和大小）
   │   ├── queue.h（修改：同 queue.c）
   │   ├── riscv.h
   │   ├── sbi.c
   │   ├── sbi.h
   │   ├── string.c
   │   ├── string.h
   │   ├── switch.S
   │   ├── sync.c（新增：实现了mutex、semaphore、condvar相关操作）
   │   ├── sync.h（新增：声明了mutex、semaphore、condvar相关操作）
   │   ├── syscall.c（修改：增加 sys_thread_create、sys_gettid、sys_waittid 以及三种同步互斥结构所用到的系统调用）
   │   ├── syscall.h（修改：同syscall.c）
   │   ├── syscall_ids.h（修改：为新增系统调用增加了调号号的宏定义）
   │   ├── timer.c
   │   ├── timer.h
   │   ├── trampoline.S
   │   ├── trap.c（修改：将进程trap改为线程trap，新增用户态虚存映射辅助函数uvmmap）
   │   ├── trap.h（修改：同trap.c）
   │   ├── types.h
   │   ├── virtio_disk.c
   │   ├── virtio.h
   │   ├── vm.c
   │   └── vm.h
   ├── README.md
   └── scripts
      └── initproc.py
