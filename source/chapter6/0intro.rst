引言
=========================================

本章导读
-----------------------------------------

在上一章中，我们引入了非常重要的进程的概念，以及与进程管理相关的 ``fork`` 、 ``exec`` 等创建新进程相关的系统调用。虽然操作系统提供新进程的动态创建和执行的服务有了很大的改进，但截止到目前为止，进程在输入和输出方面，还有不少限制。特别是进程能够进行交互的 I/O 资源还非常有限，只能接受用户在键盘上的输入，并将字符输出到屏幕上。我们一般将它们分别称为 **标准** 输入和 **标准** 输出。而且进程之间缺少信息交换的能力，这样就限制了通过进程间的合作一起完成一个大事情的能力。

其实在 **UNIX** 的早期发展历史中，也碰到了同样的问题，每个程序专注在完成一件事情上，但缺少把多个程序联合在一起完成复杂功能的机制。直到1975年UNIX v6中引入了让人眼前一亮的创新机制-- **I/O重定向** 与 **管道（pipe）** 。基于这两种机制，操作系统在不用改变应用程序的情况下，可以将一个程序的输出重新定向到另外一个程序的输入中，这样程序之间就可以进行任意的连接，并组合出各种灵活的复杂功能。

本章我们也会引入新操作系统概念 -- 管道，并进行实现。除了键盘和屏幕这样的 **标准** 输入和 **标准** 输出之外，管道其实也可以看成是一种特殊的输入和输出，而后面一章讲解的 **文件系统** 中的对持久化存储数据的抽象 **文件(file)** 也是一种存储设备的输入和输出。所以，我们可以把这三种输入输出都统一在 **文件(file)**  这个抽象之中。这也体现了在 Unix 操作系统中， ” **一切皆文件** “ (Everything is a file) 重要设计哲学。

在本章中提前引入 **文件** 这个概念，但本章不会详细讲解，只是先以最简单直白的方式对 **文件** 这个抽象进行简化的设计与实现。站在本章的操作系统的角度来看， **文件** 成为了一种需要操作系统管理的I/O资源。 

为了让应用能够基于 **文件** 这个抽象接口进行I/O操作，我们就需要对 **进程** 这个概念进行扩展，让它能够管理 **文件** 这种资源。具体而言，就是要对进程控制块进行一定的扩展。为了统一表示 **标准** 输入和 **标准** 输出和管道，我们将在每个进程控制块中增加一个 **文件描述符表** ，在表中保存着多个 **文件** 记录信息。每个文件描述符是一个非负的索引值，即对应文件记录信息的条目在文件描述符表中的索引，可方便进程表示当前使用的 **标准** 输入、 **标准** 输出和管道（当然在下一章还可以表示磁盘上的一块数据）。用户进程访问文件将很简单，它只需通过文件描述符，就可以对 **文件** 进行读写，从而完成接收键盘输入，向屏幕输出，以及两个进程之间进行数据传输的操作。

本章我们的主要目的是实现进程间的通信方式。这就意味着一个进程得到的输入和输出不一定是针对标准输入输出流了（也就是fd == 0 的 stdin 和 fd == 1 的 stdout），而可能是对应的pipe的新的fd。考虑到lab7我们就需要实现一个比较完成的文件系统，在lab6中乘着引入pipe的机会我们会先实现一个文件系统（fs）的雏形。

实践体验
-----------------------------------------

获取本章代码：

.. code-block:: console

   $ git checkout ch6

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ cd os
   $ make run

本章代码树
-----------------------------------------

