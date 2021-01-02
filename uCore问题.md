# Lab 2

## BIOS

Q：BIOS 的具体操作对操作系统是透明的吗？我从⽹上查到 BIOS 是⽤汇编码写成的，⼜是连接底层的接⼝，那OS应该有能⼒查看它吧？还是说这是以某种形式在硬件⾥写好的，OS 只能调⽤呢？

理论上讲 BIOS 可以用任意能生成二进制文件的语言编写，但为了掌控底层硬件，最合适和常用的是汇编语言，在经过汇编器生成二进制代码。BIOS 是以二进制代码的形式直接刻写在 ROM 上的，但一些计算机的 BIOS 源码并不开源，想要查看 BIOS 原理的话可以尝试一些开源 BIOS（如 coreboot）。

## ucore 地址转换

ucore 的虚拟内存

```c
/* *
 * Virtual memory map:                                          Permissions
 *                                                              kernel/user
 *
 *     4G ------------------> +---------------------------------+
 *                            |                                 |
 *                            |         Empty Memory (*)        |
 *                            |                                 |
 *                            +---------------------------------+ 0xFB000000
 *                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE
 *     VPT -----------------> +---------------------------------+ 0xFAC00000
 *                            |        Invalid Memory (*)       | --/--
 *     KERNTOP -------------> +---------------------------------+ 0xF8000000
 *                            |                                 |
 *                            |    Remapped Physical Memory     | RW/-- KMEMSIZE
 *                            |                                 |
 *     KERNBASE ------------> +---------------------------------+ 0xC0000000
 *                            |                                 |
 *                            |                                 |
 *                            |                                 |
 *                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

可见，内核的基址 KERNBASE 为 0xC0000000

### bootloader 阶段

从 bootloader 的 start 函数（boot/bootasm.S）到执行 ucore kernel 的 kern_entry 函数之前，其虚拟地址、线性地址以及物理地址之间的映射关系与 lab1 一样。lab1 中建立的段地址映射关系为**对等关系**，所以 ucore 的物理地址也是从 0x100000 开始，而ucore的入口函数 kern_init 的起始地址。

```c
virt addr = linear addr = phy addr
```

### 段映射机制阶段

从 kern_entry 函数开始，到 enable_page 函数被执行之前。

查看 kernel.ld 连接脚本，虚拟起始地址从 0xC0100000 开始，ucore 被 bootloader 放置在从 0x100000 处开始的物理内存中。

```c
// map all physical memory to linear memory with base linear addr KERNBASE
// linear_addr KERNBASE~KERNBASE+KMEMSIZE = phy_addr 0~KMEMSIZE
// But shouldn't use this map until enable_paging() & gdt_init() finished.
boot_map_segment(boot_pgdir, KERNBASE, KMEMSIZE, 0, PTE_W);
```

进行物理地址向线性地址的映射，也就是相差KERNBASE（0xC0000000）的量，所以

```c
virt addr - 0xC0000000 = linear addr = phy addr
```

### 使能页机制后

从 enable_page 函数开始，到执行 gdt_init 函数之前。

```c
static void
enable_paging(void) {
    lcr3(boot_cr3);
    // 开启分页功能
    uint32_t cr0 = rcr0();
    cr0 |= CR0_PE | CR0_PG | CR0_AM | CR0_WP | CR0_NE | CR0_TS | CR0_EM | CR0_MP;
    cr0 &= ~(CR0_TS | CR0_EM);
    lcr0(cr0);
}
```

加载 cr0 指令（即让CPU使能分页机制），则接下来的访问是基于**段页式的映射关系**了。

```c
// temporary map: 
// virtual_addr 3G~3G+4M = linear_addr 0~4M = linear_addr 3G~3G+4M = phy_addr 0~4M     
boot_pgdir[0] = boot_pgdir[PDX(KERNBASE)];
```
源码中提到的3G可以理解为KERNBASE的值，这行代码用来建立物理地址在 0~4MB 之内的三个地址间的临时映射关系。这样就只需要让页表在0~4MB的线性地址与KERNBASE ~ KERNBASE+4MB的线性地址获得相同的映射即可，都映射到 0~4MB的物理地址空间。

情况分为 0~4MB 和 KERNBASE~KERNBASE+4MB 两种：

```c
virt addr - 0xC0000000 = linear addr  = phy addr + 0xC0000000 # 物理地址在0~4MB之外的三者映射关系
virt addr - 0xC0000000 = linear addr  = phy addr # 物理地址在0~4MB之内的三者映射关系
```

### 更新段映射

```c
// reload gdt(third time,the last time) to map all physical memory
// virtual_addr 0~4G=liear_addr 0~4G
// then set kernel stack(ss:esp) in TSS, setup TSS in gdt, load TSS
gdt_init();

