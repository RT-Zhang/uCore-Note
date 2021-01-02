# Lab2 物理内存管理

## OS原理

- 基于段页式内存地址的转换机制
- 页表的建立和使用方法
- 物理内存的管理方法



## 练习0

学习meld使用，GUI化的文件比较/merge工具，方便地把Lab1中新增的代码添加到Lab2中。

## 练习1：实现 first-fit 连续物理内存分配算法

注：考虑地址连续的空闲块之间的合并操作。在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。

first-fit连续物理内存分配算法简介：分配n个字节，使用第一个可用空间比n大的空闲块。

将内存分为以下模块：

* 探测物理内存
* 物理内存管理的初始化
* 内存段页式管理
* 地址映射
* 自映射

uCore对物理内存空间按照页的大小（4KB）进行管理，页的信息保存在struct Page中，位于kern/mm/memlayout.h中

```c
struct Page {
    int ref;                        // 映射此物理页的虚拟页个数
    uint32_t flags;                 // 物理页的状态
    unsigned int property;          // 空闲时，代表以此为首的连续空闲页的数量
    list_entry_t page_link;         // 把多个连续内存空闲块链接在一起的双向链表指针
};
```

struct Page：

* ref：一旦某页表中有一个页表项设置了虚拟页到这个Page管理的物理页的映射关系，就把Page的ref加一，反之减一

* flags：有两个标志位，第一个表示是否被保留，如果被保留了则设为1（比如内核代码占用的空间）。第二个表示此页是否是free的。如果设置为1，表示这页是free的，可以被分配；如果设置为0，表示这页已经被分配出去了，不能被再二次分配。
* page_link：连续内存空闲块利用这个页的成员变量 page_link 来链接比它地址小和大的其他连续内存空闲块。

struct free_area_t：

维护一个记录空闲页的双向链表

```c
typedef struct {
    list_entry_t free_list;         // 链表头
    unsigned int nr_free;           // 空闲页数量
} free_area_t;
```

default_pmm.c

default_init函数已经完成了，不需要修改。

```c
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;  // 一开始空闲页初始化为0
}
```

```c
// 初始化一个空闲块（参数：基址，页数量）
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);  // 断言，分配的页数量n总要大于0吧
    struct Page *p = base;
    for (; p != base + n; p ++) {  // 初始化每一个空闲页，并连接
        assert(PageReserved(p));
        p->flags = 0;  // flags都设置为0，表示已经被分配出去
        SetPageProperty(p);  // 置1：所有空闲页第1位也就是PG_property位置为1  
        p->property = 0;
        set_page_ref(p, 0);  // 清除引用该物理页的引用数
        list_add_before(&free_list, &(p->page_link));  // 将这一页连接到free_list
    }
    nr_free += n;  // 空闲页个数增加
    base->property = n;  // 对整块的第1个页面，其property属性记录连续空闲页的数量
}
```

```c
// 找到空闲链表中的第一个空闲块（当然size>=n），并重新设置大小
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le, *len;  // 双指针技巧，le相当于前指针，len为后指针
    le = &free_list;

    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
      	if (p->property >= n) {
        	int i;  // 分配页数
            for (i = 0; i < n; i++) {
                  len = list_next(le);
                  struct Page *pp = le2page(le, page_link);
                  SetPageReserved(pp);  // 被内核保留
                  ClearPageProperty(pp);  // 不是头页
                  list_del(le);  // 被分配好了，要将其从freelist中删除
                  le = len;
            }
            if (p->property>n) {
                // p指针移动到顶部，修改头页的标志位
                (le2page(le,page_link))->property = p->property - n;
            }
            // 修改头页的标志位
            ClearPageProperty(p);  
            SetPageReserved(p);
            nr_free -= n;
            return p;
      	}
    }
    return NULL;
}
```

将页收回到freelist中，还会合并空闲块

```c
// 释放n个页
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    assert(PageReserved(base));  // 断言base位置的页被内核所保留

    list_entry_t *le = &free_list;
    struct Page * p;
    while ((le=list_next(le)) != &free_list) {
          p = le2page(le, page_link);  // 获取list对应的page
          if(p>base){
            	break;
          }
    }
    //list_add_before(le, base->page_link);
    for (p=base; p<base+n; p++){  // 遍历每个页
      	list_add_before(le, &(p->page_link));   // 将每个页插入到链表中
    }
    // 修改首个页的属性，仅修改首页属性即可
    base->flags = 0;
    set_page_ref(base, 0);
    ClearPageProperty(base);
    SetPageProperty(base);  // 声明是头页
    base->property = n;  // 记录空闲页的数目n
    
    // 如果是高位，则向高地址合并
    p = le2page(le,page_link) ;
    if (base+n == p ) {
        base->property += p->property;
        p->property = 0;
    }
    // 向低地址合并
    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);
    if (le!=&free_list && p==base-1) {
      	while(le!=&free_list) {
            if (p->property) {
                // p位于高地址，向低地址合并
                p->property += base->property;
                base->property = 0;
                break;
            }
            le = list_prev(le);
            p = le2page(le,page_link);
      	}
    }

    nr_free += n;
    return ;
}
```

## 练习2：实现寻找虚拟地址对应的页表项

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的get_pte函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。get_pte函数调用关系如下：

![img](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2_figs/image001.png)

在保护模式中，x86 体系结构将内存地址分成三种：逻辑地址（虚拟地址）、线性地址和物理地址。逻辑地址即是程序指令中使用的地址，物理地址是实际访问内存的地址。逻辑地址通过段式管理的地址映射可以得到线性地址，线性地址通过页式管理的地址映射得到物理地址。

