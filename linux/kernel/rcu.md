整理 http://www.ibm.com/developerworks/cn/linux/l-rcu/

##简介

        RCU（Read-Copy Update）是数据同步的一种方式，在当前的Linux内核中发挥着重要的作用。RCU主要针对的数据对象是链表，目的是提高遍历读取数据的效率，为了达到目的使用RCU机制读取数据的时候不对链表进行耗时的加锁操作。这样在同一时间可以有多个线程同时读取该链表，并且允许一个线程对链表进行修改（修改的时候，需要加锁）。RCU适用于需要频繁的读取数据，而相应修改数据并不多的情景，例如在文件系统中，经常需要查找定位目录，而对目录的修改相对来说并不多，这就是RCU发挥作用的最佳场景。

       Linux内核源码当中,关于RCU的文档比较齐全，你可以在 /Documentation/RCU/ 目录下找到这些文件。Paul E. McKenney 是内核中RCU源码的主要实现者，他也写了很多RCU方面的文章。他把这些文章和一些关于RCU的论文的链接整理到了一起。http://www2.rdrop.com/users/paulmck/RCU/

       在RCU的实现过程中，我们主要解决以下问题：

       1，在读取过程中，另外一个线程删除了一个节点。删除线程可以把这个节点从链表中移除，但它不能直接销毁这个节点，必须等到所有的读取线程读取完成以后，才进行销毁操作。RCU中把这个过程称为宽限期（Grace period）。

       2，在读取过程中，另外一个线程插入了一个新节点，而读线程读到了这个节点，那么需要保证读到的这个节点是完整的。这里涉及到了发布-订阅机制（Publish-Subscribe Mechanism）。

       3， 保证读取链表的完整性。新增或者删除一个节点，不至于导致遍历一个链表从中间断开。但是RCU并不保证一定能读到新增的节点或者不读到要被删除的节点。


##宽限期

通过例子，方便理解这个内容。以下例子修改于Paul的文章。

    struct foo {  
               int a;  
               char b;  
               long c;  
     };  
      
    DEFINE_SPINLOCK(foo_mutex);  
      
    struct foo *gbl_foo;  
      
    void foo_read (void)  
    {  
         foo *fp = gbl_foo;  
         if ( fp != NULL )  
                dosomething(fp->a, fp->b , fp->c );  
    }  
      
    void foo_update( foo* new_fp )  
    {  
         spin_lock(&foo_mutex);  
         foo *old_fp = gbl_foo;  
         gbl_foo = new_fp;  
         spin_unlock(&foo_mutex);  
         kfee(old_fp);  
    }  


如上的程序，是针对于全局变量gbl_foo的操作。假设以下场景。有两个线程同时运行 foo_ read和foo_update的时候，当foo_ read执行完赋值操作后，线程发生切换；此时另一个线程开始执行foo_update并执行完成。当foo_ read运行的进程切换回来后，运行dosomething 的时候，fp已经被删除，这将对系统造成危害。为了防止此类事件的发生，RCU里增加了一个新的概念叫宽限期（Grace period）。如下图所示：


![宽限期](rcu_grace_period.png)


图中每行代表一个线程，最下面的一行是删除线程，当它执行完删除操作后，线程进入了宽限期。宽限期的意义是，在一个删除动作发生后，它必须等待所有在宽限期开始前已经开始的读线程结束，才可以进行销毁操作。这样做的原因是这些线程有可能读到了要删除的元素。图中的宽限期必须等待1和2结束；而读线程5在宽限期开始前已经结束，不需要考虑；而3,4,6也不需要考虑，因为在宽限期结束后开始后的线程不可能读到已删除的元素。为此RCU机制提供了相应的API来实现这个功能。                     


    void foo_read(void)  
    {  
        rcu_read_lock();  
        foo *fp = gbl_foo;  
        if ( fp != NULL )  
                dosomething(fp->a,fp->b,fp->c);  
        rcu_read_unlock();  
    }  
      
    void foo_update( foo* new_fp )  
    {  
        spin_lock(&foo_mutex);  
        foo *old_fp = gbl_foo;  
        gbl_foo = new_fp;  
        spin_unlock(&foo_mutex);  
        synchronize_rcu();  
        kfee(old_fp);  
    }  


其中foo_read中增加了rcu_read_lock和rcu_read_unlock，这两个函数用来标记一个RCU读过程的开始和结束。其实作用就是帮助检测宽限期是否结束。foo_update增加了一个函数synchronize_rcu()，调用该函数意味着一个宽限期的开始，而直到宽限期结束，该函数才会返回。我们再对比着图看一看，线程1和2，在synchronize_rcu之前可能得到了旧的gbl_foo，也就是foo_update中的old_fp，如果不等它们运行结束，就调用kfee(old_fp)，极有可能造成系统崩溃。而3,4,6在synchronize_rcu之后运行，此时它们已经不可能得到old_fp，此次的kfee将不对它们产生影响。

宽限期是RCU实现中最复杂的部分,原因是在提高读数据性能的同时，删除数据的性能也不能太差。

## 订阅——发布机制 

当前使用的编译器大多会对代码做一定程度的优化，CPU也会对执行指令做一些优化调整,目的是提高代码的执行效率，但这样的优化，有时候会带来不期望的结果。如例：

    void foo_update( foo* new_fp )  
    {  
        spin_lock(&foo_mutex);  
        foo *old_fp = gbl_foo;  
          
        new_fp->a = 1;  
        new_fp->b = ‘b’;  
        new_fp->c = 100;  
          
        gbl_foo = new_fp;  
        spin_unlock(&foo_mutex);  
        synchronize_rcu();  
        kfee(old_fp);  
    }  

这段代码中，我们期望的是6，7，8行的代码在第10行代码之前执行。但优化后的代码并不对执行顺序做出保证。在这种情形下，一个读线程很可能读到 new_fp，但new_fp的成员赋值还没执行完成。当读线程执行dosomething(fp->a, fp->b , fp->c ) 的 时候，就有不确定的参数传入到dosomething，极有可能造成不期望的结果，甚至程序崩溃。可以通过优化屏障来解决该问题，RCU机制对优化屏障做了包装，提供了专用的API来解决该问题。这时候，第十行不再是直接的指针赋值，而应该改为 :

       rcu_assign_pointer(gbl_foo,new_fp);


 rcu_assign_pointer的实现比较简单，如下：



	//<include/linux/rcupdate.h>
    #define rcu_assign_pointer(p, v) \  
             __rcu_assign_pointer((p), (v), __rcu)  
      
    #define __rcu_assign_pointer(p, v, space) \  
             do { \  
                     smp_wmb(); \  
                     (p) = (typeof(*v) __force space *)(v); \  
             } while (0)  



 我们可以看到它的实现只是在赋值之前加了优化屏障 smp_wmb来确保代码的执行顺序。另外就是宏中用到的__rcu，只是作为编译过程的检测条件来使用的。

 在DEC Alpha CPU机器上还有一种更强悍的优化，如下所示：

    void foo_read(void)  
    {         
        rcu_read_lock();  
        foo *fp = gbl_foo;  
        if ( fp != NULL )  
            dosomething(fp->a, fp->b ,fp->c);  
        rcu_read_unlock();  
    }  


第六行的 fp->a,fp->b,fp->c会在第3行还没执行的时候就预先判断运行，当他和foo_update同时运行的时候，可能导致传入dosomething的一部分属于旧的gbl_foo，而另外的属于新的。这样导致运行结果的错误。为了避免该类问题，RCU还是提供了宏来解决该问题：


	//<include/linux/rcupdate.h>
    #define rcu_dereference(p) rcu_dereference_check(p, 0)  
      
      
    #define rcu_dereference_check(p, c) \  
             __rcu_dereference_check((p), rcu_read_lock_held() || (c), __rcu)  
      
    #define __rcu_dereference_check(p, c, space) \  
             ({ \  
                     typeof(*p) *_________p1 = (typeof(*p)*__force )ACCESS_ONCE(p); \  
                     rcu_lockdep_assert(c, "suspicious rcu_dereference_check()" \  
                                           " usage"); \  
                     rcu_dereference_sparse(p, space); \  
                     smp_read_barrier_depends(); \  
                     ((typeof(*p) __force __kernel *)(_________p1)); \  
             })  
      
    static inline int rcu_read_lock_held(void)  
    {  
             if (!debug_lockdep_rcu_enabled())  
                     return 1;  
             if (rcu_is_cpu_idle())  
                     return 0;  
             if (!rcu_lockdep_current_cpu_online())  
                     return 0;  
             return lock_is_held(&rcu_lock_map);  
    }  

这段代码中加入了调试信息，去除调试信息，可以是以下的形式（其实这也是旧版本中的代码）：


    #define rcu_dereference(p)     ({ \  
                        typeof(p) _________p1 = p; \  
                        smp_read_barrier_depends(); \  
                        (_________p1); \  
                        })  



 在赋值后加入优化屏障smp_read_barrier_depends()。

        我们之前的第四行代码改为 foo *fp = rcu_dereference(gbl_foo);，就可以防止上述问题。



##数据读取的完整性

还是通过例子来说明这个问题：

![rcu 链表添加](rcu_list_add.png)

 如图我们在原list中加入一个节点new到A之前，所要做的第一步是将new的指针指向A节点，第二步才是将Head的指针指向new。这样做的目的是当插入操作完成第一步的时候，对于链表的读取并不产生影响，而执行完第二步的时候，读线程如果读到new节点，也可以继续遍历链表。如果把这个过程反过来，第一步head指向new，而这时一个线程读到new，由于new的指针指向的是Null，这样将导致读线程无法读取到A，B等后续节点。从以上过程中，可以看出RCU并不保证读线程读取到new节点。如果该节点对程序产生影响，那么就需要外部调用做相应的调整。如在文件系统中，通过RCU定位后，如果查找不到相应节点，就会进行其它形式的查找，相关内容等分析到文件系统的时候再进行叙述。


 我们再看一下删除一个节点的例子：

![rcu 链表删除](rcu_list_del.png)


