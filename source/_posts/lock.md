# 锁

在修改flash驱动时，对相关寄存的操作，其中在对锁的操作中出现了bug,故又由此复习了一遍锁。

在概念上来说，处理并发的方式有信号量，互斥体，当信号量的初始值为1时，即在任意时刻只能由单个进程或者线程拥有，这是这种信号量也称为互斥体（mutex）。而在内核中几乎所有的信号量均用于互斥。

关于信号量的源码位于<asm/semaphor.h>

关于mutex的源码位于<linux/mutex.h>

```c
/*
 * Simple, straightforward mutexes with strict semantics:
 *
 * - only one task can hold the mutex at a time
 * - only the owner can unlock the mutex
 * - multiple unlocks are not permitted
 * - recursive locking is not permitted
 * - a mutex object must be initialized via the API
 * - a mutex object must not be initialized via memset or copying
 * - task may not exit with mutex held
 * - memory areas where held locks reside must not be freed
 * - held mutexes must not be reinitialized
 * - mutexes may not be used in hardware or software interrupt
 *   contexts such as tasklets and timers
 *
 * These semantics are fully enforced when DEBUG_MUTEXES is
 * enabled. Furthermore, besides enforcing the above rules, the mutex
 * debugging code also implements a number of additional features
 * that make lock debugging easier and faster:
 *
 * - uses symbolic names of mutexes, whenever they are printed in debug output
 * - point-of-acquire tracking, symbolic lookup of function names
 * - list of all locks held in the system, printout of them
 * - owner tracking
 * - detects self-recursing locks and prints out all relevant info
 * - detects multi-task circular deadlocks and prints out all affected
 *   locks and tasks (and only those tasks)
 */
struct mutex {
	/* 1: unlocked, 0: locked, negative: locked, possible waiters */
	atomic_t		count;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
	struct task_struct	*owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
	struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
#ifdef CONFIG_DEBUG_MUTEXES
	void			*magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map	dep_map;
#endif
};

```

注意：互斥锁在上锁的过程中，需要用自旋锁保证原子操作（包括修改原子量和锁的相关数据结构）

初始化
 **mutex_init(&mutex)**; //动态初始化互斥锁
 **DEFINE_MUTEX(mutexname)**; //静态定义和初始化互斥锁

上锁
 **void mutex_lock(struct mutex \*lock)\***;//无法获得锁时，睡眠等待，不会被信号中断。
 **int mutex_trylock(struct mutex \*lock)\***;//此函数是 mutex_lock()的非阻塞版本，成功返回1，失败返回0。
 **int mutex_lock_interruptible(struct mutex \*lock)\***;//和mutex_lock()一样，也是获取互斥锁。在获得了互斥锁或进入睡眠直到获得互斥锁之后会返回0。如果在等待获取锁的时候进入睡眠状态收到一个信号(被信号打断睡眠)，则返回_EINIR。

解锁
 **void mutex_unlock(struct mutex \*lock)\***;

- posix库中关于互斥锁的实现(用户态)

