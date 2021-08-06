代码框架简述
================================================

.. toctree::
   :hidden:
   :maxdepth: 5


本节导读
-------------------------------

本节会介绍代码的整体框架。我们默认大家都熟练掌握了C语言，不会从头讲C语言的基本语法qaq。
整个项目目前的代码树如下：


os文件夹
-------------------------------

os文件夹下存放了所有我们构建操作系统的源代码。也是本次实验中最最重要的一部分。在开始实验之前，大家一定要清楚我们这是自己设计的os，是无法使用C提供的官方标准库的，也就是说，就算是最简单的printf之类的函数都无法使用。但是别担心，作为一个轻量级的os，我们也用不到那么多函数。需要的函数助教们在框架之中已经为大家编写好了~~

我们的os是一个由makefile来构建的C项目。下面介绍框架之中一些重要文件的作用，以及整个项目是如何链接及编译的。

- kernel.ld
   kernel.ld是我们用于链接项目的脚本。链接脚本决定了 elf 程序的内存空间布局(严格的讲是虚存映射，注意程序中的各种绝对地址就在链接的时候确定)，由于刚进入 S 态的时候我们尚未激活虚存机制，我们必须把 os 置于物理内存的 0x80200000 处（这个地址的来由请参考rustsbi）::
      BASE_ADDRESS = 0x80200000;
      SECTIONS
      {
         . = BASE_ADDRESS;
         skernel = .;

         stext = .;
         .text : {
            *(.text.entry)   # 第一行代码
            *(.text .text.*)
         }

         ...
      }
   SECTIONS之中是从BASE_ADDRESS开始的各段。对程序内存布局还不太熟悉的同学可以翻看后面内存布局的章节。以text段为例，它是由不同文件的text组成。我们没有规定这些 text 段的具体顺序，但是我们规定了一个特殊的 text 段：.text.entry 段，该 text 段是 BASE_ADDRESS 后的第一个段，该段的第一行代码就在 0x80200000 处。这个特殊的段不是编译生成的，它在 entry.S 中人为设定。

- entry.S
   ::
      # entry.S
         .section .text.entry
         .globl _entry
      _entry:
         la sp, boot_stack
         call main

         .section .bss.stack
         .globl boot_stack
      boot_stack:
         .space 4096 * 16
         .globl boot_stack_top
      boot_stack_top:

   .text.entry 段中只有一个函数 _entry，它干的事情也十分简单，设置好 os 运行的堆栈（la sp, boot_stack语句。bootloader 并没有好心的设置好这些），然后调用 main 函数。main 函数位于 main.c 中，从此开始我们就基本进入了 C 的世界。

- main.c
   它是os的入口函数。在其中我们会完成一系列的初始化并开始运行os。
   作为第一章，它在初始化完毕之后实际上起到了一个测试的作用。如果你的main.c能够完成一系列打印并且最后成功退出(Shutdown),那么祝贺你，你完成了os的第一步。
   .. code-block:: c
      :linenos:

      void main() {
         clean_bss();
         printf("\n");
      }
   
   实际上，main.c之中众多的extern声明的内存段是在ld文件之中定义的。
   等待lab1重做。

bootloader文件夹
-------------------------------

这个文件夹是用来存放bootloader的bin文件的，这一章以及之后都无需我们做任何修改。

硬件加电之后是处于M态，而rustsbi帮助我们完成了M态的初始化，到os运行的S态的迁移，以及PC移动至我们os开始执行的位置。同时，它也帮助我们完成了异常和中断的委托。会帮助S态的os完成其发出的ecall请求等等。具体的细节就是大家这一章的练习之一。

user文件夹（本章还莫得，需移动至lab2）
---------------------------------------

大家马上就会发现我们并没有这样一个文件夹啊？实际上user文件夹是我们用于存放测例的文件夹。大家可以通过clone如下仓库的得到:

.. code-block:: bash
   
   git clone https://github.com/DeathWish5/riscvos-c-tests.git

之后直接将user文件夹复制到我们的项目之中就可以了。

我们的测例是通过cmake来编译的。具体的指令可以参见其中的readme。在使用测例的时候要注意，由于我们使用的是自己的os系统，因此所有常见的C库，比如stdlib.h，stdio.h等等都不能使用C官方的版本。这里在user的include和lib之中我们提供了搭配本次实验的对应库。大家可以看到，所有测例代码调用的函数都是使用的这里的代码。而这些库会依赖我们编写的os提供的系统调用(syscall)来运行。

user的库是如何调用到os的系统调用的呢？实际上转lab2.
