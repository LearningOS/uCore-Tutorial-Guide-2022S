.. _term-remove-std:

makefile 和 qemu
==========================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
-------------------------------

为了帮助大家进一步理解我们的项目的链接和编译的过程，这里简要介绍一下 makefile 的内容。

.. warning:: 

   注意，makefile 在整个实验过程中不可修改，否则可能导致 CI 无法通过！


makefile 内部
----------------------------------

指定编译使用的工具
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: makefile

   TOOLPREFIX = riscv64-unknown-elf-
   CC = $(TOOLPREFIX)gcc
   AS = $(TOOLPREFIX)gas
   LD = $(TOOLPREFIX)ld
   OBJCOPY = $(TOOLPREFIX)objcopy
   OBJDUMP = $(TOOLPREFIX)objdump
   GDB = $(TOOLPREFIX)gdb

这里makefile调用了大家设定好的PATH之中的riscv64工具链。如果没有设置好，那么之后的编译就会因为找不到这些文件而出错。

添加编译flag
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: makefile

   CFLAGS = -Wall -Werror -O -fno-omit-frame-pointer -ggdb
   CFLAGS += -MD
   CFLAGS += -mcmodel=medany
   CFLAGS += -ffreestanding -fno-common -nostdlib -mno-relax
   CFLAGS += -I.
   CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)

比较需要注意的是我们设置了警告也会报错，因此大家写代码的时候最好避免 warning 的出现，这是良好的编程习惯。

设置编译目标
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: makefile

   # 目录定义
   K = os
   BUILDDIR = build
   # .o 目标的确定，也就是 os 目录下所有的 .c 和 .s 都编译成 .o
   C_SRCS = $(wildcard $K/*.c)
   AS_SRCS = $(wildcard $K/*.S)
   C_OBJS = $(addprefix $(BUILDDIR)/, $(addsuffix .o, $(basename $(C_SRCS))))
   AS_OBJS = $(addprefix $(BUILDDIR)/, $(addsuffix .o, $(basename $(AS_SRCS))))
   OBJS = $(C_OBJS) $(AS_OBJS)
   # kernel 镜像由所有的 .o 按照 kernel.ld 链接而成
   $(BUILDDIR)/kernel: $(OBJS) $(K)/kernel.ld
      $(LD) $(LDFLAGS) -T kernel.ld -o kernel $(OBJS)
   
请同学们自行查阅并了解``wildcard``、``addprefix``、``addsuffix``、``basename``的意义。

运行 qemu
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: makefile

   QEMU = qemu-system-riscv64
   QEMUOPTS = \
      -nographic \
      -smp $(CPUS) \
      -machine virt \
      -bios $(BOOTLOADER) \
      -kernel kernel

   run: $(BUILDDIR)/kernel
   $(QEMU) $(QEMUOPTS)

这里和前面一致。大家不需要太关心qemu的更多细节，我们涉及它的操作已经在makefile和sbi之中处理了。
  
gdb 调试
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: makefile

   # QEMU's gdb stub command line changed in 0.11
   QEMUGDB = $(shell if $(QEMU) -help | grep -q '^-gdb'; \
      then echo "-gdb tcp::1234"; \
      else echo "-s -p 1234"; fi)

   debug: kernel .gdbinit
      $(QEMU) $(QEMUOPTS) -S $(QEMUGDB) &
      sleep 1
      $(GDB)

使用 make debug 来使用 gdb 调试 qemu。程序自身执行的机制和直接 make run 一样。在解析 bootloader 的行为时可以使用 gdb 在其中添加断点来查看对应寄存器和内存的内容。gdb的具体使用方法和汇编课程上一致。不熟悉的同学可以在训练章节查看到可能用到的gdb指令的简单用法，也十分推荐同学们自学一些基础的 gdb 使用方法，掌握　gdb 对本课程帮助很大。

LOG 支持
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: makefile

   ifeq ($(LOG), error)
   CFLAGS += -D LOG_LEVEL_ERROR
   else ifeq ($(LOG), warn)
   CFLAGS += -D LOG_LEVEL_WARN
   else ifeq ($(LOG), info)
   CFLAGS += -D LOG_LEVEL_INFO
   else ifeq ($(LOG), debug)
   CFLAGS += -D LOG_LEVEL_DEBUG
   else ifeq ($(LOG), trace)
   CFLAGS += -D LOG_LEVEL_TRACE
   endif

我们的 log 等级选择是通过 -D 参数来实现的，这也是大家 ``make run LOG=xxx`` 的原理。从这里我们也可以看到 ``LOG`` 的可选值。

.. warngin::

   FIX ME:
   大家在实际使用中会发现，由于 LOG 是静态编译是就确认的参数，所以如果想要改变 LOG 等级，就需要重新编译几乎所有的源文件。目前在需要改变 LOG 等级的时候需要 make clean 然后重新 make run。