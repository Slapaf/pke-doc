## 第六章．（实验5）进程的封装

### 6.1 实验内容

实验要求：在APP里写fork调用，其执行过程将fork出一个子进程。在代理内核中实现fork的处理例程（trap），使其能够支撑APP程序的正确执行。

在本次实验的app4.c文件中，将会测试fork（）函数。代码中170及172系统调用分别对应着sys_fork（）和sys_getpid（）系统调用。调用fork函数后，将会有两个返回。在父进程中，fork返回新创建子进程的进程ID；而在子进程中，fork返回0。你需要阅读proc.c文件，完善相关代码，是的app4.c可以正常运行。

 

**6.1.1 练习一：alloc_proc（需要编程）**

完善"pk/proc.c"中的alloc_proc()，你需要对以下属性进行初始化：

l enum proc_state state;  

l int pid;   

l int runs;   

l uintptr_t kstack; 

l volatile bool need_resched;    

l struct proc_struct *parent;       

l struct mm_struct *mm;      

l struct context context;       

l struct trapframe *tf;            

l uintptr_t cr3;               

l uint32_t flags;           

l char name[PROC_NAME_LEN + 1];   

 

**6.1.2 练习二：do_fork（需要编程）**

完善"pk/proc.c"中的do_fork函数，你需要进行以下工作：

l 调用alloc_proc()来为子进程创建进程控制块

l 调用setup_kstack来设置栈空间

l 用copy_mm来拷贝页表

l 调用copy_thread来拷贝进程

l 为子进程设置pid

l 设置子进程状态为就绪

l 将子进程加入到链表中

 

完成以上代码后，你可以进行如下测试，然后输入如下命令：

`$ riscv64-unknown-elf-gcc ../app/app5.c -o ../app/elf/app5`

`$ spike ./obj/pke app/elf/app5`

预期的输出如下：

```
PKE IS RUNNING
page fault vaddr:0x00000000000100c2
page fault vaddr:0x000000000001e17f
page fault vaddr:0x0000000000018d5a
page fault vaddr:0x000000000001a8ba
page fault vaddr:0x000000000001d218
page fault vaddr:0x000000007f7e8bf0
page fault vaddr:0x0000000000014a68
page fault vaddr:0x00000000000162ce
page fault vaddr:0x000000000001c6e0
page fault vaddr:0x0000000000012572
page fault vaddr:0x0000000000011fa6
page fault vaddr:0x0000000000019064
page fault vaddr:0x0000000000015304
page fault vaddr:0x0000000000017fd4
this is farther process;my pid = 1
sys_exit pid=1
page fault vaddr:0x0000000000010166
page fault vaddr:0x000000000001e160
page fault vaddr:0x000000000001d030
page fault vaddr:0x0000000000014a68
page fault vaddr:0x00000000000162ce
page fault vaddr:0x000000000001c6e0
page fault vaddr:0x0000000000012572
page fault vaddr:0x0000000000011fa6
page fault vaddr:0x0000000000019064
page fault vaddr:0x000000000001abb6
page fault vaddr:0x0000000000015304
page fault vaddr:0x0000000000017fd4
page fault vaddr:0x0000000000018cd4
this is child process;my pid = 2
sys_exit pid=2
```

如果你的app可以正确输出的话，那么运行检查的python脚本：

./pke-lab5

若得到如下输出，那么恭喜你，你已经成功完成了实验六！！！

 

build pk : OK

running app5 : OK

 test fork : OK

Score: 20/20

### 6.2 基础知识

**6.2.1 进程结构**

   在pk/proc.h中，我们定义进程的结构如下：

 42  struct proc_struct {

 43    enum proc_state state;         

 44    int pid;                 

 45    int runs;                

 46    uintptr_t kstack;            

 47    volatile bool need_resched;        

 48    struct proc_struct *parent;                   

 50    struct context context;           

 51    trapframe_t *tf;           

 52    uintptr_t cr3;               

 53    uint32_t flags;              

 54    char name[PROC_NAME_LEN + 1];       

 55    list_entry_t list_link;           

 56    list_entry_t hash_link;           

 57  };

