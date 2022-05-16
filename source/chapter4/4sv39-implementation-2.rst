SV39多级页表机制：OS实现
========================================================


本节导读
--------------------------


本节我们将讲述OS是如何实现页表的支持的。在深入本章的内容之前，大家一定要牢记，完成虚拟地址查询页表或TLB转换成物理地址的过程是由硬件，也就是CPU来完成的（MMU）。我们在框架之中实现的地址转换函数是为了我们在某些函数中自己计算虚拟地址到物理地址转换使用的。OS负责对页表进行建立、更改等处理，真正在程序运行时，CPU对指令、数据虚拟地址会十分机械地按照下面讲述的方法使用os创建好的页表进行地址转换。

地址相关的数据结构抽象
-----------------------------------

正如本章第一小节所说，在分页内存管理中，地址转换的核心任务在于如何维护虚拟页号到物理页号的映射——也就是页表。我们对页表的操作集中在vm.c文件之中。

首先是为了实现页表，我们新增的类型的定义：

.. code-block:: c
    
    // os/types.h

    typedef uint64 pte_t;
    typedef uint64 pde_t;
    typedef uint64 *pagetable_t;// 512 PTEs

第一小节中我们提到，在页表中以虚拟页号作为索引不仅能够查到物理页号，还能查到一组保护位，它控制了应用对地址空间每个
虚拟页面的访问权限。但实际上还有更多的标志位，物理页号和全部的标志位以某种固定的格式保存在一个结构体中，它被称为 
**页表项** (PTE, Page Table Entry) ，是利用虚拟页号在页表中查到的结果。

.. image:: sv39-pte.png

上图为 SV39 分页模式下的页表项，其中 :math:`[53:10]` 这 :math:`44` 位是物理页号，最低的 :math:`8` 位 
:math:`[7:0]` 则是标志位，它们的含义如下（请注意，为方便说明，下文我们用 *页表项的对应虚拟页面* 来表示索引到
一个页表项的虚拟页号对应的虚拟页面）：

- 仅当 V(Valid) 位为 1 时，页表项才是合法的；
- R/W/X 分别控制索引到这个页表项的对应虚拟页面是否允许读/写/取指；
- U 控制索引到这个页表项的对应虚拟页面是否在 CPU 处于 U 特权级的情况下是否被允许访问；
- G 我们暂且不理会；
- A(Accessed) 记录自从页表项上的这一位被清零之后，页表项的对应虚拟页面是否被访问过；
- D(Dirty) 则记录自从页表项上的这一位被清零之后，页表项的对应虚拟页表是否被修改过。

由于pte只有54位，每一个页表项我们用一个8字节的无符号整型来记录就已经足够。

页表实现va-->pa的转换过程
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

下面我们通过来解读我们OS的walk函数了解SV39的页表读取机制。

.. code-block:: c

    :linenos:

    // os/vm.c

    pte_t *
    walk(pagetable_t pagetable, uint64 va, int alloc) {
        if (va >= MAXVA)
            panic("walk");

        for (int level = 2; level > 0; level--) {
            pte_t *pte = &pagetable[PX(level, va)];
            if (*pte & PTE_V) {
                pagetable = (pagetable_t) PTE2PA(*pte);
            } else {
                if (!alloc || (pagetable = (pde_t *) kalloc()) == 0)
                    return 0;
                memset(pagetable, 0, PGSIZE);
                *pte = PA2PTE(pagetable) | PTE_V;
            }
        }
        return &pagetable[PX(0, va)];
    }

walk函数模拟了CPU进行MMU的过程。它的参数分别是页表，待转换的虚拟地址va，以及如果没有对应的物理地址时是否分配物理地址。
SV39的转换是由3级页表结构完成。在riscv.h之中定义的宏函数PX完成了每一级从va转换到pte的过程:

.. code-block:: c

    :linenos:

    #define PXMASK 0x1FF// 9 bits
    #define PGSHIFT 12// bits of offset within a page
    #define PXSHIFT(level) (PGSHIFT + (9 * (level)))
    #define PX(level, va) ((((uint64)(va)) >> PXSHIFT(level)) & PXMASK)

可以看到，每一次我们只需要截取队va高27位中对应级别的9位即可。一开始截取最高9位，接着是中间的9位和低9位。这9位我们如何使用呢？SV39中要求我们把页表的44位和虚拟地址对应的9位*8直接拼接在一起做为pte的地址。页表的高44位（也就是页号）拼接上12位的0实际上就是pagetable指向的物理地址。我们可以计算得到一个4096大小的页表之中有4096/8=512个页表项。因此我们得到的这9位实际上就是pte在这一页之中的偏移，也就是其下标了。

