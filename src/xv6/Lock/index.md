# 锁与并发

## 为什么需要锁

在操作系统内核中有大量的数据结构都是可以被并发访问的。并发地去访问同一片数据，可能会导致读写错误。
要让这些并发不安全的操作被序列化，可以使用锁这种同步原语。

```c
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

在运行过程中，内核的内存分配器维护一个全局的链表 `freelist`。
如果调用函数 `kalloc()` 成功，会从代表可用内存的链表中弹出一页内存。
如果调用函数 `kfree()` 成功，会向代表可用内存的链表推入一页内存。

在 xv6 中所使用的链表结构是线程不安全的，如果同时进行插入或弹出操作，可能会导致操作乱序。

## 在哪里使用了锁

| 用到锁的文件     | 为什么需要锁                                 |
| ---------------- | -------------------------------------------- |
| bcache.lock      | 保护块缓冲区缓存条目的分配                   |
| cons.lock        | 序列化了对控制台硬件的访问，避免了混合的输出 |
| ftable.lock      | 序列化文件表中文件结构体的分配               |
| itable.lock      | 保护内存中 `inode` 的分配                    |
| vdisk_lock       | 序列化对磁盘硬件和 DMA 描述符队列的访问      |
| kmem.lock        | 对内存的分配进行序列化                       |
| log.lock         | 序列化对事务日志的操作                       |
| pipe’s pi->lock  | 序列化对每个管道的操作                       |
| pid_lock         | 序列化了 `next_pid` 的增量                   |
| proc’s p->lock   | 序列化对进程状态的改变                       |
| wait_lock        | 避免 `wait` 失去唤醒                         |
| tickslock        | 序列化对 `ticks` 计数器的操作                |
| inode’s ip->lock | 序列化对每个 `inode` 和其内容的操作          |
| buf’s b->lock    | 序列化对每个块缓冲区进行的操作               |

## 锁的实现

Xv6 中实现并使用了两种锁。

### 自旋锁

在 [`kernel/spinlock.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/spinlock.h) 中定义了自旋锁的结构体。

```c
// Mutual exclusion lock.
struct spinlock {
  uint locked;       // Is the lock held?

  // For debugging:
  char *name;        // Name of lock.
  struct cpu *cpu;   // The cpu holding the lock.
};
```

自旋锁是对多处理器互斥的。也就是说，两个 CPU 不能同时对 `lk->locked` 进行修改，要达到这个目的，需要一些**原子化**的操作。
我们所学习的 xv6 操作系统目标的 RISC-V 架构指令集提供了满足这个需求的原子化原语 `amoswap r, a`。
函数 `__sync_lock_test_and_set` 便是基于这一指令定义的。
在此基础上，一个上锁函数的定义思路就出现了：将封装好的设定 `lk->locked` 的操作放在条件循环中，当条件不满足时进行循环地自旋，
以达到阻塞下一步操作的目的。

```c
// Acquire the lock.
// Loops (spins) until the lock is acquired.
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic("acquire");

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section's memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
```

注意到，上面锁的上锁操作定义中，除了调试信息之外还有一些额外的代码。

  - 条件检查 `if(holding(lk))` 获取锁的操作不应该在当前 CPU 已经持有锁的基础上发生。
    Xv6 的锁实现是不可重入的。可重入锁又被称为递归锁，意思是如果当前的线程已经持有了锁，
    这个线程还尝试再次上锁，内核是允许这种情况发生的，而不是像现在的实现一样 panic 掉。
  - 调用 `push_off` 函数（与 `pop_off` 配套）会追踪在当前 CPU 上嵌套调用多个锁的层数。当
    调用多层锁的层级被减低到 0 时，系统将会重新启动中断功能，反之启用。
    这是因为在 xv6 中，锁保护的对象，既可以被线程使用，也可以被中断处理所使用。
    在使用受保护的数据之前应该对其上锁，这是如果中断启用，可能会导致死锁。
  - 调用 `__sync_synchronize` 函数：现代的编译器会对指令进行重排和优化，也就是说，
    程序执行指令的方式和在文本中定义的顺序可能是不同或并行的；如果发生了寄存器缓存的情况，
    程序甚至可能不执行生成需要的 `load` 和 `store` 指令。如果在当前的操作中，指令有序和不被省略
    对程序的行为是关键的，可以使用函数 `__sync_synchronize` 来告诉编译器不要做这些优化。
    对需要保护的对象上锁是这一场景的典型例子。

考虑到上面的要点和实现方式，实现解锁的过程也是类似的。

```c
// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```

在 [`kernel/spinlock.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/spinlock.c) 中定义了自旋锁的初始化与基本操作，和上面提及的辅助函数。

### 睡眠锁

自旋操作会让 CPU 处于忙于等待的状态，适合用于进行一些需要时间短，顺序关键的操作。
如果符合以上要求的特性，这样的锁操作可以做到低延迟。此外，这里实现的自旋锁是和硬件相关的，最基础的高层同步原语。
如果有的上锁操作需要执行相当一段时间，例如文件操作时，便需要一类锁能够不消耗太多系统资源，
不让 CPU 忙于等待，让调度器知道当前任务在至少多少时间内无法推进。于是便引入了睡眠锁。

在 [`kernel/sleeplock.h`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sleeplock.h) 中定义了睡眠锁的数据结构。

```c
// Long-term locks for processes
struct sleeplock {
  uint locked;       // Is the lock held?
  struct spinlock lk; // spinlock protecting this sleep lock

  // For debugging:
  char *name;        // Name of lock.
  int pid;           // Process holding lock
};
```

在睡眠锁的结构体中包含了自旋锁的字段，这是为了让对睡眠锁的操作序列化。

在 [`kernel/sleeplock.c`](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sleeplock.c) 中定义的对睡眠锁的基本操作中，除了初始化函数外，
都使用内部的自旋锁来保持序列化性质。

```c

void
initsleeplock(struct sleeplock *lk, char *name)
{
  initlock(&lk->lk, "sleep lock");
  lk->name = name;
  lk->locked = 0;
  lk->pid = 0;
}

void
acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  while (lk->locked) {
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}

void
releasesleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  lk->locked = 0;
  lk->pid = 0;
  wakeup(lk);
  release(&lk->lk);
}

int
holdingsleep(struct sleeplock *lk)
{
  int r;

  acquire(&lk->lk);
  r = lk->locked && (lk->pid == myproc()->pid);
  release(&lk->lk);
  return r;
}
```

睡眠锁所能进行的基本操作与自旋锁是类似的。
在定义睡眠锁时，使用了 `sleep` 与 `wakeup` 这一对操作。
操作 `sleep` 告诉内核，线程将会停止运行，直到等待到特定事件发生；
操作 `wakeup` 告诉内核，事件发生，线程将会继续推进。
