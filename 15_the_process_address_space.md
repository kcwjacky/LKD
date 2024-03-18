# The Process Address Space

Chapter 12, “Memory Management,” looked at how the kernel manages physical memory. In addition to managing its own memory, the kernel also has to manage the memory of user-space processes.This memory is called the process address space, which is the representation of memory given to each user-space process on the system. Linux is a virtual memory operating system, and thus the resource of memory is virtualized among the processes on the system. An individual process’s view of memory is as if it alone has full access to the system’s physical memory. More important, the address space of even a single process can be much larger than physical memory.This chapter discusses how the kernel manages the process address space.

在 12 章中介紹了核心管理實體記憶體的方式。事實上，核心除了管理自己的記憶體外，還需要管理用戶空間行程的記憶體。此記憶體稱為用戶位址空間 (process address space)。Linux 是一種採用虛擬記憶體的作業系痛，因此系統上每個行程之間將以虛擬化的方式共享記憶體資源。所以每一個行程都彷彿能夠存取整個系統的實體記憶體。甚至它能使用的位址空間可以遠大於實體記憶體。

## Address Spaces

The process address space consists of the virtual memory addressable by a process and the addresses within the virtual memory that the process is allowed to use. Each process is given a flat 32- or 64-bit address space, with the size depending on the architecture.

對於一個 32 位元的位址空間，雖然理論上可以定址到 4GB 的記憶體，但事實上行程有權存取的記憶體位址區間是有限制的。這些可被行程存取的合法位址區間稱為**記憶體區 (memory area)**，行程可以透過核心對他的位址空間動態加入或移除記憶體區。

記憶體中可包含：

* A memory map of the executable file’s code, called the text section.
可執行檔案之程式碼的記憶體映射，稱為文字區 text section
* A memory map of the executable file’s initialized global variables, called the data section.
可執行檔案之已初始化全域變數的記憶體映射，稱為資料區 data section
* A memory map of the zero page (a page consisting of all zeros, used for purposes such as this) containing uninitialized global variables, called the bss section.
未初始化全域變數的頁面（由零構成）的記憶體區，稱為 BSS (block started by symbol)
* A memory map of the zero page used for the process’s user-space stack. (Do not confuse this with the process’s kernel stack, which is separate and maintained and used by the kernel.)
用於行程之用戶空間 stack 的記憶體映射。（與行程的核心堆疊不一樣，行程的核心堆疊獨立存在，且由核心使用和維護）
* An additional text, data, and bss section for each shared library, such as the C library and dynamic linker, loaded into the process’s address space. 共享函式庫像是 C 語言程式庫和動態連接器的程式碼、資料和 bss 區也會被載入行程的位址空間。
* Any memory mapped files.
* Any shared memory segments.
* Any anonymous memory mappings, such as those associated with malloc(). (新版的 glibc 用 mmap() 和 brk() 來實作 malloc())

## The Memory Descriptor

The kernel represents a process’s address space with a data structure called the memory descriptor.This structure contains all the information related to the process address space. The memory descriptor is represented by struct mm_struct and defined in <linux/mm_types.h>. Let’s look at the memory descriptor, with comments added describing each field:
核心會使用一個稱為記憶體描述器（memory descriptor）的資料結構來表示行程的位址空間。其包含了與行程位址有關的所有資訊。

