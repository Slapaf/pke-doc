## 第二章．（实验1）非法指令的截获

### 2.1 实验环境搭建

实验环境我们推荐采用Ubuntu 16.04LTS或18.04LTS（x86_64）操作系统，我们未在其他系统（如arch，RHEL等）上做过测试，但理论上只要将实验中所涉及到的安装包替换成其他系统中的等效软件包，就可完成同样效果。另外，我们在EduCoder实验平台（网址：https://www.educoder.net ）上创建了本书的同步课程，课程的终端环境中已完成实验所需软件工具的安装，所以如果读者是在EduCoder平台上选择的本课程，则可跳过本节的实验环境搭建过程，直接进入通过终端（命令行）进入实验环境。

PKE实验涉及到的软件工具有：RISC-V交叉编译器、spike模拟器，以及PKE源代码三个部分。假设读者拥有了Ubuntu 16.04LTS或18.04LTS（x86_64）操作系统的环境，以下分别介绍这三个部分的安装以及安装后的检验过程。需要说明的是，为了避免耗时耗资源的构建（build）过程，一个可能的方案是从https://toolchains.bootlin.com 下载，**但是要注意一些依赖包（如GCC）的版本号**。**我们强烈建议读者在新装环境中完整构建（build）RISC-V交叉编译器，以及spike模拟器**。如果强调环境的可移植性，可以考虑在虚拟机中安装完整系统和环境，之后将虚拟机进行克隆和迁移。

#### 2.1.1 RISC-V交叉编译器

RISC-V交叉编译器是与Linux自带的GCC编译器类似的一套工具软件集合，不同的是，x86_64平台上Linux自带的GCC编译器会将源代码编译、链接成为适合在x86_64平台上运行的二进制代码（称为native code），而RISC-V交叉编译器则会将源代码编译、链接成为在RISC-V平台上运行的代码。后者（RISC-V交叉编译器生成的二进制代码）是无法在x86_64平台（即x86_64架构的Ubuntu环境下）直接运行的，它的运行需要模拟器（我们采用的spike）的支持。

一般情况下，我们称x86_64架构的Ubuntu环境为host，而在host上执行spike后所虚拟出来的RISC-V环境，则被称为target。RISC-V交叉编译器的构建（build）、安装过程如下：

● 第一步，安装依赖库

RISC-V交叉编译器的构建需要一些本地支撑软件包，可使用以下命令安装：

`$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev device-tree-compiler`

● 第二步，获取RISC-V交叉编译器的源代码

有两种方式获得RISC-V交叉编译器的源代码：一种是通过源代码仓库获取，使用以下命令：

`$ git clone --recursive https://github.com/riscv/riscv-gnu-toolchain.git`

但由于RISC-V交叉编译器的仓库包含了Qemu模拟器的代码，下载后的目录占用的磁盘空间大小约为4.8GB，整体下载所需的时间较长。另一种方式是通过百度云盘，获取源代码压缩包，链接和提取码如下：

`链接: https://pan.baidu.com/s/1cMGt0zWhRidnw7vNUGcZhg 提取码: qbjh`

从百度云盘下载RISCV-packages/riscv-gnu-toolchain-clean.tar.gz文件（大小为2.7GB），再在Ubuntu环境下解压这个.tar.gz文件，采用如下命令行：

`$ tar xf  riscv-gnu-toolchain-clean.tar.gz`

之后就能够看到和进入当前目录下的riscv-gnu-toolchain文件夹了。

● 第三步，构建（build）RISC-V交叉编译器

`$ cd riscv-gnu-toolchain`

`$ ./configure --prefix=[your.RISCV.install.path]`

`$ make`

`$ make install`

以上命令中，[your.RISCV.install.path]指向的是你的RISC-V交叉编译器安装目录。如果安装是你home目录下的一个子目录（如~/riscv-install-dir），则最后的make install无需sudoer权限。但如果安装目录是系统目录（如/opt/ riscv-install-dir），则需要sudoer权限（即在make install命令前加上sudo）。

