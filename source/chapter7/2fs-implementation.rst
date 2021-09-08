nfs文件系统
=======================================

本节导读
---------------------------------------

本节我们简单介绍一下我们实现的nfs操作系统。本章中新增加的代码很多，但是大家如果想研读的话理解起来不太困难。很多函数只看名字也可以知道其功效，不需要再探究其实现的方式。

文件系统布局
---------------------------------------

导言中提到，我们的nfs文件系统十分类似ext4文件系统，下面我们可以看一下nfs文件系统的布局::
    // 基本信息：块大小 BSIZE = 1024B，总容量 FSSIZE = 1000 个 block = 1000 * 1024 B。
    // Layout:
    // 0号块留待后续拓展，可以忽略。superblock 固定为 1 号块，size 固定为一个块。
    // 其后是储存 inode 的若干个块，占用块数 = inode 上限 / 每个块上可以容纳的 inode 数量，
    // 其中 inode 上限固定为 200，每个块的容量 = BSIZE / sizeof(struct disk_inode)
    // 再之后是数据块相关内容，包含一个 储存空闲块位置的 bitmap 和 实际的数据块，bitmap 块
    // 数量固定为 NBITMAP = FSSIZE / (BSIZE * 8) + 1 = 1000 / 8 + 1 = 126 块。
    // [ boot block | sb block | inode blocks | free bit map | data blocks ]

注意：不推荐同学们修改该布局，除非你完全看懂了 fs 的逻辑，所以最好不要改变 disk_inode 这个结构的大小，如果想要增删字段，一定使用 pad。这个布局具体定义的位置在nfs/fs.c之中。

我们定义的inode和data blocks实际上和ext4中同名的结构功能几乎是一样的。**索引节点** (Inode, Index Node) 是文件系统中的一种重要数据结构。逻辑目录树结构中的每个文件和目录都对应一个 inode ，我们前面提到的在文件系统实现中文件/目录的底层编号实际上就是指 inode 编号。在 inode 中不仅包含了我们通过 ``stat`` 工具能够看到的文件/目录的元数据（大小/访问权限/类型等信息），还包含实际保存对应文件/目录数据的数据块（位于最后的数据块区域中）的索引信息，从而能够找到文件/目录的数据被保存在磁盘的哪些块中。从索引方式上看，同时支持直接索引和间接索引。

下面我们看一下它们在我们C中对应的具体结构体:

.. code-block:: c

    // 超级块位置固定，用来指示文件系统的一些元数据，这里最重要的是 inodestart 和 bmapstart
    struct superblock {
        uint magic;     // Must be FSMAGIC
        uint size;      // Size of file system image (blocks)
        uint nblocks;   // Number of data blocks
        uint ninodes;   // Number of inodes.
        uint inodestart;// Block number of first inode block
        uint bmapstart; // Block number of first free map block
    };

    // On-disk inode structure
    // 储存磁盘 inode 信息，主要是文件类型和数据块的索引，其大小影响磁盘布局，不要乱改，可以用 pad
    struct dinode {
        short type;             // File type
        short pad[3];
        uint size;              // Size of file (bytes)
        uint addrs[NDIRECT + 1];// Data block addresses
    };

    // in-memory copy of an inode
    // dinode 的内存缓存，为了方便，增加了 dev, inum, ref, valid 四项管理信息，大小无所谓，可以随便改。
    struct inode {
        uint dev;           // Device number
        uint inum;          // Inode number
        int ref;            // Reference count
        int valid;          // inode has been read from disk?
        short type;         // copy of disk inode
        uint size;
        uint addrs[NDIRECT+1];  // data block num
    };

    // 目录对应的数据块的内容本质是 filename 到 file inode_num 的一个 map，这里为了简单，就存为一个 `dirent` 数组，查找的时候遍历对比
    struct dirent {
        ushort inum;
        char name[DIRSIZ];
    };

    // 数据块缓存结构体。
    struct buf {
        int valid;   // has data been read from disk?
        int disk;    // does disk "own" buf?
        uint dev;
        uint blockno;
        uint refcnt;
        struct buf *prev; // LRU cache list
        struct buf *next;
        uchar data[BSIZE];
    };