```cpp
struct mm_struct {
    struct vm_area_struct   *mmap;              /* list of memory areas */
    struct rb_root struct   mm_rb;              /* red-black tree of VMAs */
    vm_area_struct          *mmap_cache;        /* last used memory area */
    unsigned long           free_area_cache;    /* 1st address space hole */
    pgd_t                   *pgd;               /* page global directory */ 
    atomic_t                mm_users;           /* address space users 使用此位址空間的用戶行程數目 */ 
    atomic_t                mm_count;           /* primary usage counter 核心管理用？ */
    int                     map_count;          /* number of memory areas */ 
    struct rw_semaphore     mmap_sem;           /* memory area semaphore */ 
    spinlock_t              page_table_lock;    /* page table lock */
    struct list_head        mmlist;             /* list of all mm_structs */ 
    unsigned long           start_code;         /* start address of code */ 
    unsigned long           end_code;           /* final address of code */
    unsigned long           start_data;         /* start address of data */ 
    unsigned long           end_data;           /* final address of data */ 
    unsigned long           start_brk;          /* start address of heap */
    unsigned long           brk;                /* final address of heap */ 
    unsigned long           start_stack;        /* start address of stack */ 
    unsigned long           arg_start;          /* start of arguments */
    unsigned long           arg_end;            /* end of arguments */
    unsigned long           env_start;          /* start of environment */ 
    unsigned long           env_end;            /* end of environment */
    unsigned long           rss;                /* pages allocated */
    unsigned long           total_vm;           /* total number of pages */ 
    unsigned long           locked_vm;          /* number of locked pages */
    unsigned long           saved_auxv[AT_VECTOR_SIZE]; /* saved auxv */
    cpumask_t               cpu_vm_mask;        /* lazy TLB switch mask */
    unsigned long           context;            /* arch-specific data */
    int                     flags;              /* status flags */
    mm_context_t            core_waiters;       /* thread core dump waiters */ 
    struct core_state       *core_state;        /* core dump support */
    spinlock_t              ioctx_lock;         /* AIO I/O list lock */ 
    struct hlist_head       ioctx_list;         /* AIO I/O list */
};
```

* mmap 和 mm_rb 這兩個欄位雖然是不同的資料結構但是所包含的內容卻是相同的東西，就是該位址空間的所有記憶體區。前者把它們存入一個鏈結串列，後者存到紅黑樹。當串列結構與樹狀結構重疊時稱為穿線樹 (threaded tree)。mmap 和 mm_rb 事實上都指向了與記憶體描述器相關聯的所有記憶體區物件 (後面會說的 vm_area_struct)
* 所有的 mm_struct 結構都會用自己的 mmlist 欄位連接成一個雙向鏈結串列。串列中的第一個元素是 init_mm 記憶體描述子，用來描述 init 行程的空間位址。
* 記憶體描述器會被存放在行程描述器的 mm 欄位中，因此可以用 current->mm 來存取
* 父行程可以選擇和子行程共享位址空間，只要給定 clone() CLONE_VM 的旗標。

### The mm_struct and Kernel Threads
Kernel threads do not have a process address space and therefore do not have an associated memory descriptor.Thus, the mm field of a kernel thread’s process descriptor is NULL. This is the definition of a kernel thread—processes that have no user context. 核心執行緒並不具有行程位址空間，所以不具有相關聯的記憶體描述器。因此，核心執行緒的 mm 欄位的值是 NULL。

## Virtual Memory Areas

**The memory area structure, vm_area_struct, represents memory areas. It is defined in <linux/mm_types.h>. In the Linux kernel, memory areas are often called virtual memory areas (abbreviated VMAs)**. 記憶體區 (memory area) 由定義於 <linux/mm_types.h> 的記憶體區結構 `vm_area_struct` 來表示，通常會被稱為虛擬記憶體區 (VMA)

The vm_area_struct structure describes a single memory area over a contiguous interval in a given address space.The kernel treats each memory area as a unique memory object. Each memory area possesses certain properties, such as permissions and a set of associated operations. In this manner, each VMA structure can represent different types of memory areas—for example, memory-mapped files or the process’s user-space stack. vm_area_struct 結構會將單一記憶體區描述成一個特定位址空間中連續的範圍。核心會把每個記憶體區視為獨一無二的記憶體物件。每個 VMA 結構都可以表示不同類型的記憶體區，例如記憶體映射檔或是行程的用戶空間 stack。

