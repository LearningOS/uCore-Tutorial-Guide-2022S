引言
=========================================

本章导读
-----------------------------------------

在第六章中，我们为进程引入了文件的抽象，使得进程能够通过一个统一的接口来读写内核管理的多种不同的 I/O 资源。作为例子，我们实现了匿名管道，并通过它进行了简单的父子进程间的单向通信。其实文件的最早起源于我们需要把数据持久保存在 **持久存储设备** 上的需求。

大家不要被 **持久存储设备** 这个词给吓住了，这就是指计算机远古时代的卡片、纸带、磁芯、磁鼓，和现在还在使用的磁带、磁盘、硬盘，还有近期逐渐普及的U盘、闪存、固态硬盘 (SSD, Solid-State Drive)等存储设备。我们可以把这些设备叫做 **外存** 。在此之前我们仅使用一种存储，也就是内存（或称 RAM）。相比内存，持久存储设备的读写速度较慢，容量较大，但内存掉电后信息会丢失，外存在掉电之后并不会丢失数据。因此，将需要持久保存的数据从内存写入到外存，或是从外存读入到内存是应用和操作系统必不可少的一种需求。


.. note::

   文件系统在UNIX操作系统有着特殊的地位，根据史料《UNIX: A History and a Memoir》记载，1969年，Ken Thompson（Unix的作者）在贝尔实验室比较闲，写了PDP-7计算机的磁盘调度算法来提高磁盘的吞吐量。为了测试这个算法，他本来想写一个批量读写数据的测试程序。但写着写着，他在某一时刻发现，这个测试程序再扩展一下，就是一个文件系统了，再再扩展一下，就是一个操作系统了。他的自觉告诉他，他离实现一个操作系统仅有 **三周之遥** 。一周：写代码编辑器；一周：写汇编器；一周写shell程序，在写这些程序的同时，需要添加操作系统的功能（如 exec等系统调用）以支持这些应用。结果三周后，为测试磁盘调度算法性能的UNIX雏形诞生了。


本章我们将实现一个简单的文件系统 -- easyfs，能够对 **持久存储设备** (Persistent Storage) 这种 I/O 资源进行管理。对于应用访问持久存储设备的需求，内核需要新增两种文件：常规文件和目录文件，它们均以文件系统所维护的 **磁盘文件** 形式被组织并保存在持久存储设备上。

同时，由于我们进一步完善了对 **文件** 这一抽象概念的实现，我们可以更容易建立 ” **一切皆文件** “ (Everything is a file) 的UNIX的重要设计哲学。我们可扩展与应用程序执行相关的 ``exec`` 系统调用，加入对程序运行参数的支持，并进一步改进了对shell程序自身的实现，加入对重定向符号 ``>`` 、 ``<`` 的识别和处理。这样我们也可以像UNIX中的shell程序一样，基于文件机制实现灵活的I/O重定位和管道操作，更加灵活地把应用程序组合在一起实现复杂功能。

实践体验
-----------------------------------------

获取本章代码：

.. code-block:: console

   $ git checkout ch7

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ make test BASE=1
   >> ch7b_usertest

.. code-block:: bash

    >> ch7b_filetest
    file_test passed!
    Shell: Process 2 exited with code 0
    >> 

它会将 ``Hello, world!`` 输出到另一个文件 ``filea`` ，并读取里面的内容确认输出正确。我们也可以通过命令行工具 ``cat`` 来更直观的查看 ``filea`` 中的内容：

.. code-block:: bash

   >> cat filea
   Hello, world!
   Shell: Process 2 exited with code 0
   >> 

此外，在本章我们为shell程序支持了输入/输出重定向功能，可以将一个应用的输出保存到一个指定的文件。例如，下面的命令可以将 ``yield`` 应用的输出保存在文件 ``fileb`` 当中，并在应用执行完毕之后确认它的输出：