> 如何使用互斥锁
>
> - 初始化
>    `int pthread_mutex_init(pthread_mutex_t *mp, const pthread_mutexattr_t *mattr);;//动态初始化互斥锁`
>    函数说明：初始化互斥锁之前，必须将其所在的内存清零。如果互斥锁已初始化，则它会处于未锁定状态
>
> - 设置锁的属性
>    `pthread_mutexattr_init(pthread_mutexattr_t *mattr);//互斥锁属性可以由该函数来初始化，然后再调用其他的函数来设置其属性`
>    `int pthread_mutexattr_setpshared(pthread_mutexattr_t *mattr, int pshared)`
>    `int pthread_mutexattr_getshared(pthread_mutexattr_t *mattr,int *pshared))//可以指定是该进程与其他进程的同步还是同一进程内不同的线程之间的同步。可以设置为PTHREAD_PROCESS_SHARE和PTHREAD_PROCESS_PRIVATE。默认是后者，表示进程内使用锁`
>    `init pthread_mutexattr_settype(pthread_mutexattr_t *attr , int type)`
>    `init pthread_mutexattr_gettype(pthread_mutexattr_t *attr , int *type)`
>
> > 互斥锁的类型,有以下几个取值空间：
> >  **PTHREAD_MUTEX_TIMED_NP**，这是缺省值，也就是普通锁。当一个线程加锁以后，其余请求锁的线程将形成一个等待队列，并在解锁后按优先级获得锁。这种锁策略保证了资源分配的公平性。
> >  **PTHREAD_MUTEX_RECURSIVE_NP**，嵌套锁，允许同一个线程对同一个锁成功获得多次，并通过多次unlock解锁。如果是不同线程请求，则在加锁线程解锁时重新竞争。
> >  **PTHREAD_MUTEX_ERRORCHECK_NP**，检错锁，如果同一个线程请求同一个锁，则返回EDEADLK，否则与PTHREAD_MUTEX_TIMED_NP类型动作相同。这样就保证当不允许多次加锁时不会出现最简单情况下的死锁。
> >  **PTHREAD_MUTEX_ADAPTIVE_NP**，适应锁，动作最简单的锁类型，仅等待解锁后重新竞争。
> >
> > - 上锁
> >    `void _int pthread_mutex_lock(pthread_mutex_t *mutex);//无法获得锁时，睡眠等待，不会被信号中断`
> >    返回值：0, 成功；其他值，失败；EAGAIN，由于已超出了互斥锁递归锁定的最大次数，因此无法获取该互斥锁；EDEADLK：当前线程已经拥有互斥锁。
>
> > - 解锁
> >    `int pthread_mutex_unlock(pthread_mutex_t *mutex);`
> >    函数说明：如果调用 pthread_mutex_unlock() 时有多个线程被 mutex 对象阻塞，则互斥锁变为可用时调度策略可确定获取该互斥锁的线程。对于 PTHREAD_MUTEX_RECURSIVE 类型的互斥锁，当计数达到零并且调用线程不再对该互斥锁进行任何锁定时，该互斥锁将变为可用.
> > - 非阻塞模式的互斥锁
> >    `int pthread_mutex_trylock(pthread_mutex_t *mutex);`
> >    函数说明：pthread_mutex_lock() 的非阻塞版本。如果 mutex 所引用的互斥对象当前被任何线程（包括当前线程）锁定，则将立即返回该调用。否则，该互斥锁将处于锁定状态，调用线程是其属主
>
> - ##### 互斥锁的特点
>
> > 1）互斥锁的特性，是一种信号量，常用来防止两个线程在同一时刻访问相同的共享资源。它有以下三个特性:
> >  **唯一性**：如果一个线程锁定了一个互斥量，在它解除锁定之前，没有其他线程可以锁定这个互斥量；
> >  **原子性**：把一个互斥量锁定为一个原子操作，这意味着操作系统（或pthread函数库）保证了如果一个线程锁定了一个互斥量，没有其他线程在同一时间可以成功锁定这个互斥量；
> >  **非繁忙等待**：如果一个线程已经锁定了一个互斥量，第二个线程又试图去锁定这个互斥量，则第二个线程将被挂起（不占用任何cpu资源），直到第一个线程解除对这个互斥量的锁定为止，第二个线程则被唤醒并继续执行，同时锁定这个互斥量。
> >  2）互斥锁的作用域
> >  互斥锁一般用在线程间，当然可以通过设置互斥锁的属性让它在进程间使用。
>
> 1. **互斥锁和信号量比较**
>
> - 互斥锁功能上基本与二元信号量一样，但是互斥锁占用空间比信号量小，运行效率比信号量高。所以，如果要用于线程间的互斥，优先选择互斥锁。
>
> > 上面这种解释是有问题了，除了开销上的区别, 我们还应该关注两者在语义上的区别：
> >  **互斥**：是指某一资源同时只允许一个访问者对其进行访问，具有唯一性和排它性。但互斥无法限制访问者对资源的访问顺序，即访问是无序的。举个例子，如果资源被锁定，另外一个线程尝试获取锁时，并阻塞线程，至于下一次什么时候线程被唤醒是未可知的，这个取决于cpu的调度。所以使用互斥锁会出现[优先级倒置（prority inversion）的问题](https://links.jianshu.com/go?to=https%3A%2F%2Fleetcode.com%2Fdiscuss%2Finterview-question%2Foperating-system%2F125169%2FMutex-vs-Semaphore)，高优先级的线程反而被延迟执行。
> >  **同步**：是指在互斥的基础上（大多数情况），通过其它机制实现访问者对资源的有序访问。在大多数情况下，同步已经实现了互斥，特别是所有写入资源的情况必定是互斥的。少数情况是指可以允许多个访问者同时访问资源。信号量一般会采用解锁通知等待队列中的线程（可以设置调度顺序，当然等待队列的开销需要额外支出的）
>
> 1. **互斥锁和自旋锁比较**
>
> - 互斥锁在无法得到资源时，内核线程会进入睡眠阻塞状态，而自旋锁处于忙等待状态。因此，如果资源被占用的时间较长，使用互斥锁较好，因为可让CPU调度去做其它进程的工作。
> - 如果被保护资源需要睡眠的话，那么只能使用互斥锁或者信号量，不能使用自旋锁。而互斥锁的效率又比信号量高，所以这时候最佳选择是互斥锁。
> - 中断里面不能使用互斥锁，因为互斥锁在获取不到锁的情况下会进入睡眠，而中断是不能睡眠的
>
> > ### 注意，为了防止死锁，不能在同一个线程里面两次申请同样的锁，同时也不推荐在不同的线程里面同时申请两个一样的锁。
>
> ### 2. 自旋锁在内核中的运用
>
> ##### 内核中自旋锁的运行机制：
>
> - **单处理器自旋锁的工作流程是**:单处理器中自旋锁不被启用，因为使用自旋锁的中断执行路径一旦被嵌套可能会造成永久等待，同步途径是关闭内核抢占->运行临界区代码->开启内核抢占。更加安全的单处理器自旋锁工作流程是:保存IF寄存器->关闭当前CPU中断->关闭内核抢占->运行临界区代码->开启内核抢占->开启当前CPU中断->恢复IF寄存器。
> - **多处理器自旋锁的工作流程是**：关闭内核抢占->（忙等待->)获取自旋锁->运行临界区代码->释放自旋锁->开启内核抢占。更加安全的多处理器自旋锁工作流程是：保存IF寄存器->关闭当前CPU中断->关闭内核抢占->（忙等待->)获取自旋锁->运行临界区代码->释放自旋锁->开启内核抢占->开启当前CPU中断->恢复IF寄存器。
>
> ##### 自旋锁的实现和特点
>
> ```c
> //CAS操作在cpu指令集中可以是原子性的
> int CompareAndExchange(int *ptr, int old, int new){
>     int actual = *ptr;
>     if (actual == old)
>     *ptr = new;
>     return actual;
> }
> void lock(lock_t *lock) {
>      while (CompareAndExchange(&lock->flag, 0, 1) == 1); // spin
> }
> void unlock(lock_t *lock) {
>      lock->flag = 0;
> }
> ```
>
> > 这个实现是借助cpu的原子指令CAS实现的，后面会介绍原子操作
>
> 自旋锁是采用忙等的状态获取锁，所以会一直占用cpu资源，但是允许不关闭中断的情况下，是可以被其他内核执行路径抢占的（中断嵌套的情况下，注意嵌套的中断不能申请同一个锁，这样会造成死等）。同时因为线程对cpu一直保持占用状态，所以对小资源加锁效率比较高，不需要做任何的线程切换，一般情况下如果加锁资源的运行延迟小于线程或者进程切换的时延则推荐使用自旋锁。如果需要等待耗时操作，则建议放弃cpu，采用信号量或者互斥锁。
>
> ***

1. <https://www.cnblogs.com/alinh/p/6905221.html>

2. <https://blog.csdn.net/mcgrady_tracy/article/details/34829019>

3. <https://leetcode.com/discuss/interview-question/operating-system/125169/Mutex-vs-Semaphore>

4. <https://blog.csdn.net/zxx901221/article/details/83033998>

