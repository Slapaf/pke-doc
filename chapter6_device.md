# 第六章．实验4：设备管理


### 目录  
- [6.1 实验4的基础知识](#fundamental)  
  - [6.1.1  内存映射I/O(MMIO)](#subsec_MMIO) 
  - [6.1.2  轮询I/O控制方式](#subsec_polling)
  - [6.1.3  中断驱动I/O控制方式](#subsec_plic)  
  - [6.1.4  设备树](#subsec_device_tree)  
- [6.2 lab4_1 POLL](#polling) 
  - [给定应用](#lab4_1_app)
  - [实验内容](#lab4_1_content)
  - [实验指导](#lab4_1_guide)
- [6.3 lab4_2_PLIC](#PLIC) 
  - [给定应用](#lab4_2_app)
  - [实验内容](#lab4_2_content)
  - [实验指导](#lab4_2_guide)
- [6.4 lab4_3_hostdevice](#hostdevice) 
  - [给定应用](#lab4_3_app)
  - [实验内容](#lab4_3_content)
  - [实验指导](#lab4_3_guide)

<a name="fundamental"></a>

## 6.1 实验4的基础知识

完成前面所有实验后，PKE内核的整体功能已经得到完善。在实验四的设备实验中，我们将结合fpga-pynq板，在rocket chip上增加uart模块和蓝牙模块，并搭载PKE内核，实现蓝牙通信控制智能小车，设计设备管理的相关实验。

<a name="subsec_MMIO"></a>

### 6.1.1 内存映射I/O(MMIO)

内存映射(Memory-Mapping I/O)是一种用于设备驱动程序和设备通信的方式，它区别于基于I/O端口控制的Port I/O方式。RICSV指令系统的CPU通常只实现一个物理地址空间，这种情况下，外设I/O端口的物理地址就被映射到CPU中单一的物理地址空间，成为内存的一部分，CPU可以像访问一个内存单元那样访问外设I/O端口，而不需要设立专门的外设I/O指令。

在MMIO中，内存和I/O设备共享同一个地址空间。MMIO是应用得最为广泛的一种IO方法，它使用相同的地址总线来处理内存和I/O设备，I/O设备的内存和寄存器被映射到与之相关联的地址。当CPU访问某个内存地址时，它可能是物理内存，也可以是某个I/O设备的内存。此时，用于访问内存的CPU指令就可以用来访问I/O设备。每个I/O设备监视CPU的地址总线，一旦CPU访问分配给它的地址，它就做出响应，将数据总线连接到需要访问的设备硬件寄存器。为了容纳I/O设备，CPU必须预留给I/O一个地址映射区域。

用户空间程序使用mmap系统调用将IO设备的物理内存地址映射到用户空间的虚拟内存地址上，一旦映射完成，用户空间的一段内存就与IO设备的内存关联起来，当用户访问用户空间的这段内存地址范围时，实际上会转化为对IO设备的访问。

<a name="subsec_polling"></a>

### 6.1.2  轮询I/O控制方式

在实验四中，我们设备管理的主要任务是控制设备与内存的数据传递，具体为从蓝牙设备读取到用户输入的指令字符（或传递数据给蓝牙在手机端进行打印），解析为小车前、后、左、右、停止等动作来传输数据给电机实现对小车的控制。在前两个实验中，我们分别需要对轮询控制方式和中断控制方式进行实现。

首先，程序直接控制方式（又称循环测试方式），每次从外部设备读取一个字的数据到存储器，对于读入的每个字，CPU需要对外设状态进行循环检查，直到确定该数据已经传入I/O数据寄存器中。在轮询的控制方式下，由于CPU的高速性和I/O设备的低速性，导致CPU浪费绝大多数时间处于等待I/O设备完成数据传输的循环测试中，会造成大量资源浪费。

轮询I/O控制方式流程如图：

<img src="pictures/fig6_1_polling.png" alt="fig6_1" style="zoom:100%;" />

<a name="subsec_plic"></a>

### 6.1.3  中断驱动I/O控制方式

在前一种轮询的控制方式中，由于没有采用中断机制，CPU需要不断测试I/O设备的状态，造成CPU资源的极大浪费。中断驱动的方式是，允许I/O设备主动打断CPU的运行并请求相应的服务，请求I/O的进程首先会进入阻塞状态，PLIC将字符读取操作转化为s态中断进行处理，向进程传递读取的数据后，唤醒进程继续运行。

采用中断驱动的控制方式，在I/O操作过程中，CPU可以执行其他的进程，CPU与设备之间达到了部分并行的工作状态，从而提升了资源利用率。

中断驱动I/O方式流程如图：

<img src="pictures/fig6_2_plic.png" alt="fig6_2" style="zoom:100%;" />

<a name="subsec_device_tree"></a>

### 6.1.4 设备树

设备树（Device Tree）是描述计算机的特定硬件设备信息的数据结构，以便于操作系统的内核可以管理和使用这些硬件，包括CPU或CPU，内存，总线和其他一些外设。

硬件的相应信息都会写在`.dts`为后缀的文件中，`dtc`是编译`dts`的工具，`dtb(Device Tree Blob)`，`dts`经过`dtc`编译之后会得到`dtb`文件，`dtb`通过`Bootloader`引导程序加载到内核。所以`Bootloader`需要支持设备树才行；Kernel也需要加入设备树的支持。

<img src="pictures/fig6_3.png" alt="fig6_3" style="zoom:130%;" />

在rocketchip中，设备即通过设备树的方式提供给pke使用。

<a name="polling"></a>

## 6.2 lab4_1 POLL

<a name="lab4_1_app"></a>

#### **给定应用**
- user/app_poll.c

```
1	/*
2	 * Below is the given application for lab4_1.
3	 * The goal of this app is to control the car via Bluetooth. 
4	 */
5	
6	#include "user_lib.h"
7	#include "util/types.h"
8	
9	int main(void) {
10	  printu("please input the instruction through bluetooth!\n");
11	  while(1)
12	  {
13	    char temp = (char)uartgetchar();
14	    uartputchar(temp);
15	    switch (temp)
16	    {
17	      case '1' : gpio_reg_write(0x2e); break; //前进
18	      case '2' : gpio_reg_write(0xd1); break; //后退
19	      case '3' : gpio_reg_write(0x63); break; //左转
20	      case '4' : gpio_reg_write(0x9c); break; //右转
21	      case 'q' : exit(0);              break;
22	      default : gpio_reg_write(0x00); break;  //停止
23	    }
24	  }
25	  exit(0);
26	  return 0;
27	}
```

应用通过轮询的方式从蓝牙端获取指令，实现对小车的控制功能。

- 切换到lab4_1，继承lab3_3及之前实验所做的修改，并make后的直接运行结果：

```
//切换到lab4_1
$ git checkout lab4_1_poll

//继承lab3_3以及之前的答案
$ git merge lab3_3_rrsched -m "continue to work on lab4_1"

//重新构造
$ make clean; make

//运行构造结果
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_poll
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
please input the instruction through bluetooth!
You need to implement the uart_getchar function in lab4_1 here!

System is shutting down with exit code -1.

```

从结果上来看，蓝牙端端口获取用户输入指令的uartgetchar系统调用未完善，所以无法进行控制小车的后续操作。按照提示，我们需要实现蓝牙uart端口的获取和打印字符系统调用，以及传送驱动数据给小车电机的系统调用，实现对小车的控制。

<a name="lab4_1_content"></a>

#### **实验内容**

如输出提示所表示的那样，需要找到并完成对uartgetchar，uartputchar，gpio_reg_write的调用，并获得以下预期结果：

```
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_poll
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Ticks 0
please input the instruction through bluetooth!
Ticks 1
going to insert process 0 to ready queue.
going to schedule process 0 to run.
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab4_1_guide"></a>

#### **实验指导**

基于实验lab1_1，你已经了解和掌握操作系统中系统调用机制的实现原理。对于本实验的应用，我们发现user/app_poll.c文件中有三个函数调用：uartgetchar，uartputchar和gpio_reg_write。对代码进行跟踪，我们发现这三个函数都在user/user_lib.c中进行了实现，对应于lab1_1的流程，我们可以在kernel/syscall.h中查看新增的系统调用以及编号：

```
16	#define SYS_user_uart_putchar (SYS_user_base + 6)
17	#define SYS_user_uart_getchar (SYS_user_base + 7)
18	#define SYS_user_gpio_reg_write (SYS_user_base + 8)
```

继续追踪，我们发现在kernel/syscall.c的do_syscall函数中新增了对应系统调用编号的实现函数，对于新增系统调用，分别有如下函数进行处理：

```
133	    case SYS_user_uart_putchar:
134	      sys_user_uart_putchar(a1);return 1;
135	    case SYS_user_uart_getchar:
136	      return sys_user_uart_getchar();
137	    case SYS_user_gpio_reg_write:
138	      return sys_user_gpio_reg_write(a1);
```

读者的任务即为在kernel/syscall.c中追踪并完善对应的函数。对于uart的函数，我们给出uart端口的地址映射如图：

<img src="pictures/fig6_3_address.png" alt="fig6_3" style="zoom:80%;" />

我们可以看到配置uart端口的偏移地址为0x60000000，对应写地址为0x60000000，读地址为0x60000004，同时对0x60000008的状态位进行轮询，检测到信号时进行读写操作。

在kernel/syscall.c中找到函数实现空缺，并根据注释完成uart系统调用：

```
84	//add uart putchar getchar syscall
85	//
86	// implement the SYS_user_uart_putchar syscall
87	//
88	void sys_user_uart_putchar(uint8 ch) {
89	    volatile uint32 *status = (void*)(uintptr_t)0x60000008;
90	    volatile uint32 *tx = (void*)(uintptr_t)0x60000004;
91	    while (*status & 0x00000008);
92	    *tx = ch;
93	}
94	
95	ssize_t sys_user_uart_getchar() {
96	  // TODO (lab4_1): implment the syscall of sys_user_uart_getchar.
97	  // hint: the functionality of sys_user_uart_getchar is to get data from UART address. therefore,
98	  // we should let a pointer point, insert it in 
99	  // the rear of ready queue, and finally, schedule a READY process to run.
100	    panic( "You have to implement sys_user_uart_getchar to get data from UART using uartgetchar in lab4_1.\n" );
101	    
102	}
103	
104	
105	
106	//car control
107	ssize_t sys_user_gpio_reg_write(uint8 val) {
108	    volatile uint32_t *control_reg = (void*)(uintptr_t)0x60001004;
109	    volatile uint32_t *data_reg = (void*)(uintptr_t)0x60001000;
110	    //*control_reg = 0;
111	    *data_reg = (uint32_t)val;
112	    return 1;
113	}
114	
```

和uart端口读写过程类似，其中电机连接端口gpio数据地址为0x60001000，根据用户程序app_poll中流程，我们需要将uart端口读到的驱动数据传递给电机。

安卓手机端验证：首先将HC-05蓝牙模块接入pynq板，接口对应关系为：

| pynq接口 | HC-05接口 |
| -------- | --------- |
| VCC      | VCC       |
| GND      | GND       |
| JA4      | RXD       |
| JA3      | TXD       |

接入时注意对应接口错位正确插入，然后在手机端下载BluetoothSerial，连接hc-05蓝牙模块，使用网线连接pynq板和电脑，打开开发板电源。

成功连接蓝牙模块后，启动连接：

```
$ ssh xilinx@192.168.2.99
```

随后使用scp指令将编译后的pke内核和用户app文件导入：

```
$ scp 文件名 xilinx@192.168.2.99:~
```

此时便成功进入pynq板环境，可对结果进行验证。

**实验完毕后，记得提交修改（命令行中-m后的字符串可自行确定），以便在后续实验中继承lab4_1中所做的工作**：

```
$ git commit -a -m "my work on lab4_1 is done."
```



<a name="PLIC"></a>

## 6.3 lab4_2_PLIC 

<a name="lab4_2_app"></a>

#### **给定应用**

- user/app_PLIC.c

```
1	/*
2	 * Below is the given application for lab4_2.
3	 * The goal of this app is to control the car via Bluetooth. 
4	 */
5	
6	#include "user_lib.h"
7	#include "util/types.h"
8	void delay(unsigned int time){
9	  unsigned int a = 0xfffff ,b = time;
10	  volatile unsigned int i,j;
11	  for(i = 0; i < a; ++i){
12	    for(j = 0; j < b; ++j){
13	      ;
14	    }
15	  }
16	}
17	int main(void) {
18	  printu("Hello world!\n");
19	  int i;
20	  int pid = fork();
21	  if(pid == 0)
22	  {
23	    while (1)
24	    {
25	      delay(3);
26	      printu("waiting for you!\n");
27	    }
28	    
29	  }
30	  else
31	  {
32	    for (;;) {
33	      char temp = (char)uartgetchar();
34	      printu("%c\n", temp);
35	      switch (temp)
36	      {
37	        case '1' : gpio_reg_write(0x2e); break; //前进
38	        case '2' : gpio_reg_write(0xd1); break; //后退
39	        case '3' : gpio_reg_write(0x63); break; //左转
40	        case '4' : gpio_reg_write(0x9c); break; //右转
41	        case 'q' : exit(0);              break;
42	        default : gpio_reg_write(0x00); break;  //停止
43	      }
44	    }
45	  }
46	  
47	
48	  exit(0);
49	
50	  return 0;
51	}
```

应用通过中断的方式从蓝牙端获取指令，实现对小车的控制功能。

- 切换到lab4_2，继承lab4_1及之前实验所做的修改，并make后的直接运行结果：

```
//切换到lab4_2
$ git checkout lab4_2_PLIC

//继承lab4_1以及之前的答案
$ git merge lab4_1_poll -m "continue to work on lab4_2"

//重新构造
$ make clean; make

//运行构造结果
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_poll
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
please input the instruction through bluetooth!
You need to implement the uart_getchar function in lab4_2 here!

System is shutting down with exit code -1.

```



<a name="lab4_2_content"></a>

#### **实验内容**

如输出提示所表示的那样，需要找到并完成对uartgetchar，do_sleep，getuartvalue的调用，并获得以下预期结果：

```
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_polling
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
Ticks 0
please input the instruction through bluetooth!
Ticks 1
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

<a name="lab4_2_guide"></a>

#### **实验指导**

对于本实验的应用，我们需要在lab4_1基础上实现基于中断的uartgetchar。对代码进行跟踪，我们可以在kernel/syscall.h中查看新增的系统调用以及编号：

```
16	#define SYS_user_uart_putchar (SYS_user_base + 6)
17	#define SYS_user_uart_getchar (SYS_user_base + 7)
18	#define SYS_user_gpio_reg_write (SYS_user_base + 8)
```

继续追踪，我们发现在kernel/syscall.c的do_syscall函数中新增了对应系统调用编号的实现函数，对于新增系统调用，分别有如下函数进行处理：

```
139	    case SYS_user_uart_getchar:
140	      return sys_user_uart_getchar();
```

你的任务即为在kernel/syscall.c中追踪并完善对应的函数。

在kernel/syscall.c中找到函数实现空缺，并根据注释完成uart系统调用：

```
88	//
89	// implement the uart syscall
90	//
103	ssize_t sys_user_uart_getchar() {
104	    panic( "You need to implement the uart_getchar function in lab4_2 here.\n" );
105	    //sleep
106	    
107	    //Wait for wake
108	    
109	    //get the value
110	    
111	    //Return the result character
112	    
113	    
114	}
```

当蓝牙有数据发送时，pke会收到外部中断，你需要完成接收到外部中断后的处理。

在kernel/strap.c中找到函数空缺，并根据注释完成中断处理函数：

```
100     case CAUSE_MEXTERNEL_S_TRAP:
101       {
102         panic( "You need to complete case CAUSE_MEXTERNEL_S_TRAP function in lab4_2 here.\n"
103         int irq = *(uint32 *)0xc201004L;
104         *(uint32 *)0xc201004L = irq;
105         volatile int *ctrl_reg = (void *)(uintptr_t)0x6000000c;
106         *ctrl_reg = *ctrl_reg | (1 << 4);
107         
108         // get the data from MMIO.
109         // send it to the process.
110         // call function to awake process[0]
111         
112         
113         
114         break;
115       }
```



在kernel/process.c中找到函数实现空缺，并根据注释完成do_sleep函数：

```
221 void do_sleep(){
222   panic( "You need to implement do_sleep function in lab4_2 here.\n"
223   // set the process BLOCKED.
224 }
```

在kernel/process.c中找到函数实现空缺，并根据注释完成do_wake函数：

```
226 void do_wake(){
227   panic( "You need to implement do_sleep function in lab4_2 here.\n"
228   //set the process READY.
229   //insert_to_ready_queue
230   
231   //schedule
232 }
```



**实验完毕后，记得提交修改（命令行中-m后的字符串可自行确定），以便在后续实验中继承lab4_2中所做的工作**：

```
$ git commit -a -m "my work on lab4_2 is done."
```

<a name="hostdevice"></a>

## 6.4 lab4_3 

<a name="lab4_3_app"></a>

#### **给定应用**
- user/app_host_device.c

```
1	 #pragma pack(4)
2	#define _SYS__TIMEVAL_H_
3	struct timeval {
4	    unsigned int tv_sec;
5	    unsigned int tv_usec;
6	};
7	
8	#include "user_lib.h"
9	#include "videodev2.h"
10	#define DARK 64
11	#define RATIO 7 / 10
12	
13	int main() {
14	    char *info = allocate_share_page();
15	    int pid = do_fork();
16	    if (pid == 0) {
17	        int f = do_open("/dev/video0", O_RDWR), r;
18	
19	        struct v4l2_format fmt;
20	        fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
21	        fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
22	        fmt.fmt.pix.width = 320;
23	        fmt.fmt.pix.height = 180;
24	        fmt.fmt.pix.field = V4L2_FIELD_NONE;
25	        r = do_ioctl(f, VIDIOC_S_FMT, &fmt);
26	        printu("Pass format: %d\n", r);
27	
28	        struct v4l2_requestbuffers req;
29	        req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
30	        req.count = 1; req.memory = V4L2_MEMORY_MMAP;
31	        r = do_ioctl(f, VIDIOC_REQBUFS, &req);
32	        printu("Pass request: %d\n", r);
33	
34	        struct v4l2_buffer buf;
35	        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
36	        buf.memory = V4L2_MEMORY_MMAP; buf.index = 0;
37	        r = do_ioctl(f, VIDIOC_QUERYBUF, &buf);
38	        printu("Pass buffer: %d\n", r);
39	
40	        int length = buf.length;
41	        char *img = do_mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_SHARED, f, buf.m.offset);
42	        unsigned int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
43	        r = do_ioctl(f, VIDIOC_STREAMON, &type);
44	        printu("Open stream: %d\n", r);
45	
46	        char *img_data = allocate_page();
47	        for (int i = 0; i < (length + 4095) / 4096 - 1; i++)
48	            allocate_page();
49	        yield();
50	
51	        for (;;) {
52	            if (*info == '1') {
53	                r = do_ioctl(f, VIDIOC_QBUF, &buf);
54	                printu("Buffer enqueue: %d\n", r);
55	                r = do_ioctl(f, VIDIOC_DQBUF, &buf);
56	                printu("Buffer dequeue: %d\n", r);
57	                r = read_mmap(img_data, img, length);
58	                int num = 0;
59	                for (int i = 0; i < length; i += 2)
60	                    if (img_data[i] < DARK) num++;
61	                printu("Dark num: %d > %d\n", num, length / 2 * RATIO);
62	                if (num > length / 2 * RATIO) {
63	                    *info = '0'; gpio_reg_write(0x00);
64	                }
65	            } else if (*info == 'q') break;
66	        }
67	
68	        for (char *i = img_data; i - img_data < length; i += 4096)
69	            free_page(i);
70	        r = do_ioctl(f, VIDIOC_STREAMOFF, &type);
71	        printu("Close stream: %d\n", r);
72	        do_munmap(img, length); do_close(f); exit(0);
73	    } else {
74	        yield();
75	        for (;;) {
76	            char temp = (char)uartgetchar();
77	            printu("From bluetooth: %c\n", temp);
78	            *info = temp;
79	            switch (temp) {
80	                case '1': gpio_reg_write(0x2e); break; //前进
81	                case '2': gpio_reg_write(0xd1); break; //后退
82	                case '3': gpio_reg_write(0x63); break; //左转
83	                case '4': gpio_reg_write(0x9c); break; //右转
84	                case 'q': exit(0); break;
85	                default: gpio_reg_write(0x00); break;  //停止
86	            }
87	        }
88	    }
89	    return 0;
90	}
```

该用户程序包含两个进程，其中主进程和实验4_2类似，负责接收蓝牙发送过来的数据，根据数据控制小车行动（前进、后退、左转、右转、停止）；子进程则负责拍摄和分析，首先初始化摄像头设备，然后是个死循环判断摄像头拍摄的图像数据：如果当前小车处于前进状态，则拍摄，然后检查数据，如果判断前面有障碍物则控制车轮停转（刹车），否则如果主进程退出了，则自己进行释放文件、内存、关闭设备等操作，再退出。在用户程序操控摄像头的过程中，使用了ioctl、mmap、munmap等系统调用，需对其进行完善从而实现小车的障碍识别和停止功能。

- 切换到lab4_3、继承lab4_2中所做修改，并make后的直接运行结果：

```
//切换到lab4_2
$ git checkout lab4_2_PLIC

//继承lab3_3以及之前的答案
$ git merge lab4_2_PLIC -m "continue to work on lab4_2"

//重新构造
$ make clean; make

//运行构造结果
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 512 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x0000000080010000, PKE kernel size: 0x0000000000010000 .
free physical memory address: [0x0000000080010000, 0x000000008003ffff] 
kernel memory manager is initializing ...
kernel pagetable addr is 0x000000008003e000
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080005000
kernel page table is on 
Switching to user mode...
in alloc_proc. user frame 0x0000000080039000, user stack 0x000000007ffff000, user kstack 0x0000000080038000 
User application is loading.
Application: app_PLIC
CODE_SEGMENT added at mapped info offset:3
Application program entry point (virtual address): 0x00000000810000de
going to insert process 0 to ready queue.
going to schedule process 0 to run.
please input the instruction through bluetooth!
You need to implement the uart_getchar function in lab4_3 here!

System is shutting down with exit code -1.

```



<a name="lab4_3_content"></a>

####  实验内容

如应用提示所表示的那样，读者需要找到并完成对ioctl的调用，使得用户能够设置设备参数，从而控制摄像头实现拍照等功能；获取图片后，检查数据，从而判断前方是否出现障碍物。

跟踪相关系统调用，在kernel/file.c里可以看到需要补充的函数：

```
25	int do_open(char *pathname, int flags) {
26	    // TODO (lab4_3): call host open through spike_file_open and then bind fd to spike_file
27	    // hint: spike_file_dup function can bind spike_file_t to an int fd.
28	    panic( "You need to finish open function in lab4_3.\n" );
29	}
```

```
39	int do_ioctl(int fd, uint64 request, char *data) {
40	    // TODO (lab4_3): call host ioctl through frontend_sycall
41	    // hint: fronted_syscall ioctl argument:
42	    // 1.call number
43	    // 2.fd
44	    // 3.the order to device
45	    // 4.data address
46	    panic( "You need to call host's ioctl by frontend_syscall in lab4_3.\n" );
47	    return frontend_syscall(HTIFSYS_ioctl, spike_file_get(fd)->kfd,
48	            request, (uint64)data, 0, 0, 0, 0);
49	}
```

实验预期结果：小车在前进过程中能够正常识别障碍物后并自动停车。

<a name="lab4_3_guide"></a>

#### 实验指导

##### 摄像头控制

USB摄像头最基础的控制方法是使用读写设备文件的方式。拍摄一张照片包含以下过程：

* 打开设备文件，使用open函数
* 设置设备参数，使用ioctl函数
* 映射内存，由于USB摄像头对应的设备文件不支持直接用read函数进行读写，所以需要用mmap函数将文件映射到一段虚拟地址，通过虚拟地址进行读写
* 拍摄，使用ioctl函数控制
* 结束和清理，包含使用ioctl函数关闭设备，使用munmap函数解映射，使用close函数关闭设备文件

其中ioctl、mmap、munmap三个函数是PKE和riscv-fesvr不支持的，需要在本设计中添加，因此本设计的重点包含以下三个内容：

* 对riscv-fesvr的修改：riscv-fesvr是PS端（Arm）的一个程序，用于控制PL端（Riscv）程序的启动以及和通信。通过riscv-fesvr，PL端上的程序也可以访问PS端的文件，调用一些PS端系统的函数。原版的riscv-fesvr不支持ioctl和mmap等函数，而操控USB摄像头的用户程序必须使用这些函数，所以需要对riscv-fesvr进行修改，使得PL端上运行的PKE和用户程序能够通过riscv-fesvr这个中间层调用宿主机的系统函数从而控制摄像头；
* 对PKE内核代码的修改：需要为riscv-fesvr新增的函数调用提供用户层接口。
* 对用户代码的修改：有了对fesvr和内核的修改，用户程序就可以调用各类系统调用函数操控摄像机了。为了实现避障的功能，程序还需要对获得的图片信息进行解码和分析，根据前方是否为障碍物选择是否刹车。

##### 图片解析

应用第21行可以看到，我们从摄像头获取的数据是YUYV格式，读者可进行查阅，它用灰度、蓝色色度、红色色度三个属性表示颜色，每个像素点都有灰度属性。由于我们分析障碍物只需要灰度图，所以取每个像素点的灰度属性即可。

因此对于获取过来的数据删去奇数索引的数据，就可以得到灰度图。对于障碍物的判断，我们使用了一个比较简单的算法：计算灰度小于64的像素点个数，如果个数大于像素点总数的7/10，即认为前方是障碍物。

**注意：对于灰度阈值的设定可根据环境亮度进行一定的调整，可以先根据摄像头返回的图像进行分析，计算出对应障碍物的灰度值；灰度阈值越精确，小车对于障碍物的识别将越灵敏，并能在合理的距离内识别到障碍物并停车。**

**实验完毕后，记得提交修改（命令行中-m后的字符串可自行确定），以便在后续实验中继承lab4_3中所做的工作**：

```
$ git commit -a -m "my work on lab4_3 is done."
```

