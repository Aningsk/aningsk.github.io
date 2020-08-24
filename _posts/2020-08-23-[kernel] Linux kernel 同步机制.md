# 概述
前几天在内核工匠的公众号上看到了两篇关于Linux kernel的同步机制的介绍，所以就想按照这两篇文章的内容整理一下。   
这两篇文章分别是：  
[Linux kernel 同步机制（上篇）][1]  
[Linux kernel 同步机制（下篇）][2]   

感谢OPPO内核团队的工作。  

# 1. 同步机制之间的比较    
Ulysses还不支持表格啊……那只能我一段一段写了……  

## 1.1 原子操作
**等待机制**：无，ldrex与strex实现内存独立访问。  
**优缺点**：性能相当高，但使用场景受限。  
**适用于**：资源计数。  

## 1.2 自旋锁
**等待机制**：忙等待，唯一持有。   
**优缺点**：多处理器下性能优异，临界区时间长会浪费。  
**适用于**：中断上下文。  

## 1.3 信号量
**等待机制**：睡眠等待（阻塞），多数持有。  
**优缺点**：相对灵活，适用于复杂情况，但是时耗长。  
**适用于**：情况复杂且耗时长的场景，比如内核与用户空间的交互。  

## 1.4 互斥锁
**等待机制**：睡眠等待（阻塞），优先自旋等待，唯一持有。  
**优缺点**：相比于信号量，高效，适用于复杂场景，但存在若干限制条件。  
**适用于**：满足使用条件下，mutex优先与信号量。  

## 1.5 读写信号量
**等待机制**：睡眠等待（阻塞），优先自旋等待，读者多数持有，写者唯一持有。  
**优缺点**：读写优化的信号量，对于读多写少的情形，性能大大提升。  
**适用于**：类似信号量，更适用于读多写少的场景，例如内存管理系统。  

## 1.6 读写锁
**等待机制**：忙等待，读者多数持有，写者唯一持有。  
**优缺点**：读写优化的自旋锁，读者阻塞写者，对于读多写少的情形性能大大提高；但写者可能受阻塞降低性能。  
**适用于**：类似自旋锁，更适用于读多写少的场景。  

## 1.7 顺序锁
**等待机制**：忙等待。  
**优缺点**：写优先于读的锁，对于读写同时比较少的情况，拥有高性能表现。  
**适用于**：不允许读者阻塞写者，写者大大少于读者的情形，例如时钟系统。  

## 1.8 RCU
**等待机制**：并不基于锁，而是一种系统强耦合同步机制。  
**优缺点**：在绝大部分为读而只有极少部分为写的情况下，是非常高效的；但延后释放内存会造成内存开销，写者阻塞比较严重。  
**适用于**：读多写少的情况、对内存消耗不敏感的情况；满足使用RCU的情况下，优先使用读写锁。对于动态分配数据结构这类引用计数的机制，也有高性能表现。  

# 2. 原子操作

## 2.1 介绍
操作的数据类型为：  
```c
typedef struct {
	in counter;
} atomic_t;
```
原子性依赖于汇编语言**ldrex**（标记独占）和**strex**（清除标记）的实现。  

## 2.2 API
```c
int atomic_read(atomic_t *v); //读操作
void atomic_set(atomic_t *v, int i); //设置变量
void atomic_add(int i, atomic_t *v); //增加i
void atomic_sub(int i, atomic_t *v); //减少i
void atomic_inc(atomic_t *v); //增加一
void atomic_dec(atomic_t *v); //减少一
bool atomic_inc_and_test(atomic_t *v); //加一是否为零
bool atomic_dec_and_test(atomic_t *v); //减一是否为零
bool atomic_add_negative(int i, atomic_t *v); //加i是否为负
int atomic_add_return(int i, atomic_t *v); //加i返回结果
int atomic_sub_return(int i, atomic_t *v); //减i返回结果
int atomic_inc_return(atomic_t *v); //加一返回结果
int atomic_dec_return(atomic_t *v); //减一返回结果
```