```cpp
struct vm_area_struct { 
    struct mm_struct *vm_mm;        /* associated mm_struct */ 
    unsigned long vm_start;         /* VMA start, inclusive */ 
    unsigned long vm_end;           /* VMA end , exclusive */ 
    struct vm_area_struct *vm_next; /* list of VMA’s */
    pgprot_t vm_page_prot;          /* access permissions */
    unsigned long vm_flags;         /* flags */
    struct rb_node vm_rb;           /* VMA’s node in the tree */
    
    union {     /* links to address_space->i_mmap or i_mmap_nonlinear */
        struct {
            struct list_head        list;
            void                    *parent;
            struct vm_area_struct   *head;
        } vm_set;
        struct prio_tree_node prio_tree_node;
    } shared;

    struct list_head                anon_vma_node;      /* anon_vma entry */
    struct anon_vma                 *anon_vma;          /* anonymous VMA object */ 
    struct vm_operations_struct     *vm_ops;            /* associated ops */
    unsigned long                   vm_pgoff;           /* offset within file */ 
    struct file                     *vm_file;           /* mapped file, if any */ 
    void                            *vm_private_data;   /* private data */
};
```

* 每個記憶體描述器 (mm_struct) 會被關聯到行程位址空間中的一個獨一無二的範圍 (vm_area_struct)，`vm_start` 與 `vm_end` 分別對應此範圍的開頭（最低）和結尾（最高）位址。
* `vm_mm` 欄位會指向此 VMA 相關聯的 mm_struct。每個 VMA 對其相關連的 mm_struct 都是獨一無二的。two separate processes map the same file into their respective address spaces, each has a unique vm_area_struct to identify its unique memory area. Conversely, two threads that share an address space also share all the vm_area_struct structures therein. 兩個獨立的 process 將一樣的檔案映射到各自的位址空間時，他們會各自有一個獨一無二的 vm_area_struct；相反的，兩個 threads 共享一個位址空間，則他們也共享其中所有的 vm_area_struct 結構。

### VMA flags
* VM_READ/VM_WRITE/VM_EXEC
* VM_SHARED 書名該記憶體區是否為多個行程共享
* etc

### VMA Operations
The vm_ops field in the vm_area_struct structure points to the table of operations associated with a given memory area, which the kernel can invoke to manipulate the VMA. The vm_area_struct acts as a generic object for representing any type of memory area, and the operations table describes the specific methods that can operate on this particular instance of the object. `vm_area_struct` 結構中的 `vm_ops` 欄位會指向與被給定記憶體相關連的操作表，核心可以調用此表中的韓式來操作 VMA。vm_area_struct 是一個通用物件，可用於表示任何類型的記憶體區，而操作表所描述的特定方式可用於操作物件的特定實例。

The operations table is represented by struct vm_operations_struct and is defined in <linux/mm.h>:
```cpp
struct vm_operations_struct {
    void (*open) (struct vm_area_struct *);
    void (*close) (struct vm_area_struct *);
    int (*fault) (struct vm_area_struct *, struct vm_fault *);
    int (*page_mkwrite) (struct vm_area_struct *vma, struct vm_fault *vmf); 
    int (*access) (struct vm_area_struct *, unsigned long, void *, int, int);
};
```

* `void open(struct vm_area_struct *area)`
This function is invoked when the given memory area is added to an address space. 
當指定的記憶體區被加入一個位址空間時，會調用此函式
* `void close(struct vm_area_struct *area)`
This function is invoked when the given memory area is removed from an address space.
當指定的記憶體區被從一個位址空間移除時，會調用此函式
* `int fault(struct vm_area_sruct *area, struct vm_fault *vmf)`
This function is invoked by the page fault handler when a page that is not present in physical memory is accessed.
當所存取的頁面不再實體記憶體中時，頁面失誤處理程序 (page fault handler) 就會調用此函式
* `int page_mkwrite(struct vm_area_sruct *area, struct vm_fault *vmf)`
This function is invoked by the page fault handler when a page that was read-only is being made writable.
當一個唯讀頁面要被變成可寫時，頁面失誤處理程序就會調用此函式
* `int access(struct vm_area_struct *vma, unsigned long address, void *buf, int len, int write)`
This function is invoked by access_process_vm() when get_user_pages() fails.
當 get_user_pages() 失敗時，access_process_vm() 就會調用此函式

### Memory Areas in Real Life

以下對一個直接 return 0 的程式，我們來觀察它的位址空間和其中所包含的記憶體區，可以用 `/proc` 檔案系統和 `pmap`：
下列可以看到文字區、資料區和 bss。若此程式與 C 程式庫動態連結，則還可以分別看到與 libc.so 和 ld.so 相應的上述三個區域。最後還可以看到行程的堆疊。

