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

 System calls also provide a return value of type long4 that signifies suc- cess or error—usually, although not always, a negative return value denotes an error. A return value of zero is usually (but again not always) a sign of success.The C library, when a system call returns an error, writes a special error code into the global errno variable.

System calls have a defined behavior. For example, the implementation of the syscall `getpid()` in the kernel is simple:
```c
SYSCALL_DEFINE0(getpid) {
    return task_tgid_vnr(current); // returns current->tgid
}

// SYSCALL_DEFINE0 is simply a macro that defines a system call with no parameters
```

Note that the asmlinkage modifier on the function definition.This is a directive to tell the compiler to look only on the stack for this function’s arguments.This is a required modifier for all system calls. Second, the func- tion returns a long. For compatibility between 32- and 64-bit systems, system calls defined to return an int in user-space return a long in the kernel.Third, note that the getpid() system call is defined as sys_getpid() in the kernel.This is the naming con- vention taken with all system calls in Linux.

### System Call Numbers

**In Linux, each system call is assigned a syscall number**.This is a unique number that is used to reference a specific system call.When a user-space process executes a system call, the syscall number identifies which syscall was executed; **the process does not refer to the syscall by name.** 

The syscall number is important; when assigned, it cannot change, or compiled applications will break.

The kernel keeps a list of all registered system calls in the **system call table**, stored in sys_call_table. This table is architecture; on x86-64 it is defined in arch/i386/kernel/syscall_64.c. This table assigns each valid syscall to a unique syscall number.

## System Call Handler

**System calls 就是當作中斷處理**

It is not possible for user-space applications to execute kernel code directly.They cannot simply make a function call to a method existing in kernel-space because the kernel exists in a protected memory space.

Instead, user-space applications must somehow signal to the kernel that they want to execute a system call and have the system switch to kernel mode, where the system call can be executed in kernel-space by the kernel on behalf of the application.

The mechanism to signal the kernel is a software interrupt: Incur an exception, and the system will switch to kernel mode and execute the exception handler.The exception handler, in this case, is actually the system call handler. *$\left( \text{software interrupt} \supset \text{system call handler} \right)$**

he defined software interrupt on x86 is interrupt number 128, which is incurred via the int $0x80 instruction. It triggers a switch to kernel mode and the execution of exception vector 128, which is the system call handler. The system call handler is the aptly named function system_call(). It is architecture-dependent.

### Denoting the Correct System Call

以上只說明了 system call 用中斷方式進入核心空間，以下說明如何指定正確的系統呼叫
Simply entering kernel-space alone is not sufficient because multiple system calls exist, all of which enter the kernel in the same manner.Thus, **the system call number must be passed into the kernel**. On x86, the syscall number is fed to the kernel via the eax register. Before causing the trap into the kernel, user-space sticks in eax the number corresponding to the desired system call.The system call handler then reads the value from eax. Other architectures do something similar.

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