// disable the map of virtual_addr 0~4M
boot_pgdir[0] = 0;
```

调整段映射关系，即重新设置新的GDT，建立对等段映射（virt addr = linear addr）。新的段页式映射已经建立好了，上面的0~4MB的线性地址与0~4MB的物理地址一一映射关系已经没有用了，把 `boot_pgdir[0]` 的第一个页目录表项（0~4MB）清零来取消临时的页映射关系。根据源码，新的映射关系如下

```c
virt addr = linear addr = phy addr + 0xC0000000
```

### 什么是自映射？

定义一个常量，

```c
#define VPT                 0xFAC00000
```

即virtual page table，页目录中的 Entry PDX(VPT)包含一个指向页目录自己的指针，所以页目录也可以被看作一个页表，所以称为自映射。

定义 vpt 和 vpd，

```c
pte_t * const vpt = (pte_t *)VPT;
pde_t * const vpd = (pde_t *)PGADDR(PDX(VPT), PDX(VPT), 0);
```

vpd 变量的值就是页目录表的起始虚地址，vpt 是页目录表中第一个目录表项指向的页表的起始虚地址。

否则，在按顺序查找页目录表和页表内容是，需要先查找页目录表的页目录表项内容，根据页目录表项内容找到页表的物理地址，转换成对应的虚地址，然后访问页表的虚地址。

# Lab 3

Q：swap.c 的 check_content_set() ⾥⾯从上⼀句的赋值，到下⼀句的 assert，到底发⽣了什么？我不明⽩的是，“访问”这个操作本⾝在哪⾥？是代码形式的吗？因为我试图找到⼀个⽆论如何都会在访问⻚时被调⽤的函数，但没有找到。问过助教，说是有同学声称找到了这个函数，但事实上打印出来的信息并不对？那如果不是代码形式的，对我们可⻅吗？

```c
static inline void check_content_set(void) {
     *(unsigned char *)0x1000 = 0x0a;
     assert(pgfault_num==1);
     *(unsigned char *)0x1010 = 0x0a;
     assert(pgfault_num==1);
     ...
}
```

这题不是很确定。

我认为这个问题是计算机硬件层面的，而不是操作系统层面。例如 `*(unsigned char *)0x1000 = 0x0a` 这句代码，本质上就是 `movl $4096, %eax` 和 `movl $10, (%eax)` 对应的机器码。CPU 执行的都是代码中的虚拟地址，所有的内存访问请求都会经过 MMU，如果没有物理页与虚拟页相关联，MMU 将向 CPU 发出页错误的信号，操作系统将进行处理。



Q：在 swap_fifo.c ⾥⾯，_fifo_tick_event 函数的意义是什么？是和 _fifo_set_unswappable 意义⼀样，属于“理论上有操作但 ucore 不涉及”吗？

_fifo_tick_event 是 tick_event 接口的实现，在 ucore 确实没有被调用。

```c
static int _fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    assert(head != NULL);
    assert(in_tick == 0);
    /* Select the victim */
    ...
	return 0;
}
```

查看 _fifo_swap_out_victim 代码，选择 victim 前会特意断言 in_tick 为 0，即 fifo 算法与这个变量无关。

查看调用处的传参，

```c
//alloc_pages - call pmm->alloc_pages to allocate a continuous n*PAGESIZE memory 
struct Page *alloc_pages(size_t n) {
    ...
    while (1)
    {
         ...
         swap_out(check_mm_struct, n, 0);
    }
    //cprintf("n %d,get page %x, No %d in alloc_pages\n",n,page,(page-pages));
    return page;
}
```

in_tick 参数确实恒为零。

swap_manager 结构体中的函数指针 tick_event 当时钟中断发生时被调用，可用于主动的swap交换策略。例如当发生时钟中断时，定时进行主动的换出操作，腾出更多的物理空闲页。



Q：关于PTE_P

kern/mm/mmu.h

```c
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
```

设一个 32bit 线性地址 la 有一个对应的 32bit 物理地址 pa，如果在以 la 的高 10 位为索引值的页目录项中的存在位（PTE_P）为0，表示缺少对应的页表空间，则可通过 alloc_page 获得一个空闲物理页给页表，页表起始物理地址是按4096字节对齐的，这样填写页目录项的内容为

```c
页目录项内容 = (页表起始物理地址 & ~0x0FFF) | PTE_U | PTE_W | PTE_P
```

对于页表中以线性地址 la 的中10位为索引值对应页表项的内容为

```c
页表项内容 = (pa & ~0x0FFF) | PTE_P | PTE_W
```



Q：关于swap分区和pte_t

如果一个页（4KB/页）被置换到了硬盘某 8 个扇区（0.5KB/扇区），该 PTE 的最低位 present 位应该为0 （即 PTE_P 标记为空，表示虚实地址映射关系不存在），接下来的 7 位暂时保留，可以用作各种扩展；而包括原来高 20 位页帧号的高 24 位数据，恰好可以用来表示此页在硬盘上的起始扇区的位置（其从第几个扇区开始）。为了在页表项中区别 0 和 swap 分区的映射，将 swap 分区的一个 page 空出来不用，也就是说一个高 24 位不为 0，而最低位为 0 的 PTE 表示了一个放在硬盘上的页的起始扇区号。



Q：关于page fault处理完之后

当缺页中断处理完后，`trap(struct trapframe *tf)` 函数返回，`kern/trap/trapentry.S` 中的代码恢复存在 trapframe 中的 CPU 的状态，然后使用 iret 指令，返回到产生页访问异常的指令处重新执行指令。

```assembly
    # call trap(tf), where tf=%esp
    call trap

    # pop the pushed stack pointer
    popl %esp

    # return falls through to trapret...
