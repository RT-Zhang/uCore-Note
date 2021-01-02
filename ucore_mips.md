# ucore on MIPS

## 实验内容

- 了解操作系统启动前的状态和要做的准备工作，完成 bootloader 代码编写；
- 了解运行操作系统所需要的硬件支持以及操作系统是如何加载到内存中的；
- 理解和实现 MIPS 的中断异常处理机制；
- 通过实现`printf`函数的串口输出了解 MIPS 对 I/O 的设定以及具体串口是如何实现的；
- 了解和实现 MIPS 的物理内存分配和虚拟内存映射。

## 实验思路

通过 QEMU 模拟 MIPS 硬件环境来验证最终移植的 ucore 能够得到正确地运行。希望借助移植 ucore 到 MIPS 架构的实验，去更好地了解 MIPS 架构以及移植系统的步骤和要考虑的问题。

参考了 QEMU的源码实现。

- 根据 MIPS 启动操作系统前需要完成的工作来实现 bootloader；
- 将操作系统中原本与 x86 相关的宏定义替换为 MIPS 相关的宏定义（该部分宏定义的文件是从 QEMU 源码中 u-boot 部分借用的），比如对 MIPS 寄存器的命名，宏定义某些特定的内存地址以及实现了 MIPS架构独有的操作）；
- 根据 MIPS 手册对虚拟内存地址映射的规定，修改了操作系统的虚拟内存分布结构来满足体系结构；
- 针对 QEMU 对于外设的具体实现，操作系统进行了相应的调整；
- 由于 x86 和 MIPS 对于中断异常的处理方式存在差异，操作系统对此进行更改。

## MIPS简介

MIPS 是一种采取精简指令集（RISC）的指令集架构，广泛被使用在许多电子产品、网络设备、个人娱乐设备与商业设备上。本文的所有实验都是在 MIPS32 Release1 的架构下进行，后续如果没有特殊说明，MIPS32都是指 MIPS32 Release1。

### RISC 与 CISC

RISC

- 由于RISC提供的指令要少于CISC，同时针对这些指令做了高度的优化，比如下面将要提到的Load/Store结构，因此RISC的流水线相对CISC简单很多。通常有指令译码以及执行流水线组成，流水线设计也相对简单，整体吞吐量理论上可以达到 1 IPC。这一点应该是RISC比CISC快的原因之一。
- RISC这样做也是有代价的：首先由于指令较少，相对于CISC，应用的一些操作需要更多的指令来完成。所以，通常RISC的应用要大于CISC平台下的应用，即需要更多的内存。当然，现在技术的发展，内存成本的下降，一定程度上缓解了这个问题。其次，RISC应用对编译器的要求很高，需要编译器能够完成复杂的优化，才能获得高性能的应用。

CISC

- 而CISC指令复杂，很多指令无法通过硬件直接完成，需要编译成微码，然后再执行微码。这样每条指令执行的时间不是固定的，流水线设计起来复杂。Intel的CPU，每个ALU前面有一个微码的Cache，尽可能减少指令解码到微码的成本，前提是在软件设计上，开发人员尽可能高效的利用一段代码处理多组数据，完成后再执行后面的操作，这样一组数据就可以将微码译指的时间分摊掉。
- Intel CPU有更加强大的分支预测、乱序执行等逻辑模块，用于提高微码的执行性能，但是占用了宝贵的片上面积。
- 由于有复杂的指令集，相对给编译器压力减少，更能够编译出高性能的应用程序。同时，应用程序占用的内存也要少于RISC。这样，CPU与内存的取指交互要好一些，对于iTLB的要求也会低一些

精简指令集架构的目的是通过精简机器指令系统来减少硬件设计的复杂程度，提高指令执行速度。

MIPS 是 load/store 架构（也叫 register-register 架构），除了 load/store 指令可以存取内存，所有的指令都在寄存器之间操作。

### 寄存器

MIPS 有32个通用寄存器

| 寄存器名  |  Number   | 用途                                                         |
| :-------: | :-------: | :----------------------------------------------------------- |
|  \$zero   |    \$0    | 永远为0                                                      |
|   \$at    |    \$1    | assembler temporary                                          |
| \$v0–\$v1 |  \$2–\$3  | 存放一个子程序的非浮点运算的结果或返回值，如果这两个寄存器不够存放需要返回的值，编译器将会通过内存来完成 |
| \$a0–\$a3 |  \$4–\$7  | 传递子函数调用时前4个非 浮点参数                             |
| \$t0–\$t7 | \$8–\$15  | 依照约定，一个子函数可以不用保存并随便地使用这些寄存器       |
| \$s0–\$s7 | \$16–\$23 | 子函数必须保证当函数返回时这些寄存器的内容必须恢复到函数调用以前的值 |
| \$t8–\$t9 | \$24–\$25 | 依照约定，一个子函数可以不用保存并随便地使用这些寄存器       |
| \$k0–\$k1 | \$26–\$27 | 被 OS 的异常或中断处理程序使用，被使用后将**不会恢复**原来的值。 |
|   \$gp    |   \$28    | global pointer                                               |
|   \$sp    |   \$29    | stack pointer                                                |
|   \$fp    |   \$30    | frame pointer                                                |
|   \$ra    |    $31    | 永远存放着正常函数调用指令(jal)的返回地址                    |

