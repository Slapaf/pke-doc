# 基于RISC-V代理内核的操作系统课程实验与课程设计



[前言](preliminary.md)

[第一章. RISC-V体系结构](chapter1_riscv.md)  
- [1.1 RISC-V发展历史](chapter1_riscv.md#history)  
- [1.2 RISC-V汇编语言](chapter1_riscv.md#assembly)  
- [1.3 机器的特权状态](chapter1_riscv.md#machinestates)  
- [1.4 中断和中断处理](chapter1_riscv.md#traps)  
- [1.5 页式虚存管理](chapter1_riscv.md#paging)  
- [1.6 什么是代理内核](chapter1_riscv.md#proxykernel)  
- [1.7 相关工具软件](chapter1_riscv.md#toolsoftware)  

[第二章. 实验环境配置与实验构成](chapter2_installation.md)  

 - [2.1 实验环境安装](chapter2_installation.md#environments)  
 - [2.2 riscv-pke（实验）代码的获取](chapter2_installation.md#preparecode)  
 - [2.3 PKE实验的组成](chapter2_installation.md#pke_experiemnts)  

[第三章. 实验1：系统调用、异常和外部中断](chapter3_traps.md)  
 - [3.1 实验1的基础知识](chapter3_traps.md#fundamental)   
 - [3.2 lab1_1 系统调用](chapter3_traps.md#syscall)  
 - [3.3 lab1_2 异常处理](chapter3_traps.md#exception)  
 - [3.4 lab1_3（外部）中断](chapter3_traps.md#irq)  

[第四章. 实验2：内存管理](chapter4_memory.md)  
 - [4.1 实验2的基础知识](chapter4_memory.md#fundamental)  
 - [4.2 lab2_1 虚实地址转换](chapter4_memory.md#lab2_1_pagetable)  
 - [4.3 lab2_2 简单内存分配和回收](chapter4_memory.md#lab2_2_allocatepage)  
 - [4.4 lab2_3 缺页异常](chapter4_memory.md#lab2_3_pagefault)  

[第五章. 实验3：进程管理](chapter5_process.md)  

 - [5.1 实验3的基础知识](chapter5_process.md#fundamental)  
 - [5.2 lab3_1 进程创建](chapter5_process.md#lab3_1_naive_fork)  
 - [5.3 lab3_2 进程yield](chapter5_process.md#lab3_2_yield)  
 - [5.4 lab3_3 循环轮转调度](chapter5_process.md#ilab3_3_rrsched)  



第六章. 实验4：设备管理



第七章. 实验5：文件系统



