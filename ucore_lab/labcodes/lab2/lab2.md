#### <center>Lab 2 实验报告</center>

<center>计71 韩荣 2016010189</center>

#### 练习1

​	在原有的代码基础上实现一个First-Fit Memory Alloc方法。首先分析原有代码的特点，原有代码一个块中仅仅将第一个page插入到free_list中，且插入无顺序性，无法通过default_check()的assert检查。

​	可以发现，这种代码的问题是，若无序，则会带来很严重的碎片化问题，且在释放后难以合并。

##### 我的First-Fit Memory Alloc方法

​	根据提示，需要修改四个程序中的内容——default_init,default_init_memmap,default_alloc_pages,default_free_pages。

​	首先，default_init是不需要修改的，因为初始化是相通的操作，即创建一个自环的free_list，然后让nr_free = 0；

​	然后修改default_init_memmap，我们可以按页为单位插入free_liist中，考虑到双向链表在加入之后是一个环形，所以在init_memmap时，使用list_add_before可以很方便地按顺序插入，且省去了索引的时间，将每一个页的flags设置为0，然后除了base外，其余的property均设置为0，base的设置为n。

​	接下来修改default_alloc_pages，在创建块时，依次寻找list中的page，直到寻找到page->property大于等于n的，即说明可以在此存放这么大的信息，之后设置reserved信息，并根据情况修改新的空闲块的大小。

​	最后修改default_free_pages，将块释放之后，需要判断能否合并，因此我们在计算中，需要确定是如下两种情况中的哪一种：1、base处于低地址而合并的块处于高地址；2、base处于高地址而合并的块处于低地址。由于我们在init时是按照地址高低init的，因此，在此我们可以找到第一次p>base时的块，其前方即为我们插入base块的位置，然后恢复base的flag和reserve信息信息。之后依次比较，判断出是上述哪种情况，即可完成合并，更新property。

##### 进一步改进的空间

​	本设计是有进一步改进空间的，因为将每一页都插入，虽然使得控制更加细粒度，但也增加了每次搜索的成本，可以用一个额外的表记录块长度信息，然后每次仅需按照长度跳转，即可略过中间的无用搜寻时间，增加效率。

​	另外，还可以将上述的长度信息中各个空闲块的大小进行排序，这样更加方便索引。同时，在碎片合并的时候，可以采用懒惰合并的方法，并不一定每一次释放都要合并，而是过一段时间合并一次，这样会减少搜索所浪费的时间。

##### 与参考答案的区别

​	观察参考答案，可以发现大体逻辑相似。在初始化时，我的方法是在循环哪判断base，而参考答案是在循环外判断，同时一些具体的实现也有所不同，总体来说都是按照注释的逻辑实现。



#### 练习2

代码如下：

```c
pte_t *get_pte(pde_t *pgdir, uintptr_t la, bool create) {
		pde_t *pdep = &pgdir[PDX(la)];
    struct Page *page;
    if((*pdep & PTE_P) == 0){
        if(!create){
            return NULL;
        }
        assert(page = alloc_page());
        set_page_ref(page,1);
        uintptr_t pa = page2pa(page);
        memset(KADDR(pa),0,PGSIZE);
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    pte_t *x = KADDR(PDE_ADDR(*pdep));
    return &x[PTX(la)];
}
```

首先找到页目录项，判断它是否存在。若不存在，则需要新建一个，当create为0时，直接返回NULL。按照页表项的格式填入，最后，需要返回页目录项中所指出的页表中页表项的内容。

##### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对 ucore 而言的潜在用处

​	PDE的各个组成部分如下：

```
|-----31~12-----|----------11~0---------| 比特
								|b|a|9|8|7|6|5|4|3|2|1|0| 占位
|-----index-----| AVL |G|P|0|A|P|P|U|R|P|	属性
												|S|		|C|W|/|/|
															|D|T|S|W|
```

​	PTE的各个组成部分如下：

```
|-----31~12-----|----------11~0---------| 比特
								|b|a|9|8|7|6|5|4|3|2|1|0| 占位
|-----index-----| AVL |G|P|D|A|P|P|U|R|P| 属性
												|A|		|C|W|/|/|
												|T|		|D|T|S|W|
```

可以发现二者结构相似，仅有部分位不同（参考链接http://www.gyarmy.com/post-424.html）

其中各个属性的含义是：

- P：有效位。0 表示当前表项无效。
- R/W: 0 表示只读。1表示可读写。
- U/S: 0 表示特技用户可访问，1表示普通用户可访问。
- A: 0 表示该页未被访问，1表示已被访问。
- D: 脏位。0表示该页未写过，1表示该页被写过。
- PS: 只存在于页目录表。0表示这是4KB页，指向一个页表。1表示这是4MB大页，直接指向物理页。

##### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

x86的MMU访存需要进行多级页表查找，若pde或是pte中的页不存在，则会产生Page Fault异常，来说CPU会进行如下操作：

​	1、保存现场并将当前寄存器存到主存储器中；

​	2、由对应寄存器记录出错程序的地址；

​	3、特权级切换；

​	4、根据异常号读取idt表，确定ISR的地址，判断是否有进入中断门的权限；

​	5、跳转到ISR的起始地址开始执行。

#### 练习3

代码如下：

```c
static inline void page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
if(*ptep & PTE_P){
        struct Page *page = pte2page(*ptep);
        page_ref_dec(page);
        if(page_ref(page) == 0){
            free_page(page);
        }
    }
    *ptep = 0;
    tlb_invalidate(pgdir,la);
}
```

首先判断页表项是否存在，如果存在，则对相应的页进行移除操作，移除后减少它的reference数量， 如果到0了，就释放这一页。然后清空pte，并对表项进行更新。

##### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系？

Page数组主要管理连续的物理内存，而页目录项和页表项均存在物理内存的特定页中，页表对应的项是物理地址，可以以此找到Page结构。Page结构储存在单独的内存区域，内核态内存管理中，页表中连续的项在Page结构中也是连续的。

##### 如果希望虚拟地址与物理地址相等，则需要如何修改Lab2，完成此事？

​	ucore在启动过程中同时有段机制和页机制，开始uCore主要使用段机制进行地址偏移、后来uCore使用页机制进行地址偏移。二者的切换过程为：uCore建立好偏移的页表后，将第0项的页表偏移取消。页机制启动后，内核处的偏移关系仍然是由段机制来维护的。启用页表之后，读取gdt表的代码能够正常执行，完成gdt表的修改之后，页机制的映射立刻起作用。之后才能删除pde索引为0的相关页目录项。

​	具体可以将entry.S中KERBASE有关的内容去除，ucore起始地址改为0x00100000。

#### 与参考答案的区别

​	按照注释逻辑实现，因此逻辑上区别不大，写法上有些区别。

#### 本实验中重要的知识点

First-Fit和Buddy算法，PTE、PDE的结构，内存管理机制，多级页表，页表自映射

#### 实验中未涉及的内容

段式存储的计算，碎片管理，Page fault的处理。

