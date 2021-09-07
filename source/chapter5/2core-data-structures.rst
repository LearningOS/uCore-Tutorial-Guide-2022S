shell与测例的加载
===================================

本节导读
-----------------------------------

本节将会展示新的bin_loader加载测例到进程的方式，并且展示我们的shell测例是如何运行的。

新的bin_loader
------------------------------------------------------------------------

exec会调用bin_loader,将对应文件名的测例加载到指定的进程p之中。

.. code-block:: c
    :linenos:

    void bin_loader(uint64 start, uint64 end, struct proc *p) {
        uint64 s = PGROUNDDOWN(start), e = PGROUNDUP(end), length = e - s;
        // proc_pagetable 完成 trapframe 和　trampoline 的映射
        p->pagetable = proc_pagetable(p);   
        // 完成 .bin 数据的映射
        for(uint64 va = BASE_ADDRESS, pa = s; pa < e; va += PGSIZE, pa += PGSIZE) {
            void* page = kalloc();
            memmove(page, (const void*)pa, PGSIZE);
            mappages(p->pagetable, va, PGSIZE, (uint64)page, PTE_U | PTE_R | PTE_W | PTE_X);
        }
        // 完成用户栈的映射
        alloc_ustack(p);    
        
        p->trapframe->epc = BASE_ADDRESS;
        p->sz = USTACK_SIZE + length;
    }

其中，对于用户栈、trapframe、trampoline 的映射没有变化，但是对 .bin 数据的映射似乎面目全非了，竟然由一个循环完成。其实，这个循环的逻辑十分简单，就是对于 .bin 的每一页，都申请一个新页并进行内容拷贝，最后建立这一页的映射。之所以这么麻烦完全是由于我们的物理内存管理过于简陋，一次只能分配一个页，如果能够分配连续的物理页，那么这个循环可以被一个 mappages 替代。

那么另一个问题是，为什么要拷贝呢？想想 lab4 我们是怎么干的，直接把虚存和物理内存映射就好了，根本没有拷贝。那么，拷贝是为了什么呢？其实，按照 lab4 的做法，程序运行之后就会修改仅有一份的程序"原像"，你会发现，lab4 的程序都是一次性的，如果第二次执行，会发现 .data 和 .bss 段数据都被上一次执行改掉了，不是初始化的状态。但是 lab4 的时候，每个程序最多执行一次，所以这么做是可以的。但在 lab5 所有程序都可能被无数次的执行，我们就必须对“程序原像”做保护，在“原像”的拷贝上运行程序了。


测例的执行
------------------------------------------------------------------------

从本章开始，大家可以发现我们的run_all_app函数有所改变:

.. code-block:: c
    :linenos:

    // os/loader.c
    int run_all_app() {
        struct proc *p = allocproc();
        p->parent = 0;
        int id = get_id_by_name("user_shell");
        if(id < 0)
            panic("no user shell");
        loader(id, p);
        p->state = RUNNABLE;
        return 0;
    }

（练习修改）我们这里只加载user_shell的bin程序。如果大家打开user_shell的测例源码，可以发现它是通过spwan系统调用来执行其他测例的。


usershell
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们也提供了一个十分简单的支持交互的usershell测例。

.. code-block:: c

    // 手搓了一个极简的 stack，用来维护用户输入，保存一行的输入
    char line[100] = {};
    int top = 0;
    void push(char c) { line[top++] = c; }
    void pop() { --top; }
    int is_empty() { return top == 0; }
    void clear() { top = 0; }

    int main() {
        printf("C user shell\n");
        printf(">> ");
        // shell 是不会结束的
        while (1) {
            // 读取一个字符
            char c = getchar();
            switch (c) {
                // 敲了回车，将输入内容解析位一个程序名，通过 fork + exec 执行 
                case LF:
                case CR:
                    printf("\n");
                    if (!is_empty()) {
                        push('\0');
                        int pid = fork();
                        if (pid == 0) {
                            // child process
                            if (exec(line) < 0) {
                                printf("no such program\n");
                                exit(0);
                            }
                            panic("unreachable!");
                        } else {
                            // 父进程 wait 执行的函数
                            int xstate = 0;
                            int exit_pid = 0;
                            exit_pid = wait(pid, &xstate);
                            assert(pid == exit_pid, -1);
                            printf("Shell: Process %d exited with code %d\n", pid, xstate);
                        }
                        // 无论如何，清空输入 buffer
                        clear();
                    }
                    printf(">> ");
                    break;
                case BS:
                case DL:
                    // 退格键
                    if (!is_empty()) {
                        putchar(BS);
                        printf(" ");
                        putchar(BS);
                        pop();
                    }
                    break;
                default:
                    // 普通输入，回显
                    putchar(c);
                    push(c);
                    break;
            }
        }
        return 0;
    }

可以看到这个测例实际上就是实现了一个简单的字符串处理的函数，并且针对解析得到的不同的指令调用不同的系统调用。要注意这需要shell支持read的系统调用。当读入用户的输入时，它会死循环的等待用户输入一个代表程序名称的字符串(通过sys_read)，当用户按下空格之后，shell 会使用 fork 和 exec 创建并执行这个程序，然后通过 sys_wait 来等待程序执行结束，并输出 exit_code。有了 shell 之后，我们可以只执行自己希望的程序，也可以执行某一个程序很多次来观察输出，这对于使用体验是极大的提升！可以说，第五章的所有努力都是为了支持 shell。

我们简单看一下sys_read的实现：

.. code-block:: c

    uint64 sys_read(int fd, uint64 va, uint64 len) {
        if (fd != 0)
            return -1;
        struct proc *p = curr_proc();
        char str[200];
        for(int i = 0; i < len; ++i) {
            int c = console_getchar();
            str[i] = c;
        }
        copyout(p->pagetable, va, str, len);
        return len;
    }

目前我们只支持标准输入stdin的输入（对应fd = 0）。console_getchar和putchar一样，在sbi.c之中实现了其系统调用的过程。