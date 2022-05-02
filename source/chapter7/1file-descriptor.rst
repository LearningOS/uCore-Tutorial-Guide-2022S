文件系统扩充
===========================================

管道的文件抽象
-------------------------------------------

.. chyyuu 可以简单介绍一下文件的起源???

在Unix操作系统之前，大多数的操作系统提供了各种复杂且不规则的设计实现来处理各种I/O设备（也可称为I/O资源），如键盘、显示器、以磁盘为代表的存储介质、以串口为代表的通信设备等，使得应用程序开发繁琐且很难统一表示和处理I/O设备。随着UNIX的诞生，一个简洁优雅的I/O设备的抽象出现了，这就是 **文件** 。在 Unix 操作系统中，”**一切皆文件**“ (Everything is a file) 是一种重要的设计思想，这种设计思想继承于 Multics 操作系统的 **通用性** 的设计理念，并进行了进一步的简化。在本章中，应用程序看到并被操作系统管理的 **文件** (File) 就是一系列的字节组合。操作系统不关心文件内容，只关心如何对文件按字节流进行读写的机制，这就意味着任何程序可以读写任何文件（即字节流），对文件具体内容的解析是应用程序的任务，操作系统对此不做任何干涉。例如，一个Rust编译器可以读取一个C语言源程序并进行编译，操作系统并并不会阻止这样的事情发生。


有了文件这样的抽象后，操作系统内核就可把能读写的I/O资源按文件来进行管理，并把文件分配给进程，让进程以统一的文件访问接口与I/O 资源进行交互。在我们目前涉及到的I/O硬件设备中，大致可以分成以下几种：

- **键盘设备** 是程序获得字符输入的一种设备，也可抽象为一种只读性质的文件，可以从这个文件中读出一系列的字节序列；
- **屏幕设备** 是展示程序的字符输出结果的一种字符显示设备，可抽象为一种只写性质的文件，可以向这个文件中写入一系列的字节序列，在显示屏上可以直接呈现出来；
- **串口设备** 是获得字符输入和展示程序的字符输出结果的一种字符通信设备，可抽象为一种可读写性质的文件，可以向这个文件中写入一系列的字节序列传给程序，也可把程序要显示的字符传输出去；还可以把这个串口设备拆分成两个文件，一个用于获取输入字符的只读文件和一个传出输出字符的只写文件。


在QEMU模拟的RV计算机和K210物理硬件上存在串口设备，操作系统通过串口设备的输入侧连接到了同学使用的计算机的键盘设备，而串口设备的输出侧这连接到了同学使用的计算机的显示器窗口上。由于RustSBI直接管理了串口设备，并给操作系统提供了两个SBI接口，从而使得操作系统可以很简单地通过这两个SBI接口输出或输入字符。

文件是提供给应用程序用的，但有操作系统来进行管理。虽然文件可代表很多种不同类型的I/O 资源，但是在进程看来，所有文件的访问都可以通过一个很简洁的统一抽象接口 ``File`` 来进行。我们看一下我们OS框架对文件结构的扩充：

.. code-block:: c

    // file.h
    struct file {
        enum { FD_NONE = 0, FD_PIPE, FD_INODE, FD_STDIO } type;
	    int ref; // reference count
	    char readable;
	    char writable;
	    struct pipe *pipe; // FD_PIPE
	    struct inode *ip; // FD_INODE
	    uint off;
    };
    



pipe管道的实现
--------------------------------------------

管道是一种进程间通信的方式。它允许管道两端的进程互相传递信息。我们OS框架对于pipe的设计十分简单: 找一块空闲内存作为 pipe 的 data buffer，两端的进程对 pipe 的读写就转化为了对这块内存的读写。虽然逻辑十分简单，但是进程读写管道实际还是通过sys_write和sys_read来实现的。sys_write 还同时需要完成屏幕输出，一个程序还可以拥有多个 pipe，而且 pipe 还要能够使得其他程序可见来完成进程通讯的功能，对每个 pipe 还要维护一些状态来记录上一次读写到的位置和 pipe 实际可读的 size等。因此我们也需要关注一下我们OS pipe实现的细节。


首先，看一下管道的结构体。

