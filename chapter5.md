# 第五章．（实验4）缺页异常的处理

## 5.1 实验内容


#### 应用： ####


app4_1.c源文件如下：


	1 #include<stdio.h>
	2
	3 int main()
	4 {
	5
	6         uintptr_t addr = 0x7f000000;
	7         *(int *)(addr)=1;
	8
	9         uintptr_t addr1_same_page = 0x7f000010;
	10         uintptr_t addr2_same_page = 0x7f000fff;
	11         *(int *)(addr1_same_page)=2;
	12         *(int *)(addr2_same_page)=3;
	13
	14         uintptr_t addr1_another_page = 0x7f001000;
	15         uintptr_t addr2_another_page = 0x7f001ff0;
	16         *(int *)(addr1_another_page)=4;
	17         *(int *)(addr2_another_page)=5;
	18
	19
	20         return 0;
	21 }

以上代码中对地址0x7f000000进行访问，将触发缺页异常。随后，对同一页内的地址0x7f000010、0x7f000fff进行访问，此时由于页0x7f000000已近完成映射，故而不会发生异常。最后有对新的一页进行访问，将再次引发缺页异常。

app4_2.c源文件如下：
	
	  1 #include <stdio.h>
	  2 void fun(int num){
	  3         if(num==0){
	  4                 return;
	  5         }
	  6         fun(num-1);
	  7 }
	  8 int main(){
	  9         int num=10000;
	 10         fun(num);
	 11         printf("end  \n");
	 12         return 0;
	 13 }


以上代码中进行了大量递归，这将产生缺页。



#### 任务一 : 缺页中断实例的完善（编程） ####

任务描述：


 **在**"pk/mmap.c"内有__handle_page_fault()函数，完善该函数，实现缺页中的处理。

```
202  static int __handle_page_fault(uintptr_t vaddr, int prot)
203  {
204   printk("page fault vaddr:0x%lx\n", vaddr);
205   //your code here
206   //start------------>
207     pte_t* pte =0;
208
209   //<-----------end
210   if (pte == 0 || *pte == 0 || !__valid_user_range(vaddr, 1))
211    return -1;
212   else if (!(*pte & PTE_V))
213   {
214
215    //your code here
216    //start--------->
217
218     uintptr_t ppn =0;
219     vmr_t* v = NULL;
220
221    //<----------end
```


预期输出：
 

当你完成__handle_page_fault()函数后，可进行如下测试：

编译app目录下，实验四相关文件：

`$ riscv64-unknown-elf-gcc  app/app4_1.c -o app/elf/app4_1`

`$ riscv64-unknown-elf-gcc  app/app4_2.c -o app/elf/app4_2`

​    使用spike运行，预期输出如下：

```
spike obj/pke app/elf/app4_1

PKE IS RUNNING
page fault vaddr:0x0000000000011000
page fault vaddr:0x000000007f7ecc00
page fault vaddr:0x00000000000100c0
page fault vaddr:0x000000007f000000
page fault vaddr:0x000000007f001000
```

 

`$ spike obj/pke app/elf/app4_1`

//递归程序可正常运行

如果你的两个测试app都可以正确输出的话，那么运行检查的python脚本：

`$ ./pke-lab4`

若得到如下输出，那么恭喜你，你已经成功完成了实验四！！！

```
build pk : OK
running app4_1 : OK
 test4_1 : OK
running app4_2 : OK
 test4_2 : OK
```

## 5.2 基础知识

**5.2.1 虚拟地址空间**

物理地址唯一，且有限的，但现在的操作系统上往往有不止一个的程序在运行。如果只有物理地址，那对于程序员来说无疑是相当繁琐的。程序不知道那些内存可用，那些内存已经被其他程序占有，这就意味着操作系统必须管理所有的物理地址，并且所有所有代码都是共用的。

