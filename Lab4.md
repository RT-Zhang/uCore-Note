# Lab4 内核线程管理

## OS原理

- 内核线程创建/执行的管理过程
- 内核线程的切换和基本调度过程

Lab2&3完成了物理和虚拟内存管理，这给创建内核线程打下了内存管理的基础。

内核线程与用户线程的区别：

- 内核线程只运行在内核态；用户进程会在在用户态和内核态交替运行
- 内核线程共用uCore内核内存空间，不需为每个内核线程维护单独的内存空间；用户进程需要维护各自的用户内存空间

## 练习1：分配并初始化一个进程控制块

alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化。

alloc_proc，分配一个proc_struct并初始化字段：

```c
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
        proc->state = PROC_UNINIT;  // 进程状态：未初始化
        proc->pid = -1;  // 未分配的进程pid是-1 先设置pid为无效值-1，用户调完alloc_proc函数后再根据实际情况设置
        proc->runs = 0;
        proc->kstack = 0;  // 内核栈位置
        proc->need_resched = 0;  // 是否需要调度
        proc->parent = NULL;  // 父进程
        proc->mm = NULL;  // 虚拟内存结构体
        memset(&(proc->context), 0, sizeof(struct context));  // 上下文结构体，用memset清空
        proc->tf = NULL;  
        proc->cr3 = boot_cr3;  // 页目录表的基址
        proc->flags = 0;
        memset(proc->name, 0, PROC_NAME_LEN);
    }
    return proc;
}
```

### 问题

proc_struct  中 struct context context 和 struct trapframe *tf 成员变量含义和在本实验中的作用是什么？

查看struct context的定义，

```c
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

context保存的是寄存器信息，主要用于进程切换。

查看struct trapframe的定义，

```c
struct trapframe {
    struct pushregs tf_regs;  // 通用寄存器
	// 段寄存器信息
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    // 异常和中断时候需要把下列信息压栈
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    // 需要特权级转换的时候，下列信息压栈
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```

\_\_attribute\_\_((packed))使结构体按照最紧凑的方式对齐（属于GCC的语法）。当进程从用户空间跳到内核空间时，trapframe保存当前被打断的进程或线程的所有信息，即保存现场。当内核需要跳回用户空间时，需要调整中断帧以恢复让进程继续执行的各寄存器值。

## 练习2：为新创建的内核线程分配资源

创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。

do_fork（kern/process/proc.c），创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。

```c
/* @clone_flags: 指导怎么克隆
 * @stack:       父进程的用户栈指针，若为0则代表fork一个内核线程
 * @tf:          会被复制到子进程的proc->tf中 */
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
    
    if ((proc = alloc_proc()) == NULL) {  // 分配一个进程控制块并初始化字段
        goto fork_out;
    }

    proc->parent = current;  // 父亲就是当前的

    if (setup_kstack(proc) != 0) {  // 分配一个内核栈
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {  // 复制原进程的内存管理信息到新进程
        goto bad_fork_cleanup_kstack;
    }
    copy_thread(proc, stack, tf);  // 复制原进程上下文到新进程

    bool intr_flag;
    local_intr_save(intr_flag);  // 关闭中断功能
    // 限定花括号中变量的作用域，使其不影响外面
    {
        proc->pid = get_pid();  // 分配一个pid
        hash_proc(proc);  // 新进程添加到hash方式组织的进程链表，便于以后对某个指定的线程的快速查找
        list_add(&proc_list, &(proc->list_link));  // 将线程加入到所有线程的链表中，便于调度
        nr_process ++;  // 全局线程的数目加1
    }
    local_intr_restore(intr_flag);  // 开启中断功能

    wakeup_proc(proc);  // 唤醒，变为RUNNABLE

    ret = proc->pid;  // 新进程号作为ret值
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

总结过程：

- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 将新进程添加到进程列表
- 唤醒新进程
- 返回新进程号

值得一提，local_intr_save(x)这个宏定义如下，

```c
#define local_intr_save(x)      do { x = __intr_save(); } while (0)
```

在Linux内核和其它一些著名的C库中有许多使用do{...}while(0)的宏，Google的Robert Love（先前从事Linux内核开发）的解释如下：do{...} while(0)在C中是唯一的构造程序，让你定义的宏总是以相同的方式工作，这样不管怎么使用宏（尤其在没有用大括号包围调用宏的语句），宏后面的分号也是相同的效果。

### 问题

ucore是否做到给每个新fork的线程一个唯一的id？

可以。在分析do_fork时可以看到，在分配pid之前禁止了中断，分配完成后再允许中断，具有事务的隔离性。

分析getpid()函数

```c
// get_pid - alloc a unique pid for process
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {  // 如果last_pid+1等于MAX_PID，意味着pid已经分配完了
                    if (last_pid >= MAX_PID) {  
                        last_pid = 1;  // 如果last_pid超出最大pid范围，则last_pid重新从1开始编号
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    } 
    return last_pid;  // last_pid作为新颁发的编号
}
```

## 练习3：理解 proc_run 函数和它调用的函数如何完成进程切换

利用grep命令搜索，发现proc_run函数在schedule.c中被调用

```c
next->runs ++;
if (next != current) {
    proc_run(next);
}
```

分析proc_run函数，作用是让进程在CPU执行

```c
void
proc_run(struct proc_struct *proc) {
    if (proc != current) {  // 如果不是当前线程
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);  // 禁止中断，目的是事务的隔离性
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);  // 将当前的cr3寄存器修改为需要运行的进程的页目录表基址
            switch_to(&(prev->context), &(next->context));  // 线程切换函数
        }
        local_intr_restore(intr_flag);
    }
}
```

### 问题

在本实验的执行过程中，创建且运行了几个内核线程？

* idleproc：最初的内核线程

* init_main：打印字符串

语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用？

在练习2&3的代码分析中已经涉及到。作用是先禁止中断，执行完下面代码后再允许中断，避免事务冲突。

