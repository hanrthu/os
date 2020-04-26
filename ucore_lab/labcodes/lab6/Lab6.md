### <center>Lab6 实验报告</center>

<center>计71 韩荣 2016010189</center>

#### 练习1 使用Round Robin调度算法

##### 请理解并分析 sched_class 中各个函数指针的用法，并结合 Round Robin 调度算法描 ucore 的调度执行过程

​	阅读代码，可以发现sched_class调度器类的主要函数如下：

```c
    init:		//初始化调度器，设置调度类和相关参数
		proc_tick:   //中断后执行的程序，用于修改进程需要被调度的信息
  	pick_next: //选择下一个程序进行执行
    enqueue:	//将执行的进程放入等待队列
    dequeue:  //将一个进程移出等待队列
```

​	当操作系统调用schedule进行调度时，结合Round Robin调度算法，首先会让当前进程进入等待队列，调用enqueue函数实现，将进程放入run_queue中。然后调用pick_next函数选出等待队列最前方的进程，并将其剔除出等待队列，将其runs加一，然后调用proc_run运行进程，当时间片用完后，又会再次调用schedule执行进程的切换。

​	在ucore中，共有如下函数调用了schedule函数：do_exit,do_wait,init_main,cpu_idle,lock，这些事主动放弃CPU使用权，trap中调用schedule则是主动打断，用于特殊的需要进程强制放弃CPU使用权的情况。

##### 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

​	首先判断进程的时间片，随着时间片的长度减少，优先级逐渐增高，管理N个队列，在Process Control Block中设置不同的优先级，代表进入不同级别的队列中，在每次入队和出队的时候，都能够进行优先级的动态修改。每次进程调度时，选择非空优先级最高的队列中最靠前的那个进程进行执行，并根据当前进程的特征（长度，PCB中的prioty信息等），来对其进行对应的enqueue。



#### 练习2 实现 Stride Scheduling 调度算法

##### 思路

​	本算法基于skew heap类实现，只需要维护和更新rq->lab6_run_pool变量即可。填写练习1中的有关函数：

```c
    init:		//初始化rq和run_pool
		proc_tick:   //时钟中断后，减小time_slice，若时间片用完，则调度下一个。
  	pick_next: //选择stride最小的进程，并更新stride，让他前进BIG_STRIDE/priority
    enqueue:	//插入堆，并且rq进程数加一
    dequeue:  //将一个移出等待堆，删除根结点
```

​	具体实现请见代码。

​	要实现“Copy on Write”机制需要在每次fork拷贝mm_struct时，不拷贝，而是将指针指向原有的mm,还需设置为只读状态。这样读操作就可以正常进行了，如果要写，则会产生pagefault，而后则拷贝mm，然后进行修改即可。修改为只读不会影响vma_struct，因此，可以区分是COW的pagefault还是真实的pagefault。

​		

#### 实现与参考答案的区别

​	没有实现list相关的函数，逻辑大体一致。

#### 实验中重要的知识点

​	本次实验中重要的知识点：处理机调度、调度方法（RR、Stride）

​	未在本次试验中出现但仍然重要的知识点：FCFS、多处理机器调度等。