​    可以看到在41行的枚举中，我们定义了进程的四种状态，其定义如下：

 11  enum proc_state {

 12    PROC_UNINIT = 0,  

 13    PROC_SLEEPING,   

 14    PROC_RUNNABLE,   

 15    PROC_ZOMBIE,   

 16  };

​    四种状态分别为未初始化(PROC_UNINIT)、睡眠（PROC_SLEEPING）、可运行（PROC_RUNNABLE）以及僵死（PROC_ZOMBIE）状态。

​    除却状态，进程还有以下重要属性：

l pid：进程id，是进程的标识符

l runs：进程已经运行的时间

l kstack：进程的内核栈空间

l need_resched：是否需要释放CPU

l parent：进程的父进程

l context：进程的上下文

l tf：当前中断的栈帧

l cr3：进程的页表地址

l name：进程名

除了上述属性，可以看到在55、56行还维护了两个进程的链表，这是操作系统内进程的组织方式，系统维护一个进程链表，以组织要管理的进程。

 

**6.2.2 设置第一个内核进程idleproc**

   在"pk/pk.c"的rest_of_boot_loader函数中调用了proc_init来设置第一个内核进程：

317  void

318  proc_init() {

319    int i;

320    extern uintptr_t kernel_stack_top;

321

322    list_init(&proc_list);

323    for (i = 0; i < HASH_LIST_SIZE; i ++) {

324      list_init(hash_list + i);

325    }

326

327    if ((idleproc = alloc_proc()) == NULL) {

328      panic("cannot alloc idleproc.\n");

329    }

330

331    idleproc->pid = 0;

332    idleproc->state = PROC_RUNNABLE;

333    idleproc->kstack = kernel_stack_top;

334    idleproc->need_resched = 1;

335    set_proc_name(idleproc, "idle");

336    nr_process ++;

337

338    currentproc = idleproc;

339

340  }

​    322行的proc_list是系统所维护的进程链表，324行的hash_list是一个大小为1024的list_entry_t的hash数组。在对系统所维护的两个list都初始化完成后，系统为idleproc分配进程结构体。然后对idleproc的各个属性进行设置，最终将currentproc改为idleproc。

​    在上述代码中，我们只是为idleproc分配了进程控制块，但并没有切换到idleproc，真正的切换代码在proc_init函数后面的run_loaded_program以及cpu_idle函数中进行。

 

**6.2.3 do_fork**

​    在run_loaded_program中有如下代码：

140  trapframe_t tf;

141  init_tf(&tf, current.entry, stack_top);

142  __clear_cache(0, 0);

143  do_fork(0,stack_top,&tf);

144  write_csr(sscratch, kstack_top);

 

   在这里，声明了一个trapframe，并且将它的gpr[2]（sp）设置为内核栈指针，将它的epc设置为current.entry，其中current.entry是elf文件的入口地址也就是app的起始执行位置，随即，我们调用了do_frok函数，其中传入参数stack为0表示我们正在fork一个内核进程。

​    在do_frok函数中，你会调用alloc_proc()来为子进程创建进程控制块、调用setup_kstack来设置栈空间，调用copy_mm来拷贝页表，调用copy_thread来拷贝进程。现在，我们来对以上函数进行分析。

​    setup_kstack函数代码如下，在函数中，我们为进程分配栈空间，然后返回：

210  static int

211  setup_kstack(struct proc_struct *proc) {

212     proc->kstack = (uintptr_t)__page_alloc();

213    return 0;

214  }

copy_mm k函数代码如下，在函数中，我们对页表进行拷贝。

228  static int

229  copy_mm(uint32_t clone_flags, struct proc_struct *proc) {

230    //assert(currentproc->mm == NULL);

231    /* do nothing in this project */

232    uintptr_t cr3=(uintptr_t)__page_alloc();

233    memcpy((void *)cr3,(void *)proc->cr3,RISCV_PGSIZE);

234    proc->cr3=cr3;

235    return 0;

236  }