查看mmu.h，查看线性地址的结构

```c
// A linear address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
//
// The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).
```

可见，高10位为页目录索引，中间10位为页表索引，低12位为页内偏移。

常用宏，根据线性地址构建PDX、PTX、PPN、PGOFF

```c
// page directory index
// 得到一级页表项对应的入口地址
#define PDX(la) ((((uintptr_t)(la)) >> PDXSHIFT) & 0x3FF)

// page table index
#define PTX(la) ((((uintptr_t)(la)) >> PTXSHIFT) & 0x3FF)

// page number field of address
#define PPN(la) (((uintptr_t)(la)) >> PTXSHIFT)

// offset in page
#define PGOFF(la) (((uintptr_t)(la)) & 0xFFF)
```

根据虚拟内存管理的Lazy Load策略，按需分配，并没有一开始就存在所有的二级页表，而是等到需要的时候再添加对应的二级页表。

当建立从一级页表到二级页表的映射时，需要注意设置控制位。这里应该设置同时设置 上PTE_U、PTE_W 和 PTE_P。如果原来就有二级页表，或者新建立了页表，则只需返回对应项的地址即可。如果 create参数为 0，则get_pte返回NULL；如果 create参数不为 0，则 get_pte 需要申请一个新的物理页。

需了解的变量类型/宏：

* pde_t（page directory entry），一级页表的表项。pgdir实际不是表项，而是一级页表本身，pgdir给出页表起始地址（相当于数组名）
* pte_t（page table entry），二级页表的表项。
* uintptr_t 表示线性地址，由于段式管理只做直接映射，所以它也是逻辑地址。
* `PTE_U`: 位3，表示用户态的软件可以读取对应地址的物理内存页内容
* `PTE_W`: 位2，表示物理内存页内容可写
* `PTE_P`: 位1，表示物理内存页存在

```c
// 得到页表索引（PTE），并返回这个PTE的内核虚拟地址，如果不存在则分配一个
// @pgdir，页目录入口 @la，要映射的线性地址 @create，是否为PT分配一个页 
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
#if 0
    pde_t *pdep = NULL;   // (1) find page directory entry
    if (0) {              // (2) check if entry is not present
                          // (3) check if creating is needed, then alloc page for page table
                          // CAUTION: this page is used for page table, not for common data page
                          // (4) set page reference
        uintptr_t pa = 0; // (5) get linear address of page
                          // (6) clear page content using memset
                          // (7) set page directory entry's permission
    }
    return NULL;          // (8) return page table entry
#endif
    pde_t *pdep = &pgdir[PDX(la)];  // 找到页目录入口
    if (!(*pdep & PTE_P)) {  // 如果入口不存在
        struct Page *page;
        // 如果不需要分配或申请新的页失败，返回NULL
        if (!create || (page = alloc_page()) == NULL) {
            return NULL;
        }
        // 到这里，如果是需要申请新的页的话一定执行过了alloc_page()函数
        set_page_ref(page, 1);  // 设置引用数目
        uintptr_t pa = page2pa(page);  // 得到该物理页的地址
        // 物理地址转为虚拟地址
        memset(KADDR(pa), 0, PGSIZE);  // KADDR(pa)将物理地址转换为内核虚拟地址，第二个参数将这一页清空，第三个参数是4096也就是一页的大小
        *pdep = pa | PTE_U | PTE_W | PTE_P;  // 设置页目录入口的许可位
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
}
```

### 问题

请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对ucore而言的潜在用处。

PDE：

* A, D, W：这些与高速缓存相关的位，记录该页是否被访问过、不允许高速缓存过或执行了写穿透策略。如果uCore需要与硬件的cache进行交互（即这些位并非由硬件设定），就需要用到这些位。
* US：决定了当前页的访问权限（内核or用户）：uCore可以通过这些位进行用户态和内核态程序访问权限的控制。
* RW：决定了当前页的是否可写属性：当uCore需要对某一页进行保护的时候，需要用到此位,用于权限控制。
* P：决定当前页是否存在：uCore需要根据这个标志确定页表是否存在，并是否建立新的相关页表。

PTE：

* PCD：与上述的D位相同。
* G：控制TLB地址的更新策略。
* D：该页是否被写过。如果uCore需要对高速缓存实现更复杂的控制则可能用到该位。同时，在页换入或是换出的时候可能需要判断是否更新高速缓存。
* P：决定是否存在

## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。page_remove_pte函数的调用关系图：

![img](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab2_figs/image002.png)

```c
// 释放一个Page，清理PTE
// 注意：页表改变后，TLB也需要刷新
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
#if 0
    if (0) {                      //(1) check if page directory is present
        struct Page *page = NULL; //(2) find corresponding page to pte
                                  //(3) decrease page reference
                                  //(4) and free this page when page reference reachs 0
                                  //(5) clear second page table entry
                                  //(6) flush tlb
    }
#endif
    if (*ptep & PTE_P) {  // 页表入口是否存在
        struct Page *page = pte2page(*ptep);  // 找到关联的页
        if (page_ref_dec(page) == 0) {  // 减少页引用
            free_page(page);  // 当引用数降至0时，释放页
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);  // 刷新快表
    }
}
```

### 问题

数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

有对应关系，PG_reserved 表示是否被内核所保留，与页表项的 PTE_U 有关联，PTE_U 表示用户态是否可访问。