MIPS 里没有状态码，CPU 状态寄存器或内部都不包含任何用户程序计算的结果状态信息。

关于 \$k0 和 \$k1 不会恢复的含义。MIPS 有个叫异常程序计数器（exception program counter,EPC) 的寄存器，属于 CP0 寄存器，用于保存造成异常的指令的地址。查看控制寄存器的方法是把它复制到通用寄存器里，指令 mfc0(move from system control) 可以将 EPC 中的地址复制到某个通用寄存器中，通过跳转语句（jr)，程序可以返回到造成异常的那条指令处继续执行。但是，为了查看 EPC 的值并跳转到造成异常的那条指令，必须把 EPC 的值复制到某个通用寄存器中，这样的话，程序返回到中断处时就无法将所有的寄存器恢复原值。如果先恢复所有的寄存器，那么从 EPC 复制过来的值就会丢失，jr 就无法返回中断处；如果我们只是恢复除有从 EPC 复制过来的返回地址外的寄存器，但这意味着程序在异常情况后某个寄存器被无端改变了，这是不行的。为了解决这个两难境地，MIPS 程序员必须保留两个寄存器 \$k0 和 \$k1，供操作系统使用。发生异常时，这两个寄存器的值不会被恢复，编译器也不使用 \$k0 和 \$k1，异常处理函数可以将返回地址放到这两个中的任何一个，然后使用 jr 跳转到造成异常的指令处继续执行。

### MIPS 指令

| 指令 | 功能                                                         |
| ---- | ------------------------------------------------------------ |
| b    | Unconditional Branch，基于当前指令地址的无条件相对跳转       |
| bal  | Branch and Link，无条件跳转                                  |
| bgez | Branch on Greater Than or Equal to Zero                      |
| jal  | Jump and Link，直接跳转指令，并带有链接功能，指令的跳转地址在指令中，跳转发生时要把返回地址存放到 R31 这个寄存器中 |
| jalr | Jump and Link Register，使用寄存器的跳转指令，并且带有链接功能，指令的跳转地址在寄存器中，跳转发生时指令的放回地址放在 R31 这个寄存器中 |
| la   | Load Address，将一个地址（标签）存入一个寄存器               |
| li   | Load Immediate，加载常数立即数指令                           |
| lw   | Load Word，从存储器中读取一个字的数据到寄存器中              |
| mtc0 | Move to Coprocessor 0，把一个通用寄存器的数据复制到协处理器0的寄存器 |
| nop  | No Operation，空操作                                         |

变量声明

```assembly
name：storage_type value（s）
```

例如

```assembly
var1: .word 3  # 创建一个初始值为 3 的整数变量
array1: .byte 'a', 'b'  # 创建一个元素初始化的 2 元素字符数组到 a 和 b
array2: .space 40  # 分配 40 个连续字节, 未初始化的空间可以用作 40 个元素的字符数组, 或者 10 个元素的整数数组
```

### 协处理器

在 MIPS 体系结构中，最多支持 4 个协处理器（Co-Processor）。其中，协处理器 CP0 是体系结构中必须实现的。它起到控制 CPU 的作用，MMU、异常处理、乘除法等功能，都依赖于协处理器 CP0 来实现。它是 MIPS 的精髓之一，也是打开 MIPS 特权级模式的大门。

MIPS 的 CP0 包含 32 个寄存器，讨论常见的一些寄存器：

* Count，一个计数器，计数频率是系统主频的 1/2（每两个系统时钟周期，Count 会增加 1）。对于操作系统来说，可以通过读取该寄存器的值来获取 tick 的时基。在系统性能测试中，利用该寄存器也可以实现打点计数。
* Compare，配合 Count 使用。当 Compare 和 Count 的值相等的时候，会触发一个硬件中断 (Hardware Interrupt)。
  * 常使用 Cause 寄存器的 IP7 位。这个特性经常用来为操作系统提供一个可靠的 tick 时脉。
