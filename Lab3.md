# Lab3 虚拟内存管理

设计的OS原理

- 虚拟内存的Page Fault异常处理实现
- 页替换算法的实现

在实验二的基础上，借助于页表机制和实验一中涉及的中断异常处理机制，完成Page Fault异常处理和FIFO页替换算法的实现，结合磁盘提供的缓存空间，从而能够支持虚存管理，提供一个比实际物理内存空间“更大”的虚拟内存空间给系统使用。

## 练习1：给未被映射的地址映射上物理页

完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限 的时候需要参考页面所在 VMA 的权限，同时需要注意映射物理页时需要操作内存控制 结构所指定的页表，而不是内核的页表。

重要的结构体：

```c
// the virtual continuous memory area(vma)
struct vma_struct {
    struct mm_struct *vm_mm; // the set of vma using the same PDT 
    uintptr_t vm_start;      //    start addr of vma    
    uintptr_t vm_end;        // end addr of vma
    uint32_t vm_flags;       // VM_READ 0x00000001，只读 VM_WRITE 0x00000002，可读写 VM_EXEC 0x00000004，可执行
    list_entry_t list_link;  // linear list link which sorted by start addr of vma
};
```

```c
// 描述一个进程的虚拟地址空间
struct mm_struct {
    list_entry_t mmap_list;        // 双向链表的头节点，根据vma起始地址排序的链表
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    void *sm_priv;                   // the private data for swap manager
};
```

vmm.c::do_pgfault​

中断，处理缺页异常

```c
// 通过addr这个线性地址返回对应的虚拟页pte 如果没有get_pte会创建一个虚拟页 同时ptep等于0
if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {  // ptep等于NULL代表alloc_page创建虚拟页失败
    cprintf("get_pte in do_pgfault failed\n");
    goto failed;
}

// 等于0是创建好虚拟页后还没有物理页与之对应
if (*ptep == 0) {
    if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {  // 尝试分配物理页
        cprintf("pgdir_alloc_page in do_pgfault failed\n");
        goto failed;
    }
}
// 执行到这里的时候，代表存在虚拟页并且原来都有与之对应的物理页，下面尝试换入
else { 
    if (swap_init_ok) {
        struct Page *page=NULL;
        if ((ret = swap_in(mm, addr, &page)) != 0) {  // 将磁盘中的页换入到内存
            cprintf("swap_in in do_pgfault failed\n");
            goto failed;
        }
        page_insert(mm->pgdir, page, addr, perm);  // 建立虚拟地址和物理地址之间的对应关系
        swap_map_swappable(mm, addr, page, 1);  // 	swap_in等于1使页面可替换
    }
    else {
        cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
        goto failed;
    }
}
```

### 问题

如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

需要将当前页访问异常的错误代码进行状态保存（快照，使用kernel stack保存），然后把CS:IP指向中断服务例程，去执行中断处理函数。

## 练习2：补充完成基于FIFO的页面替换算法

完成vmm.c中的do_pgfault函数，并且在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_victim函数。通过对swap的测试。

这两个待实现函数都是对双向链表进行维护。前面入队列，后面出队列

```c
// 根据FIFO页面置换算法，把最近使用的页插入队列
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && head != NULL);
    //record the page access situlation
    list_add(head, entry);  // 把最近使用的页插入队列
    return 0;
}
```



```c
// 根据FIFO页面置换算法，把最旧使用的页从队列删除
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
         assert(head != NULL);
     assert(in_tick==0);
     
     /* Select the tail */
     list_entry_t *le = head->prev;  // 尾部删除，因为是双向链表，所以很容易找到尾部
     assert(head!=le);
     struct Page *p = le2page(le, pra_page_link);
     list_del(le);
     assert(p !=NULL);
     *ptr_page = p;  // 把这个页的地址值赋给ptr_page
     return 0;
}
```

### 问题

如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

- 需要被换出的页的特征是什么？
  - 页表项的 Dirty Bit 为 0 
- 在ucore中如何判断具有这样特征的页？
  - 可以用D位（访问位）和A位（修改位）来实现简单的页面置换算法。当启动一个新进程时，两位都设置位0。D位（访问位）被定期的清零——以区别最近被访问和没有被访问的页
  - D0A0：未访问、未修改（需要置换出去）；D0A1：在中断刚刚产生时；D1A0：需要置换出去；
- 何时进行换入和换出操作？
  - 换入是在缺页异常的时候，换出是在物理页帧满的时候