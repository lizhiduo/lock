# Lock
---
## 主要内容
>* 原子性
>* RMW
>* atomic
>* 语义整体
>* spinlock
>* mutex
>* lookup detector

# 预备知识
>* 处理器种类： CISC(复杂指令集) 和 RISC(精简指令集)。其中CISI处理器(X86)可以直接在内存上做加法，这在单核上可以保证原子性，多核上则不能保证（因在内存的访问不是原子的,可以细分为读-改-写三个过程，所以多核上单个指令完成的操作也可以被干扰。）；而RISC处理器(ARM)的所有操作都必须在CPU内部进行（如果要对内存里的某一个数加一，则需要先从内存load这个数到CPU寄存器里，然后加一，最后再store到内存里，这是一个典型的‘读-改-写’序列）。
>
>  > 如下图所示，分别是x86以及arm架构下的**加一**汇编代码：
>  >
>  > ```
>  > #include <stdio.h>
>  > 
>  > int main()
>  > {
>  >   4004d6:	55                   	push   %rbp
>  >   4004d7:	48 89 e5             	mov    %rsp,%rbp
>  >     int a = 20;
>  >   4004da:	c7 45 fc 14 00 00 00 	movl   $0x14,-0x4(%rbp)
>  >     a++;
>  >   4004e1:	83 45 fc 01          	addl   $0x1,-0x4(%rbp)
>  >     return 0;
>  >   4004e5:	b8 00 00 00 00       	mov    $0x0,%eax
>  > }
>  > 
>  > ```
>  >
>  > ```
>  > #include <stdio.h>
>  > 
>  > int main()
>  > {
>  >     8380:	e52db004 	push	{fp}		; (str fp, [sp, #-4]!)
>  >     8384:	e28db000 	add	fp, sp, #0
>  >     8388:	e24dd00c 	sub	sp, sp, #12
>  >     int a = 20;
>  >     838c:	e3a03014 	mov	r3, #20
>  >     8390:	e50b3008 	str	r3, [fp, #-8]
>  >     a++;
>  >     8394:	e51b3008 	ldr	r3, [fp, #-8]
>  >     8398:	e2833001 	add	r3, r3, #1
>  >     839c:	e50b3008 	str	r3, [fp, #-8]
>  >     return 0;
>  >     83a0:	e3a03000 	mov	r3, #0
>  > }
>  > 
>  > ```
>

>* RISC处理器，哪怕一个整数+1操作，也不是原子的，要经过‘读-改-写（R-M-W）’。RMW序列不是原子的，所以可能引起并发问题。例如两个线程同时对一个变量做加法，会存在预期和结果不一样，程序可能会这样执行，T1做a++,第一步先将a的值LDR到寄存器中，但是此时T2抢占了CPU然后执行完a++后，将CPU让给T1，但是此时T1并不会重新去LDR a的值，导致最终和我们想要的结果不一样。
>* 部分CPU在设计上，为了避免RMW非原子序列，会将寄存器的写操作分成 SET_REG 和 CLR_REG， 此为原子操作。另外还有一种为‘bitband’的寄存器组，它将寄存器的每一个bit都映射成一个独立的寄存器（影子寄存器），对影子寄存器写0或1，就会作用于原始寄存器相应bit。这都是从硬件设计上实现原子操作。
>* 排它性的‘ldr/sttr’指令,用来实现对内存的原子操作（或者说互斥操作），比如 ARM的LDREX/STREX指令。其实现原理就是将并行的load/store变成串行的，先写会成功，后写的若失败了，会重新load/store。

>* 实际应用： 程序毫无逻辑地随机死、通常有两个原因导致：（1）内存越界； （2）加锁不完整，存在并发的问题。

# 1. 原子性
原子操作是不可分割的，在执行完毕不会被任何其它任务或事件中断。在单处理器系统(UniProcessor)中， 能够在单条指令中完成的操作都可以认为是" 原子操作"，因为中断只能发生于指令之间。