.. code-block::

    ./os/src
    Rust        28 Files    2061 Lines
    Assembly     3 Files      88 Lines

    ├── bootloader
    │   ├── rustsbi-k210.bin
    │   └── rustsbi-qemu.bin
    ├── LICENSE
    ├── os
    │   ├── build.rs
    │   ├── Cargo.lock
    │   ├── Cargo.toml
    │   ├── Makefile
    │   └── src
    │       ├── config.rs
    │       ├── console.rs
    │       ├── entry.asm
    │       ├── fs(新增：文件系统子模块 fs)
    │       │   ├── mod.rs(包含已经打开且可以被进程读写的文件的抽象 File Trait)
    │       │   ├── pipe.rs(实现了 File Trait 的第一个分支——可用来进程间通信的管道)
    │       │   └── stdio.rs(实现了 File Trait 的第二个分支——标准输入/输出)
    │       ├── lang_items.rs
    │       ├── link_app.S
    │       ├── linker-k210.ld
    │       ├── linker-qemu.ld
    │       ├── loader.rs
    │       ├── main.rs
    │       ├── mm
    │       │   ├── address.rs
    │       │   ├── frame_allocator.rs
    │       │   ├── heap_allocator.rs
    │       │   ├── memory_set.rs
    │       │   ├── mod.rs
    │       │   └── page_table.rs(新增：应用地址空间的缓冲区抽象 UserBuffer 及其迭代器实现)
    │       ├── sbi.rs
    │       ├── syscall
    │       │   ├── fs.rs(修改：调整 sys_read/write 的实现，新增 sys_close/pipe)
    │       │   ├── mod.rs(修改：调整 syscall 分发)
    │       │   └── process.rs
    │       ├── task
    │       │   ├── context.rs
    │       │   ├── manager.rs
    │       │   ├── mod.rs
    │       │   ├── pid.rs
    │       │   ├── processor.rs
    │       │   ├── switch.rs
    │       │   ├── switch.S
    │       │   └── task.rs(修改：在任务控制块中加入文件描述符表相关机制)
    │       ├── timer.rs
    │       └── trap
    │           ├── context.rs
    │           ├── mod.rs
    │           └── trap.S
    ├── README.md
    ├── rust-toolchain
    ├── tools
    │   ├── kflash.py
    │   ├── LICENSE
    │   ├── package.json
    │   ├── README.rst
    │   └── setup.py
    └── user
        ├── Cargo.lock
        ├── Cargo.toml
        ├── Makefile
        └── src
            ├── bin
            │   ├── exit.rs
            │   ├── fantastic_text.rs
            │   ├── forktest2.rs
            │   ├── forktest.rs
            │   ├── forktest_simple.rs
            │   ├── forktree.rs
            │   ├── hello_world.rs
            │   ├── initproc.rs
            │   ├── matrix.rs
            │   ├── pipe_large_test.rs(新增)
            │   ├── pipetest.rs(新增)
            │   ├── run_pipe_test.rs(新增)
            │   ├── sleep.rs
            │   ├── sleep_simple.rs
            │   ├── stack_overflow.rs
            │   ├── user_shell.rs
            │   ├── usertests.rs
            │   └── yield.rs
            ├── console.rs
            ├── lang_items.rs
            ├── lib.rs(新增两个系统调用：sys_close/sys_pipe)
            ├── linker.ld
            └── syscall.rs(新增两个系统调用：sys_close/sys_pipe)



本章代码导读
-----------------------------------------------------             

本章中引入了新的几个系统调用:

.. code-block:: c

   /// 功能：为当前进程打开一个管道。
   /// 参数：pipe 表示应用地址空间中的一个长度为 2 的 long 数组的起始地址，内核需要按顺序将管道读端和写端的文件描述符写入到数组中。
   /// 返回值：如果出现了错误则返回 -1，否则返回 0 。可能的错误原因是：传入的地址不合法。
   /// syscall ID：59
   long sys_pipe(long fd[2]);


   /// 功能：当前进程关闭一个文件。
   /// 参数：fd 表示要关闭的文件的文件描述符。
   /// 返回值：如果成功关闭则返回 0 ，否则返回 -1 。可能的出错原因：传入的文件描述符并不对应一个打开的文件。
   /// syscall ID：57
   long sys_close(int fd);

同时，为了支持对文件的支持，对sys_write和sys_read都有修改。本章的pipe被我们抽象成了文件的概念，因此，其对应的fd就是用于sys_write和sys_read的fd。我们的sys_close关闭文件，这本章也就是关闭管道。