```
rlove@wolf:~$ cat /proc/1426/maps                                       // /proc/<pid>/maps
00e80000-00faf000 r-xp 00000000 03:01 208530  /lib/tls/libc-2.5.1.so    // text of libc.so
00faf000-00fb2000 rw-p 0012f000 03:01 208530  /lib/tls/libc-2.5.1.so    // data of libc.so
00fb2000-00fb4000 rw-p 00000000 00:00 0                                 // bss of libc.so
08048000-08049000 r-xp 00000000 03:03 439029  /home/rlove/src/example   // text of the example
08049000-0804a000 rw-p 00000000 03:03 439029  /home/rlove/src/example   // data of the example
40000000-40015000 r-xp 00000000 03:01 80276   /lib/ld-2.5.1.so          // text of ld
40015000-40016000 rw-p 00015000 03:01 80276   /lib/ld-2.5.1.so          // data of ld
4001e000-4001f000 rw-p 00000000 00:00 0                                 // bss of ld
bfffe000-c0000000 rwxp fffff000 00:00 0                                 // stack of the example

The data is in the form: start-end permission offset
```

```
rlove@wolf:~$ pmap 1426
/* pmap 出來的資訊可讀性更好，細節略 */ 
```

## Manipulating Memory Areas

The kernel often has to perform operations on a memory area, such as whether a given address exists in a given VMA.These operations are frequent and form the basis of the mmap() routine, which is covered in the next section. 
核心常需要在記憶體區上進行某些操作，像是判斷某個位址是否存在於某個 VMA 中。這些操作是 mmap() 的基礎，以下先說明一個 find_vma() 就好

* `struct vm_area_struct * find_vma(struct mm_struct *mm, unsigned long addr);`
此函式會再被給定的定址空間搜尋第一個「vm_end 欄位大於 addr」的記憶體區。
* `struct vm_area_struct * find_vma_prev(struct mm_struct *mm, unsigned long addr, struct vm_area_struct **pprev)`
此函式與 find_vma 工作方式相同，另外還會傳回小於 addr 的第一個 VMA。
* `find_vma_intersection()`
The find_vma_intersection()function returns the first VMA that overlaps a given address interval.

## mmap() and do_mmap(): Creating an Address Interval
The do_mmap() function is used by the kernel to create a new linear address interval. 核心會使用 do_mmap() 函式來建立一個新的線性位址範圍。do_mmap() 將一個位址範圍加入行程的位址空間。(malloc?)

```cpp

unsigned long do_mmap(struct file *file, unsigned long addr, 
                      unsigned long len, unsigned long prot, 
                      unsigned long flag, unsigned long offset)
```

此函式會映射 file 所指定的檔案。若給定 file 為 NULL，則稱為匿名映射 (anonymous mapping)，若有則為檔案映射。若有任何參數是無效的，do_mmap 會回傳負值。否則會在虛擬記憶體中找到一個合適的範圍，如果可能的話會與相鄰的記憶體區合併再一起。否則從 slab 快取分配一個新的 vm_area_struct 結構．並通過 vma_link() 將這個新的記憶體區加入到鏈結串列和記憶體區的紅黑樹。

## munmap() and do_munmap(): Removing an Address Interval

do_munmap 函式會從所指定的行程位址空間移除一個位址範圍。
`int do_munmap(struct mm_struct *mm, unsigned long start, size_t len)`
核心提供用戶者空間 munmap() 系統呼叫，讓行程得以從自己的位址空間移除所指定的位址範圍。 （free?）

## Page Tables

Although applications operate on virtual memory mapped to physical addresses, processors operate directly on those physical addresses. Consequently, when an application accesses a virtual memory address, it must first be converted to a physical address before the processor can resolve the request. 使用者使用的是虛擬記憶體空間，而處理器需要的是實體記憶體，所以我們需要轉址。

In Linux, the page tables consist of three levels. PGD, PMD, and PTE. **The multiple levels enable a sparsely populated address space, even on 64-bit machines.**
![alt text](/image/fig15_1.png)