## 2.3 使用
一般用在引用计数。  

# 3. 自旋锁 spinlock

## 3.1 介绍
操作的数据类型为：  
```c
typedef struct {
	struct raw_spinlock rlock;
} spinlock_t;
//套娃结构体我就不写了。
```
自旋锁被别的执行者持有时，其他调用者就是原地等待并检查锁是否已被释放。  
持有自旋锁期间将不可被抢占。  

## 3.2 API
```c
spin_lock_init(); //动态初始化spin_lock
DEFINE_SPINLOCK(); //声明并初始化spin_lock

/* 普通版本 */
void spin_lock(spinlock_t *lock); //持有一个spinlock
void spin_unlock(spinlock_t *lock); //释放一个spinlock
int spin_trylock(spinlock_t *lock); //尝试获得锁，不能获得立即返回，不自旋
int spin_is_locked(spinlock_t *lock); //判断一个锁是否被持有
int spin_can_lock(spinlock_t *lock); //判断一个锁是否可以被持有

/* 保存标志关中断版本 */
void spin_lock_irqsave(spinlock_t *lock, unsigned long flags); //获得锁时，保存标志寄存器到flags变量中，关中断；释放锁反之
void spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags);

/* 关中断版本 */
void spin_lock_irq(spinlock_t *lock); //获得锁时，关中断；释放锁反之
void spin_unlock_irq(spinlock_t *lock);

/* 关软中断版本 */
void spin_lock_bh(spinlock_t *lock); //获得锁时，关软中断；释放反之
void spin_unlock_bh(spinlock_t *lock);

void spin_trylock_irqsave(spinlock_t *lock, unsigned long flags); //能够获得锁时，保存标志寄存器到flags，关中断；否则立即返回
void spin_trylock_irq(spinlock_t *lock); //能够获得锁时，关中断；否则立即返回
void spin_trylock_bh(spinlock_t *lock); //能够获得锁时，关软中断；否则立即返回

```

## 3.3 使用
性能上：  
```c
spin_lock > spin_lock_bh > spin_lock_irq > spin_lock_irqsave 
```
安全上：  
```c
spin_lock_irqsave > spin_lock_irq > spin_lock_bh > spin_lock 
```

spin\_lock用于：  
1. 阻止不同CPU上的执行单元对共享资源的同时访问。  
2. 阻止不同进程上下文相互抢占导致对共享资源的非同步访问。  
关闭中断/软中断的版本（spin\_lock\_irq, spin\_lock\_bh)用于：  
1. 阻止同一CPU上的中断/软中断对共享资源的非同步访问。  

注：tasklet和timer是软中断实现的。  
1. 如果共享资源会被进程、软中断访问，则需要使用**关闭软中断版本**。  
2. 如果共享资源只被多个软中断访问，则使用**普通版本**即可。（因为一个CPU上在运行一个软中断，其他软中断不可能在此CPU运行，我们只需防备其他CPU）  
3. 如果共享资源会被进程、软中断、硬中断访问，则在进程上下文和软中断上下文中，使用**关中断版本**。  
4. 使用**关中断版本**的位置，完全可以使用**保存标志关中断版本**。如果确信在访问共享资源前，中断是开启的，则使用关中断版本好点儿，因为更快。  

# 4. 信号量 semaphore

## 4.1 介绍
操作的数据类型为：  
```c
struct semaphore {
	raw_spinlock_t lock; //用自旋锁同步
	unsigned int count; //资源计数
	struct list_head wait_list; //等待队列
};
```
信号量基于spinlock实现。但获得不到信号量并不是忙等，而是阻塞。  
信号量在创建时设置一个初始值count。一个任务想要访问共享资源，必须先获得信号量（即，down，count - 1）；当count为负数，表示无法获得信号量，该任务将阻塞休眠并挂在此信号量的等待队列中。任务访问完后，必须释放信号量（即，up，count + 1）；此时若count为非正数，表明有任务在等待此信号量，将唤醒所有等待此信号量的任务。  

