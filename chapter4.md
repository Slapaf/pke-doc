## 第四章．（实验3）物理内存管理

### 4.1 实验内容

实验要求：了解物理内存，管理物理内存。 

**4.1.1 练习一：OS内存的初始化过程**

在"pk/mmap.c"内有 pk_vm_init()函数，阅读该函数，了解OS内存初始化的过程。

```
364  uintptr_t pk_vm_init()
365  {
366      // HTIF address signedness and va2pa macro both cap memory size to 2 GiB
         //设置物理内存大小
367      mem_size = MIN(mem_size, 1U << 31);
              //计算物理页的数量
368      size_t mem_pages = mem_size >> RISCV_PGSHIFT;
369      free_pages = MAX(8, mem_pages >> (RISCV_PGLEVEL_BITS-1));
370          
              //_end为内核结束地址
371      extern char _end;
372      first_free_page = ROUNDUP((uintptr_t)&_end, RISCV_PGSIZE);
373      first_free_paddr = first_free_page + free_pages * RISCV_PGSIZE;
374              
       //映射内核的物理空间
375      root_page_table = (void*)__page_alloc();
376      __map_kernel_range(DRAM_BASE, DRAM_BASE, first_free_paddr - DRAM_BASE, PROT_READ|PROT_WRITE|PROT_EXEC);
377              
       //crrent.mmap_max: 0x000000007f7ea000
378      current.mmap_max = current.brk_max =
379       MIN(DRAM_BASE, mem_size - (first_free_paddr - DRAM_BASE));
380          
              //映射用户栈
381      size_t stack_size = MIN(mem_pages >> 5, 2048) * RISCV_PGSIZE;
382      size_t stack_bottom = __do_mmap(current.mmap_max - stack_size, stack_size, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS|MAP_FIXED , 0, 0);
383      kassert(stack_bottom != (uintptr_t)-1);
384      current.stack_top = stack_bottom + stack_size;
385
              //开启分页
386      flush_tlb();
387      write_csr(sptbr, ((uintptr_t)root_page_table >> RISCV_PGSHIFT) | SATP_MODE_CHOICE);
388
              //分配内核栈空间，
389      uintptr_t kernel_stack_top = __page_alloc() + RISCV_PGSIZE;
390      return kernel_stack_top;
391  }
```

以上代码中，我们给出了大体的注释，请根据以上代码，读者可以尝试画一下PK的逻辑地址空间结构图，以及逻辑地址空间到物理地址空间的映射关系。

**4.1.2 练习二：first_fit内存页分配算法（需要编程）**

在"pk/pmm.c" 中，我们实现了对物理内存的管理。

构建了物理内存页管理器框架：struct pmm_manager结构如下：

```
135     const struct pmm_manager default_pmm_manager = {
136       .name = "default_pmm_manager",
137       .init = default_init,
138       .init_memmap = default_init_memmap,
139       .alloc_pages = default_alloc_pages,
140       .free_pages = default_free_pages,
141       .nr_free_pages = default_nr_free_pages,
142       .pmm_check = basic_check,
143     };
```

默认的内存管理器有如下属性：

l name:内存管理器的名字

l init:对内存管理算法所使用的数据结构进行初始化

l init_ memmap:根据物理内存设置内存管理算法的数据结构

l alloc_pages：分配物理页

l free_pages：释放物理页

l nr_free_pages：空闲物理页的数量

l pmm_check ：检查校验函数

参考已经实现的函数，完成default_alloc_pages()和default_free_pages(),实现first_fit内存页分配算法。

first_fit分配算法需要维护一个查找有序（地址按从小到大排列）空闲块（以页为最小单位的连续地址空间）的数据结构，而双向链表是一个很好的选择。pk/list.h定义了可挂接任意元素的通用双向链表结构和对应的操作，所以需要了解如何使用这个文件提供的各种函数，从而可以完成对双向链表的初始化/插入/删除等。

​    

你可以使用python脚本检查你的输出：

`./pke-lab3`

若得到如下输出，你就已经成功完成了实验三！！！

