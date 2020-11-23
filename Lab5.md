# Lab5 用户进程管理

## OS原理

- 用户进程创建
- 系统调用框架的实现
- 利用系统调用管理进程

到目前为止，所有的运行都在内核态执行。实验5将创建用户进程，让用户进程在用户态执行，且在需要ucore支持时，可通过系统调用来让ucore提供服务。为此需要构造出第一个用户进程，并通过系统调用sys_fork/sys_exec/sys_exit/sys_wait来支持运行不同的应用程序，完成对用户进程的执行过程的基本管理。

## 练习1：加载应用程序并执行

do_execv函数调用load_icode（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。

本练习要实现伪造中断返回现场，使得系统调用返回后可以正确的跳转到需要运行的程序的入口，主要分为以下步骤：

- 由于最终是在用户态下运行的，所以需要将段寄存器初始化为用户态的代码段、数据段、堆栈段；
- esp应当指向先前的步骤中创建的用户栈的栈顶；
- eip应当指向ELF可执行文件加载到内存之后的入口处；
- eflags中应当初始化为中断使能，注意eflags的第1位是恒为1的；
- 设置ret为0，表示正常返回；

load_icode函数，kern/process/proc.c：

```c
/* load_icode - load the content of binary program(ELF format) as the new content of current process
 * @binary:  the memory addr of the content of binary program
 * @size:  the size of the content of binary program
 */
static int
load_icode(unsigned char *binary, size_t size) {
    if (current->mm != NULL) {
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;
    struct mm_struct *mm;
  
    // 前5步……

    //(6) setup trapframe for user environment
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;  // 用户栈顶
    tf->tf_eip = elf->e_entry;  // 二进制程序的入口
    tf->tf_eflags = FL_IF;  // 允许产生中断
    ret = 0;  // 正常返回
out:
    return ret;
bad_cleanup_mmap:
    exit_mmap(mm);
bad_elf_cleanup_pgdir:
    put_pgdir(mm);
bad_pgdir_cleanup_mm:
    mm_destroy(mm);
bad_mm:
    goto out;
}
```

### 问题

描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

1. 操作系统从init/init.c开始执行，调用cpu_idle()函数运行一个idle process
2. cpu_idle(void) 调用 schedule()函数执行。schedule()函数本身是个调度计划表，最终会调用 proc_run(next)函数运行一个进程。
   1. 切换栈	
   2. 切换page table
   3. proc_run(struct proc_struct *proc) 调用 switch_to(&(prev->context), &(next->context))函数（切换context）
3. 执行完switch_to后，开始执行forkret(void)函数，当iret返回时，会跳到entry.S 中的 kernel_thread_entry。

## 练习2：父进程复制自己的内存空间给子进程

创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。

相关函数调用关系：do_fork()---->copy_mm()---->dup_mmap()---->copy_range()。copy_range函数把进程A的内存空间赋值给子进程B

```c
/* copy_range - 把进程A的内存空间（从start到end）复制给子进程B
 * @to:    进程B的页目录的地址
 * @from:  进程A的页目录的地址
 * @share: 指示是复制还是共享的标志位，这里只用到了复制
 *
 * 调用链: copy_mm-->dup_mmap-->copy_range
 */
int
copy_range(pde_t *to, pde_t *from, uintptr_t start, uintptr_t end, bool share) {
    assert(start % PGSIZE == 0 && end % PGSIZE == 0);
    assert(USER_ACCESS(start, end));
    // copy content by page unit.
    do {
        //call get_pte to find process A's pte according to the addr start
        pte_t *ptep = get_pte(from, start, 0), *nptep;
        if (ptep == NULL) {
            start = ROUNDDOWN(start + PTSIZE, PTSIZE);
            continue ;
        }
        //call get_pte to find process B's pte according to the addr start. If pte is NULL, just alloc a PT
        if (*ptep & PTE_P) {
            if ((nptep = get_pte(to, start, 1)) == NULL) {
                return -E_NO_MEM;
            }
        uint32_t perm = (*ptep & PTE_USER);
        //get page from ptep
        struct Page *page = pte2page(*ptep);
        // alloc a page for process B
        struct Page *npage=alloc_page();
        assert(page!=NULL);
        assert(npage!=NULL);
        int ret=0;
        /* LAB5:EXERCISE2 YOUR CODE
         * replicate content of page to npage, build the map of phy addr of nage with the linear addr start
         *
         * Some Useful MACROs and DEFINEs, you can use them in below implementation.
         * MACROs or Functions:
         *    page2kva(struct Page *page): return the kernel vritual addr of memory which page managed (SEE pmm.h)
         *    page_insert: build the map of phy addr of an Page with the linear addr la
         *    memcpy: typical memory copy function
         *
         * (1) find src_kvaddr: the kernel virtual address of page
         * (2) find dst_kvaddr: the kernel virtual address of npage
         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
         * (4) build the map of phy addr of  nage with the linear addr start
         */
        void * kva_src = page2kva(page);  // 父进程的内核虚拟页地址  
        void * kva_dst = page2kva(npage);  // 子进程的内核虚拟页地址
    
        memcpy(kva_dst, kva_src, PGSIZE);  // 复制父进程到子进程 

        ret = page_insert(to, npage, start, perm);  // 建立子进程页地址起始位置与物理地址的映射关系(prem是权限) 
        assert(ret == 0);
        }
        start += PGSIZE;
    } while (start != 0 && start < end);
    return 0;
}
```

所以copy_range函数就是调用一个`memcpy`将父进程的内存直接复制给子进程。

### 问题

简要说明如何设计实现“Copy on Write 机制”。

在创建子进程时，将父进程的PDE直接赋值给子进程的PDE，但是需要将允许写入的标志位置0；当子进程需要进行写操作时，再次出发中断调用do_pgfault()，此时应给子进程新建PTE，并取代原先PDE中的项，然后才能写入。

## 练习3：理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现

分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？

* fork：执行完毕后，如果创建新进程成功，则出现两个进程，一个是子进程，一个是父进程。在子进程中，fork函数返回0，在父进程中，fork返回新创建子进程的进程ID。我们可以通过fork返回的值来判断当前进程是子进程还是父进程

* exit：会把一个退出码error_code传递给ucore，ucore通过执行内核函数do_exit来完成对当前进程的退出处理，主要工作简单地说就是回收当前进程所占的大部分内存资源，并通知父进程完成最后的回收工作。

* execve：完成用户进程的创建工作。首先为加载新的执行码做好用户态内存空间清空准备。接下来的一步是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。
* wait：等待任意子进程的结束通知。wait_pid函数等待进程id号为pid的子进程结束通知。这两个函数最终访问sys_wait系统调用接口让ucore来完成对子进程的最后回收工作