.. code-block:: c

    // file.h，抽象成一个文件了。
    #define PIPESIZE 512

    struct pipe {
        char data[PIPESIZE];
        uint nread;     // number of bytes read
        uint nwrite;    // number of bytes written
        int readopen;   // read fd is still open
        int writeopen;  // write fd is still open
    };

可以看到，管道把数据存在了一个char数组的缓存之中来维护。这里我们以ring buffer的形式管理管道的data buffer。

我们来看一下如何创建一个管道。

.. code-block:: c

    :linenos:

    int pipealloc(struct file *f0, struct file *f1)
    {
        // 这里没有用预分配，由于 pipe 比较大，直接拿一个页过来，也不算太浪费
        struct pipe *pi = (struct pipe*)kalloc();
        // 一开始 pipe 可读可写，但是已读和已写内容为 0
        pi->readopen = 1;
        pi->writeopen = 1;
        pi->nwrite = 0;
        pi->nread = 0;
    
        // 两个参数分别通过 filealloc 得到，把该 pipe 和这两个文件关连，一端可读，一端可写。读写端控制是 sys_pipe 的要求。
        f0->type = FD_PIPE;
        f0->readable = 1;
        f0->writable = 0;
        f0->pipe = pi;
    
        f1->type = FD_PIPE;
        f1->readable = 0;
        f1->writable = 1;
        f1->pipe = pi;
        return 0;
    }

.. note::

    在内核中，我们是不能 new 一个结构体的，这是由于我们没有实现堆内存管理。但我们可以用一种略显浪费的方式，也就是直接 kalloc() 一个页，只要不大于一整个页的数据结构都可以这样 new 出来。

管道两端的输入和输出被我们抽象成了两个文件。这两个文件的创建由sys_pipe调用完成。我们在分配时就会设置管道两端哪一端可写哪一端可读，并初始化管道本身的nread和nwrite记录buffer的指针。

关闭pipe比较简单。函数其实只关闭了读写端中的一个，如果两个都被关闭，释放 pipe。

.. code-block:: c

    :linenos:

    void pipeclose(struct pipe *pi, int writable)
    {
        if(writable){
            pi->writeopen = 0;
        } else {
            pi->readopen = 0;
        }
        if(pi->readopen == 0 && pi->writeopen == 0){
            kfree((char*)pi);
        }
    }

重点是管道的读写.

.. code-block:: c

    :linenos:

    int pipewrite(struct pipe *pi, uint64 addr, int n)
    {
        // w 记录已经写的字节数
        int w = 0;
        struct proc *p = curr_proc();
        while(w < n){
            // 若不可读，写也没有意义
            if(pi->readopen == 0){
                return -1;
            }

            if(pi->nwrite == pi->nread + PIPESIZE){
                // pipe write 端已满，阻塞
                yield();
            } else {
                // 一次读的 size 为 min(用户buffer剩余，pipe 剩余写容量，pipe 剩余线性容量)
                uint64 size = MIN(
                    n - w, 
                    pi->nread + PIPESIZE - pi->nwrite, 
                    PIPESIZE - (pi->nwrite % PIPESIZE)
                );
                // 使用 copyin 读入用户 buffer 内容 
                copyin(p->pagetable, &pi->data[pi->nwrite % PIPESIZE], addr + w, size);
                pi->nwrite += size;
                w += size;
            }
        }
        return w;
    }

    int piperead(struct pipe *pi, uint64 addr, int n)
    {
        // r 记录已经写的字节数
        int r = 0;
        struct proc *p = curr_proc();
        // 若 pipe 可读内容为空，阻塞或者报错
        while(pi->nread == pi->nwrite) {
            if(pi->writeopen)
                yield();
            else
                return -1;
        }
        while(r < n && size != 0) {
            // pipe 可读内容为空，返回
            if(pi->nread == pi->nwrite)
                break;
            // 一次写的 size 为：min(用户buffer剩余，可读内容，pipe剩余线性容量)
            uint64 size = MIN(
                n - r, 
                pi->nwrite - pi->nread, 
                PIPESIZE - (pi->nread % PIPESIZE)
            );
            // 使用 copyout 写用户内存
            copyout(p->pagetable, addr + r, &pi->data[pi->nread % PIPESIZE], size);
            pi->nread += size;
            r += size;
        }
        return r;
    }