```
build pk : OK
running app3 m2048 : OK
 test3_m2048 : OK
running app3 m1024 : OK
 test3_m1024 : OK
Score: 20/20 
```

### 4.2 基础知识

**4.2.1 物理内存空间与编址**

计算机的存储结构可以抽象的看做由N个连续的字节组成的数组。想一想，在数组中我们如何找到一个元素？对了！是下标！！那么我们如何在内存中如何找打一个元素呢？自然也是‘下标’。这个下标的起始位置和位数由机器本身决定，我们称之为“物理地址”。

至于物理内存的大小，由于我们的RISC-V目标机（也就是我们的pke以及app运行的环境，这里我们假设目标机为64位机，即用到了56位的物理内存编址，虚拟地址采用Sv39方案，参见[第一章RISC-V体系结构的内容](chapter1)）是由spike模拟器构造的，构造过程中可以通过命令行的-m选项来指定物理内存的大小。而且，spike会将目标机的物理内存地址从0x8000-0000开始编制。例如，如果物理内存空间的大小为2GB（spike的默认值），则目标机的物理地址范围为：[0x8000-0000, 0x10000-0000]，其中0x10000-0000已经超过32位能够表达的范围了，但是我们目标机是64位机！再例如，如果目标机物理内存空间大小为1GB（启动spike时带入-m1024m参数），则目标机的物理地址范围为：[0x8000-0000, 0xC000-0000]。在以下的讨论中，我们用符号PHYMEM_TOP代表物理内存空间的高地址部分，在以上的两个例子中，PHYMEM_TOP分别为0x10000-0000和0xC000-0000。在定义了PHYMEM_TOP符号后，物理内存的范围就可以表示为[0x8000-0000, PHYMEM_TOP]。

我们的PK内核的逻辑编址，可以通过查看pke.lds得知，pke.lds有以下规则：

```
14  /* Begining of code and text segment */
15  . = 0x80000000;
```

可见，PK内核的逻辑地址的起始也是0x8000-0000！这也就意味着PK内核实际上采用的是直接地址映射的办法保证在未打开分页情况下，逻辑地址到物理地址的映射的。代理内核的本质也是一段程序，他本身是需要内存空间的，而这一段空间在PK的设计中是静态分配给内核使用的，不能被再分配给任何应用。那么静态分配给代理内核的内存空间具体是哪一段内存区域呢？

通过阅读PK的代码，我们可知PK内核占据了以下这一段：

```
   KERNTOP------->+---------------------------------+ 0x80816000
(first_free_paddr)|                                 |
                  |        Kern Physical Memory     | 
                  |                                 | 8M 2048pages 
 (first_free_page)|                                 |
   DRAM_BASE----> +---------------------------------+ 0x80016000
                  |        Kern Text/Data/BBS       |
       KERN------>+---------------------------------+ 0x80000000
```

也就是说，[0x8000-0000, 0x8081-6000]这段物理内存空间是被PK内核所“保留”的，余下的物理内存空间为[0x8081-6000，PHYMEM_TOP]，也就是下图中的Empty Memory（*）部分，这部分内存将会是我们的操作系统需要真正用于动态分配（给应用程序）的空间，**而本实验就是要管理这部分物理内存空间**。

```
 PHYMEM_TOP ----> +-------------------------------------------------+
 				  |                                                 |
                  |           Empty Memory (*)                      |
                  |                                                 |
 KERNTOP     ---> +-------------------------------------------------+ 0x80816000 
(first_free_paddr)|                                                 |
                  |            PK kernel resevered                  |
                  |                                                 |
                  |                                                 |
 KERN       ----> +-------------------------------------------------+ 0x80000000
```

最后，我们来看物理内存分配的单位：操作系统中，物理页是物理内存分配的基本单位。一个物理页的大小是4KB，我们使用结构体Page来表示，其结构如图：

```
struct Page {
  sint_t ref;          
  uint_t flags;        
  uint_t property;     
  list_entry_t page_link;   
};
```

l ref表示这样页被页表的引用记数

l flags表示此物理页的状态标记

l property用来记录某连续内存空闲块的大小（即地址连续的空闲页的个数）

l page_link是维持空闲物理页链表的重要结构。

