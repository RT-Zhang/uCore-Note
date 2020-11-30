#        Lab7 同步

## OS原理

- 操作系统的同步互斥的设计实现；
- 理解底层支撑技术：禁用中断、定时器、等待队列；
- 信号量机制的具体实现；
- 基于管程（monitor）的条件变量（condition variable）；
- 经典进程同步问题

## 练习1: 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

信号量结构体

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

包含了用于计数的整数值value，和一个进程等待队列wait_queue，一个等待的进程会挂在此等待队列上。

重要的信号量操作是P操作函数`down(semaphore_t *sem)`和V操作函数`up(semaphore_t *sem)`。但这两个函数的具体实现是`_down(semaphore_t *sem, uint32_t wait_state)` 函数和`__up(semaphore_t *sem, uint32_t wait_state)`函数，二者的具体实现描述如下：

```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);  // 关中断
    if (sem->value > 0) {  // 可以获得信号量
        sem->value --;
        local_intr_restore(intr_flag);  // 打开中断
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);  // 将当前进程加入等待队列
    local_intr_restore(intr_flag);  // 打开中断

    schedule();

    local_intr_save(intr_flag);  // 被唤醒，关中断
    wait_current_del(&(sem->wait_queue), wait);  // 将当前进程移出等待队列
    local_intr_restore(intr_flag);  // 打开中断

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```

