# LAB 1

对应的 OS 原理

* 连续内存的管理
* 中断机制

## 练习1：make 生成执行文件

### 练习1.1：镜像文件ucore.img如何生成

#### Makefile学习

##### 基本格式

```makefile
<target> : <prerequisites> 
[tab]  <commands>
```

##### 变量定义

```makefile
VARIABLE = value
# 在执行时扩展，允许递归扩展。
VARIABLE := value
# 在定义时扩展。
VARIABLE ?= value
# 只有在该变量为空时才设置值。
VARIABLE += value
# 将值追加到变量的尾端。
```

##### 函数

foreach

```makefile
$(foreach <var>,<list>,<text>)
```

把参数 `<list>` 中的单词逐一取出放到参数 `<var>` 所指定的变量中，然后再执行 `<text>` 所包含的表达式。每一次 `<text>` 会返回一个字符串，循环过程中， `<text>` 的所返回的每个字符串会以空格分隔，最后当整个循环结束时， `<text>` 所返回的每个字符串所组成的整个字符串（以空格分隔）将会是foreach函数的返回值。

#### 分析

```bash
# 显示make执行了哪些命令
make V=
```

gcc把c源代码编译为o目标文件，ld把目标文件转化为out执行程序，dd（用于读取、转换并输出数据）把bootloader放到虚拟硬盘

分析Makefile代码

```makefile
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

需要kernel和bootblock作为prerequisites

##### bootblock

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

观察bootfiles变量，它是boot目录下的所有c和S文件，即bootasm.S和bootmain.c

调用toobj，得到**bootasm.o**和**bootmain.o**，调用totarget，得到**bin/sign**，这三者为bootblock的prerequisites。

bootasm.o由bootasm.S编译而来，具体命令为

```makefile
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
	-c boot/bootasm.S -o obj/boot/bootasm.o
```

-ggdb  生成可供gdb使用的调试信息
-m32  生成适用于32位环境的代码
-gstabs  生成stabs格式的调试信息
-nostdinc  不使用标准库
-fno-stack-protector  不生成用于检测缓冲区溢出的代码
-Os  为减小代码大小而进行优化
-I<dir>  添加**搜索头文件**的路径。使用#include "file"的时候, gcc会先在当前目录查找头文件，如果没有找到，回到默认的头文件目录找。如果使用 -I 制定了目录，会先在指定的目录查找，然后再按常规的顺序去找

bootmain.o由bootmain.c编译而来

```makefile
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
	-fno-stack-protector -Ilibs/ -Os -nostdinc \
	-c boot/bootmain.c -o obj/boot/bootmain.o
```

-fno-builtin  除非用__builtin_前缀，否则不进行builtin函数的优化

对bin/sign，查看调用的函数

```makefile
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

```bash
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

链接生成bootblock.o

```bash
ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

-m <emulation>  模拟为i386上的连接器
-nostdlib  不使用标准库
-N  设置代码段和数据段均可读写
-e <entry>  指定入口
-Ttext  制定代码段开始位置，这里就是0x7C00

拷贝二进制代码bootblock.o到bootblock.out（一般o代表中间文件）

```bash
objcopy -S -O binary obj/bootblock.o obj/bootblock.out
```

-S  移除所有符号和重定位信息
-O <bfdname>  指定输出格式，这里是二进制

使用sign工具处理bootblock.out，生成bootblock

```bash
bin/sign obj/bootblock.out bin/bootblock
```

##### kernel

```bash
KOBJS	= $(call read_packet,kernel libs)

# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

kernel.ld已存在

```bash
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```

需要kern文件夹下的各种c文件编译为o文件，举一例

```bash
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
	-gstabs -nostdinc  -fno-stack-protector \
	-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
	-Ikern/trap/ -Ikern/mm/ 
	-c kern/init/init.c -o obj/kern/init/init.o
```

链接成bin/kernel

```bash
ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
	obj/kern/init/init.o obj/kern/libs/readline.o \
	obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
	...