.globl __trapret
__trapret:
    # restore registers from stack
    popal

    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # get rid of the trap number and error code
    addl $0x8, %esp
    iret
```

# Lab 4

Q：ucore 内核级别的线程和普通操作系统的内核级别线程是不是不⼀样？我上课的时候理解的内核线程，是依赖于进程的，但这个进程不是操作系统本⾝。也就是说，应该要先有⼀个进程，它想使⽤多线程，这个 OS 如果⽀持，就会类似于把这个线程“接管”过来，然后分配⼀系列上下⽂。但 ucore 似乎只实现了 OS ⾃⼰⽤的内核线程？idle_proc 是 OS ⾃⼰⽤来实现线程调度的，init_main 打印 helloworld也是直接分配的线程。ucore（⾄少lab4前）我没有显式的看到有哪个内核线程依赖除了 OS ⾃⼰之外的进程。  
and Q：内核线程没有依赖于任何的进程，这在操作系统里面时被允许的吗？在有用户态情况下，它和进程之间是一个什么样的关系呢？

我是这样理解的，程序是静态的可执行文件，而进程本质上就是程序与它的执行状态，需要 CPU 和内存资源。内核线程是由kernel_thread 函数在内核态下创建的。将创建时得到的函数永远执行下去，由一个循环组成。每个内核线程都拥有唯一属于自己的进程控制块，所以在内核中，它看起来就像是一个普通的进程（只是该进程和其他一些进程共享某些资源，如地址空间）。  
uCore 启动后，已经对整个内核内存空间进行了管理，通过设置页表建立了内核虚拟空间。从共享内核虚拟空间这个角度看，内核线程被 uCore 内核这个大“内核进程”所管理。

用户进程从用户态切换到内核态会保存上下文并切换上下文，处理完内核态后会恢复上下文，返回用户态继续执行。



Q：trapframe 在 lab4 里面是没有用到的，但实际上是给他在栈里面开辟了空间的，这个 trapframe 是在什么情况下会使用到？创建一个线程（do_fork)之后有一个 kernel_thread 的函数，它那里为 trapframe 开辟的空间，这个空间是属于谁的，是内核态的空间吗？助教说，trapframe 在用户态才有作用，如果 ucore 拥有用户态之后，trapframe 被创建时所占用的空间仍然是内核态空间吗？红框框的这两句代码不是特别理解。这个代码让我觉得 trapframe 是共享的，而不是每个线程独有的。

线程的创建和切换是需要利用 CPU 中断返回机制的。trapframe *tf 是中断帧的指针，总是指向内核栈的某个位置。当进程从用户空间跳到内核空间时，中断帧记录了进程在被中断前的状态。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。除此之外，uCore 内核允许嵌套中断。因此为了保证嵌套中断发生时tf 总是能够指向当前的 trapframe，uCore 在内核栈上维护了 tf 的链。

```c
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE) - 1;
    *(proc->tf) = *tf;
    proc->tf->tf_regs.reg_eax = 0;
    proc->tf->tf_esp = esp;
    proc->tf->tf_eflags |= FL_IF;

    proc->context.eip = (uintptr_t)forkret;
    proc->context.esp = (uintptr_t)(proc->tf);
}
```

函数前两行代码含义，令 proc->tf 指向 proc 内核栈顶向下偏移一个 struct trapframe 大小的位置，将参数 tf 中的结构体数据复制填入上述proc->tf 指向的位置（正好是上面 struct trapframe 指针减一腾出来的那部分空间）。

 kernel_thread 函数为 trapframe 开辟的空间位于内核栈，属于内核的地址空间。



Q：关于 get_pid。为什么我觉得这个pid是循环使用，而不是每一个线程的pid都是独一无二的……还是说，在某一时刻正在
运行的线程的pid一定都是独一无二的，但昨天和今天的线程的pid可能就不一样，有重用？

get_pid 分配 pid 时，不允许中断，不会分配当前未被销毁且已经分配过进程的 pid，循环使用 pid 不影响这一点。  
进程是动态的概念，是静态的二进制代码和执行状态，同一程序的多次执行过程对应为不同的进程



Q：关于 hash_proc。为什么不能直接链接进程，而是要进行 pid 的哈希呢？那不会出现哈希冲突吗？这个做法的意义和实际商
用os中的价值在哪里？

查看哈希函数，

```c
#define MAX_PROCESS                 4096
#define MAX_PID                     (MAX_PROCESS * 2)