* Status，用于处理器状态的控制。Bit 22 为 BEV，控制异常向量的位置。当 BEV 位等于 0 的时候，如果实现了 CP0 EBase 寄存器，异常入口为 CP0 EBase 寄存器中储存的地址。
  * 若 BEV 位为 1，异常入口为内核起始地址 0x80000000，接下来由软件来进行异常处理操作；
  * EXL，异常级别。为1时硬件/软件中断被关闭；
  * IE 位，使能中断。0 关闭中断，1 开启中断。
* Cause，在处理器异常发生时，这个寄存器标识出了异常的原因。
  * 最重要的是从 Bit2 到 Bit6，5 个 Bit 的 Excetion Code 位。它们标识出了引起异常的原因。
* Exception Program Counter，存放异常发生时，系统正在执行的指令的地址。
* EBase
  * 当 Status 的 BEV 位为 0 时，存放异常向量的基址
* WatchLo/WatchHi，这对寄存器用于设置硬件数据断点 (Hardware Data Breakpoint)。该断点一旦设定，当 CPU 存取这个地址时，系统就会发生一个异常。这个功能广泛应用于调试定位内存写坏的错误。

### 虚拟内存

MIPS 虚拟地址空间分为五个部分，

| 段    | 是否 Mapped | 是否 Cached |
| ----- | ----------- | ----------- |
| kseg3 | Mapped      | Cached      |
| ksseg | Mapped      | Cached      |
| kseg1 | Unmapped    | Uncached    |
| kseg0 | Unmapped    | Cached      |
| useg  | Mapped      | Cached      |

* useg (TLB-mapped cacheable user space, 0x00000000-0x7fffffff)：用户模式下可用的地址，大小为 2G，也就是 MIPS 约定的用户内存空间。在这段空间中的地址，需要通过 MMU 进行虚拟地址到物理地址的转换，并且对该地址内数据的读取、写入经过 cache。
* kseg0（direct-mapped cached kernel space,0x80000000-0x9fffffff）：内核地址，其内存虚存地址到物理内存地址的映射转换不通过 MMU，使用时只需要将地址的最高位清零 (&0x7fffffff)，这些地址就被转换为物理地址。也就是说，这段逻辑地址被连续地映射到物理内存开始的 512M 空间。对这段地址的存取都会通过高速缓存 (cache)，通常在没有 MMU 的系统中，这段空间用于存放大多数程序和数据；对于有 MMU 的系统，**操作系统的内核**会存放在这个区域。
* kseg1（direct-mapped uncached kernel space,0xa0000000-0xbfffffff）：与 kseg0 类似，这段地址也是内核地址，将该段虚拟地址的高 3 位清零 (&0x1fffffff)，就可以直接转换到物理地址。kseg0 和 kseg1 同样映射到同样的物理空间，区别只是在于是否经过 cache。因为 kseg1 不使用缓存 (uncached)，所以访问速度比较慢。但是对于外设来说，选择 kseg1 段的地址也就不存在 cache 一致性的问题了，同时因为 I/O 和 cache 读写的速度存在严重差距，所以这段内存通常用来被映射到 **I/O 寄存器**，实现对外设的访问。
* ksseg & kseg3（TLB-mapped cacheable kernel space, 0xc0000000-0xffffffff）：这段地址只能在内核态下使用，并且需要 MMU 地址映射来转换。

kseg1 段是unmapped 且 uncached 的，是唯一一个在系统刚上电的时候就能正常工作的地址空间，所以系统启动和复位时的入口点在这个区域内。

## 实验工具

### QEMU

常用命令参数

-kernel bzImage，使用 bzImage 作为内核镜像，直接使用指定的内核文件，而无需将内核提前安装在磁盘映像中。所以QEMU支持通过 -kernel 选项直接加载 ELF 格式的内核到内存中，所以 bootloader 可以不包含从磁盘中读取 ELF 内核、分析 ELF文件格式、确定 ELF文件的 entry point、加载内核到内存中等功能。

-bios file，指定一个文件为 BIOS。

-M ?，查看支持的具体机器类型以及默认机型。

-serial stdio 将 QEMU 从串口接收到的数据指定输出到标准输入/输出流。

### GCC

-g或-ggdb，在编译生成的可执行文件中加入调试信息，包括源代码行号到目标代码地址的映射信息，以及程序中用到的所有函数、变量、类型和数据结构的描述信息。

-Ttext org，将 org 作为 text 段的起始地址，org 需要使用十六进制。

## 常用宏

利用 C 语言的宏可以简化程序设计，这里对 MIPS 寄存器的名字定义一些宏，包括通用寄存器和 CP0 寄存器。