## 4.2 API
```c
DEFINE_SEMAPHORE(name); //声明信号量，并初始化为1
void sema_init(struct semaphore *sem, in val); //声明信号量，并初始化为val
void down(struct semaphore *sem); //获得信号量，且task不可被中断，除非是致命信号
void down_interruptible(struct semaphore *sem); //获得信号量，task可以被中断
int down_trylock(struct semaphore *sem); //在能够获得信号量时，count--；否则立即返回，不会挂起到等待队列
int down_killable(struct semaphore *sem); //获得信号量，且task可以被kill
void up(struct semaphore *sem); //释放信号量
```

## 4.3 使用
好像没啥要注意的，别忘了会休眠就行。  

# 5. 互斥锁 mutex

## 5.1 介绍
操作的结构体：  
```c
/* include/linux/mutex.h */
struct mutex {
	atomic_t count;
	spinlock_t wait_lock;
	struct list_head wait_list;
#if defined(CONFIG_DEBUG_MUTEX) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
	struct task_struct *owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
	struct optimistic_spin_queue osq; /* Spinner MCS lock */
#endif
};
```
同时只能有一个任务可以访问该锁保护的共享资源，而且，获得锁和释放锁的调用方必须一致。因此，除了锁本身进行同步，对锁的持有者也必须进行同步。  
当无法获得锁时，task会进入等待队列，休眠等待，直到可以获得锁。  
注：  
如今，互斥锁已经**可以**自旋等待：通过MCS锁机制实现（配置CONFIG\_MUTEX\_SPIN\_ON\_OWNER）。自旋等待机制的核心原理：OSQ算法。当发现持有者正在临界区执行，并且没有其他高优先级的进程要被调度时，当前期望获得锁的进程认为：持有者会很快离开临界区并释放锁。这时，当前进程选择进行自旋等待。因为短时间的自旋等待比休眠-唤醒开销小。  
MCS比较复杂。可以保证：  
1. 同一时间只有一个进程自旋等待锁的释放。  
2. 不会存在多个CPU争用锁。    
	   
如此改进后，互斥锁的性能有了很大提高；如果信号量和互斥锁都可以，则选用互斥锁。    
   
## 5.2 API
```c
DEFINE_MUTEX(name); //静态声明互斥锁并初始化状态
void mutex_init(struct mutex *lock); //动态声明互斥锁并初始化状态
void mutex_destory(struct mutex *lock); //销毁该互斥锁
bool mutex_is_locked(struct mutex *lock); //判断互斥锁是否被持有
void mutex_lock(struct mutex *lock); //获得锁，且task不可被中断，除非致命信号
void mutex_unlock(struct mutex *lock); //释放锁
int mutex_trylock(struct mutex *lock); //尝试获得锁，不能获得立即返回
int mutex_lock_interruptible(struct mutex *lock); //获得锁，task可以被中断
int mutex_lock_killable(struct mutex *lock); //获得锁，task可以被kill
```

## 5.3 使用
1. 同一时刻只有一条内核路径可以持有锁  
2. 只有锁的持有者才可以释放锁  
3. 不允许递归加锁/解锁  
4. 进程持有锁后不应退出  
5. 可能导致休眠阻塞，不可用于中断处理（包括底半部）   
	  
# 6. 读写信号量 rw\_semaphore