得到了页表项之后，我们使用PTE2PA函数将该页表项的高44位（也就是下一个页表的页号）取出和12个0拼接（通过左移和右移可以轻松实现），就得到了下一级页表的起始物理地址了。接着重复这样的操作，直到最后一个pte解析出来，就可以返回最后一个pte了（循环并没有处理最后一级）。最后一个pte中记录了物理地址的物理页号PPN，将它直接和虚拟地址的12位offset拼接就得到了对应的物理地址pa。

整个过程中要注意随时通过PTE的标志位判断每一级的pte是否是有效的（V位）。如果无效则需要kalloc分配一个新的页表，并初始化该pte在其中的位置。如果alloc参数=0或者已经没有空闲的内存了（这个情况在lab8之前不会遇到），那么遇到中途V=0的pte整个walk过程就会直接退出。当然这是OS的写法，如果CPU在MMU的时候遇到这种情况就会直接报异常了。

walk函数是我们比较底层的一个函数，但也是所有遍历页表进行地址转换函数的基础。我们还实现了两个转换函数:

.. code-block:: c

    :linenos:

    // Look up a virtual address, return the physical page,
    // or 0 if not mapped.
    // Can only be used to look up user pages.
    // Use `walk`
    uint64 walkaddr(pagetable_t pagetable, uint64 va);

    // Look up a virtual address, return the physical address.
    // Use `walkaddr`
    uint64 useraddr(pagetable_t pagetable, uint64 va);

大家可以自行阅读。注意walkaddr函数没有考虑偏移量，因此在使用的时候请首先考虑useraddr函数。

页表的建立过程
-----------------------------------

无论是CPU进行MMU，还是我们自己walk实现va到pa的转换需要的页表都是需要OS来生成的。相关函数也是本章练习涉及到的主要函数。

.. code-block:: c
    :linenos:

    // Create PTEs for virtual addresses starting at va that refer to
    // physical addresses starting at pa. va and size might not
    // be page-aligned. Returns 0 on success, -1 if walk() couldn't
    // allocate a needed page-table page.
    int mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm);

    // Remove npages of mappings starting from va. va must be
    // page-aligned. The mappings must exist.
    // Optionally free the physical memory.
    void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free);

以上是建立新映射和取消映射的函数，mappages 在 pagetable 中建立 [va, va + size) 到 [pa, pa + size) 的映射，页表属性为perm，uvmunmap 则取消一段映射，do_free 控制是否 kfree 对应的物理内存（比如这是一个共享内存，那么第一次 unmap 就不 free，最后一个 unmap 肯定要 free）。

mappages的perm是用于控制页表项的flags的。请注意它具体指向哪几位，这将极大地影响页表的可用性。因为CPU进行MMU的时候一旦权限出错，比如CPU在U态访问了flag之中U=0的页表项是会直接报异常的。

启用页表后的跨页表操作
-----------------------------------

一旦启用了页表之后，U态的测例程序就开始全部使用虚拟地址了。这就意味着它传给OS的指针参数也是虚拟地址，我们无法直接去读虚拟地址，而是要将它使用对应进程的页表转换成物理地址才能读取。

为了方便大家，我们预先准备了几个跨页表进行字符串数据交换的函数。

.. code-block:: c

    :linenos:

    // Copy from kernel to user.
    // Copy len bytes from src to virtual address dstva in a given page table.
    // Return 0 on success, -1 on error.
    int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len);

    // Copy from user to kernel.
    // Copy len bytes to dst from virtual address srcva in a given page table.
    // Return 0 on success, -1 on error.
    int copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len);

    // Copy a null-terminated string from user to kernel.
    // Copy bytes to dst from virtual address srcva in a given page table,
    // until a '\0', or max.
    // Return 0 on success, -1 on error.
    int copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max);

用于与指定页表进行数据交换，copyout 可以向页表中写东西，后续用于 sys_read，也就是给用户传数据，copyin 用户接受用户的 buffer，也就是从用户哪里读数据。
注意，用户在启用了虚拟内存之后，用户 syscall 给出的指针是不能直接用的，因为与内核的映射不一样，会读取错误的物理地址，使用指针必须通过 useraddr 转化，当然，更加推荐的是 copyin/out 接口，否则很可能损坏内存数据，同时，copyin/out 接口处理了虚存跨页的情况，useraddr 则需要手动判断并处理。跨页会在测例文件bin比较大的时候出现。如果你的程序出现了完全De不出来的BUG，可能就是跨页+使用了错误的接口导致的。

内核页表
-----------------------------------

开启页表之后，内核也需要进行映射处理。但是我们这里可以直接进行一一映射，也就是va经过MMU转换得到的pa就是va本身（但是转换过程还是会执行！）。内核需要能访问到所有的物理内存以处理频繁的操作不同进程内存的需求。内核页表建立过程在main函数之中调用。

