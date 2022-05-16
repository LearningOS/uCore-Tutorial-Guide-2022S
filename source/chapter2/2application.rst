实现应用程序以及user文件夹
===========================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
-------------------------------

本节主要讲解如何设计实现被批处理系统逐个加载并运行的应用程序。它们是假定在 U 特权级模式运行的前提下而设计、编写的。实际上，如果应用程序的代码都符合它要运行的某特权级的约束，那它完全可能在某特权级中运行。保证应用程序的代码在 U 模式运行是我们接下来将实现的批处理系统的任务。其涉及的设计实现要点是：

- 应用程序的内存布局
- 应用程序发出的系统调用

从某种程度上讲，这里设计的应用程序与第一章中的最小用户态执行环境有很多相同的地方。即设计一个应用程序，能够在用户态通过操作系统提供的服务完成自身的功能。

user文件夹以及测例简介
---------------------------------------

本章我们引入了用户程序。为了将内核与应用解耦，我们将二者分成了两个仓库，分别是存放内核程序的 ``uCore-Tutorial-Code-20xxx`` （下称代码仓库，最后几位 x 表示学期）与存放用户程序的 ``uCore-Tutorial-Test-20xxx`` （下称测例仓库）。 你首先需要进入代码仓库文件夹并 clone 用户程序仓库（如果已经执行过该步骤则不需要再重复执行）：

.. code-block:: console

   $ git clone https://github.com/LearningOS/uCore-Tutorial-Code-2022S.git
   $ cd uCore-Tutorial-Code-2022S
   $ git checkout ch2
   $ git clone https://github.com/LearningOS/uCore-Tutorial-Test-2022S.git user

上面的指令会将测例仓库克隆到代码仓库下并命名为 ``user`` ，注意 ``/user`` 在代码仓库的 ``.gitignore`` 文件中，因此不会出现 ``.git`` 文件夹嵌套的问题，并且你在代码仓库进行 checkout 操作时也不会影响测例仓库的内容。

测例实际就是批处理操作系统中一个个待执行的文件。下面我们看一个测例来理解本章以及之后测例的本质：

.. code-block:: c

    // ch2_hello_world.c
    #include <stdio.h>
    #include <unistd.h>

    int main(void)
    {
        puts("Hello world from user mode program!\nTest hello_world OK!");
        return 0;
    }

这个测例编译出来实际上就是一个可执行的打印helloworld的程序。如果是windows或者linux上它编译之后是可以直接执行的。它也可以用来检查我们操作系统的实现是否有问题。

我们的测例是通过cmake来编译的。具体编译出测例的指令可以参见其中的readme。在使用测例的时候要注意，由于我们使用的是自己的os系统，因此所有常见的C库，比如stdlib.h，stdio.h等等都不能使用C官方的版本。这里在user的include和lib之中我们提供了搭配本次实验的对应库，里面实现了所有测例所需要的函数。大家可以看到，所有测例代码调用的函数都是使用的这里的代码。而这些函数会依赖我们编写的os提供的系统调用(syscall)来完成运行。

user的库是如何调用到os的系统调用的呢？在user/lib/arch/riscv下的syscall_arch.h为我们包装好了使用riscv汇编调用系统调用ecall的函数接口。lib之中的syscall.c文件就是用这些包装好的函数来进行系统调用实现完整的函数功能。在第一章中大家已经了解了异常委托的机制。U态的ecall指令会转到S态，也就是我们编写的os来进行处理，这样整个逻辑就打通了:为了使得测例成功运行，我们必须实现处理对应ecall的函数。

那么现在我们还面临一个理解上的问题：就是测例文件在调用ecall的时候的细节：程序是如何完成特权级切换的？在ecall完毕回到U态的时候，程序又是如何恢复调用ecall之前的执行流并继续执行的呢？这里其实和汇编课程对于异常的处理是一样的，下面我们来复习一下。

应用程序的ecall处理流程
-----------------------------

ecall作为异常的一种，操作系统和CPU对它的处理方式其实和其他各种异常没什么区别。U态进行ecall调用具体的异常编号是8-Environment call from U-mode.RISCV处理异常需要引入几个特殊的寄存器——CSR寄存器。这些寄存器会记录异常和中断处理流程所需要或保存的各种信息。在上一章中我们看见的mideleg，medeleg寄存器就是CSR寄存器。

几个比较关键的CSR寄存器如下：
    - scause: 它用于记录异常和中断的原因。它的最高位为1是中断，否则是异常。其低位决定具体的种类。
    - sepc：处理完毕中断异常之后需要返回的PC值。
    - stval: 产生异常的指令的地址。
    - stvec：处理异常的函数的起始地址。
    - sstatus：记录一些比较重要的状态，比如是否允许中断异常嵌套。