如图我们希望删除B，这时候要做的就是将A的指针指向C，保持B的指针，然后删除程序将进入宽限期检测。由于B的内容并没有变更，读到B的线程仍然可以继续读取B的后续节点。B不能立即销毁，它必须等待宽限期结束后，才能进行相应销毁操作。由于A的节点已经指向了C，当宽限期开始之后所有的后续读操作通过A找到的是C，而B已经隐藏了，后续的读线程都不会读到它。这样就确保宽限期过后，删除B并不对系统造成影响。


小结

   RCU的原理并不复杂，应用也很简单。但代码的实现确并不是那么容易，难点都集中在了宽限期的检测上，后续分析源代码的时候，我们可以看到一些极富技巧的实现方式。





































将读者和写者要访问的共享数据放在一个指针上 p 中, 读者通过其中的 p
来访问其中的数据, 写者拷贝一份 p 指向的内存, 然后更新拷贝后的内存
(即写拷贝), 需要注意的是在写者写拷贝数据开始到结束期间, 如果有新
的读者, 读者必须自旋直到写者完成写操作. 在写者更新拷贝的内存之后,
之后的所有读者都读取的是新的内存空间, 而旧的内存空间不会立即释放,
而是等之前该内存的所有的读者都退出后才释放. 因此, 是无锁的. 非常
类似数据库中 MVVC 的机制


###为什么不适合多写者?

如果写者很多, 显然会存在很多个该内存的版本, 这不仅造成内存的浪费,
而且多个版本对数据的一致性非常差. 此外读者可能无法读取中间版本数据.
读者会频繁自旋.

写者除了要互斥之外, 还要考虑读者, 显然比单纯的写者之间互斥开销更大


1. removal
2. reclamation

###典型的步骤

1 Remove pointers to a data structure, so that subsequent readers cannot gain a reference to it.
2 Wait for all previous readers to complete their RCU read-side critical sections.
3 (Atomic) At this point, there cannot be any readers who hold references to the data
structure, so it now may safely be reclaimed

###关键点:

* 仅限于指针
* 对指针的操作是原子的
* 多读少写的情况
* 读者临界区不会发生进程切换(关闭内核的可抢占性)
* 写者更新期间阻塞读者, 更新之后, 保证读者读取的是最近的修改
* 写者之间是互斥的

###角色

* 读者
* 更新者
* 回收者

###问题

1. 在读者期间, 更新者进行多次更新, 那么, 待所有读者读完, 更新者的最后一次更新有效, 其他更新都是无效的, 这是期望的么?

   当更新者更新之后, 就必须等待从上次更新者到这次更新操作之前的所有读者都完成读操作, 并且同步之后,
   下一次更新者才能继续更新. 在同步之前, 下一次更新者只能等待.

2. reclamation 如何知道所有读者读完成 ?

引用计数


rcu_assign_pointer() : 更新者更新一个已经受 RCU 保护的指针为新值

rcu_dereference() : 读者获取一个已经受 RCU 保护的指针

    rcu_read_lock();
    p = rcu_dereference(head.next);
    a = p->address;
    rcu_read_unlock();
    x = p->address; /* BUG!!! */
    rcu_read_lock();
    y = p->data; /* BUG!!! */
    rcu_read_unlock();

###参考

https://github.com/torvalds/linux/blob/master/Documentation/RCU/whatisRCU.txt
http://yzcyn.blog.163.com/blog/static/38484300201343102545301/	

###附录

