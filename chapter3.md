## 第三章．（实验2）系统调用的实现

### 3.1  实验环境搭建

实验2需要在实验1的基础之上完成，所以首先你需要切换到lab2_small的branch，然后commit你的代码。

首先，查看本地拥有的branch，输入如下命令：

`$ git branch`

如果有如下输出：

```
lab1_small
lab2_small
lab3_small
lab4_small
lab5_small
```

则你本地就有实验二的branch，那么可以直接切换分支到lab2，输入如下命令：

`$ git checkout lab2_small`

当然，如果本地没有lab2的相关代码，也可以直接从远端拉取：

`$ git checkout -b lab2_small origin/ lab2_small`

然后merge实验一中你的代码：

`$ git merge -m "merge lab1" lab1_small`

完成一切后，我们就可以正式进入实验二了！

### 3.2 实验内容

实验要求：了解系统调用的执行过程，并实现一个自定义的系统调用。

**3.2.1 练习一：在app中使用系统调用**

系统调用的英文名字是System Call，用户通过系统调用来执行一些需要特权的任务，那么我们具体在app中是如何使用内核所提供的系统调用的呢？

RISC-V中提供了ecall指令，用于向运行时环境发出请求，我们可以使用内联汇编使用该指令，进行系统调用，代码如下：

```
 1     #define ecall() ({\
 2      asm volatile(\
 3         "li x17,81\n"\
 4       "ecall");\
 5     })
 6
 7     int main(void){
 8         //调用自定义的81号系统调用
 9         ecall();
 10         return 0;
 11     }
```

例3.1 ecall

以上代码中，我们将系统调用号81通过x17寄存器传给内核，再通过ecall指令进行系统调用，当然目前代理内核中并没有81号系统调用的实现，这需要我们在后面的实验中完成。

**3.2.2 练习二：系统调用过程跟踪**

在我们执行了ecall指令后，代理内核中又是如何执行的呢？

在第一章的表1.7中，我们可以看到Environment call from U-mode是exception(trap)的一种，其对应的code是8。我们在实验二中已经讨论过中断入口函数位置的设置，现在继续跟踪中断入口函数，找出系统调用的执行过程。

**3.2.3 练习三：自定义系统调用（需要编程）**

阅读pk目录system.c文件，增加一个系统调用sys_get_memsize()，系统调用返回spike设置的内存空间大小, 系统调用号为81。

提升：在pk目录下的mmap.c文件中，函数pk_vm_init中定义了代理内核的内存空间大小。

spike 通过-m来指定分配的物理内存，单位是MiB，默认为2048。如：

`$ spike obj/pke app/elf/app2_1`

得到的输出如下：

```
PKE IS RUNNING
physical mem_size = 0x80000000
```

可以看到，默认情况下，spike的物理内存是2GB

你可以修改-m选项，运行app3观察输出。

`$ spike -m1024 obj/pke app/elf/app2_1`

预计得到输出格式：

```
PKE IS RUNNING
physical mem_size = 0x40000000
```

如果你的app可以正确输出的话，那么运行检查的python脚本：

`$ ./pke-lab2`

若得到如下输出，那么恭喜你，你已经成功完成了实验二！！！

```
build pk : OK
running app3 m2048 : OK
test3_m2048 : OK
running app3 m1024 : OK
test3_m1024 : OK
Score: 20/20
```

### 3.3 基础知识

**3.3.1 系统调用**

​首先，我们要知道什么是系统调用。

例如读文件(read)、写文件(write)等，其实我们已经接触过形形色色的系统调用。系统调用和函数调用的外形相似，但他们有着本质的不同。

系统调用的英文名字是System Call。由于用户进程只能在操作系统给它圈定好的“用户环境”中执行，但“用户环境”限制了用户进程能够执行的指令，即用户进程只能执行一般的指令，无法执行特权指令。如果用户进程想执行一些需要特权指令的任务，比如通过网卡发网络包等，只能让操作系统来代劳了。系统调用就是用户模式下请求操作系统执行某些特权指令的任务的机制。

