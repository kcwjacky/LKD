# System Calls

## Communicating with the Kernel

In Linux, system calls are the only means user-space has of interfacing with the kernel; they are the only legal entry point into the kernel other than exceptions and traps. Indeed, other interfaces, such as device files or /proc, are ultimately accessed via system calls.

## API, POSIX, and the C Library

The system call interface in Linux, as with most Unix systems, is provided in part by the C library.The C library implements the main API on Unix systems, including the standard C library and the system call interface.The C library is used by all C programs and, because of C’s nature, is easily wrapped by other programming languages for use in their programs.The C library additionally provides the majority of the POSIX API.

1. C Standard Function: printf() in main.c
2. printf() in C Library
3. POSIX API: write() in C Library
4. write() in kernel space

## Syscalls

System calls (often called syscalls in Linux) are typically accessed via function calls **defined in the C library**. They can define zero, one, or more arguments (inputs) and might result in one or more side effects.

 System calls also provide a return value of type long4 that signifies success or error—usually, although not always, a negative return value denotes an error. A return value of zero is usually (but again not always) a sign of success.The C library, when a system call returns an error, writes a special error code into the global errno variable.

System calls have a defined behavior. For example, the implementation of the syscall `getpid()` in the kernel is simple:
```c
SYSCALL_DEFINE0(getpid) {
    return task_tgid_vnr(current); // returns current->tgid
}

// SYSCALL_DEFINE0 is simply a macro that defines a system call with no parameters
```

Note that the asmlinkage modifier on the function definition. This is a directive to tell the compiler to **look only on the stack for this function’s arguments**. This is a required modifier for all system calls. Second, the function returns a long. For compatibility between 32- and 64-bit systems, system calls defined to return an int in user-space return a long in the kernel. Third, note that the getpid() system call is defined as sys_getpid() in the kernel. This is the naming convention taken with all system calls in Linux.

### System Call Numbers

**In Linux, each system call is assigned a syscall number**.This is a unique number that is used to reference a specific system call. When a user-space process executes a system call, the syscall number identifies which syscall was executed; **the process does not refer to the syscall by name.** 在 x86 上，系統呼叫會通過 eax 暫存器把系統呼叫編號傳遞給核心。

The syscall number is important; when assigned, it cannot change, or compiled applications will break.

The kernel keeps a list of all registered system calls in the **system call table**, stored in sys_call_table. This table is architecture; on x86-64 it is defined in arch/i386/kernel/syscall_64.c. This table assigns each valid syscall to a unique syscall number.

## System Call Handler

**System calls 就是當作中斷處理**

It is not possible for user-space applications to execute kernel code directly. They cannot simply make a function call to a method existing in kernel-space because the kernel exists in a protected memory space.

Instead, user-space applications must somehow signal to the kernel that they want to execute a system call and have the system switch to kernel mode, where the system call can be executed in kernel-space by the kernel on behalf of the application. (核心可以代替應用程式在核心空間執行該系統呼叫)

**The mechanism to signal the kernel is a software interrupt**: Incur an exception, and the system will switch to kernel mode and execute the exception handler.The exception handler, in this case, is actually the system call handler. $$ \left( \text{software interrupt} \supset \text{system call handler} \right) $$

The defined software interrupt on x86 is interrupt number 128, which is incurred via the int $0x80 instruction. It triggers a switch to kernel mode and the execution of exception vector 128, which is the system call handler. The system call handler is the aptly named function system_call(). It is architecture-dependent.

Recently, x86 processors added a feature known as sysenter.This feature provides a faster, more specialized way of trapping into a kernel to execute a system call than using the int interrupt instruction. Support for this feature was quickly added to the kernel. Regardless of how the system call handler is invoked, however, the important notion is that somehow user-space causes an exception or trap to enter the kernel. x86 提供 sysenter 指令提供更快執行系統呼叫的核心陷入 (trap) 方法。

### Denoting the Correct System Call

以上只說明了 system call 用中斷方式進入核心空間，以下說明如何指定正確的系統呼叫
Simply entering kernel-space alone is not sufficient because multiple system calls exist, all of which enter the kernel in the same manner. Thus, **the system call number must be passed into the kernel**. On x86, the syscall number is fed to the kernel via the eax register. Before causing the trap into the kernel, user-space sticks in eax the number corresponding to the desired system call.The system call handler then reads the value from eax. Other architectures do something similar.

檢查給定的系統呼叫編號的有效性
The system_call() function checks the validity of the given system call number by comparing it to NR_syscalls. If it is larger than or equal to NR_syscalls, the function returns -ENOSYS. Otherwise, **the specified system call is invoked**:

```c
system_call() // system call handler
-> validate the given system call number 
-> call *sys_call_table(,%rax,8), if validated 
-> e.g., sys_read()
```

### Parameter Passing