```c
/* 
 * Symbolic register names for 32 bit ABI
 */
#define zero	$0	/* wired zero */
#define AT	$1	/* assembler temp  - uppercase because of ".set at" */
#define v0	$2	/* return value */
#define v1	$3
#define a0	$4	/* argument registers */
#define a1	$5
#define a2	$6
#define a3	$7
#define t0	$8	/* caller saved */
#define t1	$9
...
#define t7	$15
#define s0	$16	/* callee saved */
#define s1	$17
...
#define s7	$23
#define t8	$24	/* caller saved */
#define t9	$25
#define jp	$25	/* PIC jump register */
#define k0	$26	/* kernel scratch */
#define k1	$27
#define gp	$28	/* global pointer */
#define sp	$29	/* stack pointer */
#define fp	$30	/* frame pointer */
#define s8	$30	/* same like fp! */
#define ra	$31	/* return address */

/*
 * Coprocessor 0 register names
 */
#define CP0_INDEX $0
...
#define CP0_COUNT $9
#define CP0_ENTRYHI $10
#define CP0_COMPARE $11
#define CP0_STATUS $12
#define CP0_CAUSE $13
#define CP0_EPC $14
#define CP0_PRID $15
#define CP0_CONFIG $16
#define CP0_LLADDR $17
#define CP0_WATCHLO $18
#define CP0_WATCHHI $19
...
```

## 操作系统的启动

bootloader 参考了 QEMU源码中的 start.S 文件（roms/u-boot-sam460ex/arch/mips/cpu）。start.S 主要完成了对某些必要的 MIPS CP0 寄存器进行初始化的功能，以及为中断异常处理向量保留物理空间和初始化 cache。本次实验只针对 MIPS32 架构并且不涉及到 cache。

清除 CP0 WatchLo 和 CP0 WatchHi 寄存器，以及 CP0 Cause 寄存器，来消除 MIPS CPU 的监视功能；同时将 MIPS CPU 与时钟计数相关的两个 CP0 寄存器 Count 和 Compare 清空，避免在系统启动的时候，在意料之外的情况下发生了时钟中断导致系统运行混乱。

```assembly
.set noreorder

#define KSTACKSIZE (4096 * 2)

kernel_entry:
  b reset
  nop

	.align 4
reset:
	/* Clear watch registers */
	mtc0	zero, CP0_WATCHLO
	mtc0	zero, CP0_WATCHHI

	/* WP(Watch Pending), SW0/1 should be cleared. */
	mtc0	zero, CP0_CAUSE

	/* Init Timer */
	mtc0	zero, CP0_COUNT
	mtc0	zero, CP0_COMPARE
```

WatchLo/WatchHi这对寄存器可以用来设定“内存硬件断点”，也就是对指定点的内存进行监测。当访问的内存地址和这两个寄存器中地址一致时，会发生一个异常。Count是一个计数器，每两个系统时钟周期，Count会增加1，而当它的值和Compare相等时，会发生一个硬件中断(Hardware Interrupt)。Count/Compare这对寄存器的特性经常用来为操作系统提供一个可靠的tick时脉。所以在启动时，要先将它们重置为零。

```assembly
	/* Initialize $gp */
	bal	1f
	nop
	.word	_gp
1:
	lw	gp, 0(ra)
	la $sp, bootstacktop
# setup ram exception
    la $t0, __exception_vector
    /* Init EBase */
    mtc0 $t0, $15, 1
    ...
    .section .data
    .global bootstack
    bootstack:
    .space KSTACKSIZE
    .global bootstacktop
    bootstacktop:
    .space 32
```

初始化 gp，sp 寄存器，以及将异常处理向量基址填入 CP0 EBase 寄存器中。

C语言中有两种存储类型，自动型和静态型，自动变量是一个过程中的局部变量。静态变量是进入和退出一个过程时都是存在的。为了简化静态数据的访问，MIPS 软件保留了一个寄存器：全局指针 gp(global pointer, \$gp)，如果没有全局指针，从静态数据去装入数据需要两条指令：一条由编译器和链接器计算的32位地址常量中的有效位；另一条才真正装入数据。全局指针指向静态数据区中的运行时决定的地址，在存取位于gp值上下32KB范围内的数据时，只需要一条以gp为基指针的指令即可。在编译时，数据须在以 gp 为基指针的 64KB 范围内。

MIPS 硬件并不直接支持堆栈，例如，它没有 x86 的 SS、SP 和 BP 寄存器，MIPS 虽然定义 \$sp 为栈指针，它还是通用寄存器，只是用于特殊目的而已，可以把它用于别的目的，但为了使用别人的程序或让别人使用你的程序，还是要遵守这个约定的。x86 有单独的 PUSH 和 POP 指令，而 MIPS 没有，但这并不影响 MIPS 使用堆栈。在发生过程调用时，调用者把过程调用过后要用的寄存器压入堆栈，被调用者把返回地址寄存器 \$ra 和保留寄存器压入堆栈。同时调整堆栈指针，当返回时，从堆栈中恢复寄存器，同时调整堆栈指针。

