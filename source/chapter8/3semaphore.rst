信号量机制
=========================================

本节导读
-----------------------------------------

在上一节中，我们介绍了互斥锁（mutex 或 lock）的起因、使用和实现过程。通过互斥锁，
可以让线程在临界区执行时，独占临界资源。当我们需要更灵活的互斥访问或同步操作方式，如提供了最多只允许
N 个线程访问临界资源的情况，让某个线程等待另外一个线程执行完毕后再继续执行的同步过程等，
互斥锁这种方式就有点力不从心了。

在本节中，将介绍功能更加强大和灵活的同步互斥机制 -- 信号量（Semaphore），它的设计思路、
使用和在操作系统中的具体实现。可以看到，信号量的实现需要互斥锁和处理器原子指令的支持，
它是一种更高级的同步互斥机制。


信号量的起源和基本思路
-----------------------------------------

1963 年前后，当时的数学家（其实是计算机科学家）Edsger Dijkstra 和他的团队在为 Electrologica X8
计算机开发一个操作系统（称为 THE multiprogramming system，THE 多道程序系统）的过程中，提出了信号量
（Semphore）是一种变量或抽象数据类型，用于控制多个线程对共同资源的访问。

信号量是对互斥锁的一种巧妙的扩展。上一节中的互斥锁的初始值一般设置为 1 的整型变量，
表示临界区还没有被某个线程占用。互斥锁用 0 表示临界区已经被占用了，用 1 表示临界区为空，再通过
``lock/unlock`` 操作来协调多个线程轮流独占临界区执行。而信号量的初始值可设置为 N 的整数变量, 如果 N
大于 0， 表示最多可以有 N 个线程进入临界区执行，如果 N 小于等于 0 ，表示不能有线程进入临界区了，
必须在后续操作中让信号量的值加 1 ，才能唤醒某个等待的线程。

Dijkstra 对信号量设计了两种操作：P（Proberen（荷兰语），尝试）操作和 V（Verhogen（荷兰语），增加）操作。
P 操作是检查信号量的值是否大于 0，若该值大于 0，则将其值减 1 并继续（表示可以进入临界区了）；若该值为
0，则线程将睡眠。注意，此时 P 操作还未结束。而且由于信号量本身是一种临界资源（可回想一下上一节的锁，
其实也是一种临界资源），所以在 P 操作中，检查/修改信号量值以及可能发生的睡眠这一系列操作，
是一个不可分割的原子操作过程。通过原子操作才能保证，一旦 P 操作开始，则在该操作完成或阻塞睡眠之前，
其他线程均不允许访问该信号量。

V 操作会对信号量的值加 1 ，然后检查是否有一个或多个线程在该信号量上睡眠等待。如有，
则选择其中的一个线程唤醒并允许该线程继续完成它的 P 操作；如没有，则直接返回。注意，信号量的值加 1，
并可能唤醒一个线程的一系列操作同样也是不可分割的原子操作过程。不会有某个进程因执行 V 操作而阻塞。

如果信号量是一个任意的整数，通常被称为计数信号量（Counting Semaphore），或一般信号量（General
Semaphore）；如果信号量只有0或1的取值，则称为二值信号量（Binary Semaphore）。可以看出，
互斥锁是信号量的一种特例 --- 二值信号量，信号量很好地解决了最多允许 N 个线程访问临界资源的情况。

信号量的一种实现伪代码如下所示：

.. code-block:: c
  :linenos:

  void P(S) {
      if (S >= 1)
          S = S - 1;
      else
          <block and enqueue the thread>;
  }
  void V(S) {
      if <some threads are blocked on the queue>
          <unblock a thread>;
      else
          S = S + 1;
  }

在上述实现中，S 的取值范围为大于等于 0 的整数。S 的初值一般设置为一个大于 0 的正整数，
表示可以进入临界区的线程数。当 S 取值为 1，表示是二值信号量，也就是互斥锁了。
使用信号量实现线程互斥访问临界区的伪代码如下：