● 第四步，设置环境变量

`$ export RISCV=[your.RISCV.install.path]`

`$ export PATH=$PATH:$RISCV/bin`

以上命令设置了RISCV环境变量，指向在第三步中的安装目录，并且将交叉编译器的可执行文件所在的目录加入到了系统路径中。这样，我们就可以在PKE的工作目录调用RISC-V交叉编译器所包含的工具软件了。这时，你也可以在任意目录编写helloworld.c文件，并使用以下命令测试下安装是否成功：

`$ riscv64-unknown-elf-gcc helloworld.c -o helloworld`

若编译成功，当前的目录下会出现名为helloworld的elf文件。

#### 2.1.2 spike模拟器

接下来，安装spkie模拟器。首先取得spike的源代码，有两个途径：一个是从github代码仓库中获取：

`$ git clone https://github.com/riscv/riscv-isa-sim.git`

也可以从百度云盘中下载spike-riscv-isa-sim.tar.gz文件（约4.2MB），然后用tar命令解压缩。百度云盘的地址，以及tar命令解压缩可以参考2.1.1节RISC-V交叉编译器的安装过程。获得spike源代码或解压后，将在本地目录看到一个riscv-isa-sim目录。

接下来构建（build）spike，并安装：

`$ cd riscv-isa-sim`

`$ ./configure --prefix=$RISCV`

`$ make`

`$ make install`

在以上命令中，我们假设RISCV环境变量已经指向了RISC-V交叉编译器的安装目录。如果未建立关联，可以将$RISCV替换为2.1.1节中的[your.RISCV.install.path]。

#### 2.1.3 PKE

到码云（gitee）上下载课程仓库：

`$ git clone https://gitee.com/syivester/pke.git`

克隆完成后，将在当前目录看到pke目录。这时，可以到pke目录下查看pke的代码结构，例如：

`$ cd pke`

`$ ls`

你可以看到当前目录下有如下（部分）内容

.

├── app

├── gradelib.py

├── machine

├── Makefile

├── pk

├── pke-lab1

├── pke.lds

└── util

● 首先是app目录，里面存放的是实验的测试用例，也就是运行在User模式的应用程序，例如之前helloworld.c。

● gradelib.py、与pke-lab1是测试用的python脚本。

● machine目录，里面存放的是机器模式相关的代码，由于本课程的重点在于操作系统，在这里你可以无需详细研究。

● Makefile文件，它定义的整个工程的编译规则。

● pke.lds是工程的链接文件。

● util目录下是各模块会用到的工具函数。

● pk目录，里面存放的是pke的主要代码。

即使未开始做PKE的实验，我们的pke代码也是可构建的，可以采用以下命令（在pke目录下）生成pke代理内核：

`$ make`

以上命令完成后，会在当前目录下会生产obj子目录，里面就包含了我们的pke代理内核。pke代理内核的构建过程，将在2.2节中详细讨论。

#### 2.1.4 环境测试

全部安装完毕后，你可以对环境进行测试，在pke目录下输入：

`$ spike ./obj/pke ./app/elf/app1_2`

将得到如下输出：

```
PKE IS RUNNING
user mode test illegal instruction!
you need add your code!
```

以上命令的作用是首先采用spike模拟一个RISC-V机器，该机器支持RV64G指令集，并在该机器上运行./app/elf/app1_2应用（它的源代码在./app/app1_2.c中）。我们知道，应用是无法在“裸机”上运行的，所以测试命令使用./obj/pke作为应用的代理内核。代理内核的作用，是对spike模拟出来的RISC-V机器做简单的“包装”，使其能够在spike模拟出来的机器上顺利运行。

