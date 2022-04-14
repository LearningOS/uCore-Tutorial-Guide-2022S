shell与测例的加载
===================================

本节导读
-----------------------------------

本节将会展示新的bin_loader加载测例到进程的方式，并且展示我们的shell测例是如何运行的。

新的bin_loader
------------------------------------------------------------------------

exec会调用bin_loader,将对应文件名的测例加载到指定的进程p之中。请结合注释理解 bin_loader 的变化：

.. code-block:: c
    :linenos:

    int bin_loader(uint64 start, uint64 end, struct proc *p)
    {
        void *page;
        // 注意现在我们不要求对其了，代码的核心逻辑还是把 [start, end) 
        // 映射到虚拟内存的 [BASE_ADDRESS, BASE_ADDRESS + length)
        uint64 pa_start = PGROUNDDOWN(start);
        uint64 pa_end = PGROUNDUP(end);
        uint64 length = pa_end - pa_start;
        uint64 va_start = BASE_ADDRESS;
        uint64 va_end = BASE_ADDRESS + length;
        // 不再一次 map 很多页面，而是逐页 map，为什么？
        for (uint64 va = va_start, pa = pa_start; pa < pa_end;
            va += PGSIZE, pa += PGSIZE) {
            // 这里我们不会直接映射，而是新分配一个页面，然后使用 memmove 进行拷贝
            // 这样就不会有对其的问题了，但为何这么做其实有更深层的原因。
            page = kalloc();
            memmove(page, (const void *)pa, PGSIZE);
            // 这个 if 就是为了防止 start end 不对其导致拷贝了多余的内核数据
            // 我们需要手动把它们清空
            if (pa < start) {
                memset(page, 0, start - va);
            } else if (pa + PAGE_SIZE > end) {
                memset(page + (end - pa), 0, PAGE_SIZE - (end - pa));
            }
            mappages(p->pagetable, va, PGSIZE, (uint64)page, PTE_U | PTE_R | PTE_W | PTE_X);
        }
        // 同 lab4 map user stack
        p->ustack = va_end + PAGE_SIZE;
        for (uint64 va = p->ustack; va < p->ustack + USTACK_SIZE;
            va += PGSIZE) {
            page = kalloc();
            memset(page, 0, PGSIZE);
            mappages(p->pagetable, va, PGSIZE, (uint64)page, PTE_U | PTE_R | PTE_W);
        }
        // 设置 trapframe
        p->trapframe->sp = p->ustack + USTACK_SIZE;
        p->trapframe->epc = va_start;
        p->max_page = PGROUNDUP(p->ustack + USTACK_SIZE - 1) / PAGE_SIZE;
        p->state = RUNNABLE;
        return 0;
    }

其中，对于用户栈、trapframe、trampoline 的映射没有变化，但是对 .bin 数据的映射似乎面目全非了，竟然由一个循环完成。其实，这个循环的逻辑十分简单，就是对于 .bin 的每一页，都申请一个新页并进行内容拷贝，最后建立这一页的映射。之所以这么麻烦完全是由于我们的物理内存管理过于简陋，一次只能分配一个页，如果能够分配连续的物理页，那么这个循环可以被一个 mappages 替代。

那么另一个更重要的问题是，为什么要拷贝呢？想想 lab4 我们是怎么干的，直接把虚存和物理内存映射就好了，根本没有拷贝。那么，拷贝是为了什么呢？其实，按照 lab4 的做法，程序运行之后就会修改仅有一份的程序"原像"，你会发现，lab4 的程序都是一次性的，如果第二次执行，会发现 .data 和 .bss 段数据都被上一次执行改掉了，不是初始化的状态。但是 lab4 的时候，每个程序最多执行一次，所以这么做是可以的。但在 lab5 所有程序都可能被无数次的执行，我们就必须对“程序原像”做保护，在“原像”的拷贝上运行程序了。

测例的执行
------------------------------------------------------------------------

从本章开始，大家可以发现我们的 run_all_app 函数被 load_init_app 取代了:

