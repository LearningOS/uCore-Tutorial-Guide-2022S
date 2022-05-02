进程通讯与 fork
============================================

fork的修改
--------------------------------------------

对fork的文件支持本来应该在chapter6引入，但是为了更好的理解管道的继承机制，我们把它放在了这个章节。
fork 为什么是毒瘤呢？因为你总是要在新增加一个东西以后考虑要不要为新功能增加 fork 支持。这一章的文件就是第一个例子，那么在 fork 语境下，子进程也需要继承父进程的文件资源，也就是PCB之中的指针文件数组。我们应该如何处理呢？我们来看看 fork 在这一个 chapter 的实现：

.. code-block:: c

    int fork() {
        // ...
    +   for(int i = 3; i < FD_MAX; ++i)
    +       if(p->files[i] != 0 && p->files[i]->type != FD_NONE) {
    +           p->files[i]->ref++;
    +           np->files[i] = p->files[i];
    +       }
        // ...
    }

可以看到创建子进程时会遍历父进程，继承其所有打开的文件，并且给指定文件的ref + 1。因为我们记录的本身就只是一个指针，只需用ref来记录一个文件还有没有进程使用。

此外，进程结束需要清理的资源除了内存之外增加了文件：

.. code-block:: c 

    void freeproc(struct proc *p)
    {
        // ...
    +   for (int i = 3; i < FD_BUFFER_SIZE; i++) {
    +       if (p->files[i] != NULL) {
    +           fileclose(p->files[i]);
    +       }
    +   }
        // ...
    }

你会发现 exec 的实现竟然没有修改，注意 exec 仅仅重新加载进程执行的测例文件镜像，不会改变其他属性，比如文件。也就是说，fork 出的子进程打开了与父进程相同的文件，但是 exec 并不会把打开的文件刷掉，基于这一点，我们可以利用 pipe 进行进程间通信。

.. code-block:: c

    // user/src/ch6b_pipetest

    char STR[] = "hello pipe!";

    int main() {
        uint64 pipe_fd[2];
        int ret = pipe(&pipe_fd);
        if (fork() == 0) {
            // 子进程，从 pipe 读，和 STR 比较。
            char buffer[32 + 1];
            read(pipe_fd[0], buffer, 32);
            assert(strncmp(buffer, STR, strlen(STR) == 0);
            exit(0);
        } else {
            // 父进程，写 pipe
            write(pipe_fd[1], STR, strlen(STR));
            int exit_code = 0;
            wait(&exit_code);
            assert(exit_code == 0);
        }
        return 0;
    }

由于 fork 会拷贝所有文件而 exec 不会改变文件，所以父子进程的fd列表一致，可以直接使用创建好的pipe进行通信。