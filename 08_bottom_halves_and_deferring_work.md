# Bottom Halves and Deferring Work

## Why Bottom Halves?

為了立即處理回應硬體中斷，核心會以非同步的方式來快速且簡單的方式來執行中斷處理程序 (top halves)，而下半部就是執行與中斷相關但中斷處理程序不執行的工作。上半部負責必要的工作，如通知硬體收到中斷，或是將記憶體從硬體複製出來。然而複製出來的資料就需要在下半部去處理。

工作要放在上半部還是下半部並沒有硬性規定，但以下列出應該放在上半部的提示：

* If the work is time sensitive, perform it in the interrupt handler （工作有時限要求）
* If the work is related to the hardware, perform it in the interrupt handler （與硬體有關）
* If the work needs to ensure that another interrupt (particularly the same interrupt) does not interrupt it, perform it in the interrupt handler （該工作要確保不會被中斷，特別是同一個中斷）
* For everything else, consider performing the work in the bottom half （除此之外，都應該放在下半部）

The point of a bottom half is not to do work at some specific point in the future, but simply to defer work until any point in the future when the system is less busy and interrupts are again enabled. Often, bottom halves run immediately after the interrupt returns. The key is that they run with all interrupts enabled. 下半部不需要在某個特定的時間執行，而是只要拖延到系統比較不忙錄而且中斷再次被允許的時候執行即可。通常下半部會在中斷處理程序後立即執行，關鍵是，**下半部執行時會允許所有中斷**。

## A World of Bottom Halves

實作 ISR 的方法只有一種，但實作下半部的方法有好多種。最初的下半部直接稱為 BH (在此不贅述)，後來有用 task queue 來串接待呼叫函式，但對於一些有效能要求的系統來說，此機制不足夠靈活。

### Softirqs and Tasklets

During the 2.3 development series, the kernel developers introduced softirqs and tasklets. With the exception of compatibility with existing drivers, softirqs and tasklets could completely replace the BH interface. **Softirqs are a set of statically defined bottom halves that can run simultaneously on any processor; even two of the same type can run concurrently**. **Tasklets are flexible, dynamically created bottom halves built on top of softirqs. Two different tasklets can run concurrently on different processors, but two of the same type of tasklet cannot run simultaneously**. Thus, tasklets are a good trade-off between performance and ease of use. **For most bottom-half processing, the tasklet is sufficient. Softirqs are useful when performance is critical, such as with networking**. Using softirqs requires more care, however, because two of the same softirq can run at the same time. In addition, softirqs must be registered statically at compile time. Conversely, code can dynamically register tasklets.

softirq 是一組靜態定義的下半部，可以以在所有處理器上同事執行，相當靈活（但同時也要更小心使用）。tasklet 是一個在 softirq 之上動態建立的下半部機制。兩個不同的 tasklet 可以在不同處理器上同時執行，但兩個相同的 tasklet 不能同時執行。在效能至關重要時，使用 softirq（但要小心使用）。

(tasklet 這個命名有點糟糕，因為它與 task 其實沒有關係，可以把 tasklet 當作簡易版的 softirq。大部分的情況之下 tasklet 是夠用的)

Additionally, the task queue interface was replaced by the work queue interface. Work queues are a simple yet useful method of queuing work to later be performed in process context. We get to them later. Consequently, today 2.6 has three bottom-half mechanisms in the kernel: **softirqs**, **tasklets**, and **work queues**. The old BH and task queue interfaces are but mere memories.

總之，2.6 之後的版本有三種下半部機制：softirq, tasklet, and workqueues.

## softirq, tasklet, and workqueues

跳過細節，在此大致敘述一下三種下半部

softirq 和 tasklet 依然是在中斷環境中執行的，而 workqueues 是在行程環境中執行的。因此若被拖延的工作需要休眠的話要使用 workqueues，若不需要休眠則用 softirq 和 tasklet。另外一方面，當你不需要用一個核心執行緒來處理延後執行的工作時，可以考慮使用 tasklet (猜：因為 tasklet 在中斷環境直接執行，不需要額外的 kernel threads 的調度執行。)

Tasklet 的执行速度比 workqueue 更快，因为它们在中断上下文中直接执行，无需额外的内核线程来调度或排班。因此 tasklet 適合需要快速響應和執行的延遲任務，例如網路中斷、timer 等，而 workqueue 適合不需要立即執行的任務，例如後台處理與定期檢查等。

## Disabling Bottom Halves

下半部有可能在任何時刻執行，因此必須討論到上鎖機制。 tasklet 自行負責自己的序列化，意味著兩個同類型的 tasklet 不可以同時執行，即使他們位於不同的處理器上。這代表著不用擔心 intra-tasklet 的並行問題。而 softirq 並不提供序列化的保護，即同類型的兩個實例有可能同時執行，所以被共享的資料都需要使用恰當的鎖去保護。