# 第五章．实验3：进程管理

### 目录 

- [5.1 实验3的基础知识](#fundamental) 
  - [5.1.1 多任务环境下进程的封装](#subsec_process_structure)
  - [5.1.2 进程的换入与换出](#subsec_switch)
  - [5.1.3 就绪进程的管理与调度](#subsec_management)
- [5.2 lab3_1 进程创建（fork）](#lab3_1_naive_fork)
  - [给定应用](#lab3_1_app)
  - [实验内容](#lab3_1_content)
  - [实验指导](#lab3_1_guide)
- [5.3 lab3_2 进程yield](#lab3_2_yield) 
  - [给定应用](#lab3_2_app)
  - [实验内容](#lab3_2_content)
  - [实验指导](#lab3_2_guide)
- [5.4 lab3_3 循环轮转调度](#lab3_3_rrsched) 
  - [给定应用](#lab3_3_app)
  - [实验内容](#lab3_3_content)
  - [实验指导](#lab3_3_guide)

<a name="fundamental"></a>

## 5.1 实验3的基础知识

完成了实验1和实验2的读者，应该对PKE实验中的“进程”不会太陌生。因为实际上，我们从最开始的lab1_1开始就有了进程结构（struct process），只是在之前的实验中，进程结构中最重要的成员是trapframe和kstack，它们分别用来记录系统进入S模式前的进程上下文以及作为进入S模式后的操作系统栈。在实验3，我们将进入多任务环境，完成PKE实验环境下的进程创建、换入换出，以及进程调度相关实验。

<a name="subsec_process_structure"></a>

### 5.1.1 多任务环境下进程的封装

实验3跟之前的两个实验最大的不同，在于在实验3的3个基本实验中，PKE操作系统将需要支持多个进程的执行。为了对多任务环境进行支撑，PKE操作系统定义了一个“进程池”（见kernel/process.c文件）：

```C
 34 process procs[NPROC];
```

实际上，这个进程池就是一个包含NPROC（=32，见kernel/process.h文件）个process结构的数组。

接下来，PKE操作系统对进程的结构进行了扩充（见kernel/process.h文件）：

```C
 53   // points to a page that contains mapped_regions
 54   mapped_region *mapped_info;
 55   // next free mapped region in mapped_info
 56   int total_mapped_region;
 57
 58   // process id
 59   uint64 pid;
 60   // process status
 61   int status;
 62   // parent process
 63   struct process *parent;
 64   // next queue element
 65   struct process *queue_next;
 66
 67   // accounting
 68   int tick_count;
```

- 前两项mapped_info和total_mapped_region用于对进程的虚拟地址空间（中的代码段、堆栈段等）进行跟踪，这些虚拟地址空间在进程创建（fork）时，将发挥重要作用。同时，这也是lab3_1的内容。PKE将进程可能拥有的段分为以下几个类型：

```C
 29 enum segment_type {
 30   CODE_SEGMENT,    // ELF segment
 31   DATA_SEGMENT,    // ELF segment
 32   STACK_SEGMENT,   // runtime segment
 33   CONTEXT_SEGMENT, // trapframe segment
 34   SYSTEM_SEGMENT,  // system segment
 35 };
```

其中CODE_SEGMENT表示该段是从可执行ELF文件中加载的代码段，DATA_SEGMENT为从ELF文件中加载的数据段，STACK_SEGMENT为进程自身的栈段，CONTEXT_SEGMENT为保存进程上下文的trapframe所对应的段，SYSTEM_SEGMENT为进程的系统段，如所映射的异常处理段。

- pid是进程的ID号，具有唯一性；
- status记录了进程的状态，PKE操作系统在实验3给进程规定了以下几种状态：

```C
 20 enum proc_status {
 21   FREE,            // unused state
 22   READY,           // ready state
 23   RUNNING,         // currently running
 24   BLOCKED,         // waiting for something
 25   ZOMBIE,          // terminated but not reclaimed yet
 26 };
```

其中，FREE为自由态，表示进程结构可用；READY为就绪态，即进程所需的资源都已准备好，可以被调度执行；RUNNING表示该进程处于正在运行的状态；BLOCKED表示进程处于阻塞状态；ZOMBIE表示进程处于“僵尸”状态，进程的资源可以被释放和回收。

- parent用于记录进程的父进程；
- queue_next用于将进程链接进各类队列（比如就绪队列）；
- tick_count用于对进程进行记账，即记录它的执行经历了多少次的timer事件，将在lab3_3中实现循环轮转调度时使用。

<a name="subsec_switch"></a>

### 5.1.2 进程的启动与终止

PKE实验中，创建一个进程需要先调用kernel/process.c文件中的alloc_process()函数：

```C
 88 process* alloc_process() {
 89   // locate the first usable process structure
 90   int i;
 91
 92   for( i=0; i<NPROC; i++ )
 93     if( procs[i].status == FREE ) break;
 94
 95   if( i>=NPROC ){
 96     panic( "cannot find any free process structure.\n" );
 97     return 0;
 98   }
 99
100   // init proc[i]'s vm space
101   procs[i].trapframe = (trapframe *)alloc_page();  //trapframe, used to save context
102   memset(procs[i].trapframe, 0, sizeof(trapframe));
103
104   // page directory
105   procs[i].pagetable = (pagetable_t)alloc_page();
106   memset((void *)procs[i].pagetable, 0, PGSIZE);
107
108   procs[i].kstack = (uint64)alloc_page() + PGSIZE;   //user kernel stack top
109   uint64 user_stack = (uint64)alloc_page();       //phisical address of user stack bottom
110   procs[i].trapframe->regs.sp = USER_STACK_TOP;  //virtual address of user stack top
111
112   // allocates a page to record memory regions (segments)
113   procs[i].mapped_info = (mapped_region*)alloc_page();
114   memset( procs[i].mapped_info, 0, PGSIZE );
115
116   // map user stack in userspace
117   user_vm_map((pagetable_t)procs[i].pagetable, USER_STACK_TOP - PGSIZE, PGSIZE,
118     user_stack, prot_to_type(PROT_WRITE | PROT_READ, 1));
119   procs[i].mapped_info[0].va = USER_STACK_TOP - PGSIZE;
120   procs[i].mapped_info[0].npages = 1;
121   procs[i].mapped_info[0].seg_type = STACK_SEGMENT;
122
123   // map trapframe in user space (direct mapping as in kernel space).
124   user_vm_map((pagetable_t)procs[i].pagetable, (uint64)procs[i].trapframe, PGSIZE,
125     (uint64)procs[i].trapframe, prot_to_type(PROT_WRITE | PROT_READ, 0));
126   procs[i].mapped_info[1].va = (uint64)procs[i].trapframe;
127   procs[i].mapped_info[1].npages = 1;
128   procs[i].mapped_info[1].seg_type = CONTEXT_SEGMENT;
129
130   // map S-mode trap vector section in user space (direct mapping as in kernel space)
131   // we assume that the size of usertrap.S is smaller than a page.
132   user_vm_map((pagetable_t)procs[i].pagetable, (uint64)trap_sec_start, PGSIZE,
133     (uint64)trap_sec_start, prot_to_type(PROT_READ | PROT_EXEC, 0));
134   procs[i].mapped_info[2].va = (uint64)trap_sec_start;
135   procs[i].mapped_info[2].npages = 1;
136   procs[i].mapped_info[2].seg_type = SYSTEM_SEGMENT;
137
138   sprint("in alloc_proc. user frame 0x%lx, user stack 0x%lx, user kstack 0x%lx \n",
139     procs[i].trapframe, procs[i].trapframe->regs.sp, procs[i].kstack);
140
141   procs[i].total_mapped_region = 3;
142   // return after initialization.
143   return &procs[i];
144 }
```

通过以上代码，可以发现alloc_process()函数除了找到一个空的进程结构外，还为新创建的进程建立了KERN_BASE以上逻辑地址的映射（这段代码在实验3之前位于kernel/kernel.c文件的load_user_program()函数中），并将映射信息保存到了进程结构中。

对于给定应用，PKE将通过调用load_bincode_from_host_elf()函数载入给定应用对应的ELF文件的各个段。之后被调用的elf_load()函数在载入段后，将对被载入的段进行判断，以记录它们的虚地址映射：

```c
 62 elf_status elf_load(elf_ctx *ctx) {
 63   elf_prog_header ph_addr;
 64   int i, off;
 65   // traverse the elf program segment headers
 66   for (i = 0, off = ctx->ehdr.phoff; i < ctx->ehdr.phnum; i++, off += sizeof(ph_addr)) {
 67     // read segment headers
 68     if (elf_fpread(ctx, (void *)&ph_addr, sizeof(ph_addr), off) != sizeof(ph_addr)) return EL_EIO;
 69
 70     if (ph_addr.type != ELF_PROG_LOAD) continue;
 71     if (ph_addr.memsz < ph_addr.filesz) return EL_ERR;
 72     if (ph_addr.vaddr + ph_addr.memsz < ph_addr.vaddr) return EL_ERR;
 73
 74     // allocate memory before loading
 75     void *dest = elf_alloccb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);
 76
 77     // actual loading
 78     if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz)
 79       return EL_EIO;
 80
 81     // record the vm region in proc->mapped_info
 82     int j;
 83     for( j=0; j<PGSIZE/sizeof(mapped_region); j++ )
 84       if( (process*)(((elf_info*)(ctx->info))->p)->mapped_info[j].va == 0x0 ) break;
 85
 86     ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].va = ph_addr.vaddr;
 87     ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].npages = 1;
 88     if( ph_addr.flags == (SEGMENT_READABLE|SEGMENT_EXECUTABLE) ){
 89       ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].seg_type = CODE_SEGMENT;
 90       sprint( "CODE_SEGMENT added at mapped info offset:%d\n", j );
 91     }else if ( ph_addr.flags == (SEGMENT_READABLE|SEGMENT_WRITABLE) ){
 92       ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].seg_type = DATA_SEGMENT;
 93       sprint( "DATA_SEGMENT added at mapped info offset:%d\n", j );
 94     }else
 95       panic( "unknown program segment encountered, segment flag:%d.\n", ph_addr.flags );
 96
 97     ((process*)(((elf_info*)(ctx->info))->p))->total_mapped_region ++;
 98   }
 99
100   return EL_OK;
101 }
```

以上代码段中，第86--97行将对被载入的段的类型（ph_addr.flags）进行判断以确定它是代码段还是数据段。完成以上的虚地址空间到物理地址空间的映射后，将形成用户进程的虚地址空间结构（如[图4.5](chapter4_memory.md#user_vm_space)所示）。

接下来，将通过switch_to()函数将所构造的进程投入执行：

```c
 42 void switch_to(process *proc) {
 43   assert(proc);
 44   current = proc;
 45
 46   write_csr(stvec, (uint64)smode_trap_vector);
 47   // set up trapframe values that smode_trap_vector will need when
 48   // the process next re-enters the kernel.
 49   proc->trapframe->kernel_sp = proc->kstack;      // process's kernel stack
 50   proc->trapframe->kernel_satp = read_csr(satp);  // kernel page table
 51   proc->trapframe->kernel_trap = (uint64)smode_trap_handler;
 52
 53   // set up the registers that strap_vector.S's sret will use
 54   // to get to user space.
 55
 56   // set S Previous Privilege mode to User.
 57   unsigned long x = read_csr(sstatus);
 58   x &= ~SSTATUS_SPP;  // clear SPP to 0 for user mode
 59   x |= SSTATUS_SPIE;  // enable interrupts in user mode
 60
 61   write_csr(sstatus, x);
 62
 63   // set S Exception Program Counter to the saved user pc.
 64   write_csr(sepc, proc->trapframe->epc);
 65
 66   //make user page table
 67   uint64 user_satp = MAKE_SATP(proc->pagetable);
 68
 69   // switch to user mode with sret.
 70   return_to_user(proc->trapframe, user_satp);
 71 }
```

实际上，以上函数在[实验1](chapter3_traps.md)就有所涉及，它的作用是将进程结构中的trapframe作为进程上下文恢复到RISC-V机器的通用寄存器中，并最后调用sret指令（通过return_to_user()函数）将进程投入执行。

不同于实验1和实验2，实验3的exit系统调用不能够直接将系统shutdown，因为一个进程的结束并不一定意味着系统中所有进程的完成。以下是实验3中exit系统调用的实现：

```c
 34 ssize_t sys_user_exit(uint64 code) {
 35   sprint("User exit with code:%d.\n", code);
 36   // in lab3 now, we should reclaim the current process, and reschedule.
 37   free_process( current );
 38   schedule();
 39   return 0;
 40 }
```

可以看到，如果某进程调用了exit()系统调用，操作系统的处理方法是调用free_process()函数，将当前进程（也就是调用者）进行“释放”，然后转进程调度。其中free_process()函数的实现非常简单：

```c
149 int free_process( process* proc ) {
150   // we set the status to ZOMBIE, but cannot destruct its vm space immediately.
151   // since proc can be current process, and its user kernel stack is currently in use!
152   // but for proxy kernel, it (memory leaking) may NOT be a really serious issue,
153   // as it is different from regular OS, which needs to run 7x24.
154   proc->status = ZOMBIE;
155
156   return 0;
157 }
```

可以看到，**free_process()函数仅是将进程设为ZOMBIE状态，而不会将进程所占用的资源全部释放**！这是因为free_process()函数的调用，说明操作系统当前是在S模式下运行，而按照PKE的设计思想，S态的运行将使用当前进程的用户系统栈（user kernel stack）。此时，如果将当前进程的内存空间进行释放，将导致操作系统本身的崩溃。所以释放进程时，PKE采用的是折衷的办法，即只将其设置为僵尸（ZOMBIE）状态，而不是立即将它所占用的资源进行释放。最后，schedule()函数的调用，将选择系统中可能存在的其他处于就绪状态的进程投入运行，它的处理逻辑我们将在下一节讨论。



<a name="subsec_management"></a>

### 5.1.3 就绪进程的管理与调度

PKE的操作系统设计了一个非常简单的就绪队列管理（因为实验3的基础实验并未涉及进程的阻塞，所以未设计阻塞队列），队列头在kernel/sched.c文件中定义：

```
8 process* ready_queue_head = NULL;
```

将一个进程加入就绪队列，可以调用insert_to_ready_queue()函数：

```c
 13 void insert_to_ready_queue( process* proc ) {
 14   sprint( "going to insert process %d to ready queue.\n", proc->pid );
 15   // if the queue is empty in the beginning
 16   if( ready_queue_head == NULL ){
 17     proc->status = READY;
 18     proc->queue_next = NULL;
 19     ready_queue_head = proc;
 20     return;
 21   }
 22
 23   // ready queue is not empty
 24   process *p;
 25   // browse the ready queue to see if proc is already in-queue
 26   for( p=ready_queue_head; p->queue_next!=NULL; p=p->queue_next )
 27     if( p == proc ) return;  //already in queue
 28
 29   // p points to the last element of the ready queue
 30   if( p==proc ) return;
 31   p->queue_next = proc;
 32   proc->status = READY;
 33   proc->queue_next = NULL;
 34
 35   return;
 36 }
```

该函数首先（第16--21行）处理ready_queue_head为空（初始状态）的情况，如果就绪队列不为空，则将进程加入到队尾（第26--33行）。

PKE操作系统内核通过调用schedule()函数来完成进程的选择和换入：

```c
 45 void schedule() {
 46   if ( !ready_queue_head ){
 47     // by default, if there are no ready process, and all processes are in the status of
 48     // FREE and ZOMBIE, we should shutdown the emulated RISC-V machine.
 49     int should_shutdown = 1;
 50
 51     for( int i=0; i<NPROC; i++ )
 52       if( (procs[i].status != FREE) && (procs[i].status != ZOMBIE) ){
 53         should_shutdown = 0;
 54         sprint( "ready queue empty, but process %d is not in free/zombie state:%d\n",
 55           i, procs[i].status );
 56       }
 57
 58     if( should_shutdown ){
 59       sprint( "no more ready processes, system shutdown now.\n" );
 60       shutdown( 0 );
 61     }else{
 62       panic( "Not handled: we should let system wait for unfinished processes.\n" );
 63     }
 64   }
 65
 66   current = ready_queue_head;
 67   assert( current->status == READY );
 68   ready_queue_head = ready_queue_head->queue_next;
 69
 70   current->status == RUNNING;
 71   sprint( "going to schedule process %d to run.\n", current->pid );
 72   switch_to( current );
 73 }
```

可以看到，schedule()函数首先判断就绪队列ready_queue_head是否为空，对于为空的情况（第46--64行），schedule()函数将判断系统中所有的进程是否全部都处于被释放（FREE）状态，或者僵尸（ZOMBIE）状态。如果是，则启动关（模拟RISC-V）机程序，否则应进入等待系统中进程结束的状态。但是，由于实验3的基础实验并无可能进入这样的状态，所以我们在这里调用了panic，等后续实验有可能进入这种状态后再进一步处理。

对于就绪队列非空的情况（第66--72行），处理就简单得多：只需要将就绪队列队首的进程换入执行即可。对于换入的过程，需要注意的是，要将被选中的进程从就绪队列中摘掉。



<a name="lab3_1_naive_fork"></a>

## 5.2 lab3_1 进程创建（fork）

<a name="lab3_1_app"></a>

#### **给定应用**

- user/app_naive_fork.c

```c
  1 /*
  2  * Below is the given application for lab3_1.
  3  * It forks a child process to run .
  4  * Parent process will continue to run after child exits
  5  * So it is a naive "fork". we will implement a better one in later lab.
  6  *
  7  */
  8
  9 #include "user/user_lib.h"
 10 #include "util/types.h"
 11
 12 int main(void) {
 13   uint64 pid = fork();
 14   if (pid == 0) {
 15     printu("Child: Hello world!\n");
 16   } else {
 17     printu("Parent: Hello world! child id %ld\n", pid);
 18   }
 19
 20   exit(0);
 21 }
```

以上程序的行为非常简单：主进程调用fork()函数，后者产生一个系统调用，基于主进程这个模板创建它的子进程。

- 切换到lab3_1，继承lab2_3及之前实验所做的修改，并make后的直接运行结果：

```
//切换到lab3_1
$ git checkout lab3_1_fork

//继承lab2_3以及之前的答案
$ git merge lab2_3_pagefault -m "continue to work on lab3_1"

//重新构造
$ make clean; make

//运行构造结果
$ spike ./obj/riscv-pke ./obj/app_naive_fork
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
Switching to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
User application is loading.
Application: ./obj/app_naive_fork
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x0000000000010078
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087faf000, user stack 0x000000007ffff000, user kstack 0x0000000087fae000
You need to implement the code segment mapping of child in lab3_1.

System is shutting down with exit code -1.
```

从以上运行结果来看，应用程序的fork动作并未将子进程给创建出来并投入运行。按照提示，我们需要在PKE操作系统内核中实现子进程到父进程代码段的映射，以最终完成fork动作。

这里，既然涉及到了父进程的代码段，我们就可以先用readelf命令查看一下给定应用程序的可执行代码对应的ELF文件结构：

```
$ riscv64-unknown-elf-readelf -l ./obj/app_naive_fork

Elf file type is EXEC (Executable file)
Entry point 0x10078
There is 1 program header, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000010000 0x0000000000010000
                 0x000000000000040c 0x000000000000040c  R E    0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .text .rodata
```

可以看到，app_naive_fork可执行文件只包含一个代码段（编号为00），应该来讲是最简单的可执行文件结构了（无须考虑数据段的问题）。如果要依据这样的父进程模板创建子进程，只需要将它的代码段映射（而非拷贝）到子进程的对应虚地址即可。

<a name="lab3_1_content"></a>

#### **实验内容**

完善操作系统内核kernel/process.c文件中的do_fork()函数，并最终获得以下预期结果：

```
$ spike ./obj/riscv-pke ./obj/app_naive_fork
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
Switching to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
User application is loading.
Application: ./obj/app_naive_fork
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x0000000000010078
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087faf000, user stack 0x000000007ffff000, user kstack 0x0000000087fae000
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
Parent: Hello world! child id 1
User exit with code:0.
going to schedule process 1 to run.
Child: Hello world!
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

从以上运行结果来看，子进程已经被创建，且在其后被投入运行。

<a name="lab3_1_guide"></a>

#### **实验指导**

读者可以回顾[lab1_1](chapter3_traps.md#syscall)中所学习到的系统调用的知识，从应用程序（user/app_naive_fork.c）开始，跟踪fork()函数的实现：

user/app_naive_fork.c --> user/user_lib.c --> kernel/strap_vector.S --> kernel/strap.c --> kernel/syscall.c

直至跟踪到kernel/process.c文件中的do_fork()函数：

```c
166 int do_fork( process* parent)
167 {
168   sprint( "will fork a child from parent %d.\n", parent->pid );
169   process* child = alloc_process();
170
171   for( int i=0; i<parent->total_mapped_region; i++ ){
172     // browse parent's vm space, and copy its trapframe and data segments,
173     // map its code segment.
174     switch( parent->mapped_info[i].seg_type ){
175       case CONTEXT_SEGMENT:
176         *child->trapframe = *parent->trapframe;
177         break;
178       case STACK_SEGMENT:
179         memcpy( (void*)lookup_pa(child->pagetable, child->mapped_info[0].va),
180           (void*)lookup_pa(parent->pagetable, parent->mapped_info[i].va), PGSIZE );
181         break;
182       case CODE_SEGMENT:
183         // TODO: implment the mapping of child code segment to parent's code segment.
184         // hint: the virtual address mapping of code segment is tracked in mapped_info
185         // page of parent's process structure. use the information in mapped_info to
186         // retrieve the virtual to physical mapping of code segment.
187         // after having the mapping information, just map the corresponding virtual
188         // address region of child to the physical pages that actually store the code
189         // segment of parent process.
190         // DO NOT COPY THE PHYSICAL PAGES, JUST MAP THEM.
191         panic( "You need to implement the code segment mapping of child in lab3_1.\n" );
192
193         // after mapping, register the vm region (do not delete codes below!)
194         child->mapped_info[child->total_mapped_region].va = parent->mapped_info[i].va;
195         child->mapped_info[child->total_mapped_region].npages =
196           parent->mapped_info[i].npages;
197         child->mapped_info[child->total_mapped_region].seg_type = CODE_SEGMENT;
198         child->total_mapped_region++;
199         break;
200     }
201   }
202
203   child->status = READY;
204   child->trapframe->regs.a0 = 0;
205   child->parent = parent;
206   insert_to_ready_queue( child );
207
208   return child->pid;
209 }
```

该函数使用第171--201行的循环来拷贝父进程的逻辑地址空间到其子进程。我们看到，对于trapframe段（case CONTEXT_SEGMENT）以及堆栈段（case CODE_SEGMENT），do_fork()函数采用了简单复制的办法来拷贝父进程的这两个段到子进程中，这样做的目的是将父进程的执行现场传递给子进程。

然而，对于父进程的代码段，子进程应该如何“继承”呢？通过第184--189行的注释，我们知道对于代码段，我们不应直接复制（减少系统开销），而应通过映射的办法，将子进程中对应的逻辑地址空间映射到其父进程中装载代码段的物理页面。这里，就要回到[实验2内存管理](chapter4_memory.md#pagetablecook)部分，寻找合适的函数来实现了。

<a name="lab3_2_yield"></a>

## 5.3 lab3_2 进程yield

<a name="lab3_2_app"></a>

#### **给定应用**

- user/app_yield.c

```C
  1 /*
  2  * This app fork a child process to run.
  3  * In loops of child process and child process, they give up cpu
  4  * so that the other one can have some cpu time to run.
  5  */
  6
  7 #include "user/user_lib.h"
  8 #include "util/types.h"
  9
 10 int main(void) {
 11   uint64 pid = fork();
 12   uint64 rounds = 0xffff;
 13   if (pid == 0) {
 14     printu("Child: Hello world! \n");
 15     for (uint64 i = 0; i < rounds; ++i) {
 16       if (i % 10000 == 0) {
 17         printu("Child running %ld \n", i);
 18         yield();
 19       }
 20     }
 21   } else {
 22     printu("Parent: Hello world! \n");
 23     for (uint64 i = 0; i < rounds; ++i) {
 24       if (i % 10000 == 0) {
 25         printu("Parent running %ld \n", i);
 26         yield();
 27       }
 28     }
 29   }
 30
 31   exit(0);
 32   return 0;
 33 }
```

和lab3_1一样，以上的应用程序通过fork系统调用创建了一个子进程，接下来，父进程和子进程都进入了一个很长的循环。在循环中，无论是父进程还是子进程，在循环的次数是10000的整数倍时，除了打印信息外都调用了yield()函数，来释放自己的执行权（即CPU）。

- 切换到lab3_2，继承lab3_1及之前实验所做的修改，并make后的直接运行结果：

```bash
//切换到lab3_2
$ git checkout lab3_2_yield

//继承lab3_1以及之前的答案
$ git merge lab3_1_fork -m "continue to work on lab3_2"

//重新构造
$ make clean; make

//运行构造结果
$ spike ./obj/riscv-pke ./obj/app_yield
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
Switching to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
User application is loading.
Application: ./obj/app_yield
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x000000000001017c
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087faf000, user stack 0x000000007ffff000, user kstack 0x0000000087fae000
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
Parent: Hello world!
Parent running 0
You need to implement the yield syscall in lab3_2.

System is shutting down with exit code -1.
```

从以上输出来看，还是因为PKE操作系统中的yield()功能未完善，导致应用无法正常执行下去。

<a name="lab3_2_content"></a>

#### **实验内容**

完善yield系统调用，实现进程执行过程中的主动释放CPU的动作。实验完成后，获得以下预期结果：

```bash
$ spike ./obj/riscv-pke ./obj/app_yield
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
Switching to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
User application is loading.
Application: ./obj/app_yield
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x000000000001017c
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087faf000, user stack 0x000000007ffff000, user kstack 0x0000000087fae000
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
Parent: Hello world!
Parent running 0
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child: Hello world!
Child running 0
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 10000
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 10000
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 20000
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 20000
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 30000
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 30000
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 40000
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 40000
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 50000
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 50000
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 60000
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 60000
going to insert process 1 to ready queue.
going to schedule process 0 to run.
User exit with code:0.
going to schedule process 1 to run.
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab3_2_guide"></a>

#### **实验指导**

进程释放CPU的动作应该是：

- 将当前进程置为就绪状态（READY）；
- 将当前进程加入到就绪队列的队尾；
- 转进程调度。



<a name="lab3_3_rrsched"></a>

## 5.4 lab3_3 循环轮转调度

<a name="lab3_3_app"></a>

#### **给定应用**

```C
  1 /*
  2  * This app fork a child process to run.
  3  * Loops in parent process and child process both can have
  4  * cpu time to run because the kernel will yield when timer interrupt is triggered.
  5  */
  6
  7 #include "user/user_lib.h"
  8 #include "util/types.h"
  9
 10 int main(void) {
 11   uint64 pid = fork();
 12   uint64 rounds = 100000000;
 13   uint64 interval = 10000000;
 14   uint64 a = 0;
 15   if (pid == 0) {
 16     printu("Child: Hello world! \n");
 17     for (uint64 i = 0; i < rounds; ++i) {
 18       if (i % interval == 0) printu("Child running %ld \n", i);
 19     }
 20   } else {
 21     printu("Parent: Hello world! \n");
 22     for (uint64 i = 0; i < rounds; ++i) {
 23       if (i % interval == 0) printu("Parent running %ld \n", i);
 24     }
 25   }
 26
 27   exit(0);
 28   return 0;
 29 }
```

和lab3_2类似，lab3_3给出的应用仍然是父子两个进程，他们的执行体都是两个大循环。但与lab3_2不同的是，这两个进程在执行各自循环体时，都没有主动释放CPU的动作。显然，这样的设计会导致某个进程长期占据CPU，而另一个进程无法得到执行。

- 切换到lab3_3，继承lab3_2及之前实验所做的修改，并make后的直接运行结果：

```bash
//切换到lab3_3
$ git checkout lab3_3_rrsched

//继承lab3_2以及之前的答案
$ git merge lab3_2_yield -m "continue to work on lab3_3"

//重新构造
$ make clean; make

//运行构造结果
$ spike ./obj/riscv-pke ./obj/app_two_long_loops
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
Switching to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
User application is loading.
Application: ./obj/app_two_long_loops
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x000000000001017c
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087faf000, user stack 0x000000007ffff000, user kstack 0x0000000087fae000
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
Parent: Hello world!
Parent running 0
Parent running 10000000
Ticks 0
You need to further implement the timer handling in lab3_3.

System is shutting down with exit code -1.
```

回顾实验1的[lab1_3](chapter3_traps.md#irq)，我们看到由于进程的执行体很长，执行过程中时钟中断被触发（输出中的“Ticks 0”）。显然，我们可以通过利用时钟中断来实现进程的循环轮转调度，避免由于一个进程的执行体过长，导致系统中其他进程无法得到调度的问题！

<a name="lab3_3_content"></a>

#### **实验内容**

实现kernel/strap.c文件中的rrsched()函数，获得以下预期结果：

```bash
$ spike ./obj/riscv-pke ./obj/app_two_long_loops
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x0000000087ffffff]
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on
Switching to user mode...
in alloc_proc. user frame 0x0000000087fbc000, user stack 0x000000007ffff000, user kstack 0x0000000087fbb000
User application is loading.
Application: ./obj/app_two_long_loops
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x000000000001017c
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User call fork.
will fork a child from parent 0.
in alloc_proc. user frame 0x0000000087faf000, user stack 0x000000007ffff000, user kstack 0x0000000087fae000
do_fork map code segment at pa:0000000087fb2000 of parent to child at va:0000000000010000.
going to insert process 1 to ready queue.
Parent: Hello world!
Parent running 0
Parent running 10000000
Ticks 0
Parent running 20000000
Ticks 1
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child: Hello world!
Child running 0
Child running 10000000
Ticks 2
Child running 20000000
Ticks 3
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 30000000
Ticks 4
Parent running 40000000
Ticks 5
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 30000000
Ticks 6
Child running 40000000
Ticks 7
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 50000000
Parent running 60000000
Ticks 8
Parent running 70000000
Ticks 9
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 50000000
Child running 60000000
Ticks 10
Child running 70000000
Ticks 11
going to insert process 1 to ready queue.
going to schedule process 0 to run.
Parent running 80000000
Ticks 12
Parent running 90000000
Ticks 13
going to insert process 0 to ready queue.
going to schedule process 1 to run.
Child running 80000000
Ticks 14
Child running 90000000
Ticks 15
going to insert process 1 to ready queue.
going to schedule process 0 to run.
User exit with code:0.
going to schedule process 1 to run.
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab3_3_guide"></a>

#### **实验指导**

实际上，如果单纯为了实现进程的轮转，避免单个进程长期霸占CPU的情况，只需要简单地在时钟中断被触发时做重新调度即可。然而，为了实现时间片的概念，以及控制进程在单时间片内获得的执行长度，我们在kernel/sched.h文件中定义了“时间片”的长度：

```C
  6 //length of a time slice, in number of ticks
  7 #define TIME_SLICE_LEN  2
```

可以看到时间片的长度（TIME_SLICE_LEN）为2个ticks，这就意味着我们要每隔两个ticks触发一次进程重新调度动作。

为配合调度的实现，我们在进程结构中定义了整型成员（参见[5.1.1](#subsec_process_structure)）tick_count，完善kernel/strap.c文件中的rrsched()函数，以实现循环轮转调度时，应采取的逻辑为：

- 判断当前进程的tick_count加1后是否大于等于TIME_SLICE_LEN？
- 若答案为yes，则应将当前进程的tick_count清零，并将当前进程加入就绪队列，转进程调度；
- 若答案为no，则应将当前进程的tick_count加1，并返回。



