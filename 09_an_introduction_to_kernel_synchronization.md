# An Introduction to Kernel Synchronization

## Critical Regions and Race Conditions

Code paths that access and manipulate shared data are called critical regions (also called critical sections). It is usually unsafe for multiple threads of execution to access the same resource simultaneously. It is a bug if it is possible for two threads of execution to be simultaneously executing within the same critical region.When this does occur, we call it a race condition.

## Locking

What is needed is a way of making sure that only one thread manipulates the data structure at a time—a mechanism for preventing access to a resource while another thread of execution is in the marked region. 

Using locks can prevent concurrency and protect the queue from race conditions. Linux alone implements a handful of different locking mechanisms. The most significant difference between the various mechanisms is the behavior when the lock is unavailable because another thread already holds it— some lock variants busy wait, whereas other locks put the current task to sleep until the lock becomes available. lock 的種類主要可以分成 busy waiting 和去休眠的兩種鎖。

Fortunately, locks are implemented using atomic operations that ensure no race exists. A single instruction can verify whether the key is taken and, if not, seize it. How this is done is architecture-specific, but almost all processors implement an atomic test and set instruction that tests the value of an integer and sets it to a new value only if it is zero. A value of zero means unlocked. On the popular x86 architecture, locks are implemented using such a similar instruction called compare and exchange. 鎖的實作使用的是不可分割的操作，所以不會有 race condition 發生。幾乎大部分的處理器都會實作 test and set，該指令會測試一個變數是否為零，若為零表示未上鎖，則將它設成一個新值。在 x86 上則是 compare and exchange.

### Causes of Concurrency

user space 並行的原因是因為 CPU 是搶佔且需要重新排班的。在 critical section 可能被其它程式搶佔並進入同樣的 critical section 或是讀取共享的資源，此時就會發生 race condition。

核心則有以下會發生並行的狀況：
* Interrupts — An interrupt can occur asynchronously at almost any time, inter- rupting the currently executing code. 中斷隨時都可能發生 (非同步)
* Softirqs and tasklets — The kernel can raise or schedule a softirq or tasklet at almost any time, interrupting the currently executing code. 同樣的，核心也隨時可以引發和排班 softirq 和 tasklet
* Kernel preemption — Because the kernel is preemptive, one task in the kernel can preempt another. 核心支援搶佔
* Sleeping and synchronization with user-space — A task in the kernel can sleep and thus invoke the scheduler, resulting in the running of a new process. 核心中的任務可以進入休眠狀態，然後就會引發排班，導致新的行程進入執行
* Symmetrical multiprocessing — Two or more processors can execute kernel code at exactly the same time. 兩個或兩個以上的處理器同時執行核心程式碼

Kernel developers need to understand and prepare for these causes of concurrency. **It is a major bug if an interrupt occurs in the middle of code that is manipulating a resource and the interrupt handler can access the same resource. Similarly, it is a bug if kernel code is preemptive while it is accessing a shared resource. Likewise, it is a bug if code in the kernel sleeps while in the middle of a critical section. Finally, two processors should never simultaneously access the same piece of data.** With a clear picture of what data needs protection, it is not hard to provide the locking to keep the system stable. Rather, the hard part is identifying these conditions and realizing that to prevent concurrency, you need some form of protection.

了解並行的原因後，我們就要為此做好準備，避免以下四種瑕疵：
1. 如果程式碼正在操作某個資源時，系統產生了一個中斷，而中斷處理程序可以存取相同的資料
2. 核心程式碼存取某個共享資源時被搶佔
3. 核心程式碼在關鍵區中休眠
4. 兩個處理器同時存取相同的資料
我們需要事先就預想到這些情形從而去加鎖。另外，可以避免上述第一點發生的程式碼我們稱作 interrupt-safe 的程式碼。可以避免第四點的稱為 SMP-safe。避免第二點的稱為 preempt-safe。

### Knowing What to Protect

What does need locking? Most global kernel data structures do.A good rule of thumb is that if another thread of execution can access the data, the data needs some sort of locking; if anyone else can see it, lock it. Remember to lock data, not code.
當編寫核心程式時，我們需要問自己以下問題
* 是全域資料嗎？除了當前執行緒，有其他執行緒可以看到這些資料嗎？
* 這些資料會在行程環境和中斷環境中共享嗎？會在兩個不同的中斷處理程序之間共享嗎？
* 如果一個行程在存取某些資料時被搶佔，新排班的行程有可能存取相同資料嗎？
* 當前行程在某些資源上會休眠(受到阻擋)嗎？如果會，這會讓共享資料處在哪種狀態？
* 我要如何避免資料被釋放出來 (? 不懂，沒解釋)
* 如果這個函式在另外一個處理器上被呼叫，會發生什麼事？


## Deadlocks

* Mutual Exclusion
* Hold and Wait
* No Preemption
* Circular Wait
