.. _term-remove-std:

makefile和qemu
==========================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
-------------------------------

为了帮助大家进一步理解我们的项目的链接和编译的过程，这里简要介绍一下我们makefile的工作机制。由于涉及到CI的判定，因此大家最好不要自己修改makefile。

makefile内部
----------------------------------

指定编译使用的工具

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::
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

::
   CFLAGS = -Wall -Werror -O -fno-omit-frame-pointer -ggdb
   CFLAGS += -MD
   CFLAGS += -mcmodel=medany
   CFLAGS += -ffreestanding -fno-common -nostdlib -mno-relax
   CFLAGS += -I.
   CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)

这里添加了编译的flag。比较需要注意的是我们设置了警告也会报错，因此大家写代码的时候最好避免warning的出现。

编译出目标文件以及qemu

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::
   kernel: $(OBJS) kernel.ld
      $(LD) $(LDFLAGS) -T kernel.ld -o kernel $(OBJS)

   clean: 
      rm -f *.o *.d kernel

上面两个是编译的选项。可以使用make + name 来调用它们。比较常用的就是make clean，可以清除所有生成出来的 .o以及.d中间文件和生成的kernel。
::
   QEMU = qemu-system-riscv64
   QEMUOPTS = \
      -nographic \
      -smp $(CPUS) \
      -machine virt \
      -bios $(BOOTLOADER) \
      -kernel kernel

   run: kernel
	$(QEMU) $(QEMUOPTS)

这个就是最关键的地方:make run。我们查看这条指令的结构，它首先执行上面kernel所需要的链接以及编译操作得到一个二进制的kernel。之后执行按照QEMUOPTS变量指定的参数启动qemu。那么QEMUOPTS之中的flag有什么意义呢？
- nographic: 无图形界面
- smp 1: 单核
- machine virt: 模拟硬件 RISC-V VirtIO Board
- bios ...: 使用制定 bios，这里指向的是我们提供的rustsbi的bin文件。
- device loader ...: 增加 loader 类型的 device,　这里其实就是把我们的 os 文件放到指定位置。
因此qemu会按照上述的参数启动，使用我们的rustsbi来进行一系列初始化，并将程序计数器移动至0x80200000并开始执行我们的OS。我们之后所有执行测试都是使用的make run指令。

下面是使用gdb需要的指令，本章也会使用到这些指令。
::
   # QEMU's gdb stub command line changed in 0.11
   QEMUGDB = $(shell if $(QEMU) -help | grep -q '^-gdb'; \
      then echo "-gdb tcp::1234"; \
      else echo "-s -p 1234"; fi)

   debug: kernel .gdbinit
      $(QEMU) $(QEMUOPTS) -S $(QEMUGDB) &
      sleep 1
      $(GDB)

使用make debug来使用gdb调试qemu。程序自身执行的机制和直接make run一样。在解析bootloader的行为时可以使用gdb在其中添加断点来查看对应寄存器和内存的内容。gdb的具体使用方法和汇编课程上一致。不熟悉的同学可以在训练章节查看到可能用到的gdb指令的用法。