相交于函数调用在普通的用户模式下运行，系统调用则运行在内核模式中。

见下图：

 <img src="pictures/fig3_1.png" alt="fig3_1" style="zoom:80%;" />

图3.1系统调用过程

系统调用属于一种中断，当用户申请系统调用时，系统会从用户态陷入到内核态，完成相应的服务后，再回到原来的用户态上下文中。

**3.3.2 代理内核与应用程序的加载**

阅读pke.lds文件可以看到整个PK程序的入口为：reset_vector函数：

```
3   OUTPUT_ARCH( "riscv" )
4
5   ENTRY( reset_vector )
```

我们在machine/mentry.S中找的这个符号。

```
36   reset_vector:
37  j do_reset
```

首先初始化x0~x31共32个寄存器，其中x10（a0）寄存器与x11（a1）寄存器存储着从之前boot loader中传来的参数而不复位。

```
223     do_reset:
224     li x1, 0
  .....
255     li x31, 0
```

将mscratch寄存器置0

```
256     csrw mscratch, x0
```

将trap_vector的地址写入t0寄存器，trap_vector是mechine模式下异常处理的入口地址。再将t0的值写入mtvec寄存器中。然后读取mtvec寄存器中的地址到t1寄存器。比较t0于t1。

```
259     la t0, trap_vector
260     mtvec, t0
261   	rr t1, mtvec
262     1:bne t0, t1, 1b
```

正常情况下，t1自然是的等于t0的，于是程序顺序执行，将栈地址写入sp寄存器中

```
264     la sp, stacks + RISCV_PGSIZE - MENTRY_FRAME_SIZE
```

读取mhartid到a3寄存器，调整sp

```
266     csrr a3, mhartid
267     slli a2, a3, RISCV_PGSHIFT
268   add sp, sp, a2
```

当a3不等于0时，跳转到 init_first_hart

```
270      # Boot on the first hart
271     beqz a3, init_first_hart
```

此时进入"machine/minit.c"文件，在init_first_hart中对外设进行初始化

```
154      void init_first_hart(uintptr_t hartid, uintptr_t dtb)
155     {
         …… //初始化外设
180          boot_loader(dtb);
181     }
```

在init_first_hart的最后一行，调用boot_loader函数

```
160 void boot_loader(uintptr_t dtb)
161 {
     …….   //CSR寄存器设置
169      enter_supervisor_mode(rest_of_boot_loader, pk_vm_init(), 0);
170  }
```

在boot_loader中，经历设置中断入口地址，清零sscratch寄存器，关中断等一系列操作后。最后会调用enter_supervisor_mode函数正式切换至Supervisor模式。

```
204     void enter_supervisor_mode(void (*fn)(uintptr_t), uintptr_t arg0, uintptr_t arg1)
205   {
206         uintptr_t mstatus = read_csr(mstatus);
207      mstatus = INSERT_FIELD(mstatus, MSTATUS_MPP, PRV_S);
208      mstatus = INSERT_FIELD(mstatus, MSTATUS_MPIE, 0);
209         write_csr(mstatus, mstatus);
210      write_csr(mscratch, MACHINE_STACK_TOP() - MENTRY_FRAME_SIZE);
211       #ifndef __riscv_flen
212      uintptr_t *p_fcsr = MACHINE_STACK_TOP() - MENTRY_FRAME_SIZE; // the x0's save slot
213      *p_fcsr = 0;
214       #endif
215      write_csr(mepc, fn);
216
217         register uintptr_t a0 asm ("a0") = arg0;
218         register uintptr_t a1 asm ("a1") = arg1;
219       asm volatile ("mret" : : "r" (a0), "r" (a1));
220      __builtin_unreachable();
221     }
```