Page结构体对应着物理页，我们来看Page结构体同物理地址之间是如何转换的。首先，我们需要先了解一下物理地址。

<img src="pictures/fig4_1.png" alt="fig4_1" style="zoom:80%;" />

图4.1 RISCV64 物理地址

总的来说，物理地址分为两部分：页号（PPN）和offset

页号可以理解为物理页的编码，而offset则为页内偏移量。现在考虑一下12位的offset对应的内存大小是多少呢？

2<<12=4096也就是4KB，还记得我们讲过的物PA理页大小是多少吗？没错是4KB。12位的offset设计便是由此而来。

有了物理地址（PA）这一概念，那PA和Pages结构体又是如何转换？

实际上在初始化空闲页链表之前，系统会定义一个Page结构体的数组，而链表的节点也正是来自于这些数组，这个数组的每一项代表着一个物理页，而且它们的数组下标就代表着每一项具体代表的是哪一个物理页，就如下图所示：

 <img src="pictures/fig4_2.png" alt="fig4_2" style="zoom:80%;" />


**3.2.2** **中断的处理过程**

当程序执行到中断之前，程序是有自己的运行状态的，例如寄存器里保持的上下文数据。当中断发生，硬件在自动设置完中断原因和中断地址后，就会调转到中断处理程序，而中断处理程序同样会使用寄存器，于是当程序从中断处理程序返回时需要保存需要被调用者保存的寄存器，我们称之为callee-saved寄存器。

在PK的machine/minit.c中间中，便通过delegate_traps()，将部分中断及同步异常委托给S模式。（同学们可以查看具体是哪些中断及同步异常）

```
 43     // send S-mode interrupts and most exceptions straight to S-mode
 44     static void delegate_traps()
 45     {
 46      if (!supports_extension('S'))
 47       return;
 48
 49      uintptr_t interrupts = MIP_SSIP | MIP_STIP | MIP_SEIP;
 50      uintptr_t exceptions =
 51       (1U << CAUSE_MISALIGNED_FETCH) |
 52       (1U << CAUSE_FETCH_PAGE_FAULT) |
 53       (1U << CAUSE_BREAKPOINT) |
 54       (1U << CAUSE_LOAD_PAGE_FAULT) |
 55       (1U << CAUSE_STORE_PAGE_FAULT) |
 56       (1U << CAUSE_USER_ECALL);
 57
 58      write_csr(mideleg, interrupts);
 59      write_csr(medeleg, exceptions);
 60      assert(read_csr(mideleg) == interrupts);
 61      assert(read_csr(medeleg) == exceptions);
 62     }
```

​这里介绍一下RISCV的中断委托机制，在默认的情况下，所有的异常都会被交由机器模式处理。但正如我们知道的那样，大部分的系统调用都是在S模式下处理的，因此RISCV提供了这一委托机制，可以选择性的将中断交由S模式处理，从而完全绕过M模式。

​接下，我们继续看S模式下的中断处理。在pk目录下的pk.c文件中的boot_loader函数中将&trap_entry写入了stvec寄存器中，stvec保存着发生异常时处理器需要跳转到的地址，也就是说当中断发生，我们将跳转至trap_entry，现在我们继续跟踪trap_entry。trap_entry在pk目录下的entry.S中，其代码如下：

```
 60     trap_entry:
 61      csrrw sp, sscratch, sp
 62      bnez sp, 1f
 63      csrr sp, sscratch
 64     1:addi sp,sp,-320
 65      save_tf
 66      move a0,sp
 67      jal handle_trap
```

​在61行，交换了sp与sscratch的值，这里是为了根据sscratch的值判断该中断是来源于U模式还是S模式。

​如果sp也就是传入的sscratch值不为零，则跳转至64行，若sscratch的值为零，则恢复原sp中的值。这是因为，当中断来源于S模式是，sscratch的值为0，sp中存储的就是内核的堆栈地址。而当中断来源于U模式时，sp中存储的是用户的堆栈地址，sscratch中存储的则是内核的堆栈地址，需要交换二者，是sp指向内核的堆栈地址。