这里，代理内核的作用是只为特定的应用服务（如本例中的./app/elf/app1_2应用），所以可以做到“看菜吃饭”的效果。因为我们这里的应用非常简单，所以pke就可以做得极其精简，它没有文件系统、没有物理内存管理、没有进程调度、没有操作终端（shell）等等传统的完整操作系统“必须”具有的组件。在后续的实验中，我们将不断提升应用的复杂度，并不断完善代理内核。通过这个过程，读者将深刻体会操作系统内核对应用支持的机制，以及具体的实现细节。

  

### 2.2 实验内容

实验要求：在用户模式（APP）里调用非法指令（如S或M级别的指令），或进行非法内存访问，导致系统报错。如illegal instruction，或者内存访问越界报警。

注意：以后的实验，要基于本实验，使得代理内核能够捕捉非法指令和内存访问。

**2.2.1 练习一：hello world**

   首先进入app目录下。我们先来编写一个简单的hello world程序，我们编写hellowrold.c源文件如下：

```
 1 #include <stdio.h>
 2 int global_init=1;
 3 int global_uninit;
 4 int main(){
 5   int tmp;
 6   printf("hello world!\n");
 7   return 0;
 8 }
```

例2.1 hellowrold.c

使用riscv64-unknown-elf-gcc编译该文件，得到的ELF文件hellowrold。

`$riscv64-unknown-elf-gcc hellowrold.c -o elf/hellowrold`

   现在回到上一级目录，使用pke来运行二进制文件：

`$spike obj/pke app/elf/ hellowrold`

​    你可以得到以下输出：

```
PKE IS RUNNING
hello world!
```

**2.2.2 练习二：中断入口探寻**

CPU 运行到一些情况下会产生异常（exception） ，例如访问无效的内存地址、执行非法指令（除零）、发生缺页等。用户程序进行系统调用（syscall） ，或程序运行到断点（breakpoint） 时，也会主动触发异常。

当发生中断或异常时，CPU 会立即跳转到一个预先设置好的地址，执行中断处理程序，最后恢复原程序的执行。这个地址。我们称为中断入口地址。在RISC-V中，设有专门的CSR寄存器保存这个地址，即stvec寄存器。

下面，请你阅读pk.c文件，找出pk中设置中断入口函数的位置。

 

**2.2.3 练习三：中断过程详究**

中断的处理过程可以分为3步骤：

- 保存当前环境寄存器
- 进入具体的中断异常处理函数
- 恢复中断异常前环境的寄存器

pk中使用trapframe_t结构体（pk.h）来保存中断发生时常用的32个寄存器及部分特殊寄存器的值，其结构如下。

```
typedef struct
{
 long gpr[32]; 
 long status;
 long epc;
 long badvaddr;
 long cause;
 long insn;
} trapframe_t;
```

下面，请阅读entry.S,详细分析同上述三个过程相对应的代码。

**2.2.4 练习四：中断的具体处理（需要编程）**

当中断异常发生后，中断帧将会被传递给handlers.c中的handle_trap函数。接着，通过trapframe中的scause寄存器的值可以判断属于哪种中断异常，从而选择相对应的中断处理函数。

在pk/handlers.c中的各种中断处理函数的实现，其中segfault段错误的处理函数与illegal_instruction的处理函数并不完善。请你在pk/handlers.c中找到并完善segfault与handle_illegal_instruction两个函数。

提示：

当完成你的segfault代码后,重新make，然后输入如下命令：

`$riscv64-unknown-elf-gcc ../app/app1_1.c -o ../app/elf/app1_1`

`$spike ./obj/pke app/elf/app1_1`

预期的输出如下：

