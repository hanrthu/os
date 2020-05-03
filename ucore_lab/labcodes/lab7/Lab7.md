### <center>Lab7实验报告</center>

<center>计71 韩荣 2016010189</center>

#### 练习1 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

##### 内核级信号量设计描述和基于此的哲学家就餐问题

​	执行流程为：在init进程创建用户程序空间后，进入check_sync函数，开始测试哲学家就餐问题。在kernel_exceve生成n个哲学家进程，然后开始测试，每个哲学家的初始状态为思考（Thinking）。

​	具体的流程为：使用互斥量mutex保护临界区，保证每次仅有一个哲学家在修改fork信息和自己的状态信息state_sema。使用s信号量表示他的左右餐具是否正被其他人占用，开始进餐时，如果左右都空闲，那么便可以占用进行进餐，否则，s信号量阻塞，如果有人用餐结束，则再次检查，直到空闲，才解除s的阻塞。

​	sem->value用于表示资源的占用情况，如果value大于0，表明资源空闲。up操作是表明用餐结束，释放餐具资源，down操作则是开始用餐，消耗餐具资源。如果没有可用资源，则会进入等待队列，开启中断，重新调度，直至有资源可用为止。

```c++
void up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    // 关闭中断，保证操作原子性
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        // 没有进程在等待时，释放该资源
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            // 有进程在等待，释放了当前资源，应唤醒下一个等待占用刀叉的嵇in成
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}

uint32_t down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    // 若资源还有剩余，可以正常访问，减去信号量。
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    // 若资源没有剩余，则将进程加入等待队列
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
  	//开启中断并重新调度进程
    local_intr_restore(intr_flag)
    schedule();

    // 若该进程被重新唤醒，则资源现在已经空闲，从等待队列中删去当前进程。
    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```

##### 用户态进程/线程的信号量机制设计方案

​	可以在用户态的进程控制块中添加一些新的函数，通过系统调用完成相应操作，以便在用户态可以对信号量进行操控：

​	new_sem：参数为value，表明分配多少资源，返回值为一个数据结构，其中包括为信号量分配的id和value。

​	up：参数为信号量的id，进行up操作，即用餐完毕，释放资源或唤醒其他进程

​	down：参数为信号量的id，进行down操作，即开始用餐，请求资源或等待

​	由于上述操作中会用到原子操作，因此会使用系统调用进入内核态，在内核态关闭中断进行相关操作和进程的切换，不过这样可能会稍微降低程序的效率。同时，进程间的同步也是一个问题，需要有一些事先的约定来保证同步互斥的正确进行。

#### 练习2 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

​	条件变量的定义：

```c
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

​	每次调用函数时，需要启动管程。

```c
    down(&(mtp->mutex));
```

​	调用完毕，退出管程。

```c
    if(mtp->next_count>0)
        up(&(mtp->next));
     else
        up(&(mtp->mutex));
```

​	条件变量的两个函数需要完善：

```c
void 
cond_signal (condvar_t *cvp) {
   //LAB7 EXERCISE1: 2016010189
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
  //若条件变量大于0
   if(cvp->count>0){
     	//将自己放在特殊的next中，以便下一次能够切换回自己
       cvp->owner->next_count ++;
     	//唤醒进程，让他们执行
       up(&(cvp->sem));
      //阻塞自己
       down(&(cvp->owner->next));
       cvp->owner->next_count--;
   }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}

```

```c
void
cond_wait (condvar_t *cvp) {
    //LAB7 EXERCISE1: 2016010189
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
  	//等待队列长度加一
    cvp->count ++;
  	//如果有next
    if(cvp->owner->next_count>0)
      	//唤醒next
        up(&(cvp->owner->next));
  	//否则直接唤醒
    else
        up(&(cvp->owner->mutex));
  	//阻塞自己
    down(&(cvp->sem));
    cvp->count --;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```



##### 	用户态条件变量机制设计方案

​	具体实现类似上面的函数，但是由于在用户态，因此需要相应的系统调用来配合完成，如练习1中的分析，可以类似的设计相关的函数接口。

##### 能否不基于信号量完成条件变量

​	题目中的解法为：

```c
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

​	如果不基于信号量，则可以自己实现条件变量的占用和释放函数，即手动完成up和down操作即可，用于阻塞和唤醒进程。



#### 实验与参考答案的区别

​	练习2:由于题目中有实现逻辑的提示，因此没有太大的差别

#### 实验中重要的知识点

本实验中用到的知识点：

​	信号量、管程、三种同步互斥的实现方法、条件变量的结构和哲学家就餐问题。

本实验中没有用到但又很重要的知识点：

​	生产者-消费者模型、用户进程同步互斥的编码实现

