进程管理的核心数据结构
===================================

本节导读
-----------------------------------

本节将会展示在本章节的实验中，我们管理进程、调度进程所用到的数据结构。

进程队列
------------------------------------------------------------------------

不同于此前遍历进程池的调度方式，在本章节中，我们实现了一个简单的队列，用于存储和调度所有的就绪进程：

.. code-block:: c
    :linenos:

    // os/queue.h

    struct queue {
	    int data[QUEUE_SIZE];
	    int front;
	    int tail;
	    int empty;
    };

    void init_queue(struct queue *);
    void push_queue(struct queue *, int);
    int pop_queue(struct queue *);

队列的实现非常简单，大小为1024，具体的实现大家可以查看queue.c进行查看。我们将在后面的部分展示我们要如何使用这一数据结构

进程的调度
------------------------------------------------------------------------

进程的调度主要体现在proc.c的scheduler函数中：

.. code-block:: c
    :linenos:

    // os/proc.c

    void scheduler()
    {
	    struct proc *p;
	    for (;;) {
		    /*int has_proc = 0;
		    for (p = pool; p < &pool[NPROC]; p++) {
			    if (p->state == RUNNABLE) {
				    has_proc = 1;
    				tracef("swtich to proc %d", p - pool);
	    			p->state = RUNNING;
		    		current_proc = p;
			    	swtch(&idle.context, &p->context);
    			}
	    	}
		    if(has_proc == 0) {
			    panic("all app are over!\n");
    		}*/
	    	p = fetch_task();
		    if (p == NULL) {
			    panic("all app are over!\n");
    		}
	    	tracef("swtich to proc %d", p - pool);
		    p->state = RUNNING;
		    current_proc = p;
		    swtch(&idle.context, &p->context);
	    }
    }



可以看到，我们移除了原来遍历进程池，选出其中就绪状态的进程来运行的这种朴素调度方式，而是直接使用了fetch_task函数从队列中获取应当调度的进程，再进行相应的切换；而对于已经运行结束或时间片耗尽的进程，则将其push进入队列之中。这种调度方式，相比之前提高了调度的效率，可以在常数时间复杂度下完成一次调度。由于使用的是队列，因此大家也会发现，我们的框架代码所使用的FIFO的调度算法。