```
PKE IS RUNNING
APP: addr_u 0x7f7ecc00
APP: addr_m 0x8f000000
z 0000000000000000 ra 0000000000010192 sp 000000007f7ecb30 gp 000000000001da10
tp 0000000000000000 t0 0000000000000000 t1 000000007f7ec9f0 t2 0000219000080017
s0 000000007f7ecb50 s1 0000000000000000 a0 0000000000000017 a1 000000000001e220
a2 0000000000000017 a3 0000000000000000 a4 0000000000000001 a5 000000008f000000
a6 8080808080808080 a7 0000000000000040 s2 0000000000000000 s3 0000000000000000
s4 0000000000000000 s5 0000000000000000 s6 0000000000000000 s7 0000000000000000
s8 0000000000000000 s9 0000000000000000 sA 0000000000000000 sB 0000000000000000
t3 0000000000000000 t4 0000000000000078 t5 0000000000000000 t6 0000000000000000
pc 0000000000010198 va 000000008f000000 insn    ffffffff sr 8000000200046020
User store segfault @ 0x000000008f000000
```

接着，当你完成handle_illegal_instruction函数后，输入如下命令：

`$ riscv64-unknown-elf-gcc ../app/app1_2.c -o ../app/elf/app1_2`

`$ spike ./obj/pke app/elf/app1_2`

预期的输出如下：

```
PKE IS RUNNING
user mode test illegal instruction!
z 0000000000000000 ra 0000000000010162 sp 000000007f7ecb40 gp 0000000000013de8
tp 0000000000000000 t0 8805000503e80001 t1 0000000000000007 t2 0000219000080017
s0 000000007f7ecb50 s1 0000000000000000 a0 000000000000000a a1 0000000000014600
a2 0000000000000024 a3 0000000000000000 a4 0000000000000000 a5 0000000000000001
a6 0000000000000003 a7 0000000000000040 s2 0000000000000000 s3 0000000000000000
s4 0000000000000000 s5 0000000000000000 s6 0000000000000000 s7 0000000000000000
s8 0000000000000000 s9 0000000000000000 sA 0000000000000000 sB 0000000000000000
t3 0000000000000000 t4 000000005f195e48 t5 0000000000000000 t6 0000000000000000
pc 0000000000010162 va 0000000014005073 insn    14005073 sr 8000000200046020
An illegal instruction was executed!
```

如果你的两个测试app都可以正确输出的话，那么运行检查的python脚本：

`$./pke-lab1`

若得到如下输出，那么恭喜你，你已经成功完成了实验一！！！

```
build pk : OK
running app1 : OK
 test1 : OK
running app2 : OK
 test2 : OK
Score: 30/30
```

 

### 2.2 基础知识

**2.2.1 程序编译连接与ELF文件**

ELF的全称为Executable and Linkable Format，是一种可执行二进制文件。

在这里，我们仅仅之需要简单的了解一下ELF文件的基本组成原理，以便之后能够很好的理解内核可执行文件以及其它的一些ELF文件加载到内存的过程。首先，ELF文件可以分为这样几个部分：ELF文件头、程序头表（program header table）、节头表（section header table）和文件内容。而其中文件内容部分又可以分为这样的几个节：.text节、.rodata节、.stab节、.stabstr节、.data节、.bss节、.comment节。如果我们把ELF文件看做是一个连续顺序存放的数据块，则下图可以表明这样的一个文件的结构。

 <img src="pictures/fig2_1.png" alt="fig2_1" style="zoom:50%;" />

图2.1 ELF文件结构

从图2.1中可以看出ELF文件中需要读到内存的部分都集中在文件的中间，下面我们首先就介绍一下中间的这几个节的具体含义：

l .text节：可执行指令的部分。

l .rodata节：只读全局变量部分。

l .stab节：符号表部分。

l .stabstr节：符号表字符串部分，具体的也会在第三章做详细的介绍。

l .data节：可读可写的全局变量部分。

l .bss节：未初始化的全局变量部分，这一部分不会在磁盘有存储空间，因为这些变量并没有被初始化，因此全部默认为0，于是在将这节装入到内存的时候程序需要为其分配相应大小的初始值为0的内存空间。

l .comment节：注释部分，这一部分不会被加载到内存。

结合刚才的hellowrold.c文件分析，global_init作为初始化之后的全局变量, 存储在.data段中，而global_uninit作为为初始化的全局变量，存储在.bss段。函数中的临时变量tmp则不会被存储在ELF文件的数据段中。

