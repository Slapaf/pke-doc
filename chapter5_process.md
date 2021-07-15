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

```
 34 process procs[NPROC];
```

实际上，这个进程池就是一个包含NPROC（=32，见kernel/process.h文件）个process结构的数组。

接下来，PKE操作系统对进程的结构进行了扩充（见kernel/process.h文件）：

```
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

```
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

```
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

```
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

```
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

```
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

```
 34 ssize_t sys_user_exit(uint64 code) {
 35   sprint("User exit with code:%d.\n", code);
 36   // in lab3 now, we should reclaim the current process, and reschedule.
 37   free_process( current );
 38   schedule();
 39   return 0;
 40 }
```

可以看到，如果某进程调用了exit()系统调用，操作系统的处理方法是调用free_process()函数，将当前进程（也就是调用者）进行“释放”，然后转进程调度。其中free_process()函数的实现非常简单：

```
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

```
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

```
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

```
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

以上程序



<a name="lab3_1_content"></a>

#### **实验内容**


<a name="lab3_1_guide"></a>

#### **实验指导**



<a name="lab3_2_yield"></a>
## 5.3 lab3_2 进程yield

<a name="lab3_2_app"></a>

#### **给定应用**

<a name="lab3_2_content"></a>

#### **实验内容**


<a name="lab3_2_guide"></a>

#### **实验指导**




<a name="lab3_3_rrsched"></a>
## 5.4 lab3_3 循环轮转调度

<a name="lab3_3_app"></a>

#### **给定应用**

<a name="lab3_3_content"></a>

#### **实验内容**


<a name="lab3_3_guide"></a>

#### **实验指导**



