# Process Scheduling

## Multitasking

Multitasking operating systems come in two flavors: cooperative multitasking and preemptive multitasking. 多任務作業系統分成兩類：協同式多任務和搶佔式多任務。

In cooperative multitasking, a process does not stop running until it voluntary decides to do so.The act of a process voluntarily suspending itself is called yielding. 在協同式多任務模式中，一個行程只有在自願停止的情況之下才會讓出 CPU，這個動作稱為讓步 yielding。缺點是一個行程可能會獨佔處理器，因此現代大多數的作業系統並不採納此模式。

Linux, like all Unix variants and most modern operating systems, implements preemptive multitasking. In preemptive multitasking, the scheduler decides when a process is to cease running and a new process is to begin running. Linux 如同大部分的現代作業系統是採取搶佔式。優排班器決定一個行程何時停止，好讓其他的行程有執行的機會。最常見的方法是設置 time slice，而在現代許多作業系統中，time slice 的值是動態算出來的，會與策略有關。

## Linux’s Process Scheduler

在 Linux 2.5 中，Linux 核心的排班器大幅修改，並稱為 O(1) 排班器。但後來發現該排班器在對於互動式行程的處理並不好。因此在 2.6.23 後被 **Completely Fair Scheduler 所取代**。（CFS 的資料結構為紅黑樹）

## Policy

A scheduler’s policy often determines the overall feel of a system and is responsible for optimally utilizing processor time. 排班器策略決定了整體系統的效能

### I/O-Bound Versus Processor-Bound Processes

I/O-bound 的行程會把大部分的時間用來提交和等待 I/O 請求，而 process-bound 的行程用大部分的時間來執行程式碼。對於排班器的策略往往是降低處理器密集行程的執行頻率，但延長他們的執行時間。

Unix/Linux 系統中的排班策略往往利於 I/O 密集行程，因而可以得到良好的行程回應時間。

### Process Priority

A common type of scheduling algorithm is priority-based scheduling. 排班演算法最常見的就是基於優先權 (priority-based) 的排班。

The Linux kernel implements two separate priority ranges. The first is the nice value, a number from –20 to +19 with a default of 0. Larger nice values correspond to a lower priority—you are being “nice” to the other processes on the system. The nice value is a control over the absolute timeslice allotted to a process; in Linux. 

Linux 核心實作了兩組獨立的優先權範圍，第一組是 nice 值，從 -20 到 +19，nice 值越大表示優先權越低。在 Linux 中，nice 值控制的是 time slice 的比例。

The second range is the real-time priority. The values are configurable, but by default range from 0 to 99, inclusive. Opposite from nice values, higher real-time priority values correspond to a greater priority. All real-time processes are at a higher priority than normal processes; that is, the real-time priority and nice value are in disjoint value spaces. 第二組範圍是即時優先權，這個值是可以設定的，從 0 - 99。**較高的值代表優先權越高。並且所有的即時行程的優先權均高於一般行程**。

### Timeslice

The timeslice2 is the numeric value that represents how long a task can run until it is preempted.

Linux’s CFS scheduler, however, does not directly assign timeslices to processes. Instead, in a novel approach, CFS assigns processes a proportion of the processor. On Linux, therefore, the amount of processor time that a process receives is a function of the load of the system. This assigned proportion is further affected by each process’s nice value. The nice value acts as a weight, changing the proportion of the processor time each process receives. 

Linux 的 CFS 排班器並不直接將時間片斷分配給行程。CFS 分配給行程的是處理器時間的比例（用 nice 值）。

### The Scheduling Policy in Action

由於 Linux 是互動式的系統，在我們給予 I/O-bound 程序與 CPU-bound 程序一樣的比例處理器時間，如此一來，當 I/O-bound 程序出現時，會立刻執行，因為實際上 IO-bound 的程序一定不會用到與 CPU-bound 一樣多的比例，因此，IO-bound 的程序能夠很快的被排班到執行，確保系統互動的良好。

## The Linux Scheduling Algorithm

排班的實作細節大部分我都略過。

### The Scheduler Entry Point

The main entry point into the process schedule is the function `schedule()`, defined in kernel/sched.c. This is the function that the rest of the kernel uses to invoke the process scheduler, deciding which process to run and then running it. schedule() is generic with respect to scheduler classes. That is, it finds the highest priority scheduler class with a runnable process and asks it what to run next. 
核心中的 `schedule()` 函式是行程排班的主要進入點。核心的其餘部分會使用此函式來調用行程排班器，決定要執行的行程。該函式會從最高排班器類別中挑選出優先全最高的函式。

### Sleeping and Waking Up

當程序休眠時，會把自己標記成休眠然後移出紅黑樹，並把自己放到等待佇列中，然後呼叫 schedule()。喚醒的過程剛好相反，任務會被設成可執行的，然後從等待佇列移出，並加入到紅黑樹。

## Preemption and Context Switching

Context switching, the switching from one runnable task to another, is handled by the `context_switch()` function defined in kernel/sched.c. **It is called by schedule() when a new process has been selected to run**. It does two basic jobs: 
* Calls `switch_mm()`, which is declared in <asm/mmu_context.h>, to switch the vir- tual memory mapping from the previous process’s to that of the new process.
* Calls switch_to(), declared in <asm/system.h>, to switch the processor state from the previous process’s to the current’s.This involves saving and restoring stack infor- mation and the processor registers and any other architecture-specific state that must be managed and restored on a per-process basis.