```
/* Linux/include/linux/rcupdate.h */

/*
 * Read-Copy Update mechanism for mutual exclusion
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation; either version 2 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program; if not, you can access it online at
 * http://www.gnu.org/licenses/gpl-2.0.html.
 *
 * Copyright IBM Corporation, 2001
 *
 * Author: Dipankar Sarma <dipankar@in.ibm.com>
 *
 * Based on the original work by Paul McKenney <paulmck@us.ibm.com>
 * and inputs from Rusty Russell, Andrea Arcangeli and Andi Kleen.
 * Papers:
 * http://www.rdrop.com/users/paulmck/paper/rclockpdcsproof.pdf
 * http://lse.sourceforge.net/locking/rclock_OLS.2001.05.01c.sc.pdf (OLS2001)
 *
 * For detailed explanation of Read-Copy Update mechanism see -
 *              http://lse.sourceforge.net/locking/rcupdate.html
 *
 */

#ifndef __LINUX_RCUPDATE_H
#define __LINUX_RCUPDATE_H

#include <linux/types.h>
#include <linux/cache.h>
#include <linux/spinlock.h>
#include <linux/threads.h>
#include <linux/cpumask.h>
#include <linux/seqlock.h>
#include <linux/lockdep.h>
#include <linux/completion.h>
#include <linux/debugobjects.h>
#include <linux/bug.h>
#include <linux/compiler.h>
#include <asm/barrier.h>

extern int rcu_expedited; /* for sysctl */

#ifdef CONFIG_TINY_RCU
/* Tiny RCU doesn't expedite, as its purpose in life is instead to be tiny. */
static inline bool rcu_gp_is_expedited(void)  /* Internal RCU use. */
{
        return false;
}

static inline void rcu_expedite_gp(void)
{
}

static inline void rcu_unexpedite_gp(void)
{
}
#else /* #ifdef CONFIG_TINY_RCU */
bool rcu_gp_is_expedited(void);  /* Internal RCU use. */
void rcu_expedite_gp(void);
void rcu_unexpedite_gp(void);
#endif /* #else #ifdef CONFIG_TINY_RCU */

enum rcutorture_type {
        RCU_FLAVOR,
        RCU_BH_FLAVOR,
        RCU_SCHED_FLAVOR,
        RCU_TASKS_FLAVOR,
        SRCU_FLAVOR,
        INVALID_RCU_FLAVOR
};

#if defined(CONFIG_TREE_RCU) || defined(CONFIG_PREEMPT_RCU)
void rcutorture_get_gp_data(enum rcutorture_type test_type, int *flags,
                            unsigned long *gpnum, unsigned long *completed);
void rcutorture_record_test_transition(void);
void rcutorture_record_progress(unsigned long vernum);
void do_trace_rcu_torture_read(const char *rcutorturename,
                               struct rcu_head *rhp,
                               unsigned long secs,
                               unsigned long c_old,
                               unsigned long c);
#else
static inline void rcutorture_get_gp_data(enum rcutorture_type test_type,
                                          int *flags,
                                          unsigned long *gpnum,
                                          unsigned long *completed)
{
        *flags = 0;
        *gpnum = 0;
        *completed = 0;
}
static inline void rcutorture_record_test_transition(void)
{
}
static inline void rcutorture_record_progress(unsigned long vernum)
{
}
#ifdef CONFIG_RCU_TRACE
void do_trace_rcu_torture_read(const char *rcutorturename,
                               struct rcu_head *rhp,
                               unsigned long secs,
                               unsigned long c_old,
                               unsigned long c);
#else
#define do_trace_rcu_torture_read(rcutorturename, rhp, secs, c_old, c) \
        do { } while (0)
#endif
#endif

#define UINT_CMP_GE(a, b)       (UINT_MAX / 2 >= (a) - (b))
#define UINT_CMP_LT(a, b)       (UINT_MAX / 2 < (a) - (b))
#define ULONG_CMP_GE(a, b)      (ULONG_MAX / 2 >= (a) - (b))
#define ULONG_CMP_LT(a, b)      (ULONG_MAX / 2 < (a) - (b))
#define ulong2long(a)           (*(long *)(&(a)))

/* Exported common interfaces */

#ifdef CONFIG_PREEMPT_RCU

/**
 * call_rcu() - Queue an RCU callback for invocation after a grace period.
 * @head: structure to be used for queueing the RCU updates.
 * @func: actual callback function to be invoked after the grace period
 *
 * The callback function will be invoked some time after a full grace
 * period elapses, in other words after all pre-existing RCU read-side
 * critical sections have completed.  However, the callback function
 * might well execute concurrently with RCU read-side critical sections
 * that started after call_rcu() was invoked.  RCU read-side critical
 * sections are delimited by rcu_read_lock() and rcu_read_unlock(),
 * and may be nested.
 *
 * Note that all CPUs must agree that the grace period extended beyond
 * all pre-existing RCU read-side critical section.  On systems with more
 * than one CPU, this means that when "func()" is invoked, each CPU is
 * guaranteed to have executed a full memory barrier since the end of its
 * last RCU read-side critical section whose beginning preceded the call
 * to call_rcu().  It also means that each CPU executing an RCU read-side
 * critical section that continues beyond the start of "func()" must have
 * executed a memory barrier after the call_rcu() but before the beginning
 * of that RCU read-side critical section.  Note that these guarantees
 * include CPUs that are offline, idle, or executing in user mode, as
 * well as CPUs that are executing in the kernel.
 *
 * Furthermore, if CPU A invoked call_rcu() and CPU B invoked the
 * resulting RCU callback function "func()", then both CPU A and CPU B are
 * guaranteed to execute a full memory barrier during the time interval
 * between the call to call_rcu() and the invocation of "func()" -- even
 * if CPU A and CPU B are the same CPU (but again only if the system has
 * more than one CPU).
 */
void call_rcu(struct rcu_head *head,
              void (*func)(struct rcu_head *head));

#else /* #ifdef CONFIG_PREEMPT_RCU */

/* In classic RCU, call_rcu() is just call_rcu_sched(). */
#define call_rcu        call_rcu_sched

#endif /* #else #ifdef CONFIG_PREEMPT_RCU */

/**
 * call_rcu_bh() - Queue an RCU for invocation after a quicker grace period.
 * @head: structure to be used for queueing the RCU updates.
 * @func: actual callback function to be invoked after the grace period
 *
 * The callback function will be invoked some time after a full grace
 * period elapses, in other words after all currently executing RCU
 * read-side critical sections have completed. call_rcu_bh() assumes
 * that the read-side critical sections end on completion of a softirq
 * handler. This means that read-side critical sections in process
 * context must not be interrupted by softirqs. This interface is to be
 * used when most of the read-side critical sections are in softirq context.
 * RCU read-side critical sections are delimited by :
 *  - rcu_read_lock() and  rcu_read_unlock(), if in interrupt context.
 *  OR
 *  - rcu_read_lock_bh() and rcu_read_unlock_bh(), if in process context.
 *  These may be nested.
 *
 * See the description of call_rcu() for more detailed information on
 * memory ordering guarantees.
 */
void call_rcu_bh(struct rcu_head *head,
                 void (*func)(struct rcu_head *head));

/**
 * call_rcu_sched() - Queue an RCU for invocation after sched grace period.
 * @head: structure to be used for queueing the RCU updates.
 * @func: actual callback function to be invoked after the grace period
 *
 * The callback function will be invoked some time after a full grace
 * period elapses, in other words after all currently executing RCU
 * read-side critical sections have completed. call_rcu_sched() assumes
 * that the read-side critical sections end on enabling of preemption
 * or on voluntary preemption.
 * RCU read-side critical sections are delimited by :
 *  - rcu_read_lock_sched() and  rcu_read_unlock_sched(),
 *  OR
 *  anything that disables preemption.
 *  These may be nested.
 *
 * See the description of call_rcu() for more detailed information on
 * memory ordering guarantees.
 */
void call_rcu_sched(struct rcu_head *head,
                    void (*func)(struct rcu_head *rcu));

void synchronize_sched(void);

/*
 * Structure allowing asynchronous waiting on RCU.
 */
struct rcu_synchronize {
        struct rcu_head head;
        struct completion completion;
};
void wakeme_after_rcu(struct rcu_head *head);

/**
 * call_rcu_tasks() - Queue an RCU for invocation task-based grace period
 * @head: structure to be used for queueing the RCU updates.
 * @func: actual callback function to be invoked after the grace period
 *
 * The callback function will be invoked some time after a full grace
 * period elapses, in other words after all currently executing RCU
 * read-side critical sections have completed. call_rcu_tasks() assumes
 * that the read-side critical sections end at a voluntary context
 * switch (not a preemption!), entry into idle, or transition to usermode
 * execution.  As such, there are no read-side primitives analogous to
 * rcu_read_lock() and rcu_read_unlock() because this primitive is intended
 * to determine that all tasks have passed through a safe state, not so
 * much for data-strcuture synchronization.
 *
 * See the description of call_rcu() for more detailed information on
 * memory ordering guarantees.
 */
void call_rcu_tasks(struct rcu_head *head, void (*func)(struct rcu_head *head));
void synchronize_rcu_tasks(void);
void rcu_barrier_tasks(void);

#ifdef CONFIG_PREEMPT_RCU

void __rcu_read_lock(void);
void __rcu_read_unlock(void);
void rcu_read_unlock_special(struct task_struct *t);
void synchronize_rcu(void);

/*
 * Defined as a macro as it is a very low level header included from
 * areas that don't even know about current.  This gives the rcu_read_lock()
 * nesting depth, but makes sense only if CONFIG_PREEMPT_RCU -- in other
 * types of kernel builds, the rcu_read_lock() nesting depth is unknowable.
 */
#define rcu_preempt_depth() (current->rcu_read_lock_nesting)

#else /* #ifdef CONFIG_PREEMPT_RCU */

static inline void __rcu_read_lock(void)
{
        preempt_disable();
}

static inline void __rcu_read_unlock(void)
{
        preempt_enable();
}

static inline void synchronize_rcu(void)
{
        synchronize_sched();
}

static inline int rcu_preempt_depth(void)
{
        return 0;
}

#endif /* #else #ifdef CONFIG_PREEMPT_RCU */

/* Internal to kernel */
void rcu_init(void);
void rcu_end_inkernel_boot(void);
void rcu_sched_qs(void);
void rcu_bh_qs(void);
void rcu_check_callbacks(int user);
struct notifier_block;
void rcu_idle_enter(void);
void rcu_idle_exit(void);
void rcu_irq_enter(void);
void rcu_irq_exit(void);
int rcu_cpu_notify(struct notifier_block *self,
                   unsigned long action, void *hcpu);

#ifdef CONFIG_RCU_STALL_COMMON
void rcu_sysrq_start(void);
void rcu_sysrq_end(void);
#else /* #ifdef CONFIG_RCU_STALL_COMMON */
static inline void rcu_sysrq_start(void)
{
}
static inline void rcu_sysrq_end(void)
{
}
#endif /* #else #ifdef CONFIG_RCU_STALL_COMMON */

#ifdef CONFIG_RCU_USER_QS
void rcu_user_enter(void);
void rcu_user_exit(void);
#else
static inline void rcu_user_enter(void) { }
static inline void rcu_user_exit(void) { }
static inline void rcu_user_hooks_switch(struct task_struct *prev,
                                         struct task_struct *next) { }
#endif /* CONFIG_RCU_USER_QS */

#ifdef CONFIG_RCU_NOCB_CPU
void rcu_init_nohz(void);
#else /* #ifdef CONFIG_RCU_NOCB_CPU */
static inline void rcu_init_nohz(void)
{
}
#endif /* #else #ifdef CONFIG_RCU_NOCB_CPU */

/**
 * RCU_NONIDLE - Indicate idle-loop code that needs RCU readers
 * @a: Code that RCU needs to pay attention to.
 *
 * RCU, RCU-bh, and RCU-sched read-side critical sections are forbidden
 * in the inner idle loop, that is, between the rcu_idle_enter() and
 * the rcu_idle_exit() -- RCU will happily ignore any such read-side
 * critical sections.  However, things like powertop need tracepoints
 * in the inner idle loop.
 *
 * This macro provides the way out:  RCU_NONIDLE(do_something_with_RCU())
 * will tell RCU that it needs to pay attending, invoke its argument
 * (in this example, a call to the do_something_with_RCU() function),
 * and then tell RCU to go back to ignoring this CPU.  It is permissible
 * to nest RCU_NONIDLE() wrappers, but the nesting level is currently
 * quite limited.  If deeper nesting is required, it will be necessary
 * to adjust DYNTICK_TASK_NESTING_VALUE accordingly.
 */
#define RCU_NONIDLE(a) \
        do { \
                rcu_irq_enter(); \
                do { a; } while (0); \
                rcu_irq_exit(); \
        } while (0)

/*
 * Note a voluntary context switch for RCU-tasks benefit.  This is a
 * macro rather than an inline function to avoid #include hell.
 */
#ifdef CONFIG_TASKS_RCU
#define TASKS_RCU(x) x
extern struct srcu_struct tasks_rcu_exit_srcu;
#define rcu_note_voluntary_context_switch(t) \
        do { \
                rcu_all_qs(); \
                if (ACCESS_ONCE((t)->rcu_tasks_holdout)) \
                        ACCESS_ONCE((t)->rcu_tasks_holdout) = false; \
        } while (0)
#else /* #ifdef CONFIG_TASKS_RCU */
#define TASKS_RCU(x) do { } while (0)
#define rcu_note_voluntary_context_switch(t)    rcu_all_qs()
#endif /* #else #ifdef CONFIG_TASKS_RCU */

/**
 * cond_resched_rcu_qs - Report potential quiescent states to RCU
 *
 * This macro resembles cond_resched(), except that it is defined to
 * report potential quiescent states to RCU-tasks even if the cond_resched()
 * machinery were to be shut off, as some advocate for PREEMPT kernels.
 */
#define cond_resched_rcu_qs() \
do { \
        if (!cond_resched()) \
                rcu_note_voluntary_context_switch(current); \
} while (0)

#if defined(CONFIG_DEBUG_LOCK_ALLOC) || defined(CONFIG_RCU_TRACE) || defined(CONFIG_SMP)
bool __rcu_is_watching(void);
#endif /* #if defined(CONFIG_DEBUG_LOCK_ALLOC) || defined(CONFIG_RCU_TRACE) || defined(CONFIG_SMP) */

/*
 * Infrastructure to implement the synchronize_() primitives in
 * TREE_RCU and rcu_barrier_() primitives in TINY_RCU.
 */

typedef void call_rcu_func_t(struct rcu_head *head,
                             void (*func)(struct rcu_head *head));
void wait_rcu_gp(call_rcu_func_t crf);

#if defined(CONFIG_TREE_RCU) || defined(CONFIG_PREEMPT_RCU)
#include <linux/rcutree.h>
#elif defined(CONFIG_TINY_RCU)
#include <linux/rcutiny.h>
#else
#error "Unknown RCU implementation specified to kernel configuration"
#endif

/*
 * init_rcu_head_on_stack()/destroy_rcu_head_on_stack() are needed for dynamic
 * initialization and destruction of rcu_head on the stack. rcu_head structures
 * allocated dynamically in the heap or defined statically don't need any
 * initialization.
 */
#ifdef CONFIG_DEBUG_OBJECTS_RCU_HEAD
void init_rcu_head(struct rcu_head *head);
void destroy_rcu_head(struct rcu_head *head);
void init_rcu_head_on_stack(struct rcu_head *head);
void destroy_rcu_head_on_stack(struct rcu_head *head);
#else /* !CONFIG_DEBUG_OBJECTS_RCU_HEAD */
static inline void init_rcu_head(struct rcu_head *head)
{
}

static inline void destroy_rcu_head(struct rcu_head *head)
{
}

static inline void init_rcu_head_on_stack(struct rcu_head *head)
{
}

static inline void destroy_rcu_head_on_stack(struct rcu_head *head)
{
}
#endif  /* #else !CONFIG_DEBUG_OBJECTS_RCU_HEAD */

#if defined(CONFIG_HOTPLUG_CPU) && defined(CONFIG_PROVE_RCU)
bool rcu_lockdep_current_cpu_online(void);
#else /* #if defined(CONFIG_HOTPLUG_CPU) && defined(CONFIG_PROVE_RCU) */
static inline bool rcu_lockdep_current_cpu_online(void)
{
        return true;
}
#endif /* #else #if defined(CONFIG_HOTPLUG_CPU) && defined(CONFIG_PROVE_RCU) */

#ifdef CONFIG_DEBUG_LOCK_ALLOC

static inline void rcu_lock_acquire(struct lockdep_map *map)
{
        lock_acquire(map, 0, 0, 2, 0, NULL, _THIS_IP_);
}

static inline void rcu_lock_release(struct lockdep_map *map)
{
        lock_release(map, 1, _THIS_IP_);
}

extern struct lockdep_map rcu_lock_map;
extern struct lockdep_map rcu_bh_lock_map;
extern struct lockdep_map rcu_sched_lock_map;
extern struct lockdep_map rcu_callback_map;
int debug_lockdep_rcu_enabled(void);

int rcu_read_lock_held(void);
int rcu_read_lock_bh_held(void);

/**
 * rcu_read_lock_sched_held() - might we be in RCU-sched read-side critical section?
 *
 * If CONFIG_DEBUG_LOCK_ALLOC is selected, returns nonzero iff in an
 * RCU-sched read-side critical section.  In absence of
 * CONFIG_DEBUG_LOCK_ALLOC, this assumes we are in an RCU-sched read-side
 * critical section unless it can prove otherwise.  Note that disabling
 * of preemption (including disabling irqs) counts as an RCU-sched
 * read-side critical section.  This is useful for debug checks in functions
 * that required that they be called within an RCU-sched read-side
 * critical section.
 *
 * Check debug_lockdep_rcu_enabled() to prevent false positives during boot
 * and while lockdep is disabled.
 *
 * Note that if the CPU is in the idle loop from an RCU point of
 * view (ie: that we are in the section between rcu_idle_enter() and
 * rcu_idle_exit()) then rcu_read_lock_held() returns false even if the CPU
 * did an rcu_read_lock().  The reason for this is that RCU ignores CPUs
 * that are in such a section, considering these as in extended quiescent
 * state, so such a CPU is effectively never in an RCU read-side critical
 * section regardless of what RCU primitives it invokes.  This state of
 * affairs is required --- we need to keep an RCU-free window in idle
 * where the CPU may possibly enter into low power mode. This way we can
 * notice an extended quiescent state to other CPUs that started a grace
 * period. Otherwise we would delay any grace period as long as we run in
 * the idle task.
 *
 * Similarly, we avoid claiming an SRCU read lock held if the current
 * CPU is offline.
 */
#ifdef CONFIG_PREEMPT_COUNT
static inline int rcu_read_lock_sched_held(void)
{
        int lockdep_opinion = 0;

        if (!debug_lockdep_rcu_enabled())
                return 1;
        if (!rcu_is_watching())
                return 0;
        if (!rcu_lockdep_current_cpu_online())
                return 0;
        if (debug_locks)
                lockdep_opinion = lock_is_held(&rcu_sched_lock_map);
        return lockdep_opinion || preempt_count() != 0 || irqs_disabled();
}
#else /* #ifdef CONFIG_PREEMPT_COUNT */
static inline int rcu_read_lock_sched_held(void)
{
        return 1;
}
#endif /* #else #ifdef CONFIG_PREEMPT_COUNT */

#else /* #ifdef CONFIG_DEBUG_LOCK_ALLOC */

# define rcu_lock_acquire(a)            do { } while (0)
# define rcu_lock_release(a)            do { } while (0)

static inline int rcu_read_lock_held(void)
{
        return 1;
}

static inline int rcu_read_lock_bh_held(void)
{
        return 1;
}

#ifdef CONFIG_PREEMPT_COUNT
static inline int rcu_read_lock_sched_held(void)
{
        return preempt_count() != 0 || irqs_disabled();
}
#else /* #ifdef CONFIG_PREEMPT_COUNT */
static inline int rcu_read_lock_sched_held(void)
{
        return 1;
}
#endif /* #else #ifdef CONFIG_PREEMPT_COUNT */

#endif /* #else #ifdef CONFIG_DEBUG_LOCK_ALLOC */

#ifdef CONFIG_PROVE_RCU

/**
 * rcu_lockdep_assert - emit lockdep splat if specified condition not met
 * @c: condition to check
 * @s: informative message
 */
#define rcu_lockdep_assert(c, s)                                        \
        do {                                                            \
                static bool __section(.data.unlikely) __warned;         \
                if (debug_lockdep_rcu_enabled() && !__warned && !(c)) { \
                        __warned = true;                                \
                        lockdep_rcu_suspicious(__FILE__, __LINE__, s);  \
                }                                                       \
        } while (0)

#if defined(CONFIG_PROVE_RCU) && !defined(CONFIG_PREEMPT_RCU)
static inline void rcu_preempt_sleep_check(void)
{
        rcu_lockdep_assert(!lock_is_held(&rcu_lock_map),
                           "Illegal context switch in RCU read-side critical section");
}
#else /* #ifdef CONFIG_PROVE_RCU */
static inline void rcu_preempt_sleep_check(void)
{
}
#endif /* #else #ifdef CONFIG_PROVE_RCU */

#define rcu_sleep_check()                                               \
        do {                                                            \
                rcu_preempt_sleep_check();                              \
                rcu_lockdep_assert(!lock_is_held(&rcu_bh_lock_map),     \
                                   "Illegal context switch in RCU-bh read-side critical section"); \
                rcu_lockdep_assert(!lock_is_held(&rcu_sched_lock_map),  \
                                   "Illegal context switch in RCU-sched read-side critical section"); \
        } while (0)

#else /* #ifdef CONFIG_PROVE_RCU */

#define rcu_lockdep_assert(c, s) do { } while (0)
#define rcu_sleep_check() do { } while (0)

#endif /* #else #ifdef CONFIG_PROVE_RCU */

/*
 * Helper functions for rcu_dereference_check(), rcu_dereference_protected()
 * and rcu_assign_pointer().  Some of these could be folded into their
 * callers, but they are left separate in order to ease introduction of
 * multiple flavors of pointers to match the multiple flavors of RCU
 * (e.g., __rcu_bh, * __rcu_sched, and __srcu), should this make sense in
 * the future.
 */

#ifdef __CHECKER__
#define rcu_dereference_sparse(p, space) \
        ((void)(((typeof(*p) space *)p) == p))
#else /* #ifdef __CHECKER__ */
#define rcu_dereference_sparse(p, space)
#endif /* #else #ifdef __CHECKER__ */

#define __rcu_access_pointer(p, space) \
({ \
        typeof(*p) *_________p1 = (typeof(*p) *__force)ACCESS_ONCE(p); \
        rcu_dereference_sparse(p, space); \
        ((typeof(*p) __force __kernel *)(_________p1)); \
})
#define __rcu_dereference_check(p, c, space) \
({ \
        /* Dependency order vs. p above. */ \
        typeof(*p) *________p1 = (typeof(*p) *__force)lockless_dereference(p); \
        rcu_lockdep_assert(c, "suspicious rcu_dereference_check() usage"); \
        rcu_dereference_sparse(p, space); \
        ((typeof(*p) __force __kernel *)(________p1)); \
})
#define __rcu_dereference_protected(p, c, space) \
({ \
        rcu_lockdep_assert(c, "suspicious rcu_dereference_protected() usage"); \
        rcu_dereference_sparse(p, space); \
        ((typeof(*p) __force __kernel *)(p)); \
})

#define __rcu_access_index(p, space) \
({ \
        typeof(p) _________p1 = ACCESS_ONCE(p); \
        rcu_dereference_sparse(p, space); \
        (_________p1); \
})
#define __rcu_dereference_index_check(p, c) \
({ \
        /* Dependency order vs. p above. */ \
        typeof(p) _________p1 = lockless_dereference(p); \
        rcu_lockdep_assert(c, \
                           "suspicious rcu_dereference_index_check() usage"); \
        (_________p1); \
})

/**
 * RCU_INITIALIZER() - statically initialize an RCU-protected global variable
 * @v: The value to statically initialize with.
 */
#define RCU_INITIALIZER(v) (typeof(*(v)) __force __rcu *)(v)

/**
 * lockless_dereference() - safely load a pointer for later dereference
 * @p: The pointer to load
 *
 * Similar to rcu_dereference(), but for situations where the pointed-to
 * object's lifetime is managed by something other than RCU.  That
 * "something other" might be reference counting or simple immortality.
 */
#define lockless_dereference(p) \
({ \
        typeof(p) _________p1 = ACCESS_ONCE(p); \
        smp_read_barrier_depends(); /* Dependency order vs. p above. */ \
        (_________p1); \
})

/**
 * rcu_assign_pointer() - assign to RCU-protected pointer
 * @p: pointer to assign to
 * @v: value to assign (publish)
 *
 * Assigns the specified value to the specified RCU-protected
 * pointer, ensuring that any concurrent RCU readers will see
 * any prior initialization.
 *
 * Inserts memory barriers on architectures that require them
 * (which is most of them), and also prevents the compiler from
 * reordering the code that initializes the structure after the pointer
 * assignment.  More importantly, this call documents which pointers
 * will be dereferenced by RCU read-side code.
 *
 * In some special cases, you may use RCU_INIT_POINTER() instead
 * of rcu_assign_pointer().  RCU_INIT_POINTER() is a bit faster due
 * to the fact that it does not constrain either the CPU or the compiler.
 * That said, using RCU_INIT_POINTER() when you should have used
 * rcu_assign_pointer() is a very bad thing that results in
 * impossible-to-diagnose memory corruption.  So please be careful.
 * See the RCU_INIT_POINTER() comment header for details.
 *
 * Note that rcu_assign_pointer() evaluates each of its arguments only
 * once, appearances notwithstanding.  One of the "extra" evaluations
 * is in typeof() and the other visible only to sparse (__CHECKER__),
 * neither of which actually execute the argument.  As with most cpp
 * macros, this execute-arguments-only-once property is important, so
 * please be careful when making changes to rcu_assign_pointer() and the
 * other macros that it invokes.
 */
#define rcu_assign_pointer(p, v) smp_store_release(&p, RCU_INITIALIZER(v))

/**
 * rcu_access_pointer() - fetch RCU pointer with no dereferencing
 * @p: The pointer to read
 *
 * Return the value of the specified RCU-protected pointer, but omit the
 * smp_read_barrier_depends() and keep the ACCESS_ONCE().  This is useful
 * when the value of this pointer is accessed, but the pointer is not
 * dereferenced, for example, when testing an RCU-protected pointer against
 * NULL.  Although rcu_access_pointer() may also be used in cases where
 * update-side locks prevent the value of the pointer from changing, you
 * should instead use rcu_dereference_protected() for this use case.
 *
 * It is also permissible to use rcu_access_pointer() when read-side
 * access to the pointer was removed at least one grace period ago, as
 * is the case in the context of the RCU callback that is freeing up
 * the data, or after a synchronize_rcu() returns.  This can be useful
 * when tearing down multi-linked structures after a grace period
 * has elapsed.
 */
#define rcu_access_pointer(p) __rcu_access_pointer((p), __rcu)

/**
 * rcu_dereference_check() - rcu_dereference with debug checking
 * @p: The pointer to read, prior to dereferencing
 * @c: The conditions under which the dereference will take place
 *
 * Do an rcu_dereference(), but check that the conditions under which the
 * dereference will take place are correct.  Typically the conditions
 * indicate the various locking conditions that should be held at that
 * point.  The check should return true if the conditions are satisfied.
 * An implicit check for being in an RCU read-side critical section
 * (rcu_read_lock()) is included.
 *
 * For example:
 *
 *      bar = rcu_dereference_check(foo->bar, lockdep_is_held(&foo->lock));
 *
 * could be used to indicate to lockdep that foo->bar may only be dereferenced
 * if either rcu_read_lock() is held, or that the lock required to replace
 * the bar struct at foo->bar is held.
 *
 * Note that the list of conditions may also include indications of when a lock
 * need not be held, for example during initialisation or destruction of the
 * target struct:
 *
 *      bar = rcu_dereference_check(foo->bar, lockdep_is_held(&foo->lock) ||
 *                                            atomic_read(&foo->usage) == 0);
 *
 * Inserts memory barriers on architectures that require them
 * (currently only the Alpha), prevents the compiler from refetching
 * (and from merging fetches), and, more importantly, documents exactly
 * which pointers are protected by RCU and checks that the pointer is
 * annotated as __rcu.
 */
#define rcu_dereference_check(p, c) \
        __rcu_dereference_check((p), (c) || rcu_read_lock_held(), __rcu)

/**
 * rcu_dereference_bh_check() - rcu_dereference_bh with debug checking
 * @p: The pointer to read, prior to dereferencing
 * @c: The conditions under which the dereference will take place
 *
 * This is the RCU-bh counterpart to rcu_dereference_check().
 */
#define rcu_dereference_bh_check(p, c) \
        __rcu_dereference_check((p), (c) || rcu_read_lock_bh_held(), __rcu)

/**
 * rcu_dereference_sched_check() - rcu_dereference_sched with debug checking
 * @p: The pointer to read, prior to dereferencing
 * @c: The conditions under which the dereference will take place
 *
 * This is the RCU-sched counterpart to rcu_dereference_check().
 */
#define rcu_dereference_sched_check(p, c) \
        __rcu_dereference_check((p), (c) || rcu_read_lock_sched_held(), \
                                __rcu)

#define rcu_dereference_raw(p) rcu_dereference_check(p, 1) /*@@@ needed? @@@*/

/*
 * The tracing infrastructure traces RCU (we want that), but unfortunately
 * some of the RCU checks causes tracing to lock up the system.
 *
 * The tracing version of rcu_dereference_raw() must not call
 * rcu_read_lock_held().
 */
#define rcu_dereference_raw_notrace(p) __rcu_dereference_check((p), 1, __rcu)

/**
 * rcu_access_index() - fetch RCU index with no dereferencing
 * @p: The index to read
 *
 * Return the value of the specified RCU-protected index, but omit the
 * smp_read_barrier_depends() and keep the ACCESS_ONCE().  This is useful
 * when the value of this index is accessed, but the index is not
 * dereferenced, for example, when testing an RCU-protected index against
 * -1.  Although rcu_access_index() may also be used in cases where
 * update-side locks prevent the value of the index from changing, you
 * should instead use rcu_dereference_index_protected() for this use case.
 */
#define rcu_access_index(p) __rcu_access_index((p), __rcu)

/**
 * rcu_dereference_index_check() - rcu_dereference for indices with debug checking
 * @p: The pointer to read, prior to dereferencing
 * @c: The conditions under which the dereference will take place
 *
 * Similar to rcu_dereference_check(), but omits the sparse checking.
 * This allows rcu_dereference_index_check() to be used on integers,
 * which can then be used as array indices.  Attempting to use
 * rcu_dereference_check() on an integer will give compiler warnings
 * because the sparse address-space mechanism relies on dereferencing
 * the RCU-protected pointer.  Dereferencing integers is not something
 * that even gcc will put up with.
 *
 * Note that this function does not implicitly check for RCU read-side
 * critical sections.  If this function gains lots of uses, it might
 * make sense to provide versions for each flavor of RCU, but it does
 * not make sense as of early 2010.
 */
#define rcu_dereference_index_check(p, c) \
        __rcu_dereference_index_check((p), (c))

/**
 * rcu_dereference_protected() - fetch RCU pointer when updates prevented
 * @p: The pointer to read, prior to dereferencing
 * @c: The conditions under which the dereference will take place
 *
 * Return the value of the specified RCU-protected pointer, but omit
 * both the smp_read_barrier_depends() and the ACCESS_ONCE().  This
 * is useful in cases where update-side locks prevent the value of the
 * pointer from changing.  Please note that this primitive does -not-
 * prevent the compiler from repeating this reference or combining it
 * with other references, so it should not be used without protection
 * of appropriate locks.
 *
 * This function is only for update-side use.  Using this function
 * when protected only by rcu_read_lock() will result in infrequent
 * but very ugly failures.
 */
#define rcu_dereference_protected(p, c) \
        __rcu_dereference_protected((p), (c), __rcu)


/**
 * rcu_dereference() - fetch RCU-protected pointer for dereferencing
 * @p: The pointer to read, prior to dereferencing
 *
 * This is a simple wrapper around rcu_dereference_check().
 */
#define rcu_dereference(p) rcu_dereference_check(p, 0)

/**
 * rcu_dereference_bh() - fetch an RCU-bh-protected pointer for dereferencing
 * @p: The pointer to read, prior to dereferencing
 *
 * Makes rcu_dereference_check() do the dirty work.
 */
#define rcu_dereference_bh(p) rcu_dereference_bh_check(p, 0)

/**
 * rcu_dereference_sched() - fetch RCU-sched-protected pointer for dereferencing
 * @p: The pointer to read, prior to dereferencing
 *
 * Makes rcu_dereference_check() do the dirty work.
 */
#define rcu_dereference_sched(p) rcu_dereference_sched_check(p, 0)

/**
 * rcu_read_lock() - mark the beginning of an RCU read-side critical section
 *
 * When synchronize_rcu() is invoked on one CPU while other CPUs
 * are within RCU read-side critical sections, then the
 * synchronize_rcu() is guaranteed to block until after all the other
 * CPUs exit their critical sections.  Similarly, if call_rcu() is invoked
 * on one CPU while other CPUs are within RCU read-side critical
 * sections, invocation of the corresponding RCU callback is deferred
 * until after the all the other CPUs exit their critical sections.
 *
 * Note, however, that RCU callbacks are permitted to run concurrently
 * with new RCU read-side critical sections.  One way that this can happen
 * is via the following sequence of events: (1) CPU 0 enters an RCU
 * read-side critical section, (2) CPU 1 invokes call_rcu() to register
 * an RCU callback, (3) CPU 0 exits the RCU read-side critical section,
 * (4) CPU 2 enters a RCU read-side critical section, (5) the RCU
 * callback is invoked.  This is legal, because the RCU read-side critical
 * section that was running concurrently with the call_rcu() (and which
 * therefore might be referencing something that the corresponding RCU
 * callback would free up) has completed before the corresponding
 * RCU callback is invoked.
 *
 * RCU read-side critical sections may be nested.  Any deferred actions
 * will be deferred until the outermost RCU read-side critical section
 * completes.
 *
 * You can avoid reading and understanding the next paragraph by
 * following this rule: don't put anything in an rcu_read_lock() RCU
 * read-side critical section that would block in a !PREEMPT kernel.
 * But if you want the full story, read on!
 *
 * In non-preemptible RCU implementations (TREE_RCU and TINY_RCU),
 * it is illegal to block while in an RCU read-side critical section.
 * In preemptible RCU implementations (PREEMPT_RCU) in CONFIG_PREEMPT
 * kernel builds, RCU read-side critical sections may be preempted,
 * but explicit blocking is illegal.  Finally, in preemptible RCU
 * implementations in real-time (with -rt patchset) kernel builds, RCU
 * read-side critical sections may be preempted and they may also block, but
 * only when acquiring spinlocks that are subject to priority inheritance.
 */
static inline void rcu_read_lock(void)
{
        __rcu_read_lock();
        __acquire(RCU);
        rcu_lock_acquire(&rcu_lock_map);
        rcu_lockdep_assert(rcu_is_watching(),
                           "rcu_read_lock() used illegally while idle");
}

/*
 * So where is rcu_write_lock()?  It does not exist, as there is no
 * way for writers to lock out RCU readers.  This is a feature, not
 * a bug -- this property is what provides RCU's performance benefits.
 * Of course, writers must coordinate with each other.  The normal
 * spinlock primitives work well for this, but any other technique may be
 * used as well.  RCU does not care how the writers keep out of each
 * others' way, as long as they do so.
 */

/**
 * rcu_read_unlock() - marks the end of an RCU read-side critical section.
 *
 * In most situations, rcu_read_unlock() is immune from deadlock.
 * However, in kernels built with CONFIG_RCU_BOOST, rcu_read_unlock()
 * is responsible for deboosting, which it does via rt_mutex_unlock().
 * Unfortunately, this function acquires the scheduler's runqueue and
 * priority-inheritance spinlocks.  This means that deadlock could result
 * if the caller of rcu_read_unlock() already holds one of these locks or
 * any lock that is ever acquired while holding them; or any lock which
 * can be taken from interrupt context because rcu_boost()->rt_mutex_lock()
 * does not disable irqs while taking ->wait_lock.
 *
 * That said, RCU readers are never priority boosted unless they were
 * preempted.  Therefore, one way to avoid deadlock is to make sure
 * that preemption never happens within any RCU read-side critical
 * section whose outermost rcu_read_unlock() is called with one of
 * rt_mutex_unlock()'s locks held.  Such preemption can be avoided in
 * a number of ways, for example, by invoking preempt_disable() before
 * critical section's outermost rcu_read_lock().
 *
 * Given that the set of locks acquired by rt_mutex_unlock() might change
 * at any time, a somewhat more future-proofed approach is to make sure
 * that that preemption never happens within any RCU read-side critical
 * section whose outermost rcu_read_unlock() is called with irqs disabled.
 * This approach relies on the fact that rt_mutex_unlock() currently only
 * acquires irq-disabled locks.
 *
 * The second of these two approaches is best in most situations,
 * however, the first approach can also be useful, at least to those
 * developers willing to keep abreast of the set of locks acquired by
 * rt_mutex_unlock().
 *
 * See rcu_read_lock() for more information.
 */
static inline void rcu_read_unlock(void)
{
        rcu_lockdep_assert(rcu_is_watching(),
                           "rcu_read_unlock() used illegally while idle");
        __release(RCU);
        __rcu_read_unlock();
        rcu_lock_release(&rcu_lock_map); /* Keep acq info for rls diags. */
}

/**
 * rcu_read_lock_bh() - mark the beginning of an RCU-bh critical section
 *
 * This is equivalent of rcu_read_lock(), but to be used when updates
 * are being done using call_rcu_bh() or synchronize_rcu_bh(). Since
 * both call_rcu_bh() and synchronize_rcu_bh() consider completion of a
 * softirq handler to be a quiescent state, a process in RCU read-side
 * critical section must be protected by disabling softirqs. Read-side
 * critical sections in interrupt context can use just rcu_read_lock(),
 * though this should at least be commented to avoid confusing people
 * reading the code.
 *
 * Note that rcu_read_lock_bh() and the matching rcu_read_unlock_bh()
 * must occur in the same context, for example, it is illegal to invoke
 * rcu_read_unlock_bh() from one task if the matching rcu_read_lock_bh()
 * was invoked from some other task.
 */
static inline void rcu_read_lock_bh(void)
{
        local_bh_disable();
        __acquire(RCU_BH);
        rcu_lock_acquire(&rcu_bh_lock_map);
        rcu_lockdep_assert(rcu_is_watching(),
                           "rcu_read_lock_bh() used illegally while idle");
}

/*
  * rcu_read_unlock_bh - marks the end of a softirq-only RCU critical section
  *
  * See rcu_read_lock_bh() for more information.
  */
 static inline void rcu_read_unlock_bh(void)
 {
         rcu_lockdep_assert(rcu_is_watching(),
                            "rcu_read_unlock_bh() used illegally while idle");
         rcu_lock_release(&rcu_bh_lock_map);
         __release(RCU_BH);
         local_bh_enable();
 }
 
 /**
  * rcu_read_lock_sched() - mark the beginning of a RCU-sched critical section
  *
  * This is equivalent of rcu_read_lock(), but to be used when updates
  * are being done using call_rcu_sched() or synchronize_rcu_sched().
  * Read-side critical sections can also be introduced by anything that
  * disables preemption, including local_irq_disable() and friends.
  *
  * Note that rcu_read_lock_sched() and the matching rcu_read_unlock_sched()
  * must occur in the same context, for example, it is illegal to invoke
  * rcu_read_unlock_sched() from process context if the matching
  * rcu_read_lock_sched() was invoked from an NMI handler.
  */
 static inline void rcu_read_lock_sched(void)
 {
         preempt_disable();
         __acquire(RCU_SCHED);
         rcu_lock_acquire(&rcu_sched_lock_map);
         rcu_lockdep_assert(rcu_is_watching(),
                            "rcu_read_lock_sched() used illegally while idle");
 }
 
 /* Used by lockdep and tracing: cannot be traced, cannot call lockdep. */
 static inline notrace void rcu_read_lock_sched_notrace(void)
 {
         preempt_disable_notrace();
         __acquire(RCU_SCHED);
 }
 
 /*
  * rcu_read_unlock_sched - marks the end of a RCU-classic critical section
  *
  * See rcu_read_lock_sched for more information.
  */
 static inline void rcu_read_unlock_sched(void)
 {
         rcu_lockdep_assert(rcu_is_watching(),
                            "rcu_read_unlock_sched() used illegally while idle");
         rcu_lock_release(&rcu_sched_lock_map);
         __release(RCU_SCHED);
         preempt_enable();
 }
 
 /* Used by lockdep and tracing: cannot be traced, cannot call lockdep. */
 static inline notrace void rcu_read_unlock_sched_notrace(void)
 {
         __release(RCU_SCHED);
         preempt_enable_notrace();
 }
 
 /**
  * RCU_INIT_POINTER() - initialize an RCU protected pointer
  *
  * Initialize an RCU-protected pointer in special cases where readers
  * do not need ordering constraints on the CPU or the compiler.  These
  * special cases are:
  *
  * 1.   This use of RCU_INIT_POINTER() is NULLing out the pointer -or-
  * 2.   The caller has taken whatever steps are required to prevent
  *      RCU readers from concurrently accessing this pointer -or-
  * 3.   The referenced data structure has already been exposed to
  *      readers either at compile time or via rcu_assign_pointer() -and-
  *      a.      You have not made -any- reader-visible changes to
  *              this structure since then -or-
  *      b.      It is OK for readers accessing this structure from its
  *              new location to see the old state of the structure.  (For
  *              example, the changes were to statistical counters or to
  *              other state where exact synchronization is not required.)
  *
  * Failure to follow these rules governing use of RCU_INIT_POINTER() will
  * result in impossible-to-diagnose memory corruption.  As in the structures
  * will look OK in crash dumps, but any concurrent RCU readers might
  * see pre-initialized values of the referenced data structure.  So
  * please be very careful how you use RCU_INIT_POINTER()!!!
  *
  * If you are creating an RCU-protected linked structure that is accessed
  * by a single external-to-structure RCU-protected pointer, then you may
  * use RCU_INIT_POINTER() to initialize the internal RCU-protected
  * pointers, but you must use rcu_assign_pointer() to initialize the
  * external-to-structure pointer -after- you have completely initialized
  * the reader-accessible portions of the linked structure.
  *
  * Note that unlike rcu_assign_pointer(), RCU_INIT_POINTER() provides no
  * ordering guarantees for either the CPU or the compiler.
  */
 #define RCU_INIT_POINTER(p, v) \
         do { \
                 rcu_dereference_sparse(p, __rcu); \
                 p = RCU_INITIALIZER(v); \
         } while (0)
 
 /**
  * RCU_POINTER_INITIALIZER() - statically initialize an RCU protected pointer
  *
  * GCC-style initialization for an RCU-protected pointer in a structure field.
  */
 #define RCU_POINTER_INITIALIZER(p, v) \
                 .p = RCU_INITIALIZER(v)
 
 /*
  * Does the specified offset indicate that the corresponding rcu_head
  * structure can be handled by kfree_rcu()?
  */
 #define __is_kfree_rcu_offset(offset) ((offset) < 4096)
 
 /*
  * Helper macro for kfree_rcu() to prevent argument-expansion eyestrain.
  */
 #define __kfree_rcu(head, offset) \
         do { \
                 BUILD_BUG_ON(!__is_kfree_rcu_offset(offset)); \
                 kfree_call_rcu(head, (void (*)(struct rcu_head *))(unsigned long)(offset)); \
         } while (0)
 
 /**
  * kfree_rcu() - kfree an object after a grace period.
  * @ptr:        pointer to kfree
  * @rcu_head:   the name of the struct rcu_head within the type of @ptr.
  *
  * Many rcu callbacks functions just call kfree() on the base structure.
  * These functions are trivial, but their size adds up, and furthermore
  * when they are used in a kernel module, that module must invoke the
  * high-latency rcu_barrier() function at module-unload time.
  *
  * The kfree_rcu() function handles this issue.  Rather than encoding a
  * function address in the embedded rcu_head structure, kfree_rcu() instead
  * encodes the offset of the rcu_head structure within the base structure.
  * Because the functions are not allowed in the low-order 4096 bytes of
  * kernel virtual memory, offsets up to 4095 bytes can be accommodated.
  * If the offset is larger than 4095 bytes, a compile-time error will
  * be generated in __kfree_rcu().  If this error is triggered, you can
  * either fall back to use of call_rcu() or rearrange the structure to
  * position the rcu_head structure into the first 4096 bytes.
  *
  * Note that the allowable offset might decrease in the future, for example,
  * to allow something like kmem_cache_free_rcu().
  *
  * The BUILD_BUG_ON check must not involve any function calls, hence the
  * checks are done in macros here.
  */
 #define kfree_rcu(ptr, rcu_head)                                        \
         __kfree_rcu(&((ptr)->rcu_head), offsetof(typeof(*(ptr)), rcu_head))
 
 #if defined(CONFIG_TINY_RCU) || defined(CONFIG_RCU_NOCB_CPU_ALL)
 static inline int rcu_needs_cpu(unsigned long *delta_jiffies)
 {
         *delta_jiffies = ULONG_MAX;
         return 0;
 }
 #endif /* #if defined(CONFIG_TINY_RCU) || defined(CONFIG_RCU_NOCB_CPU_ALL) */
 
 #if defined(CONFIG_RCU_NOCB_CPU_ALL)
 static inline bool rcu_is_nocb_cpu(int cpu) { return true; }
 #elif defined(CONFIG_RCU_NOCB_CPU)
 bool rcu_is_nocb_cpu(int cpu);
 #else
 static inline bool rcu_is_nocb_cpu(int cpu) { return false; }
 #endif
 
 
 /* Only for use by adaptive-ticks code. */
 #ifdef CONFIG_NO_HZ_FULL_SYSIDLE
 bool rcu_sys_is_idle(void);
 void rcu_sysidle_force_exit(void);
 #else /* #ifdef CONFIG_NO_HZ_FULL_SYSIDLE */
 
 static inline bool rcu_sys_is_idle(void)
 {
         return false;
 }
 
 static inline void rcu_sysidle_force_exit(void)
 {
 }
 
 #endif /* #else #ifdef CONFIG_NO_HZ_FULL_SYSIDLE */
 
 
 #endif /* __LINUX_RCUPDATE_H */
 
```