> 关于 mtc0 指令，实际有 3 个参数，第三个参数代表 select，缺省值为 0。CP0 的 Register 15 Select 1 即为 EBase 寄存器。

对 CP0 Config 寄存器进行了初始化，禁止 CPU 使用 cache，防止出现意料之外的 cache读写错误导致系统运行混乱。在完成所有初
始化以后，利用 jal 指令跳转到内核的起始位置。

```assembly
# disable kernel mode cache
    mfc0 t0, CP0_CONFIG
    and t0, ~0x7
    ori t0, 0x2
    mtc0 t0, CP0_CONFIG
    
    la t9, kern_start
    jal t9
    nop

# never here
    bgez zero, .
    nop	
```

关于对 Config 寄存器的操作，先与 ~0x00000007 再或 0x00000002，将其前三位置为 010，表示禁止 cache。

关于`jal t9`，这里采用了寄存器进行跳转。因为如果采取立即数地址跳转的办法，会面临跳转范围不够的问题（因为地址
空间4G，如果是使用32位的立即数进行，就能将地址空间全部覆盖。但是实际上一条32位的指令中除去立即数还有指令的操作码等信息，所以立即数达不到32位）。当内核的起始位置在跳转范围外而又强行使用指令跳转的时候，编译器会强制修改地址，导致最终无法跳转到正确的内核起始位置，所以在此处需要利用寄存器（32bit）进行地址跳转。

根据 MIPS32 手册规定，系统上电以后 PC 寄存器默认被置为0xbfc00000，也就是意味着 CPU 从虚拟地址0xbfc00000开始读取指令。这
里需要通过 gcc 的编译选项 -Ttext 将 bootloader 的代码链接到地址0xbfc00000上。

由于通过 MMU 映射访问的地址必须在虚拟内存映射机制建立后才可以使用，而这是由操作系统内核来管理的。因此内核只能载入到不需要 MMU 的内存空间中，即 kseg0 和 kseg1这两个 unmapped 的段。其中 kseg1 是 uncached 的，一般是用于访问外部设备。所以内核应该被加载到 kseg0 段地址（0x80000000-0x9fffffff）上，这里选择 0x8000000。最终的可执行文件是由连接器 ld 生成的，因此可以通过使用 ld 链接文件将内核的代码链接到0x80000000处，并且在 ld 链接文件中使用`ENTRY()`指定内核代码的起始函数为`kern_init`。之后在
qemu-system-mips 启动的时候使用`-kernel`选项就能加载内核。

通过 qemu 的`-M ?`命令，查看 qemu-system-mips 支持的机器类型以及默认机型。

```bash
$ qemu-system-mips -machine ?
Supported machines are:
malta                MIPS Malta Core LV (default)
mips                 mips r4k platform
mipssim              MIPS MIPSsim platform
none                 empty machine
```

其中 mipssim 是 MIPS Technologies 公司编写的专门针对 MIPS 的仿真软件，它是一个相对简单的系统，只模拟了两个 UART 8250 标准的串口和一个简单的网络控制器。本次实验选择的具体模拟平台为 mipssim，在启动 qemu-system-mips 时需要加上选项`-M mipssim`。

## printf 串口输出

为了防止内存地址空间过于碎片化，MIPS保留了一块公共设备内存映射地址空间（CDMM，The Common Device Memory Map）将 MIPS处理器中的所有 I/O 设备寄存器集中映射到这块区域，同时对这些 I/O 设备寄存器同样有访问权限和内存地址转换机制。MIPS 指令集中没有设置专门的对 I/O 读取或存储数据的指令，对于 I/O 地址和内存地址采用的是统一编址。也就是说 MIPS 将 I/O 寄存器的地址映射到虚拟内存中的某一部分，在 MIPS计算机上操作系统对于 I/O 的访问就是选择对适当的存储器地址进行数据读写。

因为 I/O 设备的响应速度和传输数据的速率原本就不快，如果对外设的地址访问还涉及操作系统的分页/分段，也就是软件层面的虚实地址转换，那么就更加影响操作系统对于 I/O 的访问。在 MIPS 的虚拟内存地址空间中，kseg1 段虚拟地址的数据读写不经过 map 和 cache，MIPS 选用该段虚拟地址作为 I/O 地址的映射。只要知道外设的物理地址，在代码中加上 0xA0000000 就是外设的虚拟地址，这样访问起来速度比较快。缺点是灵活性比较差，外设只能映射在 512MB 的地址空间中，受到的限制比较多。