先关掉中断，然后判断当前信号量的value是否大于0。如果是>0，则表明可以获得信号量，故让value减一，并打开中断返回即可；如果不是>0，则表明无法获得信号量，故需要将当前的进程加入到等待队列中，并打开中断，然后运行调度器选择另外一个进程执行。如果被V操作唤醒，则把自身关联的wait从等待队列中删除（此过程需要先关中断，完成后开中断）。

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);  // 关中断
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {  // 没有进程等待
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);  // 有进程等待
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);  // 将第一个wait删除，唤醒进程
        }
    }
    local_intr_restore(intr_flag);  // 打开中断
}
```

首先关中断，如果信号量对应的wait queue中没有进程在等待，直接把信号量的value加一，然后开中断返回；如果有进程在等待且进程等待的原因是semophore设置的，则调用wakeup_wait函数将waitqueue中等待的第一个wait删除，且把此wait关联的进程唤醒，最后开中断返回。

对照信号量的原理性描述和具体实现，可以发现二者在流程上基本一致，只是具体实现采用了关中断的方式保证了对共享资源的互斥访问，通过等待队列让无法获得信号量的进程睡眠等待。另外，我们可以看出信号量的计数器value具有有如下性质：

* value>0，表示共享资源的空闲数
* vlaue<0，表示该信号量的等待队列里的进程数
* value=0，表示等待队列为空

问题：

内核级信号量的设计描述，并说明其大致执行流程。

* down()函数：先关掉中断，然后判断当前信号量的value是否大于0。如果是>0，则表明可以获得信号量，故让value减一，并打开中断返回即可；如果不是>0，则表明无法获得信号量，故需要将当前的进程加入到等待队列中，并打开中断，然后运行调度器选择另外一个进程执行。如果被V操作唤醒，则把自身关联的wait从等待队列中删除（此过程需要先关中断，完成后开中断）。

* up()函数：首先关中断，如果信号量对应的wait queue中没有进程在等待，直接把信号量的value加一，然后开中断返回；如果有进程在等待且进程等待的原因是semophore设置的，则调用wakeup_wait函数将waitqueue中等待的第一个wait删除，且把此wait关联的进程唤醒，最后开中断返回。

给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

用户态的进程/线程的信号量的数据结构和内核级的是一样的。 对于用户态的线程/进程使用信号量机制，应该首先通过系统调用进行sem的初始化，设置sem.value以及sem.wait_queue，而在初始化之后，在使用这个信号量时，通过P操作与V操作，也是通过系统调用进入到内核中进行处理，简称是否等待或者释放资源。

不同：在用户态使用信号量时，需要进行系统调用进入到内核态进行操作。

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

首先掌握管程机制，然后基于信号量实现完成条件变量实现，然后用管程机制实现哲学家就餐问题的解决方案（基于条件变量）。

条件变量（CV）

一个条件变量CV可理解为一个进程的等待队列，队列中的进程正等待某个条件C变为真。每个条件变量关联着一个断言 “断言 (程序)”)Pc。当一个进程等待一个条件变量，该进程不算作占用了该管程，因而其它进程可以进入该管程执行，改变管程的状态，通知条件变量CV其关联的断言Pc在当前状态下为真。因此对条件变量CV有两种主要操作：

wait_cv： 被一个进程调用，以等待断言Pc被满足后该进程可恢复执行. 进程挂在该条件变量上等待时，不被认为是占用了管程。
signal_cv：被一个进程调用，以指出断言Pc现在为真，从而可以唤醒等待断言Pc被满足的进程继续执行。

管程的结构体

```c++
typedef struct monitor{
    semaphore_t mutex;	// 二值信号量，只允许一个进程进入管程，初始化为1
    semaphore_t next;	// 配合cv，用于进程同步操作的信号量
    int next_count;		// 睡眠的进程数量
    condvar_t *cv;		// 条件变量cv
} monitor_t;
```

管程中的成员变量mutex是一个二值信号量，是实现每次只允许一个进程进入管程的关键元素，确保了互斥访问性质。管程中的条件变量cv通过执行wait_cv，会使得等待某个条件C为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件C为真并执行signal_cv时，能够让等待某个条件C为真的睡眠进程被唤醒，从而继续进入管程中执行。管程中的成员变量信号量next和整形变量next_count是配合进程对条件变量cv的操作而设置的，这是由于发出signal_cv的进程A会唤醒睡眠进程B，进程B执行会导致进程A睡眠，直到进程B离开管程，进程A才能继续执行，这个同步过程是通过信号量next完成的；而next_count表示了由于发出singal_cv而睡眠的进程个数。

condvar_t结构体

```c++
typedef struct condvar {
    semaphore_t sem;	// 用于发出wait_cv操作的等待某个条件C为真的进程睡眠
    int count;			// 在这个条件变量上的睡眠进程的个数
    monitor_t * owner;	// 此条件变量的宿主管程
} condvar_t;
```

条件变量的定义中也包含了一系列的成员变量，信号量sem用于让发出wait_cv操作的等待某个条件C为真的进程睡眠，而让发出signal_cv操作的进程通过这个sem来唤醒睡眠的进程。count表示等在这个条件变量上的睡眠进程的个数。owner表示此条件变量的宿主是哪个管程。

monitor_init的实现

```c++
void     
monitor_init (monitor_t * mtp, size_t num_cv) {
    int i;
    assert(num_cv>0);
    mtp->next_count = 0; //设置next_count为0
    mtp->cv = NULL;
    sem_init(&(mtp->mutex), 1); //unlocked
    sem_init(&(mtp->next), 0);
    mtp->cv =(condvar_t *) kmalloc(sizeof(condvar_t)*num_cv);
    assert(mtp->cv!=NULL);
    for(i=0; i<num_cv; i++){
        mtp->cv[i].count=0; //设置cv的count为0
        sem_init(&(mtp->cv[i].sem),0);
        mtp->cv[i].owner=mtp;
    }
}
```

对条件变量进行初始化，设置next_count为0，对mutex，next进行初始化， 并分配num个condvar_t,设置cv的count为0，初始化cv的sem和owner。

cond_wait的实现

```c++
void
cond_wait (condvar_t *cvp) {
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
      cvp->count++; // 需要睡眠的进程个数加一
      if(cvp->owner->next_count > 0) 
         up(&(cvp->owner->next)); // 唤醒进程链表中的下一个进程
      else
         up(&(cvp->owner->mutex)); // 唤醒睡在monitor.mutex上的进程 
      down(&(cvp->sem));  // 将此进程等待  
      cvp->count --;  // 睡醒后等待此条件的睡眠进程个数减一
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

可以看出如果进程A执行了`cond_wait`函数，表示此进程等待某个条件C不为真，需要睡眠。因此表示等待此条件的睡眠进程个数cv.count要加一。接下来会出现两种情况。

情况一：如果`monitor.next_count`如果大于0，表示有大于等于1个进程执行`cond_signal`函数且睡着了，就睡在了`monitor.next`信号量上。假定这些进程形成S进程链表。因此需要唤醒S进程链表中的一个进程B。然后进程A睡在`cv.sem`上，如果睡醒了，则让`cv.count`减一，表示等待此条件的睡眠进程个数少了一个，可继续执行。

情况二：如果`monitor.next_count`如果小于等于0，表示目前没有进程执行`cond_signal`函数且睡着了，那需要唤醒的是由于互斥条件限制而无法进入管程的进程，所以要唤醒睡在`monitor.mutex`上的进程。然后进程A睡在`cv.sem`上，如果睡醒了，则让`cv.count`减一，表示等待此条件的睡眠进程个数少了一个，可继续执行了！

cond_signal的实现

```c++
void 
cond_signal (condvar_t *cvp) {
   //LAB7 EXERCISE1: YOUR CODE
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
     if(cvp->count>0) { //当前存在执行cond_wait而睡眠的进程 
        cvp->owner->next_count ++; //睡眠的进程总个数加一  
        up(&(cvp->sem)); //唤醒等待在cv.sem上睡眠的进程 
        down(&(cvp->owner->next)); //自己需要睡眠
        cvp->owner->next_count --; //睡醒后等待此条件的睡眠进程个数减一
      }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

首先进程B判断`cv.count`，如果不大于0，则表示当前没有睡眠的进程，因此就没有被唤醒的对象了，直接函数返回即可； 如果大于0，这表示当前有睡眠的进程A，因此需要唤醒等待在`cv.sem`上睡眠的进程A。由于只允许一个进程在管程中执行，所以一旦进程B唤醒了别人（进程A），那么自己就需要睡眠。故让`monitor.next_count`加一，且让自己（进程B）睡在信号量`monitor.next`上。如果睡醒了，这让`monitor.next_count`减一。

哲学家问题

```c++
struct proc_struct *philosopher_proc_condvar[N]; //N个哲学家
int state_condvar[N];                            //哲学家的状态
monitor_t mt, *mtp=&mt;                          //管程123
```

信号量实现

```c++
semphore_t mutex 临界区互斥信号量
semphore_t s[N] 每个哲学家一个信号量12
```

试图得到叉子

```c
void phi_test_condvar (i) { 
   // 首先判断左右的状态是不是“eatting”
    if (state_condvar[i] == HUNGRY && state_condvar[LEFT] != EATING
        	&& state_condvar[RIGHT]!=EATING) {
        cprintf("phi_test_condvar: state_condvar[%d] will eating\n",i);
        state_condvar[i] = EATING ; //改变自己状态为eating
        cprintf("phi_test_condvar: signal self_cv[%d] \n",i);
        cond_signal(&mtp->cv[i]) ; //得到条件变量cv[i]
    }
}
```

首先判断左右的状态不是“eatting”然后是自己的状态变成“EATING”。

拿叉子

```c
void phi_take_forks_condvar(int i) {
     down(&(mtp->mutex));  //通过P操作进入临界区
      state_condvar[i]=HUNGRY; //记录哲学家i是否饥饿
      phi_test_condvar(i);   //试图拿到叉子 
      if (state_condvar[i] != EATING) {
          cprintf("phi_take_forks_condvar: %d didn't get fork and will wait\n",i);
          cond_wait(&mtp->cv[i]); //得不到叉子就睡眠
      }
      if(mtp->next_count>0)  //如果存在睡眠的进程则那么将之唤醒
         up(&(mtp->next));
      else
         up(&(mtp->mutex));
}
```

放叉子

```c
void phi_put_forks_condvar(int i) {
     down(&(mtp->mutex)); ;//通过P操作进入临界区
      state_condvar[i]=THINKING; //记录进餐结束的状态
      phi_test_condvar(LEFT); //看一下左边哲学家现在是否能进餐
      phi_test_condvar(RIGHT); //看一下右边哲学家现在是否能进餐
     if(mtp->next_count>0) //如果有哲学家睡眠就予以唤醒
        up(&(mtp->next));
     else
        up(&(mtp->mutex)); //离开临界区
}
```

### 问题

能否不用基于信号量机制来完成条件变量？

可以，条件变量维护者进程的等待队列和进程的等待数目，实际上用等待队列和锁机制同样可以实现。