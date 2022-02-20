chapter5练习
==============================================

- 本节难度： **看似唬人，其实就那样** 

本章任务
----------------------------------------
- ``make test BASE=1`` 执行 usershell，然后运行 ``ch5b_usertest``。
- merge ch4 的修改，运行 ``ch5_mergetest`` 检查 merge 是否正确。
- 结合文档和代码理解 fork, exec, wait 的逻辑。结合课堂内容回答本章问答问题（注意第二问为选做）。
- 完成本章变成作业。
- 最终，完成实验报告并 push 你的 ch5 分支到远程仓库。
  
编程作业
---------------------------------------------

进程创建
+++++++++++++++++++++++++++++++++++++++++++++

大家一定好奇过为啥进程创建要用 fork + execve 这么一个奇怪的系统调用，就不能直接搞一个新进程吗？思而不学则殆，我们就来试一试！这章的编程练习请大家实现一个完全 DIY 的系统调用 spawn，用以创建一个新进程。

spawn 系统调用定义( `标准spawn看这里 <https://man7.org/linux/man-pages/man3/posix_spawn.3.html>`_ )：

- syscall ID: 400
- C 接口： ``int spawn(char *filename)`` 
- 功能：相当于 fork + exec，新建子进程并执行目标程序。 
- 说明：成功返回子进程id，否则返回 -1。  
- 可能的错误： 
    - 无效的文件名。
    - 进程池满/内存不足等资源错误。  

实现完成之后，你应该能通过 ch5_spawn* 对应的所有测例，在 shell 中执行 ch5_usertest 来执行所有测试。

tips:

- 注意 fork 的执行流，新进程 context 的 ra 和 sp 与父进程不同。所以你不能在内核中通过 fork 和 exec 的简单组合实现 spawn。 
- 在 spawn 中不应该有任何形式的内存拷贝。

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


问答作业
--------------------------------------------

1. fork + exec 的一个比较大的问题是 fork 之后的内存页/文件等资源完全没有使用就废弃了，针对这一点，有什么改进策略？

2. [选做，不占分] 其实使用了题(1)的策略之后，fork + exec 所带来的无效资源的问题已经基本被解决了，但是今年来 fork 还是在被不断的批判，那么到底是什么正在"杀死"fork？回答合理即可。可以参考 `fork-hotos19 <https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf>`_ 。

3. 请阅读代码并分析如下程序的输出，不考虑运行错误，不考虑行缓冲，不考虑中断：
    
    .. code-block:: c 

        int main(){
            int val = 2;
            
            printf("%d", 0);
            int pid = fork();
            if (pid == 0) {
                val++;
                printf("%d", val);
            } else {
                val--;
                printf("%d", val);
                wait(NULL);
            }
            val++;
            printf("%d", val);
            return 0;
        } 


    如果 fork() 之后一定主程序先运行结果如何？如果 fork() 之后一定 child 先运行结果如何？


4. 请分析如下程序运行后最终输出 `A` 的数量(已知 ``&&`` 的优先级比 ``||`` 高)：

    .. code-block:: c 

        int main() {
            fork() && fork() && fork() || fork() && fork() || fork() && fork();
            printf("A");
            return 0' 
        }

    [选做，不占分] 更进一步，如果给出一个 ``&&`` ``||`` 的序列，你如何设计一个程序来得到答案？

5. stride 算法深入

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

报告要求
---------------------------------------

注意目录要求，报告命名 ``lab3.md``，位于 ``reports`` 目录下。 后续实验同理。

- 注明姓名学号。
- 完成 ch5 问答作业。
- [可选，不占分]你对本次实验设计及难度的看法。
