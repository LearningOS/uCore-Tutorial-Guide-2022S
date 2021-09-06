实验环境配置
============

.. toctree::
   :hidden:
   :maxdepth: 4
   
本节我们将完成环境配置并成功运行 uCore-Tutorial-v2 。整个流程分为下面几个部分：

- 系统环境配置
- Riscv下 C 开发环境配置
- Qemu 模拟器安装
- 其他工具安装
- 运行 uCore-Tutorial-v2

系统环境配置
-------------------------------

目前实验仅支持 Ubuntu18.04 + 操作系统。对于 Windows10 和 macOS 上的用户，可以使用 VMware 或 
VirtualBox 安装一台 Ubuntu18.04 虚拟机并在上面进行实验。

特别的，Windows10 的用户可以通过系统内置的 WSL2 虚拟机（请不要使用 WSL1）来安装 Ubuntu 18.04 / 20.04 。
步骤如下：

- 升级 Windows 10 到最新版（Windows 10 版本 18917 或以后的内部版本）。注意，如果
  不是 Windows 10 专业版，可能需要手动更新，在微软官网上下载。升级之后，
  可以在 PowerShell 中输入 ``winver`` 命令来查看内部版本号。
- 「Windows 设置 > 更新和安全 > Windows 预览体验计划」处选择加入 “Dev 开发者模式”。
- 以管理员身份打开 PowerShell 终端并输入以下命令：

  .. code-block:: bash

     # 启用 Windows 功能：“适用于 Linux 的 Windows 子系统”
     >> dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

     # 启用 Windows 功能：“已安装的虚拟机平台”
     >> dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

     # <Distro> 改为对应从微软应用商店安装的 Linux 版本名，比如：`wsl --set-version Ubuntu 2`
     # 如果你没有提前从微软应用商店安装任何 Linux 版本，请跳过此步骤
     >> wsl --set-version <Distro> 2

     # 设置默认为 WSL 2，如果 Windows 版本不够，这条命令会出错
     >> wsl --set-default-version 2

-  `下载 Linux 内核安装包 <https://docs.microsoft.com/zh-cn/windows/wsl/install-win10#step-4---download-the-linux-kernel-update-package>`_
-  在微软商店（Microsoft Store）中搜索并安装 Ubuntu18.04 / 20.04。

C 开发环境配置
-------------------------------------------

这一大步和下面的Docker安装方式大家可以选用一种。
首先，我们需要安装好RISC-V配套的gcc。首先选择一个位置放置gcc的可执行二进制文件。我们选择一个常用的位置。这个位置也可以大家自己指定。

.. code-block:: bash

   cd /usr/local

之后直接下载预编译好的Risc-v工具链：

.. code-block:: bash
   
   sudo wget https://static.dev.sifive.com/dev-tools/freedom-tools/v2020.08/riscv64-unknown-elf-gcc-10.1.0-2020.08.2-x86_64-linux-ubuntu14.tar.gz

解压缩：

.. code-block:: bash
   
   tar xzvf riscv64-unknown-elf-gcc-10.1.0-2020.08.2-x86_64-linux-ubuntu14.tar.gz

文件名改短：

.. code-block:: bash 

   mv riscv64-unknown-elf-gcc-10.1.0-2020.08.2-x86_64-linux-ubuntu14 riscv64-unknown-elf-gcc

这里就算安装完成了。接下来我们要把gcc的二进制文件路径添加到PATH之中，这样我们才能在任意目录直接运行它。将以下指令添加到home下的.bashrc之中可以一劳永逸地添加(如果你使用的是自己的路径请更换路径的前缀/usr/local到你自己的路径)：

.. code-block:: bash

   export PATH="/usr/local/riscv64-unknown-elf-gcc/bin:$PATH"

接下来，继续安装用于交叉编译的musl-gcc,这里我们仍然使用/usr/local存放它。下面的步骤和上一步的安装是一样的：

