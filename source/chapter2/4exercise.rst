chapter2练习
=====================================================

.. toctree::
   :hidden:
   :maxdepth: 4

- 本节难度： **低** 

本节任务
-----------------------------------
- 运行 ch2 分支的框架代码，确认 ``BASE=1``　时 os 运行正常。
- 理解目前 os 加载用户程序的整体逻辑。在 ``user`` 目录下执行 ``make CHAPTER=2_bad``　生成三个会导致错误的程序，运行并描述他们导致的现象。完成问答作业第一题。
- 阅读状态切换相关函数，思考并完成问答作业第二题。
- 完成问答作业第三题。

编程练习
-------------------------------
无

.. ch2问答作业::

问答作业
-------------------------------

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。请同学们可以自行测试这些内容（参考 `前三个测例 <https://github.com/DeathWish5/riscvos-c-tests/tree/main/user/src>`_ ，描述程序出错行为，同时注意注明你使用的 sbi 及其版本。
   
2. 请结合用例理解 `trampoline.S <https://github.com/DeathWish5/uCore-Tutorial-v2/blob/ch2/os/trampoline.S>`_ 中两个函数 `userret` 和 `uservec` 的作用，并回答如下几个问题:

    1. L79: 刚进入 `userret` 时，`a0`、`a1` 分别代表了什么值。 

    2. L87-L88: `sfence` 指令有何作用？为什么要执行该指令，当前章节中，删掉该指令会导致错误吗？

        .. code-block:: assembly

            csrw satp, a1
            sfence.vma zero, zero

    3. L96-L125: 为何注释中说要除去 `a0`？哪一个地址代表 `a0`？现在 `a0` 的值存在何处？

        .. code-block:: assembly

            # restore all but a0 from TRAPFRAME
            ld ra, 40(a0)
            ld sp, 48(a0)
            ld t5, 272(a0)
            ld t6, 280(a0)

    4. `userret`：中发生状态切换在哪一条指令？为何执行之后会进入用户态？

    5. L29： 执行之后，a0 和 sscratch 中各是什么值，为什么？

        .. code-block:: assembly

            csrrw a0, sscratch, a0     

    6. L32-L61: 从 trapframe 第几项开始保存？为什么？是否从该项开始保存了所有的值，如果不是，为什么？
        
        .. code-block:: assembly

            sd ra, 40(a0)
            sd sp, 48(a0)
            ...
            sd t5, 272(a0)
            sd t6, 280(a0)

    7. 进入 S 态是哪一条指令发生的？

    8.  L75-L76: `ld t0, 16(a0)` 执行之后，`t0`中的值是什么，解释该值的由来？
        
        .. code-block:: assembly

            ld t0, 16(a0)
            jr t0

3. 程序陷入内核的原因有中断和异常（系统调用），请问 riscv64 支持哪些中断 / 异常？如何判断进入内核是由于中断还是异常？描述陷入内核时的几个重要寄存器及其值。