In addition to the system call number, most syscalls require that one or more parameters be passed to them. Somehow, user-space must relay the parameters to the kernel during the trap.The easiest way to do this is via the same means that the syscall number is passed: **The parameters are stored in registers**. On x86-32, the registers ebx, ecx, edx, esi, and edi contain, in order, the first five arguments. In the unlikely case of six or more argu- ments, a single register is used to hold a pointer to user-space where all the parameters are stored.
The return value is sent to user-space also via register. On x86, it is written into the eax register.

### Verifying the Parameters

System calls must carefully verify all their parameters to ensure that they are valid and legal. The system call runs in kernel-space, and if the user can pass invalid input into the kernel without restraint, the system’s security and stability can suffer.

One of the most important checks is the validity of any pointers that the user provides. Imagine if a process could pass any pointer into the kernel, unchecked, with warts and all, even passing a pointer to which it did not have read access! Processes could then trick the kernel into copying data for which they did not have access permission, such as data belonging to another process or data mapped unreadable. Before following a pointer into user-space, the system must ensure that

* The pointer points to a region of memory in user-space. Processes must not be able to trick the kernel into reading data in kernel-space on their behalf. **user-space process 不得存取 kernel-space 的資料**
* The pointer points to a region of memory in the process’s address space. The process must not be able to trick the kernel into reading someone else’s data. **user-space process 不得存取其他 process 的資料**
* If reading, the memory is marked readable. If writing, the memory is marked writable. If executing, the memory is marked executable. The process must not be able to bypass memory access restrictions. **資料讀取必須符合記憶體存取的限制設定**

The kernel provides two methods for performing the requisite checks and the desired copy to and from user-space.

For writing into user-space, the method copy_to_user() is provided. For reading from user-space, the method copy_from_user().

> unsigned long copy_to_user (void __user * to, const void * from, unsigned long n) 
&nbsp;&nbsp;&nbsp;&nbsp;to: destination memory address in the process's address space
&nbsp;&nbsp;&nbsp;&nbsp;from: the source pointer in kernel-space
&nbsp;&nbsp;&nbsp;&nbsp;n: the size in bytes of the data to copy 

> unsigned long copy_from_user (void * to, const void __user * from, unsigned long n)
&nbsp;&nbsp;&nbsp;&nbsp;The function reads from the second parameter into the first parameter the number of bytes specified in the third parameter.

Both of these functions return the number of bytes they failed to copy on error. On success, they return zero. It is standard for the syscall to return -EFAULT in the case of such an error. Both copy_to_user() and copy_from_user() may block. This occurs, for example, the page containing the user data is not in physical memory. 回傳值為未能成功複製的位元組數目，因此成功時為 0。當 page fault 時會 block (休眠)。

A final possible check is for valid permission. The new system “capabilities” enables specific access checks on specific resources. A call to capable() with a valid capabilities flag returns nonzero if the caller holds the specified capability and zero otherwise. 最後使用 `capable` 函式與不同的旗標，可以用來檢查呼叫者是否具有對特定資源的權限。

## System Call Context

**The kernel is in process context during the execution of a sys- tem call.The current pointer points to the current task, which is the process that issued the syscall.** 系統呼叫執行的時後，核心是位於行程環境的，該行程即為發出系統呼叫的行程，並且此時 current 指標會指向該行程。 In process context, the kernel is capable of sleeping (for example, if the system call blocks on a call or explicitly calls schedule()) and is fully preemptible. 在行程的環境中，kernel 可以休眠並且可以被搶佔。The fact that process context is preemptible implies that, like user-space, the current task may be preempted by another task. Because the new task may then execute the same system call, care must be exercised to ensure that system calls are reentrant. 由於可休眠以及可以被搶佔的特性，當前行程可能被另外一個行程搶佔，並且呼叫相同的系統呼叫，因次，必須確保系統呼叫是可以重入的 (reentrant)。


### Accessing the System Call from User-Space

Generally, the C library provides support for system calls. User applications can pull in function prototypes from the standard headers and link with the C library to use your system call (or the library routine that, in turn, uses your syscall call).

If you just wrote the system call, however, it is doubtful that glibc already supports it! Thankfully, Linux provides a set of macros for wrapping access to system calls. It sets up the register contents and issues the trap instructions.These macros are named _syscalln(), where n is between 0 and 6. The number corresponds to the number of parameters passed into the syscall because the macro needs to know how many parame- ters to expect and, consequently, push into registers. 這一段要說明，你無法直接使用 C Library 來呼叫自己實現的 system call。但是可以利用 `_syscalln()` 這個巨集來呼叫系統呼叫，該巨集包裝了系統呼叫的調用以及 trap 指令。

### Why Not to Implement a System Call

* You need a syscall number, which needs to be officially assigned to you.
* After the system call is in a stable series kernel, it is written in stone.The interface cannot change without breaking user-space applications.
* For simple exchanges of information, a system call is overkill. The alternatives:
  * Implement a device node and read() and write() to it. Use ioctl() to manipu- late specific settings or retrieve specific information.
  * Certain interfaces, such as semaphores, can be represented as file descriptors and manipulated as such.
  * Add the information as a file to the appropriate location in sysfs.