chapter3练习
=======================================

- 本节难度： 第一次变成作业，没那么轻松了

本章任务
-----------------------------------------------------
- 老规矩，先 `make test BASE=1` 看下啥情况。
- 理解框架的多任务加载机制，了解此时用户和内核的大概内存布局。在此基础上，实现本章编程作业(1)更安全的 sys_write。
- 理解框架的调度机制，尤其要搞明白时钟中断的处理机制以及 yield 之后下一个进程的选择。在次基础上，完成本节的编程作业(2)stride 调度算法。
- 进一步思考 stride 调度算法，完成本章问答作业。
- 最终，完成实验报告并 push 你的 ch3 分支到远程仓库。ch3报告要求_ 。push 代码后会自动执行 CI，代码给分以 CI 给分为准。

编程作业
--------------------------------------

更安全的 sys_write
+++++++++++++++++++++++++++++++++++++++++

lab2 中，我们实现了第一个系统调用 ``sys_write``，这使得我们可以在用户态输出信息。但是 os 在提供服务的同时，还有保护 os 本身以及其他用户程序不受错误或者恶意程序破坏的功能。

由于还没有实现虚拟内存，我们可以在用户程序中指定一个属于其他程序或者内核的地址，并将它以字符串的形式输出，这显然是不合理的，因此我们要对 sys_write 做检查：传入缓冲区是否位于用户所属地址。也就是 ``[buf, buf + len)`` 这一段地址，是否完全属于对应用户。如果不完全属于，sys_write应该返回 -1。　

实现正确后，代码应该能够通过用户测例 ch2t_write0。你可以使用 ``make test CHAPTER=3_2`` 来测试测试你的实现是否正确，如果正确 ch2t_write1 应该正确推出并输出 "Test write0 OK!"。注意，同时你还要保证 ch2b_write1 也正确执行。

实现 tips:
- 你可以看看测例源码，并想一想，属于用户的地址空间有哪些？提示：有两段。
- 这些地址段位置是静态分配的吗？如何获得这些地址？
- 你可以修改 os 目录下，除了 Makefile 之外的所有程序。放心大胆得增加你需要的函数或者变量。

stride 调度算法
+++++++++++++++++++++++++++++++++++++++++

lab3中我们引入了任务调度的概念，可以在不同任务之间切换，目前我们实现的调度算法十分简单，存在一些问题且不存在优先级。现在我们要为我们的 os 实现一种带优先级的调度算法：stide 调度算法。

算法描述如下:

(1) 为每个进程设置一个当前 stride，表示该进程当前已经运行的“长度”。另外设置其对应的 pass 值（只与进程的优先权有关系），表示对应进程在调度后，stride 需要进行的累加值。

(2) 每次需要调度时，从当前 runnable 态的进程中选择 stride 最小的进程调度。对于获得调度的进程 P，将对应的 stride 加上其对应的步长 pass。

(3) 一个时间片后，回到上一步骤，重新调度当前 stride 最小的进程。

可以证明，如果令 P.pass = BigStride / P.priority 其中 P.pass 为进程的 pass 值，P.priority 表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。证明过程我们在这里略去，有兴趣的同学可以在网上查找相关资料。

其他实验细节：

- stride 调度要求进程优先级 :math:`\geq 2`，所以设定进程优先级 :math:`\leq 1` 会导致错误。
- 进程初始 stride 设置为 0 即可。
- 进程初始优先级设置为 16。

实验首先要求新增 syscall ``sys_set_priority``:

* 功能描述：设定进程优先级
  * syscall ID: 140
  * 功能：设定进程优先级。
  * C 接口：`int setpriority(long long prio);`
  * 说明：设定自身进程优先级，只要 prio 在 [2, isize_max] 就成功，返回 prio，否则返回 -1。
* 针对测例
  * `ch3_setprio`

实现 sys_set_priority 之后，你可以通过　``make test CHAPTER=3`` 来进行测试。

完成之后你需要调整框架的代码调度机制，是的可以设置不同进程优先级之后可以按照 stride 算法进行调度。实现正确后，代码应该能够通过用户测例 ch3t_stride*。使用 ``make test CHAPTER=3t`` 来测试测试你的实现是否正确，如果正确,ch3t_stride[x] 最终输出的 priority 和 exitcode 应该大致成正比，由于我们的时间片比较粗糙，qemu 的模拟也不是十分准确，我们最终的 CI 测试会允许最大 30% 的误差。 

