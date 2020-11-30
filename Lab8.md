# Lab8 文件系统

## OS原理

- 文件系统系统调用的实现；
- 基于索引节点组织方式的Simple FS文件系统的设计与实现；
- 文件系统抽象层-VFS的设计与实现；

## 练习1: 完成读文件操作的实现

首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，编写在sfs_inode.c中sfs_io_nolock读文件中数据的实现代码。

file结构体

```c
struct file {
    enum {
        FD_NONE, FD_INIT, FD_OPENED, FD_CLOSED,
    } status;  // 访问文件的执行状态
    bool readable;  // 文件是否可读
    bool writable;  // 文件是否可写
    int fd;  // 文件在filemap中的索引值
    off_t pos;  // 访问文件的当前位置
    struct inode *node;  // 该文件对应的内存inode指针
    int open_count;  // 打开此文件的次数
};
```

inode是文件系统中重要的数据结构，它是位于内存的索引节点，把不同文件系统的特定索引节点信息统一封装起来，避免了进程直接访问具体文件系统

```c
struct inode {
    union {
        struct device __device_info;  // 设备文件系统内存inode信息 
        struct sfs_inode __sfs_inode_info;  // SFS文件系统内存inode信息
    } in_info;  // 包含不同文件系统的特定inode信息
    enum {
        inode_type_device_info = 0x1234,
        inode_type_sfs_inode_info,
    } in_type;  // 此inode所属文件系统类型
    int ref_count;  // 引用计数
    int open_count;  // 打开此inode对应文件的个数
    struct fs *in_fs;  // 抽象的文件系统，包含访问文件系统的函数指针
    const struct inode_ops *in_ops;  // 抽象的inode操作，包含访问inode的函数指针
};
```

### 打开文件的处理流程

1. 通用文件访问接口层的处理流程。首先用户会在进程中调用 `safe_open()`函数，然后依次调用如下函数： open->sys_open->syscall，从而引起系统调用进入到内核态。到到了内核态后，通过中断处理例程，会调用到sys_open内核函数，并进一步调用sysfile_open内核函数。到了这里，需要把位于用户空间的字符串”/test/testfile”拷贝到内核空间中的字符串path中，并进入到文件系统抽象层的处理流程完成进一步的打开文件操作中。
2. 文件系统抽象层的处理流程。分配一个空闲的file数据结构变量file在文件系统抽象层的处理中（当前进程的打开文件数组current->fs_struct->filemap[]中的一个空闲元素），到了这一步还仅仅是给当前用户进程分配了一个file数据结构的变量，还没有找到对应的文件索引节点。进一步调用vfs_open函数来找到path指出的文件所对应的基于inode数据结构的VFS索引节点node。然后调用`vop_open`函数打开文件。然后层层返回，通过执行语句`file->node=node;`，就把当前进程的`current->fs_struct->filemap[fd]`（即file所指变量）的成员变量node指针指向了代表文件的索引节点node。这时返回fd。最后完成打开文件的操作。
3. SFS文件系统层的处理流程。在第二步中，vop_lookup函数调用了sfs_lookup函数。

```c
/*
 * sfs_lookup - Parse path relative to the passed directory
 *              DIR, and hand back the inode for the file it
 *              refers to.
 */
static int
sfs_lookup(struct inode *node, char *path, struct inode **node_store) {
    struct sfs_fs *sfs = fsop_info(vop_fs(node), sfs);
    assert(*path != '\0' && *path != '/');  // 以“/”为分割符，从左至右逐一分解path获得各个子目录和最终文件对应的inode节点。
    vop_ref_inc(node);
    struct sfs_inode *sin = vop_info(node, sfs_inode);
    if (sin->din->type != SFS_TYPE_DIR) {
        vop_ref_dec(node);
        return -E_NOTDIR;
    }
    struct inode *subnode;
    int ret = sfs_lookup_once(sfs, sin, path, &subnode, NULL);  // 循环进一步调用sfs_lookup_once查找inode节点

    vop_ref_dec(node);
    if (ret != 0) {
        return ret;
    }
    *node_store = subnode;  // 当无法分解path后，就意味着找到了需要对应的inode节点
    return 0;
}
```