​    最后是copy_thread函数：

240  static void

241  copy_thread(struct proc_struct *proc, uintptr_t esp, trapframe_t *tf) {

242    proc->tf = (trapframe_t *)(proc->kstack + KSTACKSIZE - sizeof(trapframe_t));

243    *(proc->tf) = *tf;

244

245    proc->tf->gpr[10] = 0;

246    proc->tf->gpr[2] = (esp == 0) ? (uintptr_t)proc->tf -4 : esp;

247

248    proc->context.ra = (uintptr_t)forkret;

249    proc->context.sp = (uintptr_t)(proc->tf);

250  }

​    在函数中，首先对传入的栈帧进行拷贝，并且将上下文中的ra设置为地址forkret，将sp设置为该栈帧。

​    完成以上几步后，我们为子进程设置pid，将其加入到进程链表当中，并且设置其状态为就绪。

​         

**6.2.3 上下文切换**

​    每个进程都有着自己的上下文，在进程间切换时，需要对上下文一并切换。

​    在pk/proc.c的cpu_idle中有以下代码：

374  void

375  cpu_idle(void) {

376    while (1) {

377      if (currentproc->need_resched) {

378        schedule();

379      }

380    }

381  }

​    在当前进程处于need_resched状态时，会执行调度算法schedule，其代码如下：

 16  void

 17  schedule(void) {

 18    list_entry_t *le, *last;

 19    struct proc_struct *next = NULL;

 20    {

 21      currentproc->need_resched = 0;

 22      last = (currentproc == idleproc) ? &proc_list : &(currentproc->list_link);

 23      le = last;

 24      do {

 25        if ((le = list_next(le)) != &proc_list) {

 26          next = le2proc(le, list_link);

 27          if (next->state == PROC_RUNNABLE) {

 28            break;

 29          }

 30        }

 31      } while (le != last);

 32      if (next == NULL || next->state != PROC_RUNNABLE) {

 33        next = idleproc;

 34      }

 35      next->runs ++;

 36      if (next != currentproc) {

 37        proc_run(next);

 38      }

 39    }

 40  }

 

​    在schedule函数中找到下一个需要执行的进程，并执行，执行代码proc_run如下：

145  void

146  proc_run(struct proc_struct *proc) {

147    if (proc != currentproc) {

148      bool intr_flag;

149      struct proc_struct *prev = currentproc, *next = proc;

150      currentproc = proc;

151      write_csr(sptbr, ((uintptr_t)next->cr3 >> RISCV_PGSHIFT) | SATP_MODE_CHOICE);

152      switch_to(&(prev->context), &(next->context));

153

154    }

155  }

​    当传入的proc不为当前进程时，执行切换操作：

7 switch_to:

 8   # save from's registers

 9   STORE ra, 0*REGBYTES(a0)

 10   STORE sp, 1*REGBYTES(a0)

 11   STORE s0, 2*REGBYTES(a0)

 12   STORE s1, 3*REGBYTES(a0)

 13   STORE s2, 4*REGBYTES(a0)

 14   STORE s3, 5*REGBYTES(a0)

 15   STORE s4, 6*REGBYTES(a0)

 16   STORE s5, 7*REGBYTES(a0)

 17   STORE s6, 8*REGBYTES(a0)

 18   STORE s7, 9*REGBYTES(a0)

 19   STORE s8, 10*REGBYTES(a0)

 20   STORE s9, 11*REGBYTES(a0)

 21   STORE s10, 12*REGBYTES(a0)

 22   STORE s11, 13*REGBYTES(a0)

 23

 24   # restore to's registers

 25   LOAD ra, 0*REGBYTES(a1)

 26   LOAD sp, 1*REGBYTES(a1)

 27   LOAD s0, 2*REGBYTES(a1)

 28   LOAD s1, 3*REGBYTES(a1)

 29   LOAD s2, 4*REGBYTES(a1)

 30   LOAD s3, 5*REGBYTES(a1)

 31   LOAD s4, 6*REGBYTES(a1)

 32   LOAD s5, 7*REGBYTES(a1)

 33   LOAD s6, 8*REGBYTES(a1)

 34   LOAD s7, 9*REGBYTES(a1)

 35   LOAD s8, 10*REGBYTES(a1)

 36   LOAD s9, 11*REGBYTES(a1)

 37   LOAD s10, 12*REGBYTES(a1)

 38   LOAD s11, 13*REGBYTES(a1)

 39

 40   ret