需要注意的是这些寄存器是S态的CSR寄存器。M态还有一套自己的CSR寄存器mcause，mtvec... 

所以当U态执行ecall指令的时候就产生了异常。此时CPU会处理上述的各个CSR寄存器，之后跳转至stvec所指向的地址，也就是我们的异常处理函数。我们的os的这个函数的具体位置是在trap_init函数之中就指定了——是uservec函数。这个函数位于trampoline.S之中，是由汇编语言编写的。在uservec之中，os保存了U态执行流的各个寄存器的值。这些值的位置其实已经由trap.h中的trapframe结构体规定好了:

.. code-block:: c

    // os/trap.h
    struct trapframe {
        /*   0 */ uint64 kernel_satp;   // kernel page table
        /*   8 */ uint64 kernel_sp;     // top of process's kernel stack
        /*  16 */ uint64 kernel_trap;   // usertrap entry
        /*  24 */ uint64 epc;           // saved user program counter
        /*  32 */ uint64 kernel_hartid; // saved kernel tp， unused in our project
        /*  40 */ uint64 ra;
        /*  48 */ uint64 sp;
        /* ... */ ....
        /* 272 */ uint64 t5;
        /* 280 */ uint64 t6;
    };

由于涉及到直接操作寄存器，因此这里只能使用汇编语言来编写。具体可以参考trampoline.S之中的代码:

.. code-block:: c

        .section .text
    .globl trampoline
    trampoline:
    .align 4
    .globl uservec
    uservec:
        #
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #

        # swap a0 and sscratch
        # so that a0 is TRAPFRAME
        csrrw a0, sscratch, a0

        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        ...
        sd t6, 280(a0)

        # save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        csrr t1, sepc
        sd t1, 24(a0)

        ld sp, 8(a0)
        ld tp, 32(a0)
        ld t1, 0(a0)
        # csrw satp, t1
        # sfence.vma zero, zero
        ld t0, 16(a0)
        jr t0

这里需要注意sscratch这个CSR寄存器的作用就是一个cache，它只负责存某一个值，这里它保存的就是TRAPFRAME结构体的位置。csrr和csrrw指令是RV特供的读写CSR寄存器的指令。我们取用它的值的时候实际把原来a0的值和其交换了，因此返回时大家可以看到我们会再交换一次得到原来的a0。这里注释了两句代码大家可以不用管，这是页表相关的处理，我们在ch4会仔细了解它。

然后我们使用jr t0,就跳转到了我们早先设定在 trapframe->kernel_trap 中的地址，也就是 trap.c 之中的 usertrap 函数。这个函数在main的初始化之中已经调用了。

.. code-block:: c

    // os/trap.c
    // set up to take exceptions and traps while in the kernel.
    void trapinit(void)
    {
        w_stvec((uint64)uservec & ~0x3);   // 写 stvec, 最后两位表明跳转模式，该实验始终为 0
    }

该函数完成异常中断处理与返回，包括执行我们写好的syscall。

从S态返回U态是由 usertrapret 函数实现的。这里设置了返回地址sepc，并调用另外一个 userret 汇编函数来恢复 trapframe 结构体之中的保存的U态执行流数据。

.. code-block:: c

    void usertrapret(struct trapframe *trapframe, uint64 kstack)
    {
        trapframe->kernel_satp = r_satp(); // kernel page table
        trapframe->kernel_sp = kstack + PGSIZE; // process's kernel stack
        trapframe->kernel_trap = (uint64)usertrap;
        trapframe->kernel_hartid = r_tp(); // hartid for cpuid()

        w_sepc(trapframe->epc); // 设置了sepc寄存器的值。
        // set up the registers that trampoline.S's sret will use
        // to get to user space.

        // set S Previous Privilege mode to User.
        uint64 x = r_sstatus();
        x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
        x |= SSTATUS_SPIE; // enable interrupts in user mode
        w_sstatus(x);

        // tell trampoline.S the user page table to switch to.
        // uint64 satp = MAKE_SATP(p->pagetable);
        userret((uint64)trapframe);
    }

同样由于涉及寄存器的恢复，以及未来页表satp寄存器的设置等，userret也必须是一个汇编函数。它基本上就是uservec函数的镜像，将保存在trapframe之中的数据依次读出用于恢复对应的寄存器，实现恢复用户中断前的状态。

.. code-block:: c

    .globl userret
    userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.在第四章才会有具体作用。
        csrw satp, a1
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ...
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

        # restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0

        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret

需要注意最后执行的sret指令执行了2个事情：从S态回到U态，并将PC移动到sepc指定的位置，继续执行用户程序。

这个过程中还有一些细节，大家将在课后习题中慢慢品味。