由于我们的管道是由ring buffer形式来管理的，其本身的容量只有PAGESIZE大小，因此需要使用nread和nwrite两个指针来记录当前两端分别写到哪里了（它们的绝对值可以大于PAGESIZE，关键是两者的差值）。由于必须写了才能读，因此有关系 nwrite >= nread。相等意味着当前已经读完了，就退出piperead。如果nwrite - nread == PAGESIZE 则说明已经写满了整个PAGESIZE，不能再写了，会覆盖住没读的部分。如果能写入，就会将数据写入data之中，注意由于是环形，如果nwrite % PAGESIZE != 0并且当前指针位置到环尾写不下要写入的数据,会从环头继续写.大家可以仔细阅读write的实现。

pipe 相关系统调用
--------------------------------------------

首先是sys_pipe.

.. code-block:: c

    :linenos:

    // os/syscall.c
    uint64 sys_pipe(uint64 fdarray) {
        struct proc *p = curr_proc();
        // 申请两个空 file
        struct file* f0 = filealloc();
        struct file* f1 = filealloc();
        // 实际分配一个 pipe，与两个文件关联
        pipealloc(f0, f1);
        // 分配两个 fd，并将之与 文件指针关联
        fd0 = fdalloc(f0);
        fd1 = fdalloc(f1);
        size_t PSIZE = sizeof(fd0);
        copyout(p->pagetable, fdarray, &fd0, sizeof(fd0));
        copyout(p->pagetable, fdarray + sizeof(uint64), &fd1, sizeof(fd1));
        return 0;
    }

这个系统调用完成了创建一个pipe并记录下两端对应file的功能。并把对应的fd写入user传入的数组地址之中传回user态。

sys_close比较简单。就只是释放掉进程的fd并且清空对应file，并且设置其种类为FD_NONE.

.. code-block:: c

    :linenos:

    uint64 sys_close(int fd)
    {
        // 目前不支持 stdio 的关闭，ch7会支持这个
        if (fd <= 2 || fd > FD_BUFFER_SIZE)
            return -1;
        struct proc *p = curr_proc();
        struct file *f = p->files[fd];
        // 目前仅支持关闭 pipe
        if (f->type == FD_PIPE) {
            fileclose(f);
        } else {
            panic("fileclose: unsupported file type %d fd = %d\n", f->type, fd);
        }
        p->files[fd] = 0;
        return 0;
    }

    void fileclose(struct file *f)
    {
        // ref == 0 才真正关闭
        if(--f->ref > 0) {
            return;
        }
        // pipe 类型需要关闭对应的 pipe
        if(f->type == FD_PIPE){
            pipeclose(f->pipe, f->writable);
        }
        // 清空其他数据
        f->off = 0;
        f->readable = 0;
        f->writable = 0;
        f->ref = 0;
        f->type = FD_NONE;
    }

原来的 sys_write 更名为 console_write，新 sys_write 根据文件类型分别调用 console_write 和 pipe_write。sys_read 同理。具体的区分是通过判断fd来进行的。

.. code-block:: c
    :linenos:

    uint64 sys_write(int fd, uint64 va, uint64 len)
    {
        if (fd == STDOUT || fd == STDERR) {
            return console_write(va, len);
        }
        if (fd <= 2 || fd > FD_BUFFER_SIZE)
            return -1;
        struct proc *p = curr_proc();
        struct file *f = p->files[fd];
        if (f->type == FD_PIPE) {
            return pipewrite(f->pipe, va, len);
        } else {
            panic("unknown file type %d\n", f->type);
        }
    }

    uint64 sys_read(int fd, uint64 va, uint64 len)
    {
        if (fd == STDIN) {
            return console_read(va, len);
        }
        if (fd <= 2 || fd > FD_BUFFER_SIZE)
            return -1;
        struct proc *p = curr_proc();
        struct file *f = p->files[fd];
        if (f->type == FD_PIPE) {
            return piperead(f->pipe, va, len);
        } else {
            panic("unknown file type %d fd = %d\n", f->type, fd);
        }
    }

注意一个文件目前fd最大就是15。