# 3. atomic
>* atomic保证整数操作的原子性。而Linux内核将这些指令封装为一组原子锁API，用于实现整数的原子操作，比如 atomic_add()/atomic_sub()/atomic_inc()/atomic_dec() ... 

>* 原子锁是Linux里最底层的锁，它是其他各种锁（如mutex、信号量等）实现的基础。 

>* 原子锁只能对整数的操作加锁、不能对复杂数据结构的操作（如结构体）加锁，而加锁一定要锁住一个语义完整的整体。因此实际编程中用处不大。

### 注：atomic.h中有着它的定义以及操作函数。typedef struct { volatile int counter; } atomic_t; 

临界区是一些语义关联的事物构成的一个语义完整的整体。临界区需要加锁，以构成一个‘自洽、完整、统一、不自相矛盾’的整体。在进入临界区的时候，要么所以事情都没开始，要么所有的事情都已经做完了。

# 4. spinlock
>* 工作逻辑：**核内锁调度，核间自旋**。 核内锁调度：在当前核内部，直接关闭调度器，使当前线程独占当前核，其它线程无法执行；核间自旋：在多核上，当前核被锁后，其它核需要忙等，直到当前核释放锁为止。 
>
>  spinlock实现原理：在核内直接锁调度，核间才是自旋，所以自旋只有在双核或多核里面才能体现出来，在单核上就是只是锁调度器。

>* spinlock的应用场景：用于锁耗时特别短的区间；被锁区间绝对不能进行睡眠。（睡眠和唤醒需要进行两次上下文切换，这种开销可能会比我死等更大，这也是为什么需要spinlock锁住的区间需要耗时特别短，而且不能存在睡眠）。
>* 如果临界资源只被线程访问，那么线程里调用spin_lock()/spin_unlock()即可。

>* 如果临界资源既要被线程访问，又要被中断访问，因为spinlock本身不会关中断，那么线程里就需要使用spinlock的修改版本 -spin_lock_irqsave()/spin_lock_irqrestore()，而在中断里仍旧使用spin_lock()/spin_unlock() 即可。
spin_lock_irqsave() = spin_lock() + local_irq_save() 【保存“中断使能配置”（即当前系统里中断的开关状态 ） + 关中断 】 
spin_lock_irqrestore () = spin_unlock() + local_irq_restore() 【恢复“中断使能配置” + 开中断】

>### 多核 + 中断存在竞争情况
            线程与线程
            线程与中断
            中断与中断
            此核与彼核

     cpu0                   cpu1
      T1                    T3
      T2                    T4
      IRQ0                  IRQ1

CPU0上有线程T1、T2和中断IRQ0都会访问某个临界区，CPU1上的线程T3、T4和IRQ1也会访问这个临界 区。这就形成了一个复杂的竞争网络,消除所以的竞态只需要记住一条编程规则: ：**在线程里统一调用spin_lock_irqsave()/spin_lock_irqrestore()，在中断里统一调用 spin_lock()/spin_unlock()** 。

注：Linux内核2.6.32以后，就不再允许中断嵌套了，也就是说，当前中断里本来就是关闭其他中断的，因此无需在中断里调用spin_lock_irqxxx()再去关中断，直接调用spin_lock()/spin_unlock()即可。 对于单核CPU来说，如果要偷懒，直接在线程里调用spin_lock_irqsave()/spin_lock_irqrestore()即可，中断里就不用管了，因为在线程里已经将本核（也就是全部的核）的中断关掉了。 当线程和中断都要访问同一临界区时，我们知道，中断里当然要使用spin_lock()，那么线程里能不能也使用spin_lock() 呢？不能！否则当线程执行临界区时来中断会导致死锁。因此线程里必须使用spin_lock_irqsave()关中断。

      1                     2           3
    cpu0 |          |               |  外设1
    cpu1 |   <----  | 中断控制器     |  外设2
    cpu2 |          |               |  外设3

local_irq_disable/save ： 让本cpu不响应所有的中断（在位置1控制）； irq_disable： 让某号中断不能发给所有的cpu（在位置2控制）。