注意几个量的概念:
- block num: 表示某一个磁盘块的编号。我们操作数据块会把它读入内存的数据块缓存之中，其结构体见上。
- inode num: 表示某一个 inode 在所有 inode 项里的编号。注意 inode blocks 其实就是一个 inode 的大数组。

同时，目录本身是一个 filename 到 file对应的inode_num的map，可以完成 filename 到 inode_num 的转化。

OS启动后是没有inode的内存缓存的。下面我们自底向上走一遍OS打开已存在在磁盘上文件的过程，让大家熟悉一下nfs的具体实现方式。

virtio 磁盘驱动
---------------------------------------

注意：这一部分代码不需要同学们详细了解细节，但需要知道大概的过程。

在 uCore-Tutorial 中磁盘块的读写是通过中断处理的。在 virtio.h 和 virtio-disk.c 中我们按照 qemu 对 virtio 的定义，实现了 virtio_disk_init 和 virtio_disk_rw 两个函数，前者完成磁盘设备的初始化和对其管理的初始化。virtio_disk_rw 实际完成磁盘IO，当设定好读写信息后会通过 MMIO 的方式通知磁盘开始写。然后，os 会开启中断并开始死等磁盘读写完成。当磁盘完成 IO 后，磁盘会触发一个外部中断，在中断处理中会把死循环条件解除。内核态只会在处理磁盘读写的时候短暂开启中断，之后会马上关闭。

.. code-block:: c

    virtio_disk_rw(struct buf *b, int write) {
        /// ... set IO config
        *R(VIRTIO_MMIO_QUEUE_NOTIFY) = 0; 		// notify the disk to carry out IO
        struct buf volatile * _b = b;   		// Make sure complier will load 'b' form memory
        intr_on();
        while(_b->disk == 1);	// _b->disk == 0 means that this IO is done 
        intr_off();
    }

    // 开启和关闭中断的函数。
    static inline void intr_on() { w_sstatus(r_sstatus() | SSTATUS_SIE); }

    // disable device interrupts
    static inline void intr_off() { w_sstatus(r_sstatus() & ~SSTATUS_SIE); }

对于内核中断处理的修改在trap.c之中。之前我们的trap from kernel会直接panic，现在我们需要添加对外部中断的处理。kerneltrap也需要类似usertrap的保存上下文以及回到原处的kernelvec以及kernelret函数。进入内核之后要单独设置stvec指向kernelvec处。

.. code-block:: riscv64

    # kernelvec.S

    kernelvec:
            // make room to save registers.
            addi sp, sp, -256
            // save the registers expect x0
            sd ra, 0(sp)
            sd sp, 8(sp)
            sd gp, 16(sp)
            // ...
            sd t4, 224(sp)
            sd t5, 232(sp)
            sd t6, 240(sp)

            call kerneltrap

    kernelret:
            // restore registers.
            // 思考：为什么直接就使用了sp？
            ld ra, 0(sp)
            ld sp, 8(sp)
            ld gp, 16(sp)
            // restore all registers expect x0
            ld t4, 224(sp)
            ld t5, 232(sp)
            ld t6, 240(sp)
            addi sp, sp, 256
            sret

kerneltrap具体的修改如下:

.. code-block:: c

    void kerneltrap() {
        // 老三样，不过在这里把处理放到了 C 代码中
        uint64 sepc = r_sepc();
        uint64 sstatus = r_sstatus();
        uint64 scause = r_scause();

        if ((sstatus & SSTATUS_SPP) == 0)
            panic("kerneltrap: not from supervisor mode");

        if (scause & (1ULL << 63)) {
            // 可能发生时钟中断和外部中断，我们的主要目标是处理外部中断
            devintr(scause & 0xff);
        } else {
            // kernel 发生异常就挣扎了，肯定出问题了，杀掉用户线程跑路
            error("invalid trap from kernel: %p, stval = %p sepc = %p\n", scause, r_stval(), sepc);
            exit(-1);
        }
    }

    // 外部中断处理函数
    void devintr(uint64 cause) {
        int irq;
        switch (cause) {
            case SupervisorTimer:
                set_next_timer();
                // 时钟中断如果发生在内核态，不切换进程，原因分析在下面
                // 如果发生在用户态，照常处理
                if((r_sstatus() & SSTATUS_SPP) == 0) {
                    yield();
                }
                break;
            case SupervisorExternal:
                irq = plic_claim();
                if (irq == UART0_IRQ) {		// UART 串口的终端不需要处理，这个 rustsbi 替我们处理好了
                    // do nothing
                } else if (irq == VIRTIO0_IRQ) {	// 我们等的就是这个中断
                    virtio_disk_intr();
                }
                if (irq)
                    plic_complete(irq);		// 表明中断已经处理完毕
                break;
        }
    }