#define HASH_SHIFT          10
#define HASH_LIST_SIZE      (1 << HASH_SHIFT)
#define pid_hashfn(x)       (hash32(x, HASH_SHIFT))

/* 2^31 + 2^29 - 2^25 + 2^22 - 2^19 - 2^16 + 1 */
#define GOLDEN_RATIO_PRIME_32       0x9e370001UL

uint32_t
hash32(uint32_t val, unsigned int bits) {
    uint32_t hash = val * GOLDEN_RATIO_PRIME_32;
    return (hash >> (32 - bits));
}
```

所以哈希值的范围是 0~2^10-1，而 MAX_PID 为 2^13，确实会有冲突。

关于哈希冲突，

```c
// has list for process set based on pid
static list_entry_t hash_list[HASH_LIST_SIZE];
```

这是哈希表的定义，list_entry_t 类型是双向链表，而 hash_list 又是一个含 HASH_LIST_SIZE 个链表的数组，所以 hash_list 实际上采用了链式地址法解决哈希冲突。对于相同的哈希值，使用链表进行连接，同时使用数组存储每一个链表。

do_fork 中会将新进程添加到以 hash 方式组织的的进程链表，以便于以后对某个指定的线程的查找。  
价值在于 hash 表的查找速度更快。



Q：我们一般理解的cr3存着应该是当前进程的页目录这个page的首地址，我们在初始化一个proc（内核线程）的时候需要将它的cr3初始化为boot_cr3。实验指导书上有这样一句话，说boot_cr3指向二级页表描述的空间，那就是boot_cr3其实是一个页表的地址？

boot_cr3 是 ucore 启动时建立的内核虚拟空间的页目录表首地址，通过这个指针可以找到二级页表描述的空间。