.. code-block:: C
    :linenos:

    static struct semaphore S = {1};

    // Thread i
    void foo() {
        ...
        P(S);
        execute Cricital Section;
        V(S);
        ...
    }

下面是另外一种信号量实现的伪代码：

.. code-block:: c
    :linenos:

    void P(S) {
        S = S - 1;
        if (S < 0)
            <block and enqueue the thread>;
    }

    void V(S) {
        S = S + 1;
        if <some threads are blocked on the queue>
            <unblock a thread>;
    }

在这种实现中，S 的初值一般设置为一个大于 0 的正整数，表示可以进入临界区的线程数。但 S
的取值范围可以是小于 0 的整数，表示等待进入临界区的睡眠线程数。

信号量的另一种用途是用于实现同步（synchronization）。比如，把信号量的初始值设置为 0 ，
当一个线程 A 对此信号量执行一个 P 操作，那么该线程立即会被阻塞睡眠。之后有另外一个线程 B
对此信号量执行一个 V 操作，就会将线程 A 唤醒。这样线程 B 中执行 V 操作之前的代码序列 B-stmts
和线程 A 中执行 P 操作之后的代码 A-stmts 序列之间就形成了一种确定的同步执行关系，即线程 B 的
B-stmts 会先执行，然后才是线程 A 的 A-stmts 开始执行。相关伪代码如下所示：

.. code-block:: C
  :linenos:

  static struct semaphore S = {1};
  //Thread A
  ...
  P(S);
  Label_2:
  A-stmts after Thread B::Label_1;
  ...

  //Thread B
  ...
  B-stmts before Thread A::Label_2;
  Label_1:
  V(S);
  ...


实现信号量
------------------------------------------

使用 semaphore 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们通过例子来看看如何实际使用信号量。下面是面向应用程序对信号量系统调用的简单使用，
可以看到对它的使用与上一节介绍的互斥锁系统调用类似。

在这个例子中，主线程先创建了信号量初值为 0 的信号量 ``SEM_SYNC`` ，然后再创建两个线程 First
和 Second 。线程 First 会先睡眠 10ms，而当线程 Second 执行时，会由于执行信号量的 P
操作而等待睡眠；当线程 First 醒来后，会执行 V 操作，从而能够唤醒线程 Second。这样线程 First
和线程 Second 就形成了一种稳定的同步关系。

.. code-block:: C
  :linenos:
  :emphasize-lines: 5,10,16,23,28,33

  const int SEM_SYNC = 0; //信号量ID
  void first() {
      sleep(10);
      puts("First work and wakeup Second");
      semaphore_up(SEM_SYNC); //信号量V操作
      exit(0);
  }
  void second() {
      puts("Second want to continue,but need to wait first");
      semaphore_down(SEM_SYNC); //信号量P操作
      puts("Second can work now");
      exit(0);
  }
  int main() {
      // create semaphores
      assert_eq(semaphore_create(0), SEM_SYNC); // 信号量初值为0
      // create first, second threads
      ...
  }

  int semaphore_create(int res_count)
  {
    return syscall(SYS_semaphore_create, res_count);
  }

  int semaphore_up(int sid)
  {
    return syscall(SYS_semaphore_up, sid);
  }

  int semaphore_down(int sid)
  {
    return syscall(SYS_semaphore_down, sid);
  }


- 第 16 行，创建了一个初值为 0 ，ID 为 ``SEM_SYNC`` 的信号量，对应的是第 23 行
  ``SYS_semaphore_create`` 系统调用；
- 第 10 行，线程 Second 执行信号量 P 操作（对应第 33 行 ``SYS_semaphore_down``
  系统调用），由于信号量初值为 0 ，该线程将阻塞；
- 第 5 行，线程 First 执行信号量 V 操作（对应第 28 行 ``SYS_semaphore_up`` 系统调用），
  会唤醒等待该信号量的线程 Second。