ucore关于串口的实现，就是在参考 mipssim.c(qemu-3.1.1.1/hw/mips)的基础上，根据 mipssim 对于串口的实现来完成的。在 mipssim.c中可以看到对于 ISA BASE 以及串口地址和串口中断请求的实现。mipssim.c部分关键代码如下：

```c
static void
mips_mipssim_init(MachineState *machine) {   
    /* Register 64 KB of ISA IO space at 0x1fd00000. */
    memory_region_init_alias(isa, NULL, "isa_mmio",
                             get_system_io(), 0, 0x00010000);
    memory_region_add_subregion(get_system_memory(), 0x1fd00000, isa);

    /* A single 16450 sits at offset 0x3f8. It is attached to
       MIPS CPU INT2, which is interrupt 4. */
    if (serial_hd(0))
        serial_init(0x3f8, env->irq[4], 115200, serial_hd(0),
                    get_system_io());
}
```

可以看出 mipssim 将 ISA IO 地址空间的起点设为 0x1fd0000，根据前面提到的虚拟映射规则，物理地址0x1fd00000是由虚拟地址0xbfd00000 直接抹去地址最高三位映射得来的。虚拟地址0xbfd00000在虚拟内存分配中属于 kseg1段，符合前面对于 I/O地址在虚拟内存分布空间的分析。

在 mipssim 中，将串口接受数据的端口寄存器映射到 ISA BASE 偏移 0x3f8 的虚拟地址上，即操作系统往虚拟地址 0xbfd003f8 上传输数据，QEMU 就能通过串口接受到相对应的数据。

因为 printf 是直接往串口上，也就是往指定地址上传输数据，为了方便采用 MIPS 内联汇编来实现操作系统对于指定地址数据的读取和写入。同时为了最小地改动原本 ucore 在 x86 架构下的代码，对 x86 的 I/O 特殊读取、写入指令（inb、outb）进行了重新定义。具体 ucore 对于 inb 和 outb 的实现代码如下：

```c
static inline uint8_t
inb(uint16_t port) {
    uint8_t data = *((volatile uint8_t*) port);
    return data;
}

static inline void
outb(uint16_t port, uint8_t data) {
    *((volatile uint8_t*) port) = data;
}
```

被 volatile 关键字声明的变量，编译器对其访问时代码就不进行任何优化，从而可以提供对特殊地址的稳定访问。如果不使用，则编译器将对所声明的语句进行优化，可能产生一些意料之外的改变。

操作系统需要根据 UART 8250 的串口标准来对串口进行输出传输，否则操作系统和 QEMU 的串口标准不一致会导致 QEMU 接受到的信息为乱码。

```c
static void
serial_init(void) { 
    // Turn off the FIFO
    outb(COM1 + COM_FCR, 0);

    // Set speed; requires DLAB latch
    outb(COM1 + COM_LCR, COM_LCR_DLAB);
    outb(COM1 + COM_DLL, (uint8_t) (115200 / 9600));
    outb(COM1 + COM_DLH, 0);

    // 8 data bits, 1 stop bit, parity off; turn off DLAB latch
    outb(COM1 + COM_LCR, COM_LCR_WLEN8 & ~COM_LCR_DLAB);

    // No modem controls
    outb(COM1 + COM_MCR, 0);
    // Enable rcv interrupts
    outb(COM1 + COM_IER, COM_IER_RDI);
}
```

FCR(FIFO Control Register)寄存器相对较新，并不是最初的 8250 UART 的一部分。随着 Windows NT以及各种 UNIX 这类的多任务操作系统进入个人电脑，8250 每个字符触发一次的高频中断成为了问题，因此16550(A)内置了 16 字节的 FIFO 数据缓存器用于收集收到的数据。在串口初始化时先将其关闭。

需要先将 LCR(Line Control Register) 寄存器的 DLAB(Divisor Latch Access Bit) 位置 1，才可以设置速度（也就是baud rate）。再通过 DLL(Divisor Latch Low Byte) 和 DLH(Divisor Latch High Byte) 两个寄存器设置速度的低位和高位。顾名思义，divisor 意味着这个值将作为除数来决定传输速率，DLB的值与传输速率遵循以下关系。
$$
DivisorLatchValue=\frac{115200}{BaudRate}
$$
所以这里实际上将传输速率为 9600。

设置每个字传输 8 比特，1 个停止比特（stop bit）。这里的停止比特其实是指时间间隔，在 50 baud 情况下，1 bit 的传输耗时 1s/50 = 20ms。关闭奇偶校验位，也就是代码注释中的 parity。最后恢复 DLAB 的值为 0

设置 IER 寄存器的 RDI 位为 1。IER(Interrupt Enable Register)寄存器用于控制中断，RDI 负责开启 Received Data Available Interrupt，这个中断告知我们有数据可以从 UART 获取。

## 异常处理

### 异常向量