.. code-block:: c

    :linenos:

    #define PTE_V (1L << 0)     // valid
    #define PTE_R (1L << 1)
    #define PTE_W (1L << 2)
    #define PTE_X (1L << 3)
    #define PTE_U (1L << 4)     // 1 -> user can access

    #define KERNBASE (0x80200000)
    extern char e_text[];     // kernel.ld sets this to end of kernel code.
    extern char trampoline[];

    pagetable_t kvmmake(void) {
        pagetable_t kpgtbl;
        kpgtbl = (pagetable_t) kalloc();
        memset(kpgtbl, 0, PGSIZE);
        kvmmap(kpgtbl, KERNBASE, KERNBASE, (uint64) e_text - KERNBASE, PTE_R | PTE_X);
        kvmmap(kpgtbl, (uint64) e_text, (uint64) e_text, PHYSTOP - (uint64) e_text, PTE_R | PTE_W);
        kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
        return kpgtbl;
    }

    void kvmmap(pagetable_t kpgtbl, uint64 va, uint64 pa, uint64 sz, int perm)
    {
        if (mappages(kpgtbl, va, sz, pa, perm) != 0)
            panic("kvmmap");
    }

用户页表的加载
-----------------------------------

用户的加载逻辑在 loader.c 中，其中唯一逻辑变化较大的就是 bin_loader 函数，请结合注释理解这个函数：

.. code-block:: c

    :linenos:

    pagetable_t bin_loader(uint64 start, uint64 end, struct proc *p)
    {
        // pg 代表根页表地址
        pagetable_t pg;
        // 根页表大小恰好为 1 个页
        pg= (pagetable_t)kalloc();
        if (pg == 0) {
            errorf("uvmcreate: kalloc error");
            return 0;
        }
        // 注意 kalloc() 分配的页为脏页，这里需要先清空。
        memset(pagetable, 0, PGSIZE);
        // 映射 trapoline（也就是 uservec 和 userret 的代码），注意这里的权限!
        if (mappages(pagetable, TRAMPOLINE, PAGE_SIZE, (uint64)trampoline,
                PTE_R | PTE_X) < 0) {
            kfree(pagetable);
            errorf("uvmcreate: mappages error");
            return 0;
        }
        // 映射 trapframe（中断帧），注意这里的权限!
        if (mappages(pg, TRAPFRAME, PGSIZE, (uint64)p->trapframe,
                PTE_R | PTE_W) < 0) {
            panic("mappages fail");
        }
        
        // 接下来映射用户实际地址空间，也就是把 physics address [start, end)　
        // 映射到虚拟地址 [BASE_ADDRESS, BASE_ADDRESS + length)

        // riscv 指令有对齐要求，同时,如果不对齐直接映射的话会把部分内核地址映射到用户态，很不安全
        // ch5我们就不需要这个限制了。
        if (!PGALIGNED(start)) {
            // Fix in ch5
            panic("user program not aligned, start = %p", start);
        }
        end = PGROUNDUP(end);
        // 实际的 map 指令。
        uint64 length = end - start;
        if (mappages(pg, BASE_ADDRESS, length, start,
                PTE_U | PTE_R | PTE_W | PTE_X) != 0) {
            panic("mappages fail");
        }
        p->pagetable = pg;
        // 接下来 map user stack， 注意这里的虚存选择，想想为何要这样？
        uint64 ustack_bottom_vaddr = BASE_ADDRESS + length + PAGE_SIZE;
        mappages(pg, ustack_bottom_vaddr, USTACK_SIZE, (uint64)kalloc(),
            PTE_U | PTE_R | PTE_W | PTE_X);
        p->ustack = ustack_bottom_vaddr;
        // 设置 trapframe
        p->trapframe->epc = BASE_ADDRESS;
        p->trapframe->sp = p->ustack + USTACK_SIZE;
        // exit 的时候会 unmap 页表中 [BASE_ADDRESS, max_page * PAGE_SIZE) 的页
        p->max_page = PGROUNDUP(p->ustack + USTACK_SIZE - 1) / PAGE_SIZE;
        return pg;
    }

这里大家也要注意，每一个测例进程都有一套自己的页表。因此在进程切换或者异常中断处理返回U态的时候需要设置satp的值为其对应的值才能使用正确的页表。具体的实现其实之前几章已经先做好了。

我们需要重点关注一下trapframe 和 trampoline 代码的位置。在前面两节我们看到了memory_layout文件。这两块内存用户特权级切换，必须用户态和内核态都能访问。所以它们在内核和用户页表中都有 map，注意所有 kalloc() 分配的内存内核都能访问，这是因为我们已经预先设置好页表了。

.. code-block:: c
    :linenos:

    #define USER_TOP (MAXVA)
    #define TRAMPOLINE (USER_TOP - PGSIZE)
    #define TRAPFRAME (TRAMPOLINE - PGSIZE)

至于为何要这么设定，留给读者思考。