​接着在64,65行保存上下文，最后跳转至67行处理trap。handle_trap在pk目录下的handlers.c文件中，代码如下：

```
112     void handle_trap(trapframe_t* tf)
113     {
114      if ((intptr_t)tf->cause < 0)
115       return handle_interrupt(tf);
116
117      typedef void (*trap_handler)(trapframe_t*);
118
119      const static trap_handler trap_handlers[] = {
120       [CAUSE_MISALIGNED_FETCH] = handle_misaligned_fetch,
121       [CAUSE_FETCH_ACCESS] = handle_instruction_access_fault,
122       [CAUSE_LOAD_ACCESS] = handle_load_access_fault,
123       [CAUSE_STORE_ACCESS] = handle_store_access_fault,
124       [CAUSE_FETCH_PAGE_FAULT] = handle_fault_fetch,
125       [CAUSE_ILLEGAL_INSTRUCTION] = handle_illegal_instruction,
126       [CAUSE_USER_ECALL] = handle_syscall,
127       [CAUSE_BREAKPOINT] = handle_breakpoint,
128       [CAUSE_MISALIGNED_LOAD] = handle_misaligned_load,
129       [CAUSE_MISALIGNED_STORE] = handle_misaligned_store,
130       [CAUSE_LOAD_PAGE_FAULT] = handle_fault_load,
131       [CAUSE_STORE_PAGE_FAULT] = handle_fault_store,
132      };
```

​handle_trap函数中实现了S模式下各类中断的处理。可以看到，代码的126行就对应着系统调用的处理，handle_syscall的实现如下：

```
100     static void handle_syscall(trapframe_t* tf)
101     {
102      tf->gpr[10] = do_syscall(tf->gpr[10], tf->gpr[11], tf->gpr[12], tf->gpr[13],
103                  tf->gpr[14], tf->gpr[15], tf->gpr[17]);
104      tf->epc += 4;
105     }
```

​还记得我们在例3.1中是将中断号写入x17寄存器嘛？其对应的就是这里do_syscall的最后一个参数，我们跟踪进入do_syscall函数，其代码如下：

```
313 long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, unsigned long n)
314     {
315      const static void* syscall_table[] = {
316      // your code here:
317      // add get_init_memsize syscall
318       [SYS_init_memsize ] = sys_get_init_memsize,
319       [SYS_exit] = sys_exit,
320       [SYS_exit_group] = sys_exit,
321       [SYS_read] = sys_read,
322       [SYS_pread] = sys_pread,
323       [SYS_write] = sys_write,
324       [SYS_openat] = sys_openat,
325       [SYS_close] = sys_close,
326       [SYS_fstat] = sys_fstat,
327       [SYS_lseek] = sys_lseek,
328       [SYS_renameat] = sys_renameat,
329       [SYS_mkdirat] = sys_mkdirat,
330       [SYS_getcwd] = sys_getcwd,
331       [SYS_brk] = sys_brk,
332       [SYS_uname] = sys_uname,
333       [SYS_prlimit64] = sys_stub_nosys,
334       [SYS_rt_sigaction] = sys_rt_sigaction,
335       [SYS_times] = sys_times,
336       [SYS_writev] = sys_writev,
337       [SYS_readlinkat] = sys_stub_nosys,
338       [SYS_rt_sigprocmask] = sys_stub_success,
339       [SYS_ioctl] = sys_stub_nosys,
340       [SYS_getrusage] = sys_stub_nosys,
341       [SYS_getrlimit] = sys_stub_nosys,
342       [SYS_setrlimit] = sys_stub_nosys,
343       [SYS_set_tid_address] = sys_stub_nosys,
344       [SYS_set_robust_list] = sys_stub_nosys,
345      };
346
347      syscall_t f = 0;
348
349      if (n < ARRAY_SIZE(syscall_table))
350       f = syscall_table[n];
351      if (!f)
352       panic("bad syscall #%ld!",n);
353
354      return f(a0, a1, a2, a3, a4, a5, n);
355     }
```

​do_syscall中通过传入的系统调用号n，查询syscall_table得到对应的函数，并最终执行系统调用。
