
.. _term-batchos:

实现批处理操作系统的细节
==============================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
-------------------------------

前面一节中我们明白了os是如何执行应用程序的。但是os是如何”找到“这些应用程序并允许它们的呢？在引言之中我们简要介绍了这是由link_app.S以及kernel_app.ld完成的。实际上，能够在批处理操作系统与应用程序之间建立联系的纽带。这主要包括两个方面：

- 静态编码：通过一定的编程技巧，把应用程序代码和批处理操作系统代码“绑定”在一起。
- 动态加载：基于静态编码留下的“绑定”信息，操作系统可以找到应用程序文件二进制代码的起始地址和长度，并能加载到内存中运行。

这里与硬件相关且比较困难的地方是如何让在内核态的批处理操作系统启动应用程序，且能让应用程序在用户态正常执行。

将应用程序链接到内核
--------------------------------------------

makefile更新
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们首先看一看本章的makefile改变了什么

.. code-block:: Makefile 

    link_app.o: link_app.S
    link_app.S: pack.py
        @$(PY) pack.py
    kernel_app.ld: kernelld.py
        @$(PY) kernelld.py

    kernel: $(OBJS) kernel_app.ld link_app.S
        $(LD) $(LDFLAGS) -T kernel_app.ld -o kernel $(OBJS)
        $(OBJDUMP) -S kernel > kernel.asm
        $(OBJDUMP) -t kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > kernel.sym

可以看到makefile执行了两个python脚本生成了我们提到的link_app.S和kernel_app.ld。这里选择python只是因为比较好写生成的代码，我们的os和python没有任何关系。link_app.S的大致内容如下::

        .align 4
        .section .data
        .global _app_num
    _app_num:
        .quad 2
        .quad app_0_start
        .quad app_1_start
        .quad app_1_end

        .global _app_names
    _app_names:
    .string "hello.bin"
    .string "matrix.bin"

        .section .data.app0
        .global app_0_start
    app_0_start:
        .incbin "../user/target/bin/ch2t_write0.bin" 

        .section .data.app1
        .global app_1_start
    app_1_start:
        .incbin "../user/target/bin/ch2b_write1.bin"
    app_1_end:

pack.py会遍历../user/target/bin，并将该目录下的目标用户程序*.bin包含入 link_app.S中，同时给每一个bin文件记录其地址和名称信息。最后，我们在 Makefile 中会将内核与 link_app.S 一同编译并链接。这样，我们在内核中就可以通过 extern 指令访问到用户程序的所有信息，如其文件名等。

由于 riscv 要求程序指令必须是对齐的，我们对内核链接脚本也作出修改，保证用户程序链接时的指令对齐，这些内容见 os/kernelld.py。这个脚本也会遍历../user/target/，并对每一个bin文件分配对齐的空间。最终修改后的kernel_app.ld脚本中多了如下对齐要求::

    .data : {
        *(.data)
        . = ALIGN(0x1000);
        *(.data.app0)
        . = ALIGN(0x1000);
        *(.data.app1)
        . = ALIGN(0x1000);
        *(.data.app2)
        . = ALIGN(0x1000);
        *(.data.app3)
        . = ALIGN(0x1000);
        *(.data.app4)

        *(.data.*)
    }

编译出的kernel已经包含了bin文件的信息。熟悉汇编的同学可以去看看生成的kernel.asm（kernel整体的汇编代码）来加深理解。

内核的relocation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

内核中通过访问 link_app.S 中定义的 _app_num、app_0_start 等符号来获得用户程序位置.

.. code-block:: c

    // os/batch.c
    extern char _app_num[]; // 在link_app.S之中已经定义
    void batchinit() {
        app_info_ptr = (uint64*) _app_num;
        app_num = *app_info_ptr;
        app_info_ptr++;
        // from now on:
        // app_n_start = app_info_ptr[n]
        // app_n_end = app_info_ptr[n+1]
    }

然而我们并不能直接跳转到 app_n_start 直接运行，因为用户程序在编译的时候，会假定程序处在虚存的特定位置，而由于我们还没有虚存机制，因此我们在运行之前还需要将用户程序加载到规定的物理内存位置。为此我们规定了用户的链接脚本，并在内核完成程序的 "搬运"

.. code-block:: c 

    # user/lib/arch/riscv/user.ld
    SECTIONS {
        . = 0x80400000;                 #　规定了内存加载位置

        .startup : {
            *crt.S.o(.text)             #　确保程序入口在程序开头
        }

        .text : { *(.text) }
        .data : { *(.data .rodata) }

        /DISCARD/ : { *(.eh_*) }
    }

这样之后，我们就可以在读取指定内存位置的bin文件来执行它们了。下面是os内核读取link_app.S的info并把它们搬运到0x80400000开始位置的具体过程。

.. code-block:: c

    // os/batch.c
    const uint64 BASE_ADDRESS = 0x80400000, MAX_APP_SIZE = 0x20000;
    int load_app(uint64* info) {
        uint64 start = info[0], end = info[1], length = end - start;
        memset((void*)BASE_ADDRESS, 0, MAX_APP_SIZE);
        memmove((void*)BASE_ADDRESS, (void*)start, length);
        return length;
    }


用户栈与内核栈
--------------------------------------------

我们自己的OS内核运行时，是需要一个栈来存放自己需要的变量的，这个栈我们称之为内核栈。在RV之中，我们使用sp寄存器来记录当前栈顶的位置。因此，在进入OS之前，我们需要告诉qemu我们OS的内核栈的起始位置。这个在entry.S之中有实现::

    // entry.S
     _entry:
        la sp, boot_stack_top
        call main

        .section .bss.stack
        .globl boot_stack
    boot_stack:
        .space 4096 * 16
        .globl boot_stack_top

一个应用程序肯定也需要内存空间来存放执行时需要的种种变量（实际上就是执行程序对应的用户栈），同时我们在上一章节提到了trapframe，这个也需要一个空间存放。那么OS是如何给应用程序分配这些对应的空间的呢？

实际上，我们采用一个静态分配的方式来给程序分配对应的一定大小的空间,并在run_next_app函数初始化应用程序对应的trapframe，并将用户栈对应的起始位置写入trapframe之中的sp寄存器，来让程序找到自己用户栈起始的位置。（注意栈在空间是高到低位，因此这里起始位置的初始化是在静态分配数组的尾部)。

.. code-block:: c

    // loader.c
    __attribute__((aligned(4096))) char user_stack[USER_STACK_SIZE];
    __attribute__((aligned(4096))) char trap_page[TRAP_PAGE_SIZE];

    int run_next_app()
    {
        struct trapframe *trapframe = (struct trapframe *)trap_page;
        ...
        memset(trapframe, 0, 4096);
        trapframe->epc = BASE_ADDRESS;
        trapframe->sp = (uint64)user_stack + USER_STACK_SIZE; 
        usertrapret(trapframe, (uint64)boot_stack_top);
        ...
    }

到这里，一个应用程序就算真正完全加载进入了内存之中进入就绪状态，可以随时运行了。


    