sfs_lookup函数先以“/”为分割符，从左至右逐一分解path获得各个子目录和最终文件对应的inode节点。然后循环进一步调用sfs_lookup_once查找以“test”子目录下的文件“testfile1”所对应的inode节点。最后当无法分解path后，就意味着找到了需要对应的inode节点，就可顺利返回了。

sfs_io_nolock（kern/fs/sfs/sfs_inode.c），分为三部分来读取文件，每次通过`sfs_bmap_load_nolock`函数获取文件索引编号，然后调用`sfs_buf_op`完成实际的文件读写操作。

```c
static int
sfs_io_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, void *buf, off_t offset, size_t *alenp, bool write) { 
    
    ...

    if ((blkoff = offset % SFS_BLKSIZE) != 0) {
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);  // 计算第一个数据块的大小
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {  // 找到内存文件索引对应的block的编号ino
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
            goto out;
        }
        // 完成实际的读写操作
        alen += size;
        if (nblks == 0) {
            goto out;
        }
        buf += size, blkno ++, nblks --;
    }
    // 读取中间部分的数据，将其分为size大学的块，然后一次读一块直至读完
    size = SFS_BLKSIZE;
    while (nblks != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0) {
            goto out;
        }
        alen += size, buf += size, blkno ++, nblks --;
    }
	// 读取第三部分的数据
    if ((size = endpos % SFS_BLKSIZE) != 0) {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {
            goto out;
        }
        alen += size;
    }
out:
    *alenp = alen;
    if (offset + alen > sin->din->size) {
        sin->din->size = offset + alen;
        sin->dirty = 1;
    }
    return ret;
}
```

### 问题

给出设计实现“UNIX的PIPE机制”的概要设方案

管道本质上就是一个操作系统内核管理的环形缓冲区，所以需要一块内存作为缓冲区，然后需要记录环形缓冲区的头部和尾部。当一个进程尝试从空管道读取数据或者向满管道写入数据的时候，操作系统内核需要将进程阻塞，所以还需要一个读取等待队列和一个写入等待队列。缓冲区大小通常设为一页的大小4KB。

## 练习2: 完成基于文件系统的执行程序机制的实现

改写proc.c中的load_icode函数和其他相关函数，实现基于文件系统的执行程序机制。

需要先初始化fs中的进程控制结构，即在`alloc_proc`函数中我们需要做一下修改，加上一句proc->filesp = NULL从而完成初始化。 修改之后`alloc_proc`函数如下：

```c
// alloc_proc - alloc a proc_struct and init all fields of proc_struct
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
        ...
        proc->filesp = NULL;  // 文件相关信息
    }
    return proc;
}
```