>>### Linux也提供了直接控制中断是否使能的API： 
>* local_irq_disable()/local_irq_enable() ： 关闭/开启当前核的中断，有副作用 （直接去改CPU里面的CPRS寄存器，禁用所有中断） 
>* local_irq_save()/local_irq_restore() ： 关闭/开启当前核的中断，无副作用 
>* irq_disable()/irq_enable()：关闭/开启所有核的中断，有副作用。
>* irq_save()/irq_restore() ：关闭/开启所有核的中断，无副作用 Linux里没有一个API，能让你关掉其他核的中断。
>* 副作用：如果调用local_irq_disable()之前，中断已经是关闭的，那么之后调用local_irq_enable()会导致中断被误开启，而不是恢复之前的关闭状态。因此，local_irq_save()/local_irq_restore()通过“保存中断状态-关中断-恢复中断状 态”来解决这个问题。

>* local_irq_disable()和local_irq_save()关掉了当前核的中断，间接导致当前核的调度器被关闭，因为调度器是依赖中断的。**local_irq_disable()和local_irq_save()一般很少使用，除非你非常清楚你正在做什么！**
* spin_lock()/spin_unlock() 和spin_lock_irqsave()/spin_lock_irqrestore()，是内核里并行编程时最常使用的API。

  |                        |          核内          |  核间  |
  | ---------------------- | :--------------------: | :----: |
  | spin_lock              |        锁住调度        |  自旋  |
  | local_irq_disable/save | 锁住中断  间接锁住调度 | 无意义 |
  | spin_lock_irqsave      |   锁住中断 锁住调度    |  自旋  |


# 6. mutex

mutex的用法：
>   mutex_lock(&lock);
    临界区...
    可以睡眠
    mutex_unlock(&lock);

* mutex工作机制：如果线程T1拿到了mutex，于是线程T2拿不到，它就会睡眠直到T1释放mutex，则T2被唤醒 -这里涉及两次contex switch。 
* 如果临界区里需要睡眠，或者临界区很长，则应使用mutex。
* 加锁原则：同一把锁、语义整体、粒度最小；同一把锁和语义整体可以保证代码运行的正确性，粒度可以决定你的代码性能。软件设计时需要即要保证粒度最小也要语义整体完整。


# 7. lockup detector
>* lockup detector的实现源码：kernel/watchdog.c。注意这个不是硬件看门狗（drivers/watchdog/watchdog.c）， 而是用来监测系统里有没有锁住调度器和中断的。
>>* lockup detector的工作原理：kernel/watchdog.c里，为每个CPU核都启动了一个高优先级的RT线程，这个线程会周 期性地对某个计数值执行+1操作。而另一个定时器中断会周期性地侦测这个计数值是否在自增。 
* NMI中断 + 定时器中断 + 高优先级RT线程
* 用定时器中断，检测高优先级线程有无机会执行->soft lockup(只锁调度器)
* 用NMI，检测定时器中断有无机会执行->hard lockup（中断和调度器都被锁住）
>>>* 如果发生了soft lockup，则定时器中断会监测到计数值停滞，于是打印出栈的backtrace（通过CPU的SP指针 来回溯），将事故现场报告给用户。 
* 如果发生了hard lockup，比较难debug，只有CPU支持NMI（不可屏蔽的中断，一般由性能检测单元PMU来 实现），才能进行debug。X86是支持NMI的。但是ARM不支持NMI，所以Linux官方内核是不支持ARM的 hard lockup detector的。有两种补丁可以实现ARM上的hard lockup detector： （1）使用FIQ来模拟NMI：由Linaro开发。注意，在Linux内核里，FIQ只作为特殊的debug用途，常规代 码里基本不会用到FIQ。 （2）使用另一个CPU核来检测：缺点是这个CPU无法访问hang死的CPU的栈（因为一个CPU无法访问别的 CPU的SP寄存器），因此无法进行栈回溯。


* 内核hang死，通常是由关中断或者spinlock引起的。