为了解决上述问题，操作系统引入了虚拟地址的概念。每个进程拥有着独立的虚拟地址空间，这个空间是连续的，一个虚拟页可以映射到不同或者相同的物理页。这就是我们所说的虚拟地址。在程序中，我们所使用的变量的地址均为虚拟地址。

**5.2.2 虚拟地址同物理地址之间的转换**

​    虚拟地址只是一个逻辑上的概念，在计算机中，最后正真被访问的地址仍是物理地址。所以，我们需要在一个虚拟地址访问内存之前将它翻译成物理地址，这个过程称为地址翻译。CPU上的内存管理单元（MMU）会利用存放在主存的页表完成这一过程。

​    RISCV的S模式下提供了基于页面的虚拟内存管理机制，内存被划分为固定大小的页。我们使用物理地址对这些页进行描述，我们在此回顾上一章所讲到的RISCV物理地址的定义：

<img src="pictures/fig5_1.png" alt="fig5_1" style="zoom:80%;" />

图5.1 RISCV64 物理地址

​可以看到，物理地址由PPN（物理页号）与Offset（偏移量）组成。这里的PPN就对应着上述的物理页。

​现在，我们来看RISCV虚拟地址的定义： 

<img src="pictures/fig5_2.png" alt="fig5_2" style="zoom:80%;" />

图5.2 RISCV64 虚拟地址

​    可以看到虚拟地址同样由页号和偏移量组成。而这二者之间是如何转换的呢？RV64支持多种分页方案，如Sv32、Sv39、Sv48，它们的原理相似，这里我们对pke中所使用的Sv39进行讲述。Sv39中维护着一个三级的页表，其页表项定义如下：

  <img src="pictures/fig1_7.png" alt="fig1_7" style="zoom:80%;" />

图5.3 Sv39页表项 

​    当启动分页后，MMU会对每个虚拟地址进行页表查询，页表的地址由satp寄存器保存。在"pk/mmap.c"中的pk_vm_init函数中有如下一行代码其中，sptbr即为satp的曾用名，在这行代码中，我们将页表地址写入satp寄存器。

```
458  write_csr(sptbr, ((uintptr_t)root_page_table >> RISCV_PGSHIFT) | SATP_MODE_CHOICE);
```



 <img src="pictures/fig5_4.png" alt="fig5_4" style="zoom:80%;" />

 图5.4 地址转换

​    于是，当需要进行页表转换时，我们变从satp所存储的页表地址开始，逐级的转换。

在pke中，位于"pk/mmap.c"中的转换代码如下：

```
112  static size_t pt_idx(uintptr_t addr, int level)
113  {
114   size_t idx = addr >> (RISCV_PGLEVEL_BITS*level + RISCV_PGSHIFT);
115   return idx & ((1 << RISCV_PGLEVEL_BITS) - 1);
116  }
```

 

​    首先，我们来看pt_idx函数，函数中将虚拟地址addr右移RISCV_PGLEVEL_BITS*level + RISCV_PGSHIFT位，其中RISCV_PGSHIFT对应着VPN中的Offset，而level则对应着各级VPN，pt_idx通过level取出指定的VPN。当level = 2, 得到vpn[2]，即页目录项在一级页表的序号，，当level = 1, 得到vpn[1]，即页目录项在二级页表的序号，同理，当level = 0, 则得到vpn[0]，即页表项在三级页表的序号。

```
125  static pte_t* __walk_internal(uintptr_t addr, int create)
126  {
127   pte_t* t = root_page_table;
128   for (int i = (VA_BITS - RISCV_PGSHIFT) / RISCV_PGLEVEL_BITS - 1; i > 0; i--) {
129    size_t idx = pt_idx(addr, i);
130    if (unlikely(!(t[idx] & PTE_V)))
131     return create ? __continue_walk_create(addr, &t[idx]) : 0;
132    t = (pte_t*)(pte_ppn(t[idx]) << RISCV_PGSHIFT);
133   }
134   return &t[pt_idx(addr, 0)];
135  }
```

