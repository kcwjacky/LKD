# Kernel Synchronization Methods

## Atomic Operations

The kernel provides two sets of interfaces for atomic operations — one that operates on integers and another that operates on individual bits.
Kernel 對不可分割操作提供了兩個介面，一組是針對整數操作，一組是針對位元操作。

### Atomic Integer Operations
The atomic integer methods operate on a special data type, `atomic_t`. 對於不可分割的整數操作，我們會需要一個特殊的資料型態 `atomic_t`，而不是 C 語言中的 int。首先是避免與非不可分割的函式所誤用，另外也會確保編譯器不會對其做優化。

```cpp
typedef struct {
    volatile int counter; 
} atomic_t;

atomic_t v;                     /* define v */
atomic_t u = ATOMIC_INIT(0);    /* define u and initialize it to zero */

atomic_set(&v, 4);          /* v = 4 (atomically) */
atomic_add(2, &v);          /* v = v + 2 = 6 (atomically) */
atomic_inc(&v);             /* v = v + 1 = 7 (atomically) */
atmoic_dec(&v);             /* v = v - 1 = 6 (atomically) */
```

The atomic operations are typically implemented as inline functions with inline assembly. In the case where a specific function is inherently atomic, the given function is usually just a macro. 不可分割操作通常被實作成 inline function，而且是通過 inline assembly 來實作的。

In your code, it is usually preferred to choose atomic operations over more complicated locking mechanisms. On most architectures, one or two atomic operations incur less overhead and less cache-line thrashing than a more complicated synchronization method. 不可分割操作比上鎖機制簡單且帶來的開銷比較小，應該優先選擇不可分割操作，不要殺雞焉用牛刀。

### Atomic Bitwise Operations

**What might be surprising is that the bitwise functions operate on generic memory addresses**. 不可分割的逐位元操作的對象是記憶體位址。所以對象不必使用特殊的資料結構 atomic_t。

```cpp
set_bit(0, &word);      /* bit zero is now set (atomically) */
set_bit(1, &word);      /* bit one is now set (atomically) */
printk(“%ul\n”, word);  /* will print “3” */
clear_bit(1, &word);    /* bit one is now unset (atomically) */
change_bit(0, &word);   /* bit zero is flipped; now it is unset (atomically) */
```

Kernel 也提供了一系列非不可分割的逐位元操作，其函式名稱與不可分割幾乎一樣，只是在前面會有兩個底線符號。所以當今天已經有鎖保護時，就可以使用非不可分割版本的操作。

## Spin Locks

The most common lock in the Linux kernel is the spin lock. A spin lock is a lock that can be held by at most one thread of execution. If a thread of execution attempts to acquire a spin lock while it is already held, which is called contended, the thread busy loops—spins—waiting for the lock to become available. If the lock is not contended, the thread can immediately acquire the lock and continue. The spinning prevents more than one thread of execution from entering the critical region at any one time.The same lock can be used in multiple locations, so all access to a given data structure, for example, can be protected and synchronized. 重點就是 spin lock 是 busy waiting

It is not wise to hold a spin lock for a long time.This is the nature of the spin lock: a lightweight single-holder lock that should be held for short durations. An al- ternative behavior when the lock is contended is to put the current thread to sleep and wake it up when it becomes available.Then the processor can go off and execute other code.This incurs a bit of overhead—most notably the two context switches required to switch out of and back into the blocking thread, which is certainly a lot more code than the handful of lines used to implement a spin lock.Therefore, it is wise to hold spin locks for less than the duration of two context switches. 由於 spin lock 是 busy waiting 所以要用在已知不會 lock 太久的狀況之下才用 spin lock，不然應該選擇會休眠的 semaphore 才對，但因為後者會需要兩次的 context switch，所以對於不同的情形，自己要選擇。 

```cpp
DEFINE_SPINLOCK(mr_lock);
spin_lock(&mr_lock);
/* critical region ... */ 
spin_unlock(&mr_lock);
```

On uniprocessor machines, the locks compile away and do not exist; they simply act as markers to disable and enable kernel preemption. If kernel preempt is turned off, the locks compile away entirely. 在單處理器機器上，編譯的時候不會加入自旋鎖，而是會被當作禁止核心搶佔機制的標記。

Spin locks can be used in interrupt handlers, whereas semaphores cannot be used because they sleep. If a lock is used in an interrupt handler, you must also disable local interrupts (interrupt requests on the current processor) before obtaining the lock. Otherwise, it is possible for an interrupt handler to interrupt kernel code while the lock is held and attempt to reacquire the lock.The interrupt handler spins, waiting for the lock to become available. The lock holder, however, does not run until the interrupt handler completes. This is an example of the double-acquire deadlock discussed in the previous chapter. 
在中斷處理程序當中，是可以使用 spin lock 而不能使用 semaphore 的，因為 ISR 非在行程環境中不能休眠。並且在使用 spin lock 上鎖之前，還應該要禁止本地中斷，以免重複上鎖導致死鎖發生。禁止中斷與上鎖的搭配可以用下面的代（巨集）來實現。

