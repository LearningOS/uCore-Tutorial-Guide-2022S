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

   $ make test BASE=1

   # ....
   app list:
   ch2b_exit
   ch2b_hello_world
   ch2b_power
   ch2b_write1
   ch3b_sleep
   ch3b_sleep1
   ch3b_yield0
   ch3b_yield1
   ch3b_yield2
   ch5b_exec_simple
   ch5b_exit
   ch5b_forktest0
   ch5b_forktest1
   ch5b_forktest2
   ch5b_getpid
   ch5b_usertest
   usershell
   C user shell
   >>

不出意外，你将最终运行进入 C suer shell，这里，你可以输入 app list 中的一个应用，敲击回车之后就可以运行。其中 ``ch5b_usertest`` 打包了很多应用，只要执行它就能够自动执行所有基础测试：

.. code-block:: bash

   >> ch2b_exit
   Shell: Process 2 exited with code 1234
   >> ch2b_hello_world
   Hello world from user mode program!
   Test hello_world OK!
   Shell: Process 3 exited with code 0

当应用执行完毕后，将继续回到shell程序的命令输入模式。另外，这个命令行支持退格键。


本章代码导读
-----------------------------------------------------

本章对于框架没有大量修改的代码。由于添加的系统调用是针对进程方面的，除了在syscall.c之中添加了相关接口的定义之外，主要函数的实现都在proc.c之中完成。此外，为了方便大家理解本章的进程调度部分内容，加入了queue.c文件，定义了一个就绪进程的一个队列。

我们已经完成了对上述几系统调用的支持，在开始本章的练习之前，大家需要仔细研究它们的实现细节，可以复习课堂上的知识，并且大大降低练习的难度。