.. code-block:: bash

   >> yield > fileb
   Shell: Process 2 exited with code 0
   >> cat fileb
   Hello, I am process 2.
   Back in process 2, iteration 0.
   Back in process 2, iteration 1.
   Back in process 2, iteration 2.
   Back in process 2, iteration 3.
   Back in process 2, iteration 4.
   yield pass.

   Shell: Process 2 exited with code 0
   >> 

本章代码树
-----------------------------------------

.. code-block:: bash 

   .
   ├── bootloader
   │   └── rustsbi-qemu.bin
   ├── LICENSE
   ├── Makefile
   ├── nfs (新增，辅助程序，要来将 .bin 打包为 os 可以识别的文件镜像)
   │   ├── fs.c
   │   ├── fs.h
   │   ├── Makefile
   │   └── types.h
   ├── os
   │   ├── bio.c (新增，IO buffer 的实现)
   │   ├── bio.h
   │   ├── console.c
   │   ├── console.h
   │   ├── const.h
   │   ├── defs.h
   │   ├── entry.S
   │   ├── fcntl.h (新增，文件相关的一些抽象)
   │   ├── file.c (更加完成的文件操作)
   │   ├── file.h (更加完成的文件定义)
   │   ├── fs.c (新增，文件系统实际逻辑)
   │   ├── fs.h
   │   ├── kalloc.c
   │   ├── kalloc.h
   │   ├── kernel.ld
   │   ├── kernelvec.S
   │   ├── link_app.S
   │   ├── loader.c
   │   ├── loader.h
   │   ├── log.h
   │   ├── main.c
   │   ├── pipe.c
   │   ├── plic.c (新增，用来处理磁盘中断)
   │   ├── plic.h (新增，用来处理磁盘中断)
   │   ├── printf.c
   │   ├── printf.h
   │   ├── proc.c
   │   ├── proc.h
   │   ├── riscv.h
   │   ├── sbi.c
   │   ├── sbi.h
   │   ├── string.c
   │   ├── string.h
   │   ├── switch.S
   │   ├── syscall.c
   │   ├── syscall.h
   │   ├── syscall_ids.h
   │   ├── timer.c
   │   ├── timer.h
   │   ├── trampoline.S
   │   ├── trap.c
   │   ├── trap.h
   │   ├── types.h
   │   ├── virtio_disk.c (新增，用来处理磁盘中断)
   │   ├── virtio.h (新增，用来处理磁盘中断)
   │   ├── vm.c
   │   └── vm.h
   ├── README.md
   ├── scripts
   │   └── initproc.py (弱化的 pack.py，仅仅用来插入 INIT_PROC 符号)
   └── user

本章代码导读
-----------------------------------------------------          

本章涉及的代码量相对较多，且与进程执行相关的管理还有直接的关系。其实我们是参考经典的UNIX基于索引的文件系统，设计了一个简化的有一级目录并支持创建/打开/读写/关闭文件一系列操作的文件系统，也就是说本章。本章采用的文件系统和ext4文件系统比较类似。其中也涉及到了inode这个概念。进入本章之后，我们的测例文件一开始是存放在我们生成的“磁盘”上的，需要我们实现磁盘的读写来进行操作了。我们实现了一个简单的 nfs 文件系统，具体的结构将在下面的章节中说明。大家可以看一看我们本章对 makefile 文件的改动.

.. code-block:: Makefile

   QEMU = qemu-system-riscv64
   QEMUOPTS = \
      -nographic \
      -smp $(CPUS) \
      -machine virt \
      -bios $(BOOTLOADER) \
      -kernel kernel	\
   +	-drive file=$(U)/fs.img,if=none,format=raw,id=x0 \       # 以 user/fs.img 作为磁盘镜像
   +  -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0      # 虚拟 virtio 磁盘设备

我们OS的读写文件操作均在内核态进行，由于不确定读写磁盘的结束时间，这意味着我们需要新的中断方式——外部中断来提醒OS读写结束了。而要在内核态引入中断意味着我们不得不短暂开启在内核态的嵌套中断。一旦OS打开了文件，那么我们就可以获得文件对应的fd了(实际上lab6中我们做了类似的事情），就可以使用sys_write/sys_read对文件进行读写操作。