```
/* rculist.h */
    #ifndef _LINUX_RCULIST_H
    #define _LINUX_RCULIST_H
    
    #ifdef __KERNEL__
    
    /*
     * RCU-protected list version
     */
    #include <linux/list.h>
    #include <linux/rcupdate.h>
    
    /*
     * Why is there no list_empty_rcu()?  Because list_empty() serves this
     * purpose.  The list_empty() function fetches the RCU-protected pointer
     * and compares it to the address of the list head, but neither dereferences
     * this pointer itself nor provides this pointer to the caller.  Therefore,
     * it is not necessary to use rcu_dereference(), so that list_empty() can
     * be used anywhere you would want to use a list_empty_rcu().
     */
    
    /*
     * INIT_LIST_HEAD_RCU - Initialize a list_head visible to RCU readers
     * @list: list to be initialized
     *
     * You should instead use INIT_LIST_HEAD() for normal initialization and
     * cleanup tasks, when readers have no access to the list being initialized.
     * However, if the list being initialized is visible to readers, you
     * need to keep the compiler from being too mischievous.
     */
    static inline void INIT_LIST_HEAD_RCU(struct list_head *list)
    {
            ACCESS_ONCE(list->next) = list;
            ACCESS_ONCE(list->prev) = list;
    }
    
    /*
     * return the ->next pointer of a list_head in an rcu safe
     * way, we must not access it directly
     */
    #define list_next_rcu(list)     (*((struct list_head __rcu **)(&(list)->next)))
    
    /*
     * Insert a new entry between two known consecutive entries.
     *
     * This is only for internal list manipulation where we know
     * the prev/next entries already!
     */
    #ifndef CONFIG_DEBUG_LIST
    static inline void __list_add_rcu(struct list_head *new,
                    struct list_head *prev, struct list_head *next)
    {
            new->next = next;
            new->prev = prev;
            rcu_assign_pointer(list_next_rcu(prev), new);
            next->prev = new;
    }
    #else
    void __list_add_rcu(struct list_head *new,
                        struct list_head *prev, struct list_head *next);
    #endif
    
    /**
     * list_add_rcu - add a new entry to rcu-protected list
     * @new: new entry to be added
     * @head: list head to add it after
     *
     * Insert a new entry after the specified head.
     * This is good for implementing stacks.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as list_add_rcu()
     * or list_del_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * list_for_each_entry_rcu().
     */
    static inline void list_add_rcu(struct list_head *new, struct list_head *head)
    {
            __list_add_rcu(new, head, head->next);
    }
    
    /**
     * list_add_tail_rcu - add a new entry to rcu-protected list
     * @new: new entry to be added
     * @head: list head to add it before
     *
     * Insert a new entry before the specified head.
     * This is useful for implementing queues.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as list_add_tail_rcu()
     * or list_del_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * list_for_each_entry_rcu().
     */
    static inline void list_add_tail_rcu(struct list_head *new,
                                            struct list_head *head)
    {
            __list_add_rcu(new, head->prev, head);
    }
    
    /**
     * list_del_rcu - deletes entry from list without re-initialization
     * @entry: the element to delete from the list.
     *
     * Note: list_empty() on entry does not return true after this,
     * the entry is in an undefined state. It is useful for RCU based
     * lockfree traversal.
     *
     * In particular, it means that we can not poison the forward
     * pointers that may still be used for walking the list.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as list_del_rcu()
     * or list_add_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * list_for_each_entry_rcu().
     *
     * Note that the caller is not permitted to immediately free
     * the newly deleted entry.  Instead, either synchronize_rcu()
     * or call_rcu() must be used to defer freeing until an RCU
     * grace period has elapsed.
     */
    static inline void list_del_rcu(struct list_head *entry)
    {
            __list_del_entry(entry);
            entry->prev = LIST_POISON2;
    }
    
    /**
     * hlist_del_init_rcu - deletes entry from hash list with re-initialization
     * @n: the element to delete from the hash list.
     *
     * Note: list_unhashed() on the node return true after this. It is
     * useful for RCU based read lockfree traversal if the writer side
     * must know if the list entry is still hashed or already unhashed.
     *
     * In particular, it means that we can not poison the forward pointers
     * that may still be used for walking the hash list and we can only
     * zero the pprev pointer so list_unhashed() will return true after
     * this.
     *
     * The caller must take whatever precautions are necessary (such as
     * holding appropriate locks) to avoid racing with another
     * list-mutation primitive, such as hlist_add_head_rcu() or
     * hlist_del_rcu(), running on this same list.  However, it is
     * perfectly legal to run concurrently with the _rcu list-traversal
     * primitives, such as hlist_for_each_entry_rcu().
     */
    static inline void hlist_del_init_rcu(struct hlist_node *n)
    {
            if (!hlist_unhashed(n)) {
                    __hlist_del(n);
                    n->pprev = NULL;
            }
    }
    
    /**
     * list_replace_rcu - replace old entry by new one
     * @old : the element to be replaced
     * @new : the new element to insert
     *
     * The @old entry will be replaced with the @new entry atomically.
     * Note: @old should not be empty.
     */
    static inline void list_replace_rcu(struct list_head *old,
                                    struct list_head *new)
    {
            new->next = old->next;
            new->prev = old->prev;
            rcu_assign_pointer(list_next_rcu(new->prev), new);
            new->next->prev = new;
            old->prev = LIST_POISON2;
    }
    
    /**
     * list_splice_init_rcu - splice an RCU-protected list into an existing list.
     * @list:       the RCU-protected list to splice
     * @head:       the place in the list to splice the first list into
     * @sync:       function to sync: synchronize_rcu(), synchronize_sched(), ...
     *
     * @head can be RCU-read traversed concurrently with this function.
     *
     * Note that this function blocks.
     *
     * Important note: the caller must take whatever action is necessary to
     *      prevent any other updates to @head.  In principle, it is possible
     *      to modify the list as soon as sync() begins execution.
     *      If this sort of thing becomes necessary, an alternative version
     *      based on call_rcu() could be created.  But only if -really-
     *      needed -- there is no shortage of RCU API members.
     */
    static inline void list_splice_init_rcu(struct list_head *list,
                                            struct list_head *head,
                                            void (*sync)(void))
    {
            struct list_head *first = list->next;
            struct list_head *last = list->prev;
            struct list_head *at = head->next;
    
            if (list_empty(list))
                    return;
    
            /*
             * "first" and "last" tracking list, so initialize it.  RCU readers
             * have access to this list, so we must use INIT_LIST_HEAD_RCU()
             * instead of INIT_LIST_HEAD().
             */
    
            INIT_LIST_HEAD_RCU(list);
    
            /*
             * At this point, the list body still points to the source list.
             * Wait for any readers to finish using the list before splicing
             * the list body into the new list.  Any new readers will see
             * an empty list.
             */
    
            sync();
    
            /*
             * Readers are finished with the source list, so perform splice.
             * The order is important if the new list is global and accessible
             * to concurrent RCU readers.  Note that RCU readers are not
             * permitted to traverse the prev pointers without excluding
             * this function.
             */
    
            last->next = at;
            rcu_assign_pointer(list_next_rcu(head), first);
            first->prev = head;
            at->prev = last;
    }
    
    /**
     * list_entry_rcu - get the struct for this entry
     * @ptr:        the &struct list_head pointer.
     * @type:       the type of the struct this is embedded in.
     * @member:     the name of the list_head within the struct.
     *
     * This primitive may safely run concurrently with the _rcu list-mutation
     * primitives such as list_add_rcu() as long as it's guarded by rcu_read_lock().
     */
    #define list_entry_rcu(ptr, type, member) \
    ({ \
            typeof(*ptr) __rcu *__ptr = (typeof(*ptr) __rcu __force *)ptr; \
            container_of((typeof(ptr))rcu_dereference_raw(__ptr), type, member); \
    })
    
    /**
     * Where are list_empty_rcu() and list_first_entry_rcu()?
     *
     * Implementing those functions following their counterparts list_empty() and
     * list_first_entry() is not advisable because they lead to subtle race
     * conditions as the following snippet shows:
     *
     * if (!list_empty_rcu(mylist)) {
     *      struct foo *bar = list_first_entry_rcu(mylist, struct foo, list_member);
     *      do_something(bar);
     * }
     *
     * The list may not be empty when list_empty_rcu checks it, but it may be when
     * list_first_entry_rcu rereads the ->next pointer.
     *
     * Rereading the ->next pointer is not a problem for list_empty() and
     * list_first_entry() because they would be protected by a lock that blocks
     * writers.
     *
     * See list_first_or_null_rcu for an alternative.
     */
    
    /**
     * list_first_or_null_rcu - get the first element from a list
     * @ptr:        the list head to take the element from.
     * @type:       the type of the struct this is embedded in.
     * @member:     the name of the list_head within the struct.
     *
     * Note that if the list is empty, it returns NULL.
     *
     * This primitive may safely run concurrently with the _rcu list-mutation
     * primitives such as list_add_rcu() as long as it's guarded by rcu_read_lock().
     */
    #define list_first_or_null_rcu(ptr, type, member) \
    ({ \
            struct list_head *__ptr = (ptr); \
            struct list_head *__next = ACCESS_ONCE(__ptr->next); \
            likely(__ptr != __next) ? list_entry_rcu(__next, type, member) : NULL; \
    })
    
    /**
     * list_for_each_entry_rcu      -       iterate over rcu list of given type
     * @pos:        the type * to use as a loop cursor.
     * @head:       the head for your list.
     * @member:     the name of the list_head within the struct.
     *
     * This list-traversal primitive may safely run concurrently with
     * the _rcu list-mutation primitives such as list_add_rcu()
     * as long as the traversal is guarded by rcu_read_lock().
     */
    #define list_for_each_entry_rcu(pos, head, member) \
            for (pos = list_entry_rcu((head)->next, typeof(*pos), member); \
                    &pos->member != (head); \
                    pos = list_entry_rcu(pos->member.next, typeof(*pos), member))
    
    /**
     * list_for_each_entry_continue_rcu - continue iteration over list of given type
     * @pos:        the type * to use as a loop cursor.
     * @head:       the head for your list.
     * @member:     the name of the list_head within the struct.
     *
     * Continue to iterate over list of given type, continuing after
     * the current position.
     */
    #define list_for_each_entry_continue_rcu(pos, head, member)             \
            for (pos = list_entry_rcu(pos->member.next, typeof(*pos), member); \
                 &pos->member != (head);    \
                 pos = list_entry_rcu(pos->member.next, typeof(*pos), member))
    
    /**
     * hlist_del_rcu - deletes entry from hash list without re-initialization
     * @n: the element to delete from the hash list.
     *
     * Note: list_unhashed() on entry does not return true after this,
     * the entry is in an undefined state. It is useful for RCU based
     * lockfree traversal.
     *
     * In particular, it means that we can not poison the forward
     * pointers that may still be used for walking the hash list.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as hlist_add_head_rcu()
     * or hlist_del_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * hlist_for_each_entry().
     */
    static inline void hlist_del_rcu(struct hlist_node *n)
    {
            __hlist_del(n);
            n->pprev = LIST_POISON2;
    }
    
    /**
     * hlist_replace_rcu - replace old entry by new one
     * @old : the element to be replaced
     * @new : the new element to insert
     *
     * The @old entry will be replaced with the @new entry atomically.
     */
    static inline void hlist_replace_rcu(struct hlist_node *old,
                                            struct hlist_node *new)
    {
            struct hlist_node *next = old->next;
    
            new->next = next;
            new->pprev = old->pprev;
            rcu_assign_pointer(*(struct hlist_node __rcu **)new->pprev, new);
            if (next)
                    new->next->pprev = &new->next;
            old->pprev = LIST_POISON2;
    }
    
    /*
     * return the first or the next element in an RCU protected hlist
     */
    #define hlist_first_rcu(head)   (*((struct hlist_node __rcu **)(&(head)->first)))
    #define hlist_next_rcu(node)    (*((struct hlist_node __rcu **)(&(node)->next)))
    #define hlist_pprev_rcu(node)   (*((struct hlist_node __rcu **)((node)->pprev)))
    
    /**
     * hlist_add_head_rcu
     * @n: the element to add to the hash list.
     * @h: the list to add to.
     *
     * Description:
     * Adds the specified element to the specified hlist,
     * while permitting racing traversals.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as hlist_add_head_rcu()
     * or hlist_del_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * hlist_for_each_entry_rcu(), used to prevent memory-consistency
     * problems on Alpha CPUs.  Regardless of the type of CPU, the
     * list-traversal primitive must be guarded by rcu_read_lock().
     */
    static inline void hlist_add_head_rcu(struct hlist_node *n,
                                            struct hlist_head *h)
    {
            struct hlist_node *first = h->first;
    
            n->next = first;
            n->pprev = &h->first;
            rcu_assign_pointer(hlist_first_rcu(h), n);
            if (first)
                    first->pprev = &n->next;
    }
    
    /**
     * hlist_add_before_rcu
     * @n: the new element to add to the hash list.
     * @next: the existing element to add the new element before.
     *
     * Description:
     * Adds the specified element to the specified hlist
     * before the specified node while permitting racing traversals.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as hlist_add_head_rcu()
     * or hlist_del_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * hlist_for_each_entry_rcu(), used to prevent memory-consistency
     * problems on Alpha CPUs.
     */
    static inline void hlist_add_before_rcu(struct hlist_node *n,
                                            struct hlist_node *next)
    {
            n->pprev = next->pprev;
            n->next = next;
            rcu_assign_pointer(hlist_pprev_rcu(n), n);
            next->pprev = &n->next;
    }
    
    /**
     * hlist_add_behind_rcu
     * @n: the new element to add to the hash list.
     * @prev: the existing element to add the new element after.
     *
     * Description:
     * Adds the specified element to the specified hlist
     * after the specified node while permitting racing traversals.
     *
     * The caller must take whatever precautions are necessary
     * (such as holding appropriate locks) to avoid racing
     * with another list-mutation primitive, such as hlist_add_head_rcu()
     * or hlist_del_rcu(), running on this same list.
     * However, it is perfectly legal to run concurrently with
     * the _rcu list-traversal primitives, such as
     * hlist_for_each_entry_rcu(), used to prevent memory-consistency
     * problems on Alpha CPUs.
     */
    static inline void hlist_add_behind_rcu(struct hlist_node *n,
                                            struct hlist_node *prev)
    {
            n->next = prev->next;
            n->pprev = &prev->next;
            rcu_assign_pointer(hlist_next_rcu(prev), n);
            if (n->next)
                    n->next->pprev = &n->next;
    }
    
    #define __hlist_for_each_rcu(pos, head)                         \
            for (pos = rcu_dereference(hlist_first_rcu(head));      \
                 pos;                                               \
                 pos = rcu_dereference(hlist_next_rcu(pos)))
    
    /**
     * hlist_for_each_entry_rcu - iterate over rcu list of given type
     * @pos:        the type * to use as a loop cursor.
     * @head:       the head for your list.
     * @member:     the name of the hlist_node within the struct.
     *
     * This list-traversal primitive may safely run concurrently with
     * the _rcu list-mutation primitives such as hlist_add_head_rcu()
     * as long as the traversal is guarded by rcu_read_lock().
     */
    #define hlist_for_each_entry_rcu(pos, head, member)                     \
            for (pos = hlist_entry_safe (rcu_dereference_raw(hlist_first_rcu(head)),\
                            typeof(*(pos)), member);                        \
                    pos;                                                    \
                    pos = hlist_entry_safe(rcu_dereference_raw(hlist_next_rcu(\
                            &(pos)->member)), typeof(*(pos)), member))
    
    /**
     * hlist_for_each_entry_rcu_notrace - iterate over rcu list of given type (for tracing)
     * @pos:        the type * to use as a loop cursor.
     * @head:       the head for your list.
     * @member:     the name of the hlist_node within the struct.
     *
     * This list-traversal primitive may safely run concurrently with
     * the _rcu list-mutation primitives such as hlist_add_head_rcu()
     * as long as the traversal is guarded by rcu_read_lock().
     *
     * This is the same as hlist_for_each_entry_rcu() except that it does
     * not do any RCU debugging or tracing.
     */
    #define hlist_for_each_entry_rcu_notrace(pos, head, member)                     \
            for (pos = hlist_entry_safe (rcu_dereference_raw_notrace(hlist_first_rcu(head)),\
                            typeof(*(pos)), member);                        \
                    pos;                                                    \
                    pos = hlist_entry_safe(rcu_dereference_raw_notrace(hlist_next_rcu(\
                            &(pos)->member)), typeof(*(pos)), member))
    
    /**
     * hlist_for_each_entry_rcu_bh - iterate over rcu list of given type
     * @pos:        the type * to use as a loop cursor.
     * @head:       the head for your list.
     * @member:     the name of the hlist_node within the struct.
     *
     * This list-traversal primitive may safely run concurrently with
     * the _rcu list-mutation primitives such as hlist_add_head_rcu()
     * as long as the traversal is guarded by rcu_read_lock().
     */
    #define hlist_for_each_entry_rcu_bh(pos, head, member)                  \
            for (pos = hlist_entry_safe(rcu_dereference_bh(hlist_first_rcu(head)),\
                            typeof(*(pos)), member);                        \
                    pos;                                                    \
                    pos = hlist_entry_safe(rcu_dereference_bh(hlist_next_rcu(\
                            &(pos)->member)), typeof(*(pos)), member))
    
    /**
     * hlist_for_each_entry_continue_rcu - iterate over a hlist continuing after current point
     * @pos:        the type * to use as a loop cursor.
     * @member:     the name of the hlist_node within the struct.
     */
    #define hlist_for_each_entry_continue_rcu(pos, member)                  \
            for (pos = hlist_entry_safe(rcu_dereference_raw(hlist_next_rcu( \
                            &(pos)->member)), typeof(*(pos)), member);      \
                 pos;                                                       \
                 pos = hlist_entry_safe(rcu_dereference_raw(hlist_next_rcu( \
                            &(pos)->member)), typeof(*(pos)), member))
    
    /**
     * hlist_for_each_entry_continue_rcu_bh - iterate over a hlist continuing after current point
     * @pos:        the type * to use as a loop cursor.
     * @member:     the name of the hlist_node within the struct.
     */
    #define hlist_for_each_entry_continue_rcu_bh(pos, member)               \
            for (pos = hlist_entry_safe(rcu_dereference_bh(hlist_next_rcu(  \
                            &(pos)->member)), typeof(*(pos)), member);      \
                 pos;                                                       \
                 pos = hlist_entry_safe(rcu_dereference_bh(hlist_next_rcu(  \
                            &(pos)->member)), typeof(*(pos)), member))
    
    /**
     * hlist_for_each_entry_from_rcu - iterate over a hlist continuing from current point
     * @pos:        the type * to use as a loop cursor.
     * @member:     the name of the hlist_node within the struct.
     */
    #define hlist_for_each_entry_from_rcu(pos, member)                      \
            for (; pos;                                                     \
                 pos = hlist_entry_safe(rcu_dereference((pos)->member.next),\
                            typeof(*(pos)), member))
    
    #endif  /* __KERNEL__ */
    #endif


```