```cpp
DEFINE_SPINLOCK(mr_lock); 
unsigned long flags;
spin_lock_irqsave(&mr_lock, flags);         // 保存中斷當前狀態，禁止本地中斷，取得鎖
/* critical region ... */ 
spin_unlock_irqrestore(&mr_lock, flags);    // 解鎖，讓中斷恢復成上鎖之前的狀態
```

## Reader-Writer Spin Locks

就是有 `read_lock` `read_unlock` `write_lock` `write_unlock` 用來支援讀著寫者的操作

---

## Semaphores 
Semaphores in Linux are sleeping locks. When a task attempts to acquire a semaphore that is unavailable, the semaphore places the task onto a **wait queue** and puts the task to sleep. The processor is then free to execute other code. When the semaphore becomes available, one of the tasks on the wait queue is awakened so that it can then acquire the semaphore. This provides better processor utilization than spin locks because there is no time spent busy looping, but semaphores have much greater overhead than spin locks. Life is always a trade-off. 

* Because the contending tasks sleep while waiting for the lock to become available, semaphores are well suited to locks that are held for a long time. 適合用在鎖會被長期持有的情況下。
* Conversely, semaphores are not optimal for locks that are held for short periods be- cause the overhead of sleeping, maintaining the wait queue, and waking back up can easily outweigh the total lock hold time. 不適合短期持有的使用情形，因為休眠、維護等待佇列和喚醒的開銷。
* Because a thread of execution sleeps on lock contention, semaphores must be ob- tained only in process context because interrupt context is not schedulable. 因為會休眠，所以只能用在行程環境下，**因為中斷環境是不可排班的**。
* You can (although you might not want to) sleep while holding a semaphore be- cause you will not deadlock when another process acquires the same semaphore. (It will just go to sleep and eventually let you continue.) 不會變成死鎖
* You cannot hold a spin lock while you acquire a semaphore, because you might have to sleep while waiting for the semaphore, and you cannot sleep while holding a spin lock. 不能在持有 spin lock 下去取得號誌，因為號誌可能會休眠，而 spin lock 不能休眠。

### Counting and Binary Semaphores

分兩種，不贅述

### Creating and Initializing Semaphores

```cpp
struct semaphore name; 
sema_init(&name, count);

static DECLARE_MUTEX(name);     
// a shortcut to create a binary semaphore

sema_init(sem, count);
init_MUTEX(sem);
// 比較常見的用法，把 semaphore 動態建立成比較大結構的一部分。給定的 sem 是一個指標，使用 sema_init () 來初始化

```

### Using Semaphores

```cpp
/* attempt to acquire the semaphore ... */ 
if (down_interruptible(&mr_sem)) {
    /* signal received, semaphore not acquired ... */
}

/* critical region ... */

/* release the given semaphore */ 
up(&mr_sem);
```

### Reader-Writer Semaphores

略

## Mutexes

Mutex 會被表示成 `struct mutex`。它的行為類似於計數等於一的 semaphore。但是他的介面比較簡單，效率較高，同時他的使用也會有比較多的限制。 

```cpp
DEFINE_MUTEX(name);
mutex_init(&mutex);

mutex_lock(&mutex);
/* critical region ... */ 
mutex_unlock(&mutex);
```

相比於 semaphore 有以下更嚴格的限制：
* Only one task can hold the mutex at a time.That is, the usage count on a mutex is always one. （計數為一）
* Whoever locked a mutex must unlock it.That is, you cannot lock a mutex in one context and then unlock it in another.This means that the mutex isn’t suitable for more complicated synchronizations between kernel and user-space. Most use cases, however, cleanly lock and unlock from the same context.（在哪個環境上鎖，就需要在同一個環境中解鎖。但這符合大部分的使用情景，不過這意味著 mutex 不適合核心與用戶空間之間較複雜的同步）
* Recursive locks and unlocks are not allowed.That is, you cannot recursively acquire the same mutex, and you cannot unlock an unlocked mutex.（不能遞迴地上鎖和解鎖）
* A process cannot exit while holding a mutex.（一個持有互斥鎖的行程無法退出）
* A mutex cannot be acquired by an interrupt handler or bottom half, even with mutex_trylock().（中斷與下半部無法取得 mutex）
* A mutex can be managed only via the official API: It must be initialized via the methods described in this section and cannot be copied, hand initialized, or reinitialized.（用上述的 API 使用互斥鎖，互斥鎖不能夠複製）

### Semaphores Versus Mutexes

盡量使用 Mutexes

### Spin Locks Versus Mutexes

只有 spin locks 可以使用在中斷環境，會休眠藥選擇 mutex