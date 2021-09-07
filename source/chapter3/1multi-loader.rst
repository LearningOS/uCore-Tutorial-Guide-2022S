多道程序放置与加载
=====================================

本节导读
--------------------------

在本章的引言中我们提到每个应用都需要按照它的编号被分别放置并加载到内存中不同的位置。本节我们就来介绍它是如何实现的。通过具体实现，可以看到多个应用程序被一次性地加载到内存中，这样在切换到另外一个应用程序执行会很快，不像前一章介绍的操作系统，还要有清空前一个应用，然后加载当前应用的过程与开销。

但我们也会了解到，每个应用程序需要知道自己运行时在内存中的不同位置，这对应用程序的编写带来了一定的麻烦。而且操作系统也要知道每个应用程序运行时的位置，不能任意移动应用程序所在的内存空间，即不能在运行时根据内存空间的动态空闲情况，把应用程序调整到合适的空闲空间中。

..
  chyyuu：有一个ascii图，画出我们做的OS在本节的部分。

多道程序的放置
----------------------------

我们仍然使用pack.py以及kernel.py来生成链接测例bin文件的link_app.S以及kernel_app.ld两个文件。这些内容相较第二章并没有任何改变。主要改变的是对多道程序的加载上，要求同时加载多个程序了。

多道程序加载
----------------------------

从本章开始不再使用上一章批处理操作系统的run_next_app函数。让我们看看loader.c文件之中修改了什么。

.. code-block:: c
   :linenos:

    // os/loader.c

    int load_app(int n, uint64* info) {
        uint64 start = info[n], end = info[n+1], length = end - start;
        memset((void*)BASE_ADDRESS + n * MAX_APP_SIZE, 0, MAX_APP_SIZE);
        memmove((void*)BASE_ADDRESS + n * MAX_APP_SIZE, (void*)start, length);
        return length;
    }

    // load all apps and init the corresponding `proc` structure.
    // 这个函数的更过细节在之后讲解
    int run_all_app()
    {
        for (int i = 0; i < app_num; ++i) {
            struct proc *p = allocproc();
            struct trapframe *trapframe = p->trapframe;
            load_app(i, app_info_ptr);
            uint64 entry = BASE_ADDRESS + i * MAX_APP_SIZE;
            trapframe->epc = entry;
            trapframe->sp = (uint64)p->ustack + USER_STACK_SIZE;
            p->state = RUNNABLE;
        }
        return 0;
    }

可以看到，进程 load 的逻辑其实没有变化。但是我们在run_all_app函数之中一次性加载了所有的测例程序。具体的方式是遍历每一个app获取其放置的位置，并根据其序号i设置相对于BASE_ADDRESS的偏移量作为程序的起始位置。我们认为设定每个进程所使用的空间是 [0x80400000 + i*0x20000, 0x80400000 + (i+1)*0x20000)，每个进程的最大 size 为 0x20000，i 即为进程编号。需要注意此时同时完成了每一个程序对应的进程的初始化以及状态的设置（对进程还不熟悉的同学可以阅读下一节）。

现在应用程序加载完毕了。不同于批处理操作系统，我们该如何执行它们呢？