.. code-block:: bash
   
   cd /usr/local
   sudo wget https://more.musl.cc/9.2.1-20190831/x86_64-linux-musl/riscv64-linux-musl-cross.tgz
   tar xzvf riscv64-linux-musl-cross.tgz

将路径添加到PATH之中:

.. code-block:: bash

   export PATH="/usr/local/riscv64-linux-musl-cross/bin:$PATH"

我们的项目使用cmake搭建，因此还需要安装cmake。

.. code-block:: bash

   sudo apt install cmake

如果是第一次启动虚拟机，需要执行：

.. code-block:: bash

   sudo apt update
   sudo apt upgrade

至于 C 开发环境，推荐直接使用Vscode作为IDE来进行。可以在其中安装C的插件来使用自动补全和lint。

Docker 环境安装(可选，已完成上述步骤的可以忽略)
---------------------------------------------------------------------

使用配置好的Docker容器可以免于自己安装上面的一系列包，当然会多出配置Docker的工作量。

- 首先安装Docker: https://www.docker.com/get-started

- 接着拉取我们的Docker镜像文件：

   .. code-block:: bash

      docker pull nzpznk/oslab-c-env
      docker pull registry.cn-hangzhou.aliyuncs.com/nzpznk/oslab-c-env

- 查看下载的镜像文件：docker image ls (nzpznk/oslab-c-env or registry.cn-hangzhou.aliyuncs.com/nzpznk/oslab-c-env)

- 使用下载的镜像文件创建一个名字叫container_name(可自己指定)的容器，并获得容器的bash shell: docker run -it --name container_name image_name /bin/bash

- 一些常用的指令：

   .. code-block:: bash

      # 检查运行中的容器
      docker ps 
      # 检查所有容器（包括停止了的）
      docker ps -a
      # 停止 / 启动容器
      docker stop/start container_name
      # 获取一个运行中的docker容器的bash shell:
      docker exec -it container_name /bin/bash
      # 删除一个已经停止的容器:
      docker rm container_name

- 之后就是把我们宿主机的文件挂载在docker容器上.在使用 docker run 启动容器时，你可以将目录挂载到容器上，这样就可以从docker容器访问到本地主机的某个文件夹.

   .. code-block:: bash

      docker run -it --name container_name --mount type=bind,src=[absolute path of folder in host machine],dst=[absolute path in container] image_name /bin/bas

Qemu 模拟器安装
----------------------------------------

我们需要使用 Qemu 5.x.x 版本进行实验，而很多 Linux 发行版的软件包管理器默认软件源中的 Qemu 版本过低，因此
我们需要从源码手动编译安装 Qemu 模拟器。

首先我们安装依赖包，获取 Qemu 源代码并手动编译：

.. code-block:: bash

   # 安装编译所需的依赖包
   sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
                 gawk build-essential bison flex texinfo gperf libtool patchutils bc \
                 zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3
   # 下载源码包 
   # 如果下载速度过慢可以使用我们提供的百度网盘链接：https://pan.baidu.com/s/1z-iWIPjxjxbdFS2Qf-NKxQ
   # 提取码 8woe
   wget https://download.qemu.org/qemu-5.0.0.tar.xz
   # 解压
   tar xvJf qemu-5.0.0.tar.xz
   # 编译安装并配置 RISC-V 支持
   cd qemu-5.0.0
   ./configure --target-list=riscv64-softmmu,riscv64-linux-user
   make -j$(nproc)

.. note::
   
   注意，上面的依赖包可能并不完全，比如在 Ubuntu 18.04 上：

   - 出现 ``ERROR: pkg-config binary 'pkg-config' not found`` 时，可以安装 ``pkg-config`` 包；
   - 出现 ``ERROR: glib-2.48 gthread-2.0 is required to compile QEMU`` 时，可以安装 
     ``libglib2.0-dev`` 包；
   - 出现 ``ERROR: pixman >= 0.21.8 not present`` 时，可以安装 ``libpixman-1-dev`` 包。

   另外一些 Linux 发行版编译 Qemu 的依赖包可以从 `这里 <https://risc-v-getting-started-guide.readthedocs.io/en/latest/linux-qemu.html#prerequisites>`_ 
   找到。