virtio_disk_intr() 会把 buf->disk 置零，这样中断返回后死循环条件解除，程序可以继续运行。具体代码在 virtio-disk.c 中。

这里还需要注意的一点是，为什么始终不允许内核发生进程切换呢？只是由于我们的内核并没有并发的支持，相关的数据结构没有锁或者其他机制保护。考虑这样一种情况，一个进程读写一个文件，内核处理等待磁盘相应时，发生时钟中断切换到了其他进程，然而另一个进程也要读写同一个文件，这就可能发生数据访问上的冲突，甚至导致磁盘出现错误的行为。这也是为什么内核态一直不处理时钟中断，我们必须保证每一次内核的操作都是原子的，不能被打断。大家可以想一想，如果内核可以随时切换，当前有那些数据结构可能被破坏。提示：想想 kalloc 分配到一半，进程 switch 切换到一半之类的。

磁盘块缓存
---------------------------------------

为了加快磁盘访问的速度，在内核中设置了磁盘缓存 struct buf，一个 buf 对应一个磁盘 block，这一部分代码也不要求同学们深入掌握。大致的作用机制是，对磁盘的读写都会被转化为对 buf 的读写，当 buf 有效时，读写 buf，buf 无效时（类似页表缺页和 TLB 缺失），就实际读写磁盘，将 buf 变得有效，然后继续读写 buf。详细的内容在 buf.h 和 bio.c 中。buf 写回的时机是 buf 池满需要替换的时候(类似内存的 swap 策略) 手动写回。如果 buf 没有写回，一但掉电就 GG 了，所以手动写回还是挺重要的。

.. code-block:: c

    // os/bio.c
    struct buf *
    bread(uint dev, uint blockno) {
        struct buf *b;
        b = bget(dev, blockno);
        if (!b->valid) {
            virtio_disk_rw(b, R);
            b->valid = 1;
        }
        return b;
    }

    // Write b's contents to disk.
    void bwrite(struct buf *b) {
        virtio_disk_rw(b, W);
    }

读取文件数据实际就是读取文件inode指向数据块的数据。读数据块到缓存的数据需要使用bread，而写回缓存需要用到bwrite函数。文件系统首先使用bget去查缓存中是否已有对应的block，如果没有会分配内存来缓存对应的块。之后会调用bread/bwrite进行从磁盘读数据块、写回数据块。要注意释放块缓存的brelse函数。

.. code-block:: c
    
    // os/bio.c
    void brelse(struct buf *b) {
        b->refcnt--;
        if (b->refcnt == 0) {
            b->next->prev = b->prev;
            b->prev->next = b->next;
            b->next = bcache.head.next;
            b->prev = &bcache.head;
            bcache.head.next->prev = b;
            bcache.head.next = b;
        }
    }

需要特别注意的是 brelse 不会真的如字面意思释放一个 buf。它的准确含义是暂时不操作该 buf 了并把它放置在bcache链表的首部，buf 的真正释放会被推迟到 buf 池满，无法分配的时候，就会把最近最久未使用的 buf 释放掉（释放 = 写回 + 清空）。这是为了尽可能保留内存缓存，因为读写磁盘真的太太太太慢了。

此外，brelse 的数量必须和 bget 相同，因为 bget 会是的引用计数加一。如果没有相匹配的 brelse，就好比 new 了之后没有 delete。千万注意。

inode的操作
---------------------------------------

现在我们来看看nfs如何读取磁盘上的dinode到内存之中。我们通过file name对应的inode num去从磁盘读取对应的inode。为了解决共享问题（不同进程可以打开同一个磁盘文件），也有一个全局的 inode table，每当新打开一个文件的时候，会把一个空闲的　inode 绑定为对应 dinode 的缓存，这一步通过 iget　完成。

