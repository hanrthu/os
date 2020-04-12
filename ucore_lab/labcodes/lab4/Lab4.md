### <center>Lab4</center>

<center>计71 韩荣 2016010189</center>

#### 练习1

##### 思路

​	本练习是对一个struct proc_struct进行初始化，由于输入参数为空，可以推测其实仅需要给一个最基本的初始值，即将一些基本量清零，然后给一些特殊量如pid赋值为-1，proc->state设置为uninit，cr3设置为bootcr3即可，具体代码如下：

```c
		proc->state = PROC_UNINIT;
    proc->pid = -1;
    proc->runs = 0;
    proc->kstack = 0;
    proc->need_resched = 0;
    proc->parent = NULL;
    proc->mm = NULL;
    memset(&(proc->context),0,sizeof(struct context));	//将上下文清零
    proc->tf = NULL;
    proc->cr3 = boot_cr3;
    proc->flags = 0;
    memset(&(proc->name),0,sizeof(PROC_NAME_LEN));	//将名字清零
    }
    return proc;
```



##### 请说明 proc_struct 中`struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？

context保存进程的上下文，里面有一些寄存器，用于存放栈指针、参数等信息，在进程切换的时候会用到（具体是在switch_to时会用到），tf时中断时用于储存用户态上下文的结构体，用于从用户态到内核态切换时，在iret时会被用到。即context是用于内核态切换，tf用于中断，即用户态向内核态切换。



#### 练习2

##### 思路

​	本练习是在上一个练习后进行，需要完成do_fork函数，创建新的进程，在实验过程中，仅需要按照提示信息的思路来进行代码编写，值得注意的是，需要进行必要的合法性检查，以保证成功创建了新的进程，具体代码如下：

```c
 		//创建一个初始化的进程
		if((proc = alloc_proc()) == NULL){
        panic("canot alloc child proc");
        goto fork_out;
    }
		//为新进程创建栈
    if((setup_kstack(proc)) < 0){
        panic("cannot setup kstack");
        ret = - E_BAD_PROC;
        goto bad_fork_cleanup_proc;
    }
		//复制mm，在本实验中为NULL
    if(copy_mm(clone_flags,proc) < 0){
        panic("cannot copy mm");
        goto bad_fork_cleanup_kstack;
    }
		//复制父进程的trapfram并将子进程返回值设为0
    copy_thread(proc,stack,tf);
    bool intr;
		//将父进程设为当前进程
    proc->parent = current;
		//在临界区进行操作（即屏蔽中断）
    local_intr_restore(intr);
    proc->pid = get_pid();
    hash_proc(proc);
    list_add(&proc_list,&(proc->list_link));
    nr_process++;
    local_intr_save(intr);
		//改变进程状态，设置返回值为pid
    proc->state = PROC_RUNNABLE;
    ret = proc->pid;
    return ret;
```

##### 请说明 ucore 是否做到给每个新 fork 的线程一个唯一的 id？请说明你的分析和理由

​	是的，可以发现，在get_pid时，是屏蔽了中断的，也就是说当前操作具有原子性，且在get_pid函数内部保证了不会分配已有的pid号，因此是一个唯一的id。



#### 练习3

​	本段代码并不长，其作用仅为将当前进程切换到下一个进程proc，具体的做法是：

​	（1）关闭中断

​	（2）将current设置为proc

​	（3）读取proc的相关信息，如esp0，cr3等

​	（4）调用switch_to切换到proc，switch_to是一个汇编代码，储存于switch.S

​	（5）重新打开中断

##### 在本实验的执行过程中，创建且运行了几个内核线程？

​	创建且运行了两个内核线程，分别是idle和init，idle作为第零个线程被创建，然后切换到init，打印出需要打印的内容之后，线程结束，程序退出。

##### 语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

​	可以观察两个函数的定义如下：

```c
static inline bool
__intr_save(void) {
    if (read_eflags() & FL_IF) {
        intr_disable();
        return 1;
    }
    return 0;
}

static inline void
__intr_restore(bool flag) {
    if (flag) {
        intr_enable();
    }
}
```

​	即save是屏蔽中断，而restore是打开中断，其目的是保证两个命令之间的内容是一个原子操作，类似的用法在do_fork中也用到了。



#### 实验中的重要知识点

​	练习1：进程的概念，进程控制块的结构及初始化的方法

​	练习2：创建新进程的方法，内核进程，中断屏蔽

​	练习3：进程切换，以及相关参数的修改

#### 实现中与参考答案的区别

​	练习1:由于都是初始化，与实验答案逻辑上差不多

​	练习2:在出错的情况中，增加了新的panic提供出错信息，以及更改了ret的值