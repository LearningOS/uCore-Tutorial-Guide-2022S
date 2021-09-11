chapter7练习
================================================

- 本节难度： **理解文件系统比较费事，编程难度适中** 

本章任务
-----------------------------------------------
- ``ch7b_usertest`` ``ch7_mergetest`` 
- merge ch6 的改动，然后再次测试 ch6_usertest 和 ch7b_usertest。
- 完成本章问答作业。
- 完成本章编程作业。
- 最终，完成实验报告并 push 你的 ch7 分支到远程仓库。

编程作业
-------------------------------------------------

硬链接
++++++++++++++++++++++++++++++++++++++++++++++++++

你的电脑桌面是咋样的？是放满了图标吗？反正我的 windows 是这样的。显然很少人会真的把可执行文件放到桌面上，桌面图标其实都是一些快捷方式。或者用 unix 的术语来说：软链接。为了减少工作量，我们今天来实现软链接的兄弟： `硬链接 <https://en.wikipedia.org/wiki/Hard_link>`_ 。

硬链接要求两个不同的目录项指向同一个文件，在我们的文件系统中也就是两个不同名称目录项指向同一个磁盘块。本节要求实现三个系统调用 ``sys_linkat、sys_unlinkat、sys_stat`` 。注意在测例中 ``sys_open`` 的接口定义也发生了变化。

**open**

    - syscall ID: 56
    - 功能：打开一个文件，并返回可以访问它的文件描述符。
    - 接口： ``int open(int dirfd, char* path, unsigned int flags, unsigned int mode);``
    - 参数：
        - **dirfd** : 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)。可以忽略。
        - **path** 描述要打开的文件的文件名（简单起见，文件系统不需要支持目录，所有的文件都放在根目录 ``/`` 下）
        - **flags** 描述打开文件的标志，具体含义（其他参数不考虑）：
          
          .. code-block:: c

                #define O_RDONLY  0x000
                #define O_WRONLY  0x001
                #define O_RDWR    0x002		// 可读可写
                #define O_CREATE  0x200

        - **mode** 仅在创建文件时有用，表示创建文件的访问权限，为了简单，本次实验中中统一为 *O_RDWR* 。
    - 说明：
        - 有 create 标志但文件存在时，忽略 create 标志，直接打开文件。
    - 返回值：如果出现了错误则返回 -1，否则返回可以访问给定文件的文件描述符。
    - 可能的错误：
        - 文件不存在且无 create 标志。
        - 标志非法（低两位为 0x3）
        - 打开文件数量达到上限。
  
**linkat**：

    * syscall ID: 37
    * 功能：创建一个文件的一个硬链接， `linkat标准接口 <https://linux.die.net/man/2/linkat>`_ 。
    * 接口： ``int linkat(int olddirfd, char* oldpath, int newdirfd, char* newpath, unsigned int flags)``
    * 参数：
        * olddirfd，newdirfd: 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)，可以忽略。
        * flags: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。
        * oldpath：原有文件路径
        * newpath: 新的链接文件路径。
    * 说明：
        * 为了方便，不考虑新文件路径已经存在的情况（属于未定义行为），除非链接同名文件。
        * 返回值：如果出现了错误则返回 -1，否则返回 0。
    * 可能的错误
        * 链接同名文件。

**unlinkat**:

    * syscall ID: 35
    * 功能：取消一个文件路径到文件的链接, `unlinkat标准接口 <https://linux.die.net/man/2/unlinkat>`_ 。
    * 接口： ``int unlinkat(int dirfd, char* path, unsigned int flags)``
    * 参数：
        * dirfd: 仅为了兼容性考虑，本次实验中始终为 AT_FDCWD (-100)，可以忽略。
        * flags: 仅为了兼容性考虑，本次实验中始终为 0，可以忽略。
        * path：文件路径。
    * 说明：
        * 为了方便，不考虑使用 unlink 彻底删除文件的情况。
    * 返回值：如果出现了错误则返回 -1，否则返回 0。
    * 可能的错误
        * 文件不存在。

**fstat**:

    * syscall ID: 80
    * 功能：获取文件状态。
    * Ｃ接口： ``int fstat(int fd, struct Stat* st)``
    * Rust 接口： ``fn fstat(fd: i32, st: *mut Stat) -> i32``
    * 参数：
        * fd: 文件描述符
        * st: 文件状态结构体

        .. code-block:: rust

            #[repr(C)]
            #[derive(Debug)]
            pub struct Stat {
                /// 文件所在磁盘驱动器号
                pub dev: u64,
                /// inode 文件所在 inode 编号
                pub ino: u64,
                /// 文件类型
                pub mode: StatMode,
                /// 硬链接数量，初始为1
                pub nlink: u32,
                /// 无需考虑，为了兼容性设计
                pad: [u64; 7],
            }
            
            /// StatMode 定义：
            bitflags! {
                pub struct StatMode: u32 {
                    const NULL  = 0;
                    /// directory
                    const DIR   = 0o040000;
                    /// ordinary regular file
                    const FILE  = 0o100000;
                }
            }

正确实现后，你的 os 应该能够正确运行 ch7_file* 对应的测试用例，在 shell 中执行 ch7_usertest 来执行测试。

Tips
++++++++++++++++++++++++++++++++++++++++++++++++++++++++

- 需要给 inode 和 dinode 都增加 link 的计数，但强烈建议不要改变整个数据结构的大小，事实上，推荐你修改一个 pad。
- os 和 nfs 的修改需要同步，只不过 nfs 比较简单，只需要初始化 link 计数为 1 就行（可以通过修改 ``ialloc``来实现）。
- 理论上讲，unlink 有删除文件的语义，如果 link 计数为 0，需要删除 inode 和对应的数据块，但我们没有设置相关的测试，如果仅仅是想拿到测试分数，可以不实现这一点。
- 理论上讲，想要测试文件系统的属性需要重启机器，但我们没有这样做。一方面是为了测试的方便，另一方面也留了一条后路。。。。。。。。。


问答作业
----------------------------------------------------------

1. 目前的文件系统只有单级目录，假设想要支持多级文件目录，请描述你设想的实现方式，描述合理即可。

2. 在有了多级目录之后，我们就也可以为一个目录增加硬链接了。在这种情况下，文件树中是否可能出现环路(软硬链接都可以，鼓励多尝试)？你认为应该如何解决？请在你喜欢的系统上实现一个环路，描述你的实现方式以及系统提示、实际测试结果。

报告要求
-----------------------------------------------------------
* 简单总结本次实验与上个实验相比你增加的东西。（控制在5行以内，不要贴代码）
* 完成问答问题
* (optional) 你对本次实验设计及难度的看法。