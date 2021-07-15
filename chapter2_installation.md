# 第二章．实验环境的安装与使用

### 目录  
- [2.1 头歌平台](#educoder)  
- [2.2 Ubuntu环境](#ubuntu)  
- [2.3 openEuler操作系统](#openeuler)  


<a name="educoder"></a>
## 2.1 头歌平台

PKE实验在[头歌平台](https://www.educoder.net/)上进行了部署，但因为仍在测试阶段，所以没有开放全局选课（感兴趣的读者可以尝试邀请码：2T8MA）。PKE实验（2.0版本）将于2021年秋季在头歌平台重新上线，届时将开放全局选课。

<img src="pictures/fig2_install_1.png" alt="fig2_install_1" style="zoom:80%;" />

图1.1 头歌课程界面。

头歌平台为每个选课的学生提供了一个docker虚拟机，该虚拟机环境中已经配置好了所有开发套件（包括交叉编译器、Spike模拟器等），用户可以通过shell选项（*详细使用方法将待2.0上线时更新*）进入该docker环境在该docker环境中完成实验任务。


<a name="ubuntu"></a>
## 2.2 Ubuntu环境

实验环境我们推荐采用Ubuntu 16.04LTS或18.04LTS（x86_64）操作系统，我们未在其他系统（如arch，RHEL等）上做过测试，但理论上只要将实验中所涉及到的安装包替换成其他系统中的等效软件包，就可完成同样效果。另外，我们在EduCoder实验平台（网址：https://www.educoder.net ）上创建了本书的同步课程，课程的终端环境中已完成实验所需软件工具的安装，所以如果读者是在EduCoder平台上选择的本课程，则可跳过本节的实验环境搭建过程，直接进入通过终端（命令行）进入实验环境。

PKE实验涉及到的软件工具有：RISC-V交叉编译器、spike模拟器，以及PKE源代码三个部分。假设读者拥有了Ubuntu 16.04LTS或18.04LTS（x86_64）操作系统的环境，以下分别介绍这三个部分的安装以及安装后的检验过程。需要说明的是，为了避免耗时耗资源的构建（build）过程，一个可能的方案是从https://toolchains.bootlin.com 下载，**但是要注意一些依赖包（如GCC）的版本号**。

**我们强烈建议读者在新装环境中完整构建（build）RISC-V交叉编译器，以及spike模拟器**。如果强调环境的可移植性，可以考虑在虚拟机中安装完整系统和环境，之后将虚拟机进行克隆和迁移。

### 2.1.1 RISC-V交叉编译器

RISC-V交叉编译器是与Linux自带的GCC编译器类似的一套工具软件集合，不同的是，x86_64平台上Linux自带的GCC编译器会将源代码编译、链接成为适合在x86_64平台上运行的二进制代码（称为native code），而RISC-V交叉编译器则会将源代码编译、链接成为在RISC-V平台上运行的代码。后者（RISC-V交叉编译器生成的二进制代码）是无法在x86_64平台（即x86_64架构的Ubuntu环境下）直接运行的，它的运行需要模拟器（我们采用的spike）的支持。

一般情况下，我们称x86_64架构的Ubuntu环境为host，而在host上执行spike后所虚拟出来的RISC-V环境，则被称为target。RISC-V交叉编译器的构建（build）、安装过程如下：

● 第一步，安装依赖库

RISC-V交叉编译器的构建需要一些本地支撑软件包，可使用以下命令安装：

`$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev device-tree-compiler`

● 第二步，获取RISC-V交叉编译器的源代码

有两种方式获得RISC-V交叉编译器的源代码：一种是通过源代码仓库获取，使用以下命令：

`$ git clone --recursive https://github.com/riscv/riscv-gnu-toolchain.git`

但由于RISC-V交叉编译器的仓库包含了Qemu模拟器的代码，下载后的目录占用的磁盘空间大小约为4.8GB，（从国内下载）整体下载所需的时间较长。为了方便国内用户，我们提供了另一种方式就是通过百度云盘获取源代码压缩包，链接和提取码如下：

`链接: https://pan.baidu.com/s/1cMGt0zWhRidnw7vNUGcZhg 提取码: qbjh`

从百度云盘下载RISCV-packages/riscv-gnu-toolchain-clean.tar.gz文件（大小为2.7GB），再在Ubuntu环境下解压这个.tar.gz文件，采用如下命令行：

`$ tar xf  riscv-gnu-toolchain-clean.tar.gz`

之后就能够看到和进入当前目录下的riscv-gnu-toolchain文件夹了。

● 第三步，构建（build）RISC-V交叉编译器

`$ cd riscv-gnu-toolchain`

`$ ./configure --prefix=[your.RISCV.install.path]`

`$ make`

以上命令中，[your.RISCV.install.path]指向的是你的RISC-V交叉编译器安装目录。如果安装是你home目录下的一个子目录（如~/riscv-install-dir），则最后的make install无需sudoer权限。但如果安装目录是系统目录（如/opt/riscv-install-dir），则需要sudoer权限（即在make install命令前加上sudo）。

● 第四步，设置环境变量

`$ export RISCV=[your.RISCV.install.path]`

`$ export PATH=$PATH:$RISCV/bin`

以上命令设置了RISCV环境变量，指向在第三步中的安装目录，并且将交叉编译器的可执行文件所在的目录加入到了系统路径中。这样，我们就可以在PKE的工作目录调用RISC-V交叉编译器所包含的工具软件了。

### 2.1.2 spike模拟器

接下来，安装spkie模拟器。首先取得spike的源代码，有两个途径：一个是从github代码仓库中获取：

`$ git clone https://github.com/riscv/riscv-isa-sim.git`

也可以从百度云盘中下载spike-riscv-isa-sim.tar.gz文件（约4.2MB），然后用tar命令解压缩。百度云盘的地址，以及tar命令解压缩可以参考2.1.1节RISC-V交叉编译器的安装过程。获得spike源代码或解压后，将在本地目录看到一个riscv-isa-sim目录。

接下来构建（build）spike，并安装：

`$ cd riscv-isa-sim`

`$ ./configure --prefix=$RISCV`

`$ make`

`$ make install`

在以上命令中，我们假设RISCV环境变量已经指向了RISC-V交叉编译器的安装目录。如果未建立关联，可以将$RISCV替换为2.1.1节中的[your.RISCV.install.path]。

### 2.1.3 PKE

到github下载课程仓库：

`$ git clone https://github.com/MrShawCode/pke.git`

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

### 2.1.4 环境测试

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

<a name="openeuler"></a>
## 2.3 openEuler操作系统

PKE实验将提供基于华为openEuler操作系统的开发方法，*具体的华为云使用方法待续*，但在openEuler操作系统环境中的交叉编译器安装方法，以及其他环节都可参考[2.2 Ubuntu环境](#ubuntu)的命令进行。 