​    可以看到，在switch_to中，我们正真执行了上一个进程的上下文保存，以及下一个进程的上下文加载。在switch_to的最后一行，我们执行ret指令，该指令是一条从子过程返回的伪指令，会将pc设置为x1（ra）寄存器的值，还记得我们在copy_thread中层将ra设置为forkret嘛？现在程序将从forkret继续执行：

160  static void

161  forkret(void) {

162    extern elf_info current;

163    load_elf(current.file_name,&current);

164

165    int pid=currentproc->pid;

166    struct proc_struct * proc=find_proc(pid);

167    write_csr(sscratch, proc->tf);

168    set_csr(sstatus, SSTATUS_SUM | SSTATUS_FS);

169    currentproc->tf->status = (read_csr(sstatus) &~ SSTATUS_SPP &~ SSTATUS_SIE) | SSTATUS_SPIE;

170    forkrets(currentproc->tf);

171  }

 

​    我们进入forkrets：

121  forkrets:

122    # set stack to this new process's trapframe

123    move sp, a0

124    addi sp,sp,320

125    csrw sscratch,sp

126    j start_user

 

 

 76  .globl start_user

 77 start_user:

 78  LOAD t0, 32*REGBYTES(a0)

 79  LOAD t1, 33*REGBYTES(a0)

 80  csrw sstatus, t0

 81  csrw sepc, t1

 82

 83  # restore x registers

 84  LOAD x1,1*REGBYTES(a0)

 85  LOAD x2,2*REGBYTES(a0)

 86  LOAD x3,3*REGBYTES(a0)

 87  LOAD x4,4*REGBYTES(a0)

 88  LOAD x5,5*REGBYTES(a0)

 89  LOAD x6,6*REGBYTES(a0)

 90  LOAD x7,7*REGBYTES(a0)

 91  LOAD x8,8*REGBYTES(a0)

 92  LOAD x9,9*REGBYTES(a0)

 93  LOAD x11,11*REGBYTES(a0)

 94  LOAD x12,12*REGBYTES(a0)

 95  LOAD x13,13*REGBYTES(a0)

 96  LOAD x14,14*REGBYTES(a0)

 97  LOAD x15,15*REGBYTES(a0)

 98  LOAD x16,16*REGBYTES(a0)

 99  LOAD x17,17*REGBYTES(a0)

100  LOAD x18,18*REGBYTES(a0)

101  LOAD x19,19*REGBYTES(a0)

102  LOAD x20,20*REGBYTES(a0)

103  LOAD x21,21*REGBYTES(a0)

104  LOAD x22,22*REGBYTES(a0)

105  LOAD x23,23*REGBYTES(a0)

106  LOAD x24,24*REGBYTES(a0)

107  LOAD x25,25*REGBYTES(a0)

108  LOAD x26,26*REGBYTES(a0)

109  LOAD x27,27*REGBYTES(a0)

110  LOAD x28,28*REGBYTES(a0)

111  LOAD x29,29*REGBYTES(a0)

112  LOAD x30,30*REGBYTES(a0)

113  LOAD x31,31*REGBYTES(a0)

114  # restore a0 last

115  LOAD x10,10*REGBYTES(a0)

116

117  # gtfo

118  sret

​    可以看到在forkrets最后执行了sret，程序就此由内核切换至用户程序执行！！

​      