.. code-block:: c
    :linenos:

    // os/loader.c

    // load all apps and init the corresponding `proc` structure.
    int load_init_app()
    {
        int id = get_id_by_name(INIT_PROC);
        if (id < 0)
            panic("Cannpt find INIT_PROC %s", INIT_PROC);
        struct proc *p = allocproc();
        if (p == NULL) {
            panic("allocproc\n");
        }
        debugf("load init proc %s", INIT_PROC);
        loader(id, p);
        return 0;
    }

这个 load_init_app load 的 INIT_PROC 一般来说就是我们在本章第一节展示的那个 usershell，不过可以通过在 Makefile 中传入 INIT_PROC 参数而改变，大部分情况下，不推荐修改，这是由于 usershell 具有不错的灵活性。


usershell
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``user/src/usershell.c`` 就是 usershell 的代码了，有兴趣的同学可以研究下这个 shell:

.. code-block:: c

    const unsigned char LF = 0x0a;
    const unsigned char CR = 0x0d;
    const unsigned char DL = 0x7f;
    const unsigned char BS = 0x08;

    // 手搓了一个极简的 stack，用来维护用户输入，保存一行的输入
    char line[100] = {};
    int top = 0;
    void push(char c){ line[top++] = c; }
    void pop() { --top; }
    int is_empty() { return top == 0;}
    void clear() { top = 0; }

    int main()
    {
        printf("C user shell\n");
        printf(">> ");
        fflush(stdout);
        while (1) {
            char c = getchar();
            switch (c) {
            // 回车，执行当前 stack 中字符串对应的程序
            case LF:
            case CR:
                printf("\n");
                if (!is_empty()) {
                    push('\0');
                    int pid = fork();
                    if (pid == 0) {
                        // child process
                        if (exec(line, NULL) < 0) {
                            printf("no such program: %s\n",
                                line);
                            exit(0);
                        }
                        panic("unreachable!");
                    } else {
                        int xstate = 0;
                        int exit_pid = 0;
                        exit_pid = waitpid(pid, &xstate);
                        assert(pid == exit_pid);
                        printf("Shell: Process %d exited with code %d\n",
                            pid, xstate);
                    }
                    clear();
                }
                printf(">> ");
                fflush(stdout);
                break;
            // 退格建，pop一个char
            case BS:
            case DL:
                if (!is_empty()) {
                    putchar(BS);
                    printf(" ");
                    putchar(BS);
                    fflush(stdout);
                    pop();
                }
                break;
            // 普通输入，回显并 push 一个 char
            default:
                putchar(c);
                fflush(stdout);
                push(c);
                break;
            }
        }
        return 0;
    }


可以看到这个测例实际上就是实现了一个简单的字符串处理的函数，并且针对解析得到的不同的指令调用不同的系统调用。要注意这需要shell支持read的系统调用。当读入用户的输入时，它会死循环的等待用户输入一个代表程序名称的字符串(通过sys_read)，当用户按下空格之后，shell 会使用 fork 和 exec 创建并执行这个程序，然后通过 sys_wait 来等待程序执行结束，并输出 exit_code。有了 shell 之后，我们可以只执行自己希望的程序，也可以执行某一个程序很多次来观察输出，这对于使用体验是极大的提升！可以说，第五章的所有努力都是为了支持 shell。

我们简单看一下sys_read的实现，它与 sys_write 有点相似：

.. code-block:: c

    uint64 sys_read(int fd, uint64 va, uint64 len)
    {
        if (fd != STDIN)
            return -1;
        struct proc *p = curr_proc();
        char str[MAX_STR_LEN];
        len = MIN(len, MAX_STR_LEN);
        for (int i = 0; i < len; ++i) {
            // consgetc() 会阻塞式的等待读取一个 char
            int c = consgetc();
            str[i] = c;
        }
        copyout(p->pagetable, va, str, len);
        return len;
    }

目前我们只支持标准输入stdin的输入（对应fd = STDIN）。