实现 tips:
- 你应该给 proc 结构体加入新的字段来支持优先级。
- 我们的测例运行时间不很长，不要求处理 stride 的溢出（详见问答作业，当然处理了更好）。
- 为了减少整数除的误差，BIG_STRIDE 一般需要很大，但测例中的优先级都是 2 的整数次幂，结合第二点，BIG_STRIDE不需要太大，65536 是一个不错的数字。
- 用户态的 printf 支持了行缓冲，所以如果你想要增加用户程序的输出，记得换行。
- stride 算法要找到　stride 最小的进程，使用优先级队列是效率不错的办法，但是我们的实验测例很简单，所以效率完全不是问题。事实上，我很推荐使用暴力扫一遍的办法找最小值。
- 注意设置进程的初始优先级。

.. ch3问答作业::

问答作业
--------------------------------------------

1、stride 算法深入

    stride 算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride， p1.stride = 255, p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

    - 实际情况是轮到 p1 执行吗？为什么？

    我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明，**在不考虑溢出的情况下**, 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。

    - 为什么？尝试简单说明（传达思想即可，不要求严格证明）。
    
    已知以上结论，**在考虑溢出的情况下**，假设我们通过逐个比较得到 Stride 最小的进程，请设计一个合适的比较函数，用来正确比较两个 Stride 的真正大小：

    .. code-block:: c
    
        typedef unsigned long long Stride_t;
        const Stride_t BIG_STRIDE = 0xffffffffffffffffULL;
        int cmp(Stride_t a, Stride_t b) {
            // YOUR CODE HERE
            // return 1 if a > b
            // return -1 if a < b
            // return 0 if a == b
        }


    例子：假设使用 8 bits 储存 stride, BigStride = 255。那么：
    * `cmp(125, 255) == 1`
    * `cmp(129, 255) == -1`


实验目录要求
------------------------------------------

.. code-block::

   ├── os(内核实现)
   │   └── ...
   ├── reports (不是 report)
   │   ├── ch3.pdf
   │   └── ...
   ├── ...


.. ch3报告要求::

报告要求
-------------------------------
- pdf 格式，CI 网站提交，注明姓名学号。 
- 注意目录要求，报告命名 ``ch3.pdf``，位于 ``reports`` 目录下。命名错误视作没有提交。后续实验同理。
- 完成问答问题。
 
    + ch1: ch1问答作业_ 。
    + ch2: ch2问答作业_ 。
    + ch3: ch3问答作业_ 。

- [可选，不占分]你对本次实验设计及难度/工作量的看法，以及有哪些需要改进的地方，欢迎畅所欲言。

.. warning::

    请勿抄袭，报告会进行抽样查重！


参考信息
-------------------------------
如果有兴趣进一步了解 stride 调度相关内容，可以尝试看看：

- `作者 Carl A. Waldspurger 写这个调度算法的原论文 <https://people.cs.umass.edu/~mcorner/courses/691J/papers/PS/waldspurger_stride/waldspurger95stride.pdf>`_
- `作者 Carl A. Waldspurger 的博士生答辩slide <http://www.waldspurger.org/carl/papers/phd-mit-slides.pdf>`_ 
- `南开大学实验指导中对Stride算法的部分介绍 <https://nankai.gitbook.io/ucore-os-on-risc-v64/lab6/tiao-du-suan-fa-kuang-jia#stride-suan-fa>`_
- `NYU OS课关于Stride Scheduling的Slide <https://cs.nyu.edu/~rgrimm/teaching/sp08-os/stride.pdf>`_

如果有兴趣进一步了解用户态线程实现的相关内容，可以尝试看看：

- `user-multitask in rv64 <https://github.com/chyyuu/os_kernel_lab/tree/v4-user-std-multitask>`_
- `绿色线程 in x86 <https://github.com/cfsamson/example-greenthreads>`_
- `x86版绿色线程的设计实现 <https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/>`_
- `用户级多线程的切换原理 <https://blog.csdn.net/qq_31601743/article/details/97514081?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control>`_