```c
// load_icode -  called by sys_exec-->do_execve 
static int
load_icode(int fd, int argc, char **kargv) {
    assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);
	// (1)建立内存管理器
    if (current->mm != NULL) {
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;  // E_NO_MEM代表因为存储设备产生的请求错误
    struct mm_struct *mm;  // 建立内存管理器
    if ((mm = mm_create()) == NULL) {
        goto bad_mm;
    }
    // (2)建立页目录
    if (setup_pgdir(mm) != 0) {
        goto bad_pgdir_cleanup_mm;
    }

    struct Page *page;
	// (3)从文件加载程序到内存
    struct elfhdr __elf, *elf = &__elf;
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {  // 读取elf文件头
        goto bad_elf_cleanup_pgdir;
    }

    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }

    struct proghdr __ph, *ph = &__ph;
    uint32_t vm_flags, perm, phnum;
    for (phnum = 0; phnum < elf->e_phnum; phnum ++) {  // e_phnum代表程序段入口地址数目
        off_t phoff = elf->e_phoff + sizeof(struct proghdr) * phnum;  // 循环读取程序的每个段的头部
        if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff)) != 0) {
            goto bad_cleanup_mmap;
        }
        if (ph->p_type != ELF_PT_LOAD) {
            continue ;
        }
        if (ph->p_filesz > ph->p_memsz) {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
        }
        if (ph->p_filesz == 0) {
            continue ;
        }
        vm_flags = 0, perm = PTE_U;  // 建立虚拟地址与物理地址之间的映射
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        if (vm_flags & VM_WRITE) perm |= PTE_W;
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0) {
            goto bad_cleanup_mmap;
        }
        off_t offset = ph->p_offset;
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);

        ret = -E_NO_MEM;
        // 复制数据段和代码段
        end = ph->p_va + ph->p_filesz;  // 计算数据段和代码段终止地址
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            if ((ret = load_icode_read(fd, page2kva(page) + off, size, offset)) != 0) {
                goto bad_cleanup_mmap;
            }
            start += size, offset += size;
        }
        // 建立BSS段
        end = ph->p_va + ph->p_memsz;

        if (start < la) {
            /* ph->p_memsz == ph->p_filesz */
            if (start == end) {
                continue ;
            }
            off = start + PGSIZE - la, size = PGSIZE - off;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
            assert((end < la && start == end) || (end >= la && start == la));
        }
        while (start < end) {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la) {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
        }
    }
    sysfile_close(fd);  // 关闭文件，加载程序结束
    // (4)建立相应的虚拟内存映射表
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
    // (5)设置用户栈
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

    // (6)处理用户栈中传入的参数，其中argc对应参数个数，uargv[]对应参数的具体内容的地址
    //setup argc, argv
    uint32_t argv_size=0, i;
    for (i = 0; i < argc; i ++) {
        argv_size += strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }

    uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
    char** uargv=(char **)(stacktop  - argc * sizeof(char *));
    
    argv_size = 0;
    for (i = 0; i < argc; i ++) {
        uargv[i] = strcpy((char *)(stacktop + argv_size ), kargv[i]);
        argv_size +=  strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }
    
    stacktop = (uintptr_t)uargv - sizeof(int);
    *(int *)stacktop = argc;

    // (7)设置进程的中断帧  
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = stacktop;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
    ret = 0;
    // (8)错误处理部分
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

`load_icode`主要是将文件加载到内存中执行：

- 建立内存管理器
- 建立页目录
- 将文件逐个段加载到内存中，这里要注意设置虚拟地址与物理地址之间的映射
- 建立相应的虚拟内存映射表
- 建立并初始化用户堆栈
- 处理用户栈中传入的参数
- 最后很关键的一步是设置用户进程的中断帧
- 发生错误还需要进行错误处理。

### 问题

给出设计实现基于“UNIX的硬链接和软链接机制”的概要设方案

**硬链接** 在SFS文件系统中已经实现了`nlinks`数据结构，代表了指向这个`inode`的硬链接个数，因此只需要添加一个系统调用（例如`SYS_link`），该系统调用首先找到被链接文件对应的`inode`，然后在目标文件夹的控制块中增加一个描述符即可，二者的`inode`指针应该相同，同时`nlinks`数据结构应该相应增加

创建：

1. 将目录项的名字设定为传入参数，目录项的inode号设置为目标文件的inode号。
2. 目标inode的引用计数增加。

删除：

1. 减少目标inode的引用计数。若减为0，清除目标inode及其数据块。

**软链接** 需要在`inode`上增加标记位确认这个文件是普通文件还是软链接文件，在进行打开文件或是保存文件的时候，操作系统需要根据软链接指向的地址再次在文件目录中进行查询，寻找或创建相应的`inode`，注意与硬链接不同，创建软链接的时候不涉及对`nlinks`的修改。如果需要创建软链接这个特殊的文件，也需要增加一个系统调用（例如`SYS_symlink`）在完成相应的功能。

创建：将inode的类型设置为符号链接，文件内容（数据）设置为目标路径字符串。

删除：不需要额外的操作。