在enter_supervisor_mode函数中，将 mstatus的MPP域设置为1，表示中断发生之前的模式是Superior，将mstatus的MPIE域设置为0，表示中段发生前MIE的值为0。随机将机器模式的内核栈顶写入mscratch寄存器中，设置mepc为rest_of_boot_loader的地址，并将kernel_stack_top与0作为参数存入a0和a1。

最后，执行mret指令，该指令执行时，程序从机器模式的异常返回，将程序计数器pc设置为mepc，即rest_of_boot_loader的地址；将特权级设置为mstatus寄存器的MPP域，即方才所设置的代表Superior的1，MPP设置为0；将mstatus寄存器的MIE域设置为MPIE，即方才所设置的表示中断关闭的0，MPIE设置为1。

于是，当mret指令执行完毕，程序将从rest_of_boot_loader继续执行。

```
144     static void rest_of_boot_loader(uintptr_t kstack_top)
145     {
146      arg_buf args;
147      size_t argc = parse_args(&args);
148      if (!argc)
149       panic("tell me what ELF to load!");
150
151      // load program named by argv[0]
152      long phdrs[128];
153      current.phdr = (uintptr_t)phdrs;
154      current.phdr_size = sizeof(phdrs);
155      load_elf(args.argv[0], &current);
156
157      run_loaded_program(argc, args.argv, kstack_top);
158     }
```

这个函数中，我们对应用程序的ELF文件进行解析，并且最终运行应用程序。

**3.3.3 中断的处理过程**

考虑一下，当程序执行到中断之前，程序是有自己的运行状态的，例如寄存器里保持的上下文数据。当中断发生，硬件在自动设置完中断原因和中断地址后，就会调转到中断处理程序，而中断处理程序同样会使用寄存器，于是当程序从中断处理程序返回时需要保存需要被调用者保存的寄存器，我们称之为callee-saved寄存器。

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

这里介绍一下RISCV的中断委托机制，在默认的情况下，所有的异常都会被交由机器模式处理。但正如我们知道的那样，大部分的系统调用都是在S模式下处理的，因此RISCV提供了这一委托机制，可以选择性的将中断交由S模式处理，从而完全绕过M模式。

接下，我们继续看S模式下的中断处理。在pk目录下的pk.c文件中的boot_loader函数中将&trap_entry写入了stvec寄存器中，stvec保存着发生异常时处理器需要跳转到的地址，也就是说当中断发生，我们将跳转至trap_entry，现在我们继续跟踪trap_entry。trap_entry在pk目录下的entry.S中，其代码如下：

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

在61行，交换了sp与sscratch的值，这里是为了根据sscratch的值判断该中断是来源于U模式还是S模式。

如果sp也就是传入的sscratch值不为零，则跳转至64行，若sscratch的值为零，则恢复原sp中的值。这是因为，当中断来源于S模式是，sscratch的值为0，sp中存储的就是内核的堆栈地址。而当中断来源于U模式时，sp中存储的是用户的堆栈地址，sscratch中存储的则是内核的堆栈地址，需要交换二者，是sp指向内核的堆栈地址。

接着在64,65行保存上下文，最后跳转至67行处理trap。handle_trap在pk目录下的handlers.c文件中，代码如下：

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

handle_trap函数中实现了S模式下各类中断的处理。可以看到，代码的126行就对应着系统调用的处理，handle_syscall的实现如下：

```
100     static void handle_syscall(trapframe_t* tf)
101     {
102      tf->gpr[10] = do_syscall(tf->gpr[10], tf->gpr[11], tf->gpr[12], tf->gpr[13],
103                  tf->gpr[14], tf->gpr[15], tf->gpr[17]);
104      tf->epc += 4;
105     }
```

还记得我们在例3.1中是将中断号写入x17寄存器嘛？其对应的就是这里do_syscall的最后一个参数，我们跟踪进入do_syscall函数，其代码如下：

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

do_syscall中通过传入的系统调用号n，查询syscall_table得到对应的函数，并最终执行系统调用。
 