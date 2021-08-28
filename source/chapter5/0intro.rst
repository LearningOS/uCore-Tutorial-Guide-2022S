引言
===========================================

本章导读
-------------------------------------------

本章不同于前面几章对OS框架的大量修改，主要着力于增加OS对于进程管理的支持。因此内容和难度相比前面都会轻松很多~~.

支持了页表之后，我们的操作系统在硬件上的支持就告一段落了。但是目前我们应用测例的执行方式还是十分机械化的，并且无法和用户交互。目前为止，所有的应用都是在内核初始化阶段被一并加载到内存中的，之后也无法对应用进行动态增删，从用户的角度来看这和第二章的批处理系统似乎并没有什么不同。

于是，本章我们会开发一个用户 **终端** (Terminal) 或称 **命令行** 应用(Command Line Application, 俗称 **Shell** ) ，形成用户与操作系统进行交互的命令行界面(Command Line Interface)，它就和我们今天常用的 OS 中的命令行应用（如 Linux中的bash，Windows中的CMD等）没有什么不同：只需在其中输入命令即可启动或杀死应用，或者监控系统的运行状况。这自然是现代 OS 中不可缺少的一部分，并大大增加了系统的 **可交互性** ，使得用户可以更加灵活地控制系统。

我们想一下shell执行命令的过程。首先，shell必须支持读入用户的输入，并且如果我们在shell之中运行一个测例程序，它需要创建一个新的进程来执行这个命令对应的执行流。这里shell本身对于OS来说，也是一个进程。这就意味着我们需要支持进程创建进程的系统调用。实际上，在第四章添加了页表支持之后，现在我们可以开始实现几个进程非常关键的系统调用了，它们都是大家在课堂上已经耳熟能详的函数::

   sys_read(int fd, char* buf, int size): 从标准输入读取若干个字节。
   sys_fork(): 创建一个与当前进程几乎完全一致的进程。
   sys_exec(char* filename): 修改当前进程，使其从头开始执行指定程序。
   sys_wait(int pid, int* exit_code): 等待某一个或者任意一个子进程结束，获取其 exit_code。


实践体验
-------------------------------------------

获取本章代码：

.. code-block:: console

   $ git checkout ch5

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ cd os
   $ make run

将 Maix 系列开发板连接到 PC，并在上面运行本章代码：

.. code-block:: console

   $ cd os
   $ make run BOARD=k210

待内核初始化完毕之后，将在屏幕上打印可用的应用列表并进入shell程序（以 K210 平台为例）：

.. code-block::

   [rustsbi] RustSBI version 0.1.1
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

   [rustsbi] Platform: K210 (Version 0.1.0)
   [rustsbi] misa: RV64ACDFIMSU
   [rustsbi] mideleg: 0x22
   [rustsbi] medeleg: 0x1ab
   [rustsbi] Kernel entry: 0x80020000
   [kernel] Hello, world!
   last 808 Physical Frames.
   .text [0x80020000, 0x8002e000)
   .rodata [0x8002e000, 0x80032000)
   .data [0x80032000, 0x800c7000)
   .bss [0x800c7000, 0x802d8000)
   mapping .text section
   mapping .rodata section
   mapping .data section
   mapping .bss section
   mapping physical memory
   remap_test passed!
   after initproc!
   /**** APPS ****
   exit
   fantastic_text
   forktest
   forktest2
   forktest_simple
   forktree
   hello_world
   initproc
   matrix
   sleep
   sleep_simple
   stack_overflow
   user_shell
   usertests
   yield
   **************/
   Rust user shell
   >>  

其中 ``usertests`` 打包了很多应用，只要执行它就能够自动执行一系列应用。

只需输入应用的名称并回车即可在系统中执行该应用。如果输入错误的话可以使用退格键 (Backspace) 。以应用 ``exit`` 为例：

.. code-block::

    >> exit
    I am the parent. Forking the child...
    I am the child.
    I am parent, fork a child pid 3
    I am the parent, waiting now..
    waitpid 3 ok.
    exit pass.
    Shell: Process 2 exited with code 0
    >> 

当应用执行完毕后，将继续回到shell程序的命令输入模式。


本章代码导读
-----------------------------------------------------

本章对于框架没有大量修改的代码。由于添加的系统调用是针对进程方面的，除了在syscall.c之中添加了相关接口的定义之外，主要函数的实现都在proc.c之中完成。

（训练可能会改）我们已经完成了对上述几系统调用的支持，在开始本章的练习之前，大家需要仔细研究它们的实现细节，可以复习课堂上的知识，并且大大降低练习的难度。