在pke中，ELF文件头结构的定义如下：

```
typedef struct {
	uint8_t e_ident[16];   //ELF文件标识，包含用以表示ELF文件的字符
	uint16_t e_type;     //文件类型
	uint16_t e_machine;    //体系结构信息 
	uint32_t e_version;    //版本信息
	uint64_t e_entry;     //程序入口点
	uint64_t e_phoff;     //程序头表偏移量
	uint64_t e_shoff;           //节头表偏移量
	uint32_t e_flags;      //处理器特定标志
	uint16_t e_ehsize;     //文件头长度
	uint16_t e_phentsize;    //程序头部长度
	uint16_t e_phnum;     //程序头部个数
	uint16_t e_shentsize;    //节头部长度
	uint16_t e_shnum;     //节头部个数
	uint16_t e_shstrndx;     //节头部字符索引
} Elf64_Ehdr;
```

ELF文件头比较重要的几个结构体成员是e_entry、e_phoff、e_phnum、e_shoff、e_shnum。其中e_entry是可执行程序的入口地址，即从内存的这个闻之开始执行，在这里入口地址是虚拟地址，也就是链接地址；e_phoff和e_phnum可以用来找到所有的程序头表项，e_phoff是程序头表的第一项相对于ELF文件的开始位置的偏移，而e_phnum则是表项的个数；同理e_ shoff和e_ shnum可以用来找到所有的节头表项。

以例1.1为例，我们可以使用riscv64-unknown-elf-objdump工具，查看该ELF文件。

首先，使用-x选项查看显示整体的头部内容。

`$riscv64-unknown-elf-objdump -x  hellowrold >> helloworld.txt`

​    得到的输入文件helloworld.txt的主要内容如下：

```
hellowrold:   file format elf64-littleriscv
architecture: riscv:rv64, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x00000000000100c2

Program Header:
  LOAD off  0x0000000000000000 vaddr 0x0000000000010000 paddr 0x0000000000010000 align 2**12 filesz 0x000000000000258a memsz 0x000000000000258a flags r-x
  LOAD off  0x000000000000258c vaddr 0x000000000001358c paddr 0x000000000001358c align 2**12 filesz 0x0000000000000fb4 memsz 0x000000000000103c flags rw-
hellowrold:   file format elf64-littleriscv

Sections:
Idx Name     Size   VMA        LMA        File off Algn
 0 .text     000024cc 00000000000100b0 00000000000100b0 000000b0 2**1
         CONTENTS, ALLOC, LOAD, READONLY, CODE
 1 .rodata    0000000a 0000000000012580 0000000000012580 00002580 2**3
         CONTENTS, ALLOC, LOAD, READONLY, DATA
 2 .eh_frame   00000004 000000000001358c 000000000001358c 0000258c 2**2
         CONTENTS, ALLOC, LOAD, DATA
 3 .init_array  00000010 0000000000013590 0000000000013590 00002590 2**3
         CONTENTS, ALLOC, LOAD, DATA
 4 .fini_array  00000008 00000000000135a0 00000000000135a0 000025a0 2**3
         CONTENTS, ALLOC, LOAD, DATA
 5 .data     00000f58 00000000000135a8 00000000000135a8 000025a8 2**3
         CONTENTS, ALLOC, LOAD, DATA
 6 .sdata    00000040 0000000000014500 0000000000014500 00003500 2**3
         CONTENTS, ALLOC, LOAD, DATA
 7 .sbss     00000020 0000000000014540 0000000000014540 00003540 2**3
         ALLOC
 8 .bss     00000068 0000000000014560 0000000000014560 00003540 2**3
         ALLOC
 9 .comment   00000011 0000000000000000 0000000000000000 00003540 2**0
         CONTENTS, READONLY
 10 .riscv.attributes 00000035 0000000000000000 0000000000000000 00003551 2**0
         CONTENTS, READONLY
```