.. code-block:: c

    // 找到 inum 号 dinode 绑定的 inode，如果不存在新绑定一个
    static struct inode *
    iget(uint dev, uint inum) {
        struct inode *ip, *empty;
        // 遍历查找 inode table
        for (ip = &itable.inode[0]; ip < &itable.inode[NINODE]; ip++) {
            // 如果有对应的，引用计数 +1并返回
            if (ip->ref > 0 && ip->dev == dev && ip->inum == inum) {
                ip->ref++;
                return ip;
            }
        }
        // 如果没有对于的，找一个空闲 inode 完成绑定
        empty = find_empty()
        // GG，inode 表满了，果断自杀.lab7正常不会出现这个情况。
        if (empty == 0)
            panic("iget: no inodes");
        // 注意这里仅仅是写了元数据，没有实际读取，实际读取推迟到后面
        ip = empty;
        ip->dev = dev;
        ip->inum = inum;
        ip->ref = 1;
        ip->valid = 0;  // 没有实际读取，valid = 0
        return ip;
    }

当已经得到一个文件对应的 inode 后，可以通过 ivalid 函数确保其是有效的。

.. code-block:: c

    // Reads the inode from disk if necessary.
    void ivalid(struct inode *ip) {
        struct buf *bp;
        struct dinode *dip;
        if (ip->valid == 0) {
            // bread　可以完成一个块的读取，这个在将 buf 的时候说过了
            // IBLOCK 可以计算 inum 在几个 block
            bp = bread(ip->dev, IBLOCK(ip->inum, sb));
            // 得到 dinode 内容
            dip = (struct dinode *) bp->data + ip->inum % IPB;
            // 完成实际读取
            ip->type = dip->type;
            ip->size = dip->size;
            memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
            // buf 暂时没用了
            brelse(bp);
            // 现在有效了
            ip->valid = 1;
        }
    }

在 inode 有效之后，可以通过 writei, readi 完成读写。这又是bwrite和bread的上级接口了。和其他OS支持的文件系统一样，我们首先计算出文件的偏移量，并通过bmap得到对应的block num。之后调用bwrite/bread来进行文件的读写操作。

.. code-block:: c

    // 从 ip 对应文件读取 [off, off+n) 这一段数据到 dst
    int readi(struct inode *ip, char* dst, uint off, uint n) {
        uint tot, m;
        // 还记得 buf 吗？
        struct buf *bp;
        for (tot = 0; tot < n; tot += m, off += m, dst += m) {
            // bmap 完成 off 到 block num 的对应，见下
            bp = bread(ip->dev, bmap(ip, off / BSIZE));
            // 一次最多读一个块，实际读取长度为 m
            m = MIN(n - tot, BSIZE - off % BSIZE);
            memmove(dst, (char*)bp->data + (off % BSIZE), m);
            brelse(bp);
        }
        return tot;
    }

    // 同 readi
    int writei(struct inode *ip, char* src, uint off, uint n) {
        uint tot, m;
        struct buf *bp;

        for (tot = 0; tot < n; tot += m, off += m, src += m) {
            bp = bread(ip->dev, bmap(ip, off / BSIZE));
            m = MIN(n - tot, BSIZE - off % BSIZE);
            memmove(src, (char*)bp->data + (off % BSIZE), m);
            bwrite(bp);
            brelse(bp);
        }

        // 文件长度变长，需要更新 inode 里的 size 字段
        if (off > ip->size)
            ip->size = off;

        // 有可能 inode 信息被更新了，写回
        iupdate(ip);

        return tot;
    }

其中bmap函数是连接inode和block的重要函数。但由于我们支持了间接索引，同时还设计到文件大小的改变，所以也拉出来看看:

.. code-block:: c
    // bn = off / BSIZE
    uint bmap(struct inode *ip, uint bn) {
        uint addr, *a;
        struct buf *bp;
        // 如果 bn < 12，属于直接索引, block num = ip->addr[bn]
        if (bn < NDIRECT) {
            // 如果对应的 addr, 也就是　block num = 0，表明文件大小增加，需要给文件分配新的 data block
            // 这是通过 balloc 实现的，具体做法是在 bitmap 中找一个空闲 block，置位后返回其编号
            if ((addr = ip->addrs[bn]) == 0)    
                ip->addrs[bn] = addr = balloc(ip->dev);
            return addr;
        }
        bn -= NDIRECT;
        // 间接索引块，那么对应的数据块就是一个大　addr 数组。
        if (bn < NINDIRECT) {
            // Load indirect block, allocating if necessary.
            if ((addr = ip->addrs[NDIRECT]) == 0)
                ip->addrs[NDIRECT] = addr = balloc(ip->dev);
            bp = bread(ip->dev, addr);
            a = (uint *) bp->data;
            if ((addr = a[bn]) == 0) {
                a[bn] = addr = balloc(ip->dev);
                bwrite(bp);
            }
            brelse(bp);
            return addr;
        }

        panic("bmap: out of range");
        return 0;
    }

balloc(位于nfs/fs.c)会分配一个新的buf缓存。而iupdate函数则是把修改之后的inode重新写回到磁盘上。不然掉电了就凉了。

.. code-block:: c
    // Copy a modified in-memory inode to disk.
    // Must be called after every change to an ip->xxx field
    // that lives on disk.
    void iupdate(struct inode *ip) {
        struct buf *bp;
        struct dinode *dip;

        bp = bread(ip->dev, IBLOCK(ip->inum, sb));
        dip = (struct dinode *) bp->data + ip->inum % IPB;
        dip->type = ip->type;
        dip->size = ip->size;
        memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
        bwrite(bp);
        brelse(bp);
    }


获取文件对应的inode
---------------------------------------

现在我们回到文件-inode的关系上。我们怎么获取文件对应的inode呢？上文中提到了我们是去查file name对应inode的表来实现这个过程的。这个功能由目录来提供。我们看一下代码是如何实现这个过程的。

首先用户程序要打开指定文件名文件，发起系统调用sys_openat:
.. code-block:: c
    #define O_RDONLY  0x000     // 只读
    #define O_WRONLY  0x001     // 只写
    #define O_RDWR    0x002     // 可读可写
    #define O_CREATE  0x200 　　//　如果不存在，创建
    #define O_TRUNC   0x400 　　// 舍弃原有内容，从头开始写

    uint64 sys_openat(uint64 va, uint64 omode, uint64 _flags) {
        // 还记得flags的定义吗？看看上面。
        struct proc *p = curr_proc();
        char path[200];
        copyinstr(p->pagetable, path, va, 200);
        return fileopen(path, omode);
    }

    int fileopen(char *path, uint64 omode) {
        int fd;
        struct file *f;
        struct inode *ip;
        if (omode & O_CREATE) {
            // 新常见一个路径为 path 的文件
            ip = create(path, T_FILE);
        } else {
            // 尝试寻找一个路径为 path 的文件
            ip = namei(path);
            ivalid(ip);
        }
        // 还记得吗？从全局文件池和进程 fd 池中找一个空闲的出来，参考 lab6
        f = filealloc();
        fd = fdalloc(f);
        // 初始化
        f->type = FD_INODE;
        f->off = 0;
        f->ip = ip;
        f->readable = !(omode & O_WRONLY);
        f->writable = (omode & O_WRONLY) || (omode & O_RDWR);
        if ((omode & O_TRUNC) && ip->type == T_FILE) {
            itrunc(ip);
        }
        return fd;
    }

打开文件的方式根据flags有很多种。我们先来看最简单的，就是打开已经存在的文件的方法。fileopen在处理这类打开时调用了namei这个函数。

.. code-block:: c
    // namei = 获得根目录，然后在其中遍历查找 path
    struct inode *namei(char *path) {
    struct inode *dp = root_dir();
        return dirlookup(dp, path, 0);
    }

    // root_dir 位置固定
    struct inode *root_dir() {
        struct inode* r = iget(ROOTDEV, ROOTINO);
        ivalid(r);
        return r;
    }

    // 便利根目录所有的 dirent，找到 name 一样的 inode。
    struct inode *
    dirlookup(struct inode *dp, char *name, uint *poff) {
        uint off, inum;
        struct dirent de;
        // 每次迭代处理一个 block，注意根目录可能有多个 data block
        for (off = 0; off < dp->size; off += sizeof(de)) {
            readi(dp, 0, (uint64) &de, off, sizeof(de));
            if (strncmp(name, de.name, DIRSIZ) == 0) {
                if (poff)
                    *poff = off;
                inum = de.inum;
                // 找到之后，绑定一个内存 inode 然后返回
                return iget(dp->dev, inum);
            }
        }

        return 0;
    }