## 6.1 介绍
操作的结构体：  
```c
/* include/linux/rwsem.h */
struct rw_semaphore {
	long count;
	struct list_head wait_list;
	raw_spinlock_t wait_lock;
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
	struct optimistic_spin_queue osq; /* Spinner MCS lock */
	struct task_struct *owner;
#endif
};
```
类似信号量，但是区分了读者与写者。  
写者时排他的、独占的；读者不排他，数量不受限制。  
如果读写信号量没有被“写者”持有或等待，那么，“读者”就可以获得读写信号量；否则，“读者”必须等待直到“写者”释放读写信号量。  
如果读写信号量没有被“读者”持有，也没有被“写者”持有或等待，那么，“写者”可以获得读写信号量；否则，“写者”必须等待读写信号量被释放，即“没有其他访问者”。  
不能同时既又写。  

注：  
类似互斥锁，读写信号量除了可以休眠阻塞，也可能自旋等待。  

## 6.2 API
```c
DECLARE_RWSEM(name); //静态声明并初始化
void init_rwsem(struct rw_semaphore *sem); //动态声明并初始化
void down_read(struct rw_semaphore *sem); //读者获得读写信号量
int down_read_trylock(struct rw_semphore *sem); //读者尝试获得读写信号量，不能获得立即返回
void down_wirte(struct rw_semaphore *sem); //写者获得读写信号量
int down_write_trylock(struct rw_semaphore *sem); //写者尝试获得读写信号量，不能获得立即返回
void up_read(struct rw_semaphore *sem); //读者释放读写信号量
void up_write(struct rw_semaphore *sem); //写者释放读写信号量
void downgrade_write(struct rw_semaphore *sem); //写者身份降为读者
```

## 6.3 使用
好像没有什么特别的。  

# 7. 读写锁 rw\_lock

## 7.1 介绍
操作的结构体：  
```c
typedef struct {
	arch_rwlock_t raw_lock;
} rwlock_t;
/* 套娃的结构体不写了 */
```
一种特殊的自旋锁，把共享资源的访问者分为“读者”和“写者”。  
多处理器中，允许同时有多个“读者”访问共享资源，数量最大为实际的CPU数目。  “写者”是排他的。不能同时既读又写。  

## 7.2 API
```c
rwlock_init(lock); //静态声明并初始化

read_lock(lock); //获得读者锁
read_unlock(lock); //释放读者锁
read_trylock(lock); //尝试获得读者锁，不能获得立即返回

write_lock(lock); //获得写者锁
write_unlock(lock); //释放写者锁
write_trylock(lock); //尝试获得写者锁，不能获得立即返回

read_lock_irqsave(lock， flags); //获得读者锁，将标志寄存器内容写入flags；解锁反之
read_unlock_irqrestore(lock, flags);
read_lock_irq(lock);
read_unlock_irq(lock);
read_lock_bh(lock);
read_unlock_bh(lock);

write_lock_irqsave(lock, flags);
write_unlock_irqrestore(lock, flags);
write_lock_irq(lock);
write_unlock_irq(lock);
write_lock_bh(lock);
write_unlock_bh(lock);

write_trylock_irqsave(lock, flags); //将标志寄存器内容存入flags，并尝试获得写者锁，成功获得则关闭中断；失败则恢复标志寄存器，并立即返回

```

## 7.3 使用
好像也没有什么要说的。  

# 8. 顺序锁 seqlock

## 8.1 介绍
操作的结构体：  
```c
typedef struct {
	struct seqcount seqcount;
	spinlock_t lock;
} seqlock_t;
```
依然是基于自旋锁。  
顺序锁是对读写锁的一种优化：读者绝对不会被阻塞。  
“读者”在“写者”持有锁的情况下依然可以读。  
“写者”在“读者”持有锁的情况下依然可以写。  
但是“写者”之间依然是互斥的。写操作的优先级大于读操作。  

注：  
顺序锁的限制——被保护的共享资源不可含有指针。因为写者可能让指针失效，但读者可能正要访问这个指针，这将导致Oops。  
如果在“读者”读操作期间，有“写者”进行了写操作；那么，“读者”必须重新读取数据，以确保数据完整。  
顺序锁适用于读多写少的情况。  