之后我们可以在同目录下 ``sudo make install`` 将 Qemu 安装到 ``/usr/local/bin`` 目录下，但这样经常会引起
冲突。个人来说更习惯的做法是，编辑 ``~/.bashrc`` 文件（如果使用的是默认的 ``bash`` 终端），在文件的末尾加入
几行：

.. code-block:: bash

   # 请注意，qemu-5.0.0 的父目录可以随着你的实际安装位置灵活调整
   export PATH=$PATH:/home/shinbokuow/Downloads/built/qemu-5.0.0
   export PATH=$PATH:/home/shinbokuow/Downloads/built/qemu-5.0.0/riscv64-softmmu
   export PATH=$PATH:/home/shinbokuow/Downloads/built/qemu-5.0.0/riscv64-linux-user

随后即可在当前终端 ``source ~/.bashrc`` 更新系统路径，或者直接重启一个新的终端。

此时我们可以确认 Qemu 的版本：

.. code-block:: bash

   qemu-system-riscv64 --version
   qemu-riscv64 --version


GDB 调试支持
------------------------------

在 ``os`` 目录下 ``make debug`` 可以调试我们的内核，这里需要基于 riscv64 平台的 gdb 调试器 ``riscv64-unknown-elf-gdb`` 。该调试器包含在 riscv64 gcc 工具链中，工具链的预编译版本可以在如下链接处下载：

- `Ubuntu 平台 <https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2020.04.1-x86_64-linux-ubuntu14.tar.gz>`_
- `macOS 平台 <https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2020.04.1-x86_64-apple-darwin.tar.gz>`_
- `Windows 平台 <https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2020.04.1-x86_64-w64-mingw32.zip>`_
- `CentOS 平台 <https://static.dev.sifive.com/dev-tools/riscv64-unknown-elf-gcc-8.3.0-2020.04.1-x86_64-linux-centos6.tar.gz>`_

解压后在 ``bin`` 目录下即可找到 ``riscv64-unknown-elf-gdb`` 以及另外一些常用工具 ``objcopy/objdump/readelf`` 等。

在 Qemu 平台上运行 uCore-Tutorial-v2
------------------------------------------------------------

到这里，恭喜你完成了实验环境的配置，可以开始阅读教程的正文部分了！可以直接clone下面的仓库来开始OS之旅：

.. code-block:: bash

   git clone https://github.com/DeathWish5/uCore-Tutorial-v2.git --recursive
   cd uCore-Tutorial-v2

其他的章节需要处理用户代码，我们可以先运行 ch1 分支：

.. code-block:: bash

   > git checkout ch1
   > make run LOG=debug

   [rustsbi] RustSBI version 0.1.1
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

   [rustsbi] Platform: QEMU (Version 0.1.0)
   [rustsbi] misa: RV64ACDFIMSU
   [rustsbi] mideleg: 0x222
   [rustsbi] medeleg: 0xb1ab
   [rustsbi-dtb] Hart count: cluster0 with 1 cores
   [rustsbi] Kernel entry: 0x80200000

   hello wrold!
   [ERROR 0]stext: 0x0000000080200000
   [WARN 0]etext: 0x0000000080201000
   [INFO 0]sroda: 0x0000000080201000
   [DEBUG 0]eroda: 0x0000000080202000
   [DEBUG 0]sdata: 0x0000000080202000
   [INFO 0]edata: 0x0000000080202000
   [WARN 0]sbss : 0x0000000080212000
   [ERROR 0]ebss : 0x0000000080212000
   [PANIC 0] os/main.c:39: ALL DONE

忽略掉编译输出后，你应该得到如上的输出，这表示 uCore 已经成功运行。

.. note::

   **退出 qemu 的方法**

   如果是正常推出，uCore 会自动关闭 qemu，但如果 os 跑飞了，我们不能通过 ``Ctrl + C`` 来推出。此时可以先按下 ``Ctrl+A`` ，再按下 ``X`` 来退出 Qemu。


