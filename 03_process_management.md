# Process Management

## The Process

A process is a program (object code stored on some media) in the midst of execution. Processes are, however, more than just the executing program code (often called the text section in Unix). They also include a set of resources such as open files and pending signals, internal kernel data, processor state, a memory address space with one or more memory mappings, one or more threads of execution, and a data section containing global variables. Processes, in effect, are the living result of running program code.The kernel needs to manage all these details efficiently and transparently.

行程不只是實行中的程式（text section），還包含一些資源，例如：已開啟的檔案、待處理的訊號、核心內部的資料、處理器的狀態、具有一或多個記憶體映射的記憶體空間、一個或多個執行緒，而資料區 (data section) 所包含的是全域變數。在 Linux 中 process 有另外一個名字叫做 task。

Threads of execution, often shortened to threads, are the objects of activity within the process. Each thread includes a unique program counter, process stack, and set of processor registers. The kernel schedules individual threads, not processes. In traditional Unix systems, each process consists of one thread. In modern systems, however, multithreaded programs—those that consist of more than one thread—are common. As you will see later, Linux has a unique implementation of threads: It does not differentiate between threads and processes. To Linux, a thread is just a special kind of process.

Thread 則是在行程中活動的物件。每一個執行緒包含了一個獨一無二的 PC、stack、及一組暫存器。核心會對個別的執行緒安排執行順序。Linux 並沒有另外區分 process 與 thead，而是 thread 就是一種特殊的 process。

在 Linux 中使用了 `fork` system call，來建立一個行程。而系統建立新的行程的方式是透過複製既有的行程，也就是父行程，新建立的就是子行程。在 Linux 中，是呼叫 `clone` 來實作 fork。最後透過 `exit()` 系統呼叫來退出執行狀態並且釋放出它的資源。該退出執行的父行程可以通過 `wait4()` 來詢問一個已終止的子行程。子行程在退出執行狀態時，會進入 Zombie 狀態，直到父行程使用 wait() 收屍。

## Process Descriptor and the Task Structure

The kernel stores the list of processes in a **circular doubly linked list** called the task list.2 Each element in the task list is a process descriptor of the type `struct task_struct`, which is defined in <linux/sched.h>.The process descriptor contains all the information about a specific process. 行程描述器中包含了開啟的檔案、行程的位址空間、待處理的訊號、行程狀態、PID 等資訊。

![alt text](/image/fig3_1.png)

![alt text](/image/fig3_2.png)

task_struct 是用 slab 非配的而不是放在 kernel stack 當中。kernel stack 的最底部（lowest memory address）放 `struct thread_info`，其中會有一個 `struct task_struct` 的指標指向其行程描述子。

```cpp
struct thread_info { 
    struct task_struct *task; 
    struct exec_domain *exec_domain; 
    __u32 flags; 
    __u32 status;
    __u32 cpu; 
    int preempt_count;
    mm_segment_t addr_limit; 
    struct restart_block restart_block; 
    void *sysenter_return; 
    int uaccess_err;
}
```

### Process State

![alt text](/image/fig3_3.png)

* TASK_RUNNING: 可執行，不是正在執行就是在 ready queue 中排隊。
* TASK_INTERRUPTIBLE: 行程正在休眠（blocked）等待某個條件成立。當此條件成立後，核心會把該行程的狀態設定為 TASK_RUNNING
* TASK_UNINTERRUPTIBLE: 此狀態如同 TASK_INTERRUPTIBLE，但是如果收到一個訊號，他不會被喚醒或是變成可執行的。或是因為所等待的是件很快就會發生，所以不會回應訊號。
* __TASK_TRACED: 另一個行程正在透過 ptrace 追蹤此行程。
* __TASK_STOPPED: 任務沒有處於執行狀態，也沒有資格排入執行。收到 SIGSTOP,SIGTSTP,SIGTTIN, 或是 SIGTTOU 訊號的程序會進入此狀態。

### Process Context (重要！)

One of the most important parts of a process is the executing program code.This code is read in from an executable file and executed within the program’s address space. Normal program execution occurs in user-space.When a program executes a system call (see Chapter 5,“System Calls”) or triggers an exception, it enters kernel-space.At this point, the kernel is said to be “executing on behalf of the process” and is in process context.