```assembly
#define RVECENT(f,n) \
	b f; nop
#define XVECENT(f, n) \
	b f; nop
	
.extern ramReserved
.extern ramExcHandle_tlbmiss
.extern ramExcHandle_general

.set noreorder
.global __exception_vector
__exception_vector:
    RVECENT(ramExcHandle_tlbmiss,0)
    RVECENT(ramReserved,1)
    RVECENT(ramReserved,2)
    ...
    RVECENT(ramReserved,47)
    XVECENT(ramExcHandle_general, 0x180)
    RVECENT(ramReserved,49)
    ...
    RVECENT(ramReserved,126)
    RVECENT(ramReserved,127)
.global __exception_vector_end
__exception_vector_end:
/* table end */
/*
* We hope there are no more reserved vectors!
* 128 * 8 == 1024 == 0x400
* so this is address R_VEC+0x400 == 0xbfc00400
*/
```

RVECENT 和 XVECENT 这两个宏实际上都是无条件跳转。

### 异常处理流程

* 硬件部分
  * 设置CP0 EPC寄存器，指向异常处理结束后CPU继续执行的指令位置；
  * 设置 CP0 Status寄存器的EXL位，使CPU进入kernel态，并禁止中断；
  * 设置Cause寄存器，以使软件可以得到异常的原因；
  * CPU开始从异常入口取指令，当 CP0 Status寄存器的 BEV位等于1时，异常入口固定为0xbfc00000；当 BEV位等于0的时候，如果实现了 CP0 EBase寄存器，异常入口为 CP0 EBase寄存器中储存的地址，否则为内核起始地址0x80000000，以后的事情都交由软件处理
* 软件部分
  * 保存寄存器现场。在堆栈中开辟一段区域，将32个通用寄存器和CP0的相关寄存器。从CP0到内存的数据传输必须通过通用寄存器。一般地，编程时的约定是使用k0和k1这两个寄存器暂存；
  * 异常处理。根据 Cause寄存器的 Exc Code值来判断发生的异常的具体类型，并执行相应的异常处理程序；
  * 恢复现场，将栈中被保存的寄存器值写回；
  * 从异常返回，执行 eret指令，清除 CP0 Status寄存器 EXL位（开启中断），跳转到 CP0 EPC中保存的地址。

```assembly
# +0x000: R4000 tlbmiss vector
	.global ramExcHandle_tlbmiss
ramExcHandle_tlbmiss:
    b ramExcHandle_general
    nop
    
	.global ramReserved
ramReserved:
    b ramReserved
    nop

# +0x180: R4000 general vector
    .global ramExcHandle_general
    .type ramExcHandle_general, @function
    .ent ramExcHandle_general
ramExcHandle_general:
    move k1, sp /* Save previous stack pointer in k1 */
    mfc0 k0, CP0_CAUSE /* Now, load the exception cause. */
    
    addiu sp, sp, -164
    sw ra, 160(sp)
    sw s8, 156(sp)
    sw sp, 152(sp)
    sw gp, 148(sp)
    mfc0 k1, CP0_EPC /* Copr.0 reg 13 == PC for exception */
    sw k1, 160(sp)	/* real saved PC */
    sw t9, 136(sp)
    sw t8, 132(sp)
   	...
    sw AT, 40(sp)
    sw ra, 36(sp)
    /*
    * Save remaining exception context information.
    */
    sw k0, 24(sp) /* k0 was loaded with cause earlier*/
    mfc0 t1, CP0_STATUS /* Copr.0 reg 11 == status */
    sw t1, 20(sp)
    /*
    * Prepare to call mips_trap(struct trapframe *)
    */
    addiu a0, sp, 16 /* set argument */
    la t9, mips_trap
    jal t9 /* call it */
    nop /* delay slot */
    
    /*
 	* Now restore stuff and return from the exception.
 	* Interrupts should be off.
 	*/  
    /*     16(sp)		   no need to restore tf_vaddr */
    lw t0, 20(sp)		/* load status register value into t0 */
    ori t0, t0, ST0_EXL
    nop				/* load delay slot */
    mtc0 t0, CP0_STATUS /* store it back to coprocessor 0 */
    /*     24(sp)		   no need to restore tf_cause */

    /* restore special registers */
    lw t1, 28(sp)
    lw t0, 32(sp)
    mtlo t1
    mthi t0

    /* load the general registers */
    lw ra, 36(sp)

    lw AT, 40(sp)
    lw v0, 44(sp)
    ...
    lw t8, 132(sp)
    lw t9, 136(sp)
    /*     140(sp)		   "saved" k0 was dummy garbage anyway */
    /*     144(sp)		   "saved" k1 was dummy garbage anyway */
    lw gp, 148(sp)		/* restore gp */
    /*     152(sp)		   stack pointer - below */
    lw s8, 156(sp)		/* restore s8 */
    lw k0, 160(sp)		/* fetch exception return PC into k0 */
    lw sp, 152(sp)		/* fetch saved sp (must be last) */
    /* done */
    mtc0 k0, CP0_EPC /* Copr.0 reg 13 == PC for exception */
    nop
    eret
```