由于我们是单目录结构。因此首先我们调用root_dir获取根目录对应的inode。之后就遍历这个inode索引的数据块中存储的文件信息到dirent结构体之中，比较名称和给定的文件名是否一致。dirlookup的逻辑对于我们本章的练习十分重要。

fileopen 还可能会导致文件 truncate，也就是截断，具体做法是舍弃全部现有内容，释放inode所有 data block 并添加到 free bitmap 里。这也是目前 nfs 中唯一的文件变短方式。

比较复杂的就是使用fileopen以创建的方式打开一个文件。fileopen函数调用了create这个函数。

.. code-block:: c
    static struct inode *
    create(char *path, short type) {
        struct inode *ip, *dp;
        if(ip = namei(path) != 0) {
            // 已经存在，直接返回
            return ip;
        }
        // 创建一个文件,首先分配一个空闲的 disk inode, 绑定内存 inode 之后返回
        ip = ialloc(dp->dev, type);
        // 注意 ialloc 不会执行实际读取，必须有 ivalid
        ivalid(ip);
        // 在根目录创建一个 dirent 指向刚才创建的 inode 
        dirlink(dp, path, ip->inum);
        // dp 不用了，iput 就是释放内存 inode，和 iget 正好相反。
        iput(dp);
        return ip;
    }

    // nfs/fs.c
    uint ialloc(ushort type) {
        uint inum = freeinode++;
        struct dinode din;

        bzero(&din, sizeof(din));
        din.type = xshort(type);
        din.size = xint(0);
        winode(inum, &din);
        return inum;
    }

    // os/fs.c
    // Write a new directory entry (name, inum) into the directory dp.
    int
    dirlink(struct inode *dp, char *name, uint inum)
    {
        int off;
        struct dirent de;
        struct inode *ip;
        // Check that name is not present.
        if((ip = dirlookup(dp, name, 0)) != 0){
            iput(ip);
            return -1;
        }

        // Look for an empty dirent.
        for(off = 0; off < dp->size; off += sizeof(de)){
            if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
            panic("dirlink read");
            if(de.inum == 0)
            break;
        }
        strncpy(de.name, name, DIRSIZ);
        de.inum = inum;
        if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
            panic("dirlink");
        return 0;
    }

ialloc 干的事情：遍历 inode blocks 找到一个空闲的inode，初始化并返回。dirlink对于本章的练习也十分重要。和dirlookup不同，我们没有现成的dirent存储在磁盘上，而是要在磁盘上创建一个新的dirent。他遍历根目录数据块，找到一个空的 dirent，设置 dirent = {inum, filename} 然后返回，注意这一步可能找不到空位，这时需要找一个新的数据块，并扩大 root_dir size，这是由　bmap 自动完成的。需要注意本章创建硬链接时对应inode num的处理。

文件关闭
---------------------------------------

文件读写结束后需要fclose释放掉其inode，同时释放OS中对应的file结构体和fd。其实 inode 文件的关闭只需要调用 iput 就好了，iput 的实现简单到让人感觉迷惑，就是 inode 引用计数减一。诶？为什么没有计数为 0 就写回然后释放 inode 的操作？和 buf 的释放同理，这里会等 inode 池满了之后自行被替换出去，重新读磁盘实在太太太太慢了。对了，千万记得 iput 和 iget 数量相同，一定要一一对应，否则你懂的。

.. code-block:: c

    void
    fileclose(struct file *f)
    {
        if(--f->ref > 0) {
            return;
        }
        // 暂时不支持标准输入输出文件的关闭
        if(f->type == FD_PIPE){
            pipeclose(f->pipe, f->writable);
        } else if(f->type == FD_INODE) {
            iput(f->ip);
        }

        f->off = 0;
        f->readable = 0;
        f->writable = 0;
        f->ref = 0;
        f->type = FD_NONE;
    }

    void iput(struct inode *ip) {
        ip->ref--;
    }