程式碼的執行是行程很重要的一部分。程式碼會從可執行檔讀入，並在城市的位址空間中執行。一般程式會在用戶空間中 (user-space) 執行。當一個程式執行系統呼叫或觸發一個例外，便會進入到核心空間。這時候我們會稱核心代替行程執行，而且處於行程環境 (process context)。

### The Process Family Tree

init 的 PID 為 1，所有行程都是 init 的後代。每個 task_struct 都有一個名為 parent 的指標，用於指向父行程的 task_struct，也會有一個名為 children 的子行程串列。

## Process Creation

Unix 把行程的建立分成兩個函式：`fork()` 和 `exec()`。前者負責複製當前任務來建立一個子行程，主要的不同的是 PPID 與原本的 PID 不一樣。第二個函式會把新的可執行檔載入位址空間並開始執行。

### Copy-on-Write

把父行程的所有資訊都複製給子行程是相當沒有效率的，因此 Linux 的實作使用了 COW。因此，Linux 不會複製行程位址空間，而是讓父行程和子行程共享同一個副本。而資源的複製只會發生在共享的資料被寫入時。這樣使 fork 到 exec 時，避免掉不必要的複製行為。

`fork()` 中使用 `clone()` 來實作。透過旗標可以指定父子行程應該如何共享資源。`clone()` 中會呼叫 `do_fork()`。`do_fork()` 的內容略，子行程會先被喚醒進入排班，核心會故意讓子行程先執行，一般情況之下，子行程就會執行 exec()避免掉 COW 的成本(因為父行程有比較大機率會寫入共享的資料，如此就會複製)。

### vfork

vfork 的功能如同 fork，但 vfork 不會複製，並且子行程一定會先執行，而父行程會被阻擋直到 子行程呼叫 exec() 或是退出。但 Linux 有 COW 就不需要 vfork 了，尤其執行 vfork 若在 exec() 之前當掉，就會變得很麻煩。

## The Linux Implementation of Threads

Threads are a popular modern programming abstraction. They provide multiple threads of execution within the same program in a shared memory address space. They can also share open files and other resources. Threads enable concurrent programming and, on multiple processor systems, true parallelism.

執行執行緒是現代編程技術中一種常見的抽象概念。此機制可以提供多個共享同一位址空間（開啟檔案與其他資源）的執行緒。執行緒的機制還可以支援並行編程，以及在多處理器上支援真正的平行處理。

對 Linux 而言，threads 就是用標準行程來實作，指示跟其他的行程分享某些資源的行程。每一個執行緒都有獨一無二的 `task_struct`。指定共享的方法就是在 clone 函式中給定不同的旗標(細節略)。

### Kernel Threads
It is often useful for the kernel to perform some operations in the background. The kernel accomplishes this via kernel threads—standard processes that exist solely in kernel-space.The significant difference between kernel threads and normal processes is that kernel threads do not have an address space. (Their mm pointer, which points at their address space, is NULL.) They operate only in kernel-space and do not context switch into user-space. Kernel threads, however, are schedulable and preemptable, the same as normal processes.

核心經常需要在背景進行一些工作，為了這個目的核心會使用 kernel thread。核心執行緒與一般程序執行緒最大的差異在，核心執行緒不具有位址空間 (mm 指標被設成 NULL)，因為核心執行緒僅會運作在核心空間，而且不會把環境切換到用戶空間。 kernel thread 也可以排班也可以被插隊，如同一般的 process。只要執行 `ps -ef` 就可以看到核心系統上有哪些執行緒。


## Process Termination

忽略掉 `exit()` 中的細節。exit() 最後會呼叫 schedule() 以便切換到新的行程，但因為處於 EXIT_ZOMBIE 的退出狀態，所以該行程不會再被排班。而處於 EXIT_ZOMBIE 的行程，現在只剩下 kernel stack/thread_info/task_struct 等記憶體資源。剩下來唯一的存在目的就是讓父行程來取回資訊，或是通知核心那是它不感興趣的資訊後，行程會釋放他所把持的剩餘記憶體資源給系統。

## The Dilemma of the Parentless Task

如果父行程在子行程之前就先退出，必須要有一個機制讓子行程找到一個新的父行程，否則會沒有人來幫他收屍。解決方案就是在當前執行緒的群組中找到另一個行程作為父行程，如果失敗了就讓 init 行程當其父行程。