MIPS 的 trapframe 定义如下，注意结构体的定义要符合异常处理汇编代码中的参数压栈顺序。

```c
/* $1 - $30 */
struct pushregs {
    uint32_t reg_r[30];
};
/*
 * Structure describing what is saved on the stack during entry to
 * the exception handler.
 * This must agree with the code in exception.S.
 */
struct trapframe {
	uint32_t tf_vaddr;	/* coprocessor 0 vaddr register */
	uint32_t tf_status;	/* coprocessor 0 status register */
	uint32_t tf_cause;	/* coprocessor 0 cause register */
	uint32_t tf_lo;
	uint32_t tf_hi;
	uint32_t tf_ra;	/* Saved register 31 */
    struct pushregs tf_regs;
	uint32_t tf_epc;	/* coprocessor 0 epc register */
};
```

异常分发，读取 CP0 Cause 寄存器的 ExcCode，根据 ExcCode 的不同执行不同的异常处理。

```c
static void trap_dispatch(struct trapframe *tf) {
    int code = GET_CAUSE_EXCODE(tf->tf_cause);
    switch(code) {
        case EX_IRQ:
            interrupt_handler(tf);
            break;
        ...
        case EX_OVF:
            print_trapframe(tf);
            if (trap_in_kernel(tf)) {
                panic("Ov exception occur in kernel mode!");
            }
            do_exit(-E_KILLED);
            break;
        case EX_SYS:
            print_trapframe(tf);
            tf->tf_epc += 4;
            syscall();
            break;
        ...
        default:
            print_trapframe(tf);
            panic("Unhandled Exception");
    }
}
```

同样的，打印 trapframe 的函数也要在 x86 版本上进行改动。

```c
void
print_trapframe(struct trapframe *tf) {
    cprintf("trapframe at %p\n", tf);
    print_regs(&tf->tf_regs);
    cprintf("  $ra      0x----%x\n", tf->tf_ra);
    cprintf("  BadVA    0x----%x\n", tf->tf_vaddr);
    cprintf("  Status   0x----%x\n", tf->tf_status);
    cprintf("  Cause    0x----%x\n", tf->tf_cause);
    cprintf("  EPC      0x----%x\n", tf->tf_epc);
    cprintf(trapname(GET_CAUSE_EXCODE(tf->tf_cause)));
    cputchar('\n');
}

void print_regs(struct pushregs *regs) {
    int i;
    for (i = 0; i < 30; i++) {
        cprintf(" $%d", i + 1);
        cprintf("\t: 0x%x", regs->reg_r[i]);
        cputchar('\n');
    }
}
```

提供 MIPS 各种异常的信息

```c
static const char *
trapname(int trapno) {
    static const char * const excnames[] = {
        "Interrupt",
        ...
        "Syscall",
        "Breakpoint",
        ...
    };
    if (trapno <= 12)
        return excnames[trapno];
    else
        return "Unknown";
}
```

### 开启/关闭中断

在操作系统运行的某些情况下，需要进行必要的开关中断，防止在中断异常处理的时候发生新的中断（不支持嵌套中断）。此时操作系统需要对 CP0 的 Status 寄存器进行读取，然后对读取的数据某些位进行改写，再将值写入Status 寄存器。此时 CPU 读取 CP0 Status 寄存器，根据操作系统修改后的值来判断是否开启中断。对于 CP0 Status 寄存器的读写需要借用 asm volatile 内联汇编来直接进行指定的操作，

```c
#define get_status(x) __asm volatile("mfc0 %0,$12" : "=r" (x))
#define set_status(x) __asm volatile("mtc0 %0,$12" :: "r" (x))

/* intr_enable - enable irq interrupt */
void intr_enable(void) {
    uint32_t x;
    get_status(x);
    x |= ST0_IE;
    set_status(x);
}

/* intr_disable - disable irq interrupt */
void intr_disable(void) {
    uint32_t x;
    get_status(x);
    x &= ~ST0_IE;
    set_status(x);
}
```

开启中断，是通过将 Status 的值按位或 0x00000001（即 Status 寄存器的第 0 位，置为 1 时中断开启），将 Status 的最后一位置为 1，再写入 Status 寄存器。关闭中断，是通过将 Status 的值按位与 ~0x00000001，将 Status 的最后一位置为 0，再写入 Status 寄存器。