```

-T <scriptfile>  链接器使用指定的脚本，会替代默认的

##### dd命令

```bash
dd if=/dev/zero of=bin/ucore.img count=10000
# /dev/zero是一个输入设备，可用来初始化文件。该设备无穷尽地提供0，可以用于向设备或文件写入字符串0。
# /dev/null，空设备，也称为位桶（bit bucket）。任何写入它的输出都会被抛弃。如果不想让消息以标准输出显示或写入文件，那么可以将消息重定向到位桶。
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
# notrunc，不截开输出文件
# seek=1，在输出文件跳过1*obs个块（obs默认为512）
```

所以这是先用0填充一个有10000个块的文件，每个块默认512字节。从bin中写入，先把bootblock中的内容写到第一个块，从第二个块开始写kernel中的内容。

### 练习1.2：被系统认为是符合规范的硬盘主引导扇区的特征

分析tools/sign.c文件，查看特征标记

在Makefile中sign这样被使用

```bash
bin/sign obj/bootblock.out bin/bootblock
# obj/bootblock.out bin/bootblock作为argv参数
```

sign.c关键代码

```c
char buf[512];
memset(buf, 0, sizeof(buf));
FILE *ifp = fopen(argv[1], "rb");
int size = fread(buf, 1, st.st_size, ifp);

fclose(ifp);
buf[510] = 0x55;
buf[511] = 0xAA;
FILE *ofp = fopen(argv[2], "wb+");
size = fwrite(buf, 1, 512, ofp);
```

512字节buf，先从obj/bootblock.out读取st_size大小，再把倒数第二个字节设置为0x55，最后一个字节设置为0xAA，最后全部写入bin/bootblock。

## 练习2：使用qemu执行并调试lab1中的软件

### 练习2.1：从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行

## 练习3：bootloader进入保护模式的过程

为什么使用保护模式？更好的保护机制和更大的寻址空间

* 实模式：分段，指针都是指向实际的物理地址，所以用户程序的一个指针如果指向了操作系统区域或其他用户程序区域，可以修改内容。通过修改A20地址线可以完成从实模式到保护模式的转换。
* 保护模式：分段/分页。提供4个特权级和完善的特权检查机制，既能实现资源共享又能保证代码数据的安全及任务的隔离。

从0x7C00开始执行，将flag置0和将段寄存器置0

```assembly
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment
    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

### A20

8086通过segment:offset寻址，寻址空间最大为0ffff0h + 0ffffh = 10ffefh ≈ 1088KB > 1MB，当寻址到超过1MB的内存时，会发生“回卷”。而下一代的寻址空间为16MB，也有保护模式，可以访问到1MB以上的内存了，此时如果遇到“寻址超过1MB”的情况，系统不会再“回卷”了 。为了保持向下兼容，防止“回卷”，进入A20 Gate。

查阅手册，写Output Port先向64h发送0d1h命令，然后向60h写入Output Port的数据。

```assembly
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

### 初始化GDT表

全局描述符表（Global Descriptor Table），段表可以包含8192 (2^13)个描述符，每个段最大空间为2^32字节。看起来很大，但实际上段并不能扩展物理地址空间，很大程度上各个段的地址空间是相互重叠的。没有太多实际意义。

静态储存在引导区中，直接载入：

```assembly
lgdt gdtdesc
```

### 进入保护模式

将cr0寄存器的PE置1

```assembly
movl %cr0, %eax
orl $CR0_PE_ON, %eax  # CR0_PE_ON = 0x1
movl %eax, %cr0
```

通过长跳转更新cs的基地址

```assembly
ljmp $PROT_MODE_CSEG, $protcseg  # PROT_MODE_CSEG kernel code 段选择器
```

设置段寄存器，并建立堆栈

```assembly
movw $PROT_MODE_DSEG, %ax  # PROT_MODE_DSEG kernel data 段选择器
movw %ax, %ds
movw %ax, %es
movw %ax, %fs
movw %ax, %gs
movw %ax, %ss
movl $0x0, %ebp
movl $start, %esp
```

转到保护模式完成，进入boot主方法

```assembly
 call bootmain
```