`context_switch()` 會在 `schedule()` 中被呼叫，並完成以下兩件事：
* 呼叫 `switch_mm()`，該函式負責從上一個行程的虛擬記憶體切換到新行程的虛擬記憶體空間
* **呼叫 `switch_to()` 該函式負責從上一個行程的處理器狀態切換到當前行程的處理器狀態。這包含了儲存和恢復堆疊資訊，以及處理器暫存器和必須管理的任何其他架構特有的狀態。**

很重要的是，核心如何知道什麼時候該呼叫 schedule()? 核心提供了一個旗標 `need_resched` 來表示是否應該進行一次排班。例如一個行程應該被搶佔時，會通過 `scheduler_tick()` 來設定旗標，或是喚醒的行程優先權高於現在正在執行的行程時，會利用 `try_to_wake_up()` 來設定該旗標。核心會檢查此旗標然後呼叫 schedule()。另外在返回用戶空間或是從中斷返回的時候，核心也會檢查此旗標。

### User Preemption

在以下兩種情形下，會發生用戶搶佔：
* 從系統呼叫返回用戶空間時
* 從中斷處理程序返回用戶空間時
核心在這時候都會去檢查旗標然後調用排班器

### Kernel Preemption

The Linux kernel, unlike most other Unix variants and many other operating systems, is a fully preemptive kernel. In nonpreemptive kernels, kernel code runs until completion. That is, the scheduler cannot reschedule a task while it is in the kernel—kernel code is scheduled cooperatively, not preemptively. Kernel code runs until it finishes (returns to user-space) or explicitly blocks. In the 2.6 kernel, however, the Linux kernel became preemptive.

與其它多數的 Unix 和作業系統不同，Linux 是充分支援搶佔的核心。在不支援搶佔的核心中，核心程式碼會一直執行直到完成或是明確受到阻擋為止。Linux 在 2.6 之後成為了支援搶佔的核心。

那麼支援搶佔的核心怎樣才是安全的？核心可以搶佔一個「為持有鎖」的核心。因此每一個 `thread_info` 中都有一個 `preempt_count`，取得一個鎖時，該值加一，反之減一。

另外，當核心中的任務受到阻擋，或是明確了呼叫 schedule() 時，也會明確地發生核心搶佔。這種搶佔是一直都支援的，所以不需要額外的邏輯來確認是否可以安全地被搶佔。如果程式碼明確的呼叫 schedule()，會假設程式碼知道自己可以安全地被搶佔。

以下為核心搶佔發生的時機：
- When an interrupt handler exits, before returning to kernel-space (從中斷處理程序退出，返回核心空間之前)
- When kernel code becomes preemptible again (?)
- If a task in the kernel explicitly calls schedule() (明確呼叫 schedule())
- If a task in the kernel blocks (which results in a call to schedule()) (受到阻擋)

## Real-Time Scheduling Policies

Linux provides two real-time scheduling policies,SCHED_FIFO and SCHED_RR. The normal, not real-time scheduling policy is SCHED_NORMAL. Via the scheduling classes framework, these real-time policies are managed not by the Completely Fair Scheduler, but by a special real-time scheduler, defined in kernel/sched_rt.c.

Linux 的非即時排班策略：SCHED_NORMAL (由 CFS 負責)
Linux 的即時排班策略：SCHED_FIFO 與 SCHED_RR (非由 CFS 負責)
  * SCHED_FIFO: 一定比 SCHED_NORMAL 優先，執行直到被阻擋，或是主動讓出處理器，非基於時間片段，優先權較高的 SCHED_FIFO 和 SCHED_RR 可以搶佔
  * SCHED_RR: 與 SCHED_FIFO 大致相同，但是是基於時間片段
  * **以上兩種即時排班策略實作的都是靜態優先權，核心並不會為即時任務計算動態優先權值。這可以確保特定優先權的即時行程總是可以搶佔優先權較低的行程。**
  
Linux 的即時排班策略所提供的是軟性的即時性能。軟性即時會嘗試在時限 (timing deadline) 之內執行任務，但核心無法保證可以滿足這些任務的排班要求。相對於硬性即時系統保證在時限內可以滿足排班要求，Linux 對即時任務的排班不提供任何保證。

## Scheduler-Related System Calls
### Processor Affinity System Calls

The Linux scheduler enforces hard processor affinity. That is, although it tries to provide soft or natural affinity by attempting to keep processes on the same processor. Linux 排班器提供的是硬性的處理器偏好 (processsor affinity) 機制來試著將形成保持在相同的處理器上面。

核心所提供的硬性偏好機制很簡單。首先行程被建立時會繼承父行程的偏好遮罩。其次，當處理器偏好遭到改變時，核心會使用遷移執行緒將任務推到合法的處理器上。最後附載平衡器只會把任務拉到被允許的處理器上。因此行程只能執行在被允許的處理器上，這個資訊是在行程描述器 cpus_allowed 欄位中所設定的。