## 8.2 API
```c
seqlock_init(x); //声明并初始化顺序锁
DEFINE_SEQLOCK(x); 

void write_seqlock(seqlock_t *sl); //写者获得顺序锁
void write_sequnlock(seqlock_t *sl); //写者释放顺序锁

unsigned read_seqbegin(const seqlock_t *sl); //读取开始前,获得seqlock的写操作次数seqcount
unsigned read_seqretry(const seqlock_t *sl, unsigned start); //读取结束后，判断当前的seqcount是否与start一致

/* 通常读者不需要获得顺序锁，但内核也提供了，实际实习就是spin_lock/unlock */
void read_seqlock_excl(seqlock_t *sl);
void read_sequnlock_excl(seqlock_t *sl);

/* 其他irqsave, irq, bh版本与其他锁类似 */
```

## 8.3 使用
前面已经说了，共享资源不能有指针。  

# 9. RCU (Read, Copy, Update)

## 9.1 介绍
与前面的均不同的是，RCU并不基于锁机制，RCU字段是耦合在“进程描述符”和“CPU变量”中的，是一种与系统强耦合的同步机制。  
RCU既允许多个“读者”同时访问共享资源，也允许多个“读者”和多个“写者”同时访问共享资源。这里只是说RCU对“写者”也不做限制，但通常情况，“写者”之间会使用其他的同步机制。  
“读者”没有任何同步开销，“写者”的同步开销取决于使用的“写者”间同步机制。  

## 9.2 过程
对于被RCU保护的共享资源，“读者”不需要获得任何锁就可以访问。但是，“写者”在访问共享资源时，先拷贝一个副本，然后“写者”对副本进行修改，在“适当的时机”调用一个回调，将原共享资源更新。  
**适当的时机**：所有引用该共享资源的CPU都退出了对此数据的操作。  
**原共享资源更新**：有专门的垃圾收集器探测“读者”的信号。一旦所有“读者”都已发送信号，告知不再使用共享资源，即达到“适当的时机”，垃圾收集器就调用回调完成最后的数据释放或修改。  

## 9.3 API
```c
void rcu_read_lock(void); //读者在读取RCU保护的共享资源时，使用此函数标记它进入临界区
void rcu_read_unlock(void); //与上函数配对使用，表示读者离开临界区

void synchronize_rcu(void); //同步RCU，即：所有的读者已完成读端临界区，写者才可以继续下一步操作。由于该函数会阻塞写者，只能在进程上下文中使用

void call_rcu(struct rcu_head *head, rcu_callback_t func); //将回调函数注册到RCU回调函数链，然后立即返回

rcu_assign_pointer(p, v); //用于RCU指针赋值
rcu_dereference(p); //用于RCU指针取值

void list_add_rcu(struct list_head *new, struct list_head *head); //向RCU注册一个链表结构
void list_del_rcu(struct list_head *entry); //从RCU移除一个链表结构
```

## 9.4 使用
RCU限制条件：  
1. RCU只保护动态分配，并通过指针引用的数据结构  
2. 在被RCU保护的临界区中，任何内核路径都不能休眠（在经典实现中）  

然后，我并不知道怎么用……    

```c
/* driver/input/input.c为例 */

/* 1. 将共享资源加入RCU保护 */
rcu_assign_pointer(dev->grab, handle);

/* 2. 标记进入临界区 */
rcu_read_lock();

/* 3. 读出临界区，进行操作 */
handle = rcu_dereference(dev->grab);

/* 4. 退出临界区 */
rcu_read_unlock();

/* 5. 释放，然后通知所有写者 */
rcu_assign_pointer(dev->grab, NULL);
synchronize_rcu();

```


[1]:	https://mp.weixin.qq.com/s/mosYi_W-Rp1-HgdtxUqSEg "Linux kernel 同步机制（上篇）"
[2]:	https://mp.weixin.qq.com/s/-GnR-nryH_7xkNVhJ8AMNw "Linux kernel 同步机制（下篇）"