可以看到，解析出来的文件结构与图2.1的结构相对应。其中值得注意的是，.bss节与.comment节在文件中的偏移是一样的，这就说明.bss在硬盘中式不占用空间的，仅仅只是记载了它的长度。

Program Header：程序头表实际上是将文件的内容分成了好几个段，而每个表项就代表了一个段，这里的段是不同于之前节的概念，有可能就是同时几个节包含在同一个段里。程序头表项的数据结构如下所示：

```
typedef struct {
 uint32_t p_type;  //段类型
 uint32_t p_flags;  //段标志
 uint64_t p_offset;  //段相对于文件开始处的偏移量
 uint64_t p_vaddr;  //段在内存中地址（虚拟地址）
 uint64_t p_paddr;  //段的物理地址
 uint64_t p_filesz;  //段在文件中的长度
 uint64_t p_memsz; //段在内存中的长度
 uint64_t p_align;  //段在内存中的对齐标志
} Elf64_Phdr;
```

下面我们通过一个图来看看用ELF文件头与程序头表项如何找到文件的第i段。

 <img src="pictures/fig2_2.png" alt="fig2_2" style="zoom:50%;" />

图2.2 找到文件第i段的过程

Sections：而另一个节头表的功能则是让程序能够找到特定的某一节，其中节头表项的数据结构如下所示：

```
typedef struct {
 uint32_t sh_name;        //节名称
 uint32_t sh_type;         //节类型
 uint64_t sh_flags;        //节标志
 uint64_t sh_addr;        //节在内存中的虚拟地址
 uint64_t sh_offset;       //相对于文件首部的偏移
 uint64_t sh_size;       //节大小
 uint32_t sh_link;       //与其他节的关系
 uint32_t sh_info;       //其他信息
 uint64_t sh_addralign;    //字节对齐标志
 uint64_t sh_entsize;      //表项大小
} Elf64_Shdr;
```

而通过ELF文件头与节头表找到文件的某一节的方式和之前所说的找到某一段的方式是类似的。

  

**2.2.2** 代理内核与应用程序的加载

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
268    add sp, sp, a2
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
180         boot_loader(dtb);
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

​    在boot_loader中，经历设置中断入口地址，清零sscratch寄存器，关中断等一系列操作后。最后会调用enter_supervisor_mode函数正式切换至Supervisor模式。

```
204     void enter_supervisor_mode(void (*fn)(uintptr_t), uintptr_t arg0, uintptr_t arg1)
205   {
206         uintptr_t mstatus = read_csr(mstatus);
207      mstatus = INSERT_FIELD(mstatus, MSTATUS_MPP, PRV_S);
208      mstatus = INSERT_FIELD(mstatus, MSTATUS_MPIE, 0);
209         write_csr(mstatus, mstatus);
210      write_csr(mscratch, MACHINE_STACK_TOP() - MENTRY_FRAME_SIZE);
211        #ifndef __riscv_flen
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

​    在enter_supervisor_mode函数中，将 mstatus的MPP域设置为1，表示中断发生之前的模式是Superior，将mstatus的MPIE域设置为0，表示中段发生前MIE的值为0。随机将机器模式的内核栈顶写入mscratch寄存器中，设置mepc为rest_of_boot_loader的地址，并将kernel_stack_top与0作为参数存入a0和a1。

​    最后，执行mret指令，该指令执行时，程序从机器模式的异常返回，将程序计数器pc设置为mepc，即rest_of_boot_loader的地址；将特权级设置为mstatus寄存器的MPP域，即方才所设置的代表Superior的1，MPP设置为0；将mstatus寄存器的MIE域设置为MPIE，即方才所设置的表示中断关闭的0，MPIE设置为1。

​    于是，当mret指令执行完毕，程序将从rest_of_boot_loader继续执行。

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

​    这个函数中，我们对应用程序的ELF文件进行解析，并且最终运行应用程序。