接着，我们进一步分析__walk_internal函数，首先VA_BITS即虚拟地址的位数为39，RISCV_PGSHIFT即代表虚拟地址中Offset的位数，二者相减，剩下的就是VPN0、VPN1……VPNX的位数，在除以VPN的位数，得到就是VPN的数量。由于pke中式Sv39，故而VPN的数量为3，即VPN0、VPN1、VPN2。

接着我们使用pt_idx函数得到各级VPN的值，依据图5.2所示逐级查询，一直找到该虚拟地址对应的页表项，而该页表项中存着该虚拟地址所对应的物理页号，再加上虚拟地址中的偏离量，我们就可以找到最终的物理地址了！！

  

**5.2.3** **缺页异常处理**

```
 1 #include<stdio.h>
 2
 3 int main()
 4 {
 5
 6     uintptr_t addr = 0x7f000000;
 7     *(int *)(addr)=1;
 8
 9     uintptr_t addr1_same_page = 0x7f000010;
 10     uintptr_t addr2_same_page = 0x7f000fff;
 11     *(int *)(addr1_same_page)=2;
 12     *(int *)(addr2_same_page)=3;
 13
 14     uintptr_t addr1_another_page = 0x7f001000;
 15     uintptr_t addr2_another_page = 0x7f001ff0;
 16     *(int *)(addr1_another_page)=4;
 17     *(int *)(addr2_another_page)=5;
 18
 19
 20     return 0;
 21 }
```

以上程序中，我们人为的访问虚拟地址0x7f000000与虚拟地址0x7f001000所对应的物理页，由于操作系统并没有事先加载这些页面，于是会出发缺页中断异常。进入pk/mmap.c文件下的__handle_page_fault函数中，其代码如下：

```
203  static int __handle_page_fault(uintptr_t vaddr, int prot)
204  {
205   printk("page fault vaddr:0x%lx\n", vaddr);
206   //your code here
207   //start------------>
208     pte_t* pte =0;
209
210   //<-----------end
211   if (pte == 0 || *pte == 0 || !__valid_user_range(vaddr, 1))
212    return -1;
213   else if (!(*pte & PTE_V))
214   {
215
216    //your code here
217    //start--------->
218
219     uintptr_t ppn =0;
220     vmr_t* v = NULL;
221
222    //<----------end
223
224    if (v->file)
225    {
226     size_t flen = MIN(RISCV_PGSIZE, v->length - (vaddr - v->addr));
227    // ssize_t ret = file_pread(v->file, (void*)vaddr, flen, vaddr - v->addr + v->offset);
228     ssize_t ret = file_pread_pnn(v->file, (void*)vaddr, flen, ppn, vaddr - v->addr + v->offset);
229     kassert(ret > 0);
230     if (ret < RISCV_PGSIZE)
231      memset((void*)vaddr + ret, 0, RISCV_PGSIZE - ret);
232    }
233    else
234     memset((void*)vaddr, 0, RISCV_PGSIZE);
235    __vmr_decref(v, 1);
236    *pte = pte_create(ppn, prot_to_type(v->prot, 1));
237   }
238
239   pte_t perms = pte_create(0, prot_to_type(prot, 1));
240   if ((*pte & perms) != perms)
241    return -1;
242
243   flush_tlb();
244   return 0;
245  }
```

​对于一个没有对应物理地址的虚拟地址，我们需要进行如下的处理。首先，找到该物理地址所对应的pte，这里你可能会使用到__walk函数，__walk中调用了上文中我们讨论过的__walk_internal函数，对于一个给定的虚拟地址，返回其对于的pte，其定义如下：

```
138  pte_t* __walk(uintptr_t addr)
139  {
140   return __walk_internal(addr, 0);
141  }
```

其次，使用操作系统为该虚拟地址分配一个相对应的物理页，还记得物理内存管理中的内存分配嘛？现在它有用武之地了；最后将该物理地址写入第一步的得到的pte中，这里你会用到page2ppn和pte_create函数。

以上，就是本次实验需要大家完成的部分了！