实现 semaphore 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

操作系统如何实现信号量系统调用呢？我们还是采用通常的分析做法：数据结构+方法，
即首先考虑一下与此相关的核心数据结构，然后考虑与数据结构相关的相关函数/方法的实现。

在线程的眼里，信号量是一种每个线程能看到的共享资源，且在一个进程中，可以存在多个不同信号量资源，
所以我们可以把所有的信号量资源放在一起让进程来管理，如下面代码第 4 行所示。这里需要注意的是：
``struct semaphore semaphore_pool[LOCK_POOL_SIZE]`` 表示的是信号量资源的列表。而 ``struct semaphore``
是信号量的内核数据结构，由信号量值和等待队列组成。操作系统需要显式地施加某种控制，来确定当一个线程执行
P 操作和 V 操作时，如何让线程睡眠或唤醒线程。在这里，P 操作是由 ``semaphore_down``
方法实现，而 V 操作是由 ``semaphore_up`` 方法实现。

.. code-block:: c
  :linenos:
  :emphasize-lines: 4,9-12,31-35,42-47

  struct proc {
    int pid; // Process ID
    uint next_semaphore_id;
    struct semaphore semaphore_pool[LOCK_POOL_SIZE];
    ...
  };

  struct semaphore {
    int count;
    struct queue wait_queue;
    // "alloc" data for wait queue
    int _wait_queue_data[WAIT_QUEUE_MAX_LENGTH];
  };

  struct semaphore *semaphore_create(int count)
  {
    struct proc *p = curr_proc();
    if (p->next_semaphore_id >= LOCK_POOL_SIZE) {
      return NULL;
    }
    struct semaphore *s = &p->semaphore_pool[p->next_semaphore_id];
    p->next_semaphore_id++;
    s->count = count;
    init_queue(&s->wait_queue, WAIT_QUEUE_MAX_LENGTH, s->_wait_queue_data);
    return s;
  }

  void semaphore_up(struct semaphore *s)
  {
    s->count++;
    if (s->count <= 0) {
      // count <= 0 after up means wait queue not empty
      struct thread *t = id_to_task(pop_queue(&s->wait_queue));
      t->state = RUNNABLE;
      add_task(t);
    }
  }

  void semaphore_down(struct semaphore *s)
  {
    s->count--;
    if (s->count < 0) {
      // s->count < 0 means need to wait (state=SLEEPING)
      struct thread *t = curr_thread();
      push_queue(&s->wait_queue, task_to_id(t));
      t->state = SLEEPING;
      sched();
    }
  }

首先是核心数据结构：

- 第 4 行，进程控制块中管理的信号量列表。
- 第 9~12 行，信号量的核心数据成员：信号量值和等待队列。

然后是重要的三个成员函数：

- 第 15 行，创建信号量，信号量初值为参数 ``res_count`` ，信号量池的使用原理同互斥锁。
- 第 28 行，实现 V 操作的 ``up`` 函数，第 31 行，当信号量值小于等于 0 时，
  将从信号量的等待队列中弹出一个线程放入线程就绪队列。
- 第 39 行，实现 P 操作的 ``down`` 函数，第 225 行，当信号量值小于 0 时，
  将把当前线程放入信号量的等待队列，设置当前线程为挂起状态并选择新线程执行。


Dijkstra, Edsger W. Cooperating sequential processes (EWD-123) (PDF). E.W. Dijkstra Archive.
Center for American History, University of Texas at Austin. (transcription) (September 1965)
https://www.cs.utexas.edu/users/EWD/transcriptions/EWD01xx/EWD123.html

Downey, Allen B. (2016) [2005]. "The Little Book of Semaphores" (2nd ed.). Green Tea Press.

Leppäjärvi, Jouni (May 11, 2008). "A pragmatic, historically oriented survey on the universality
of synchronization primitives" (pdf). University of Oulu, Finland.