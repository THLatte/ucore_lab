#lab6 实验报告

## 练习0：填写已有实验

```
本实验中改进的代码包括proc.c中的alloc_proc函数和trap.c中的trap_dispatch函数。
alloc_proc函数中，增加了对进程控制块中增加的新变量的初始化，包括运行队列rq、运行队列链接点run_link、时间片time_slice、优先队列节点lab6_run_pool、步进值lab6_stride和优先级lab6_priority。
trap.c函数中，修改了对时间中断的处理，增加了进程调度相关的处理。每一次时钟中断触发时，将计时器加1，同时删除了lab5中针对FCFS调度策略所做的处理，将其修改为每一次时钟中断都进行以此调度判断，调用函数sched_class_proc_tick来修改当前进程的time_slice数值。由于原代码中sched_class_proc_tick函数为static函数，无法访问非静态的current，所以将sched.c中定义的该函数前的static删除，以便于调用该函数进行处理。
```

## 练习1: 使用 Round Robin 调度算法（不需要编码）

### 请理解并分析sched_class中各个函数指针的用法，并接合Round Robin调度算法描述ucore的调度执行过程

```
.init:指向RR_init函数，实现RR算法调度器的初始化
.enqueue:指向RR_enqueue，实现将一个进程控制块加入到就绪进程队列中，必要时设置其时间片。同时增加就绪进程计数
.dequeue：指向RR_dequeue，实现将一个进程控制块从就绪进程队列中删除，同时将就绪进程计数减1
.pick_next:指向RR_pick_next，实现将就绪进程队列的头元素取出并转化为进程控制块
.proc_tick:指向RR_proc_tick，实现将正在运行的进程的时间片减一并判断时间片是否已经用完，如果用完则进行调度，修改调度标记

调度执行过程在kern/schedule/sched.c文件的schedule函数中。首先关中断，如果当前进程是就绪状态，则调用函数sched_class_enqueue将其插入倒就绪队列中，然后调用sched_class_pick_next函数从就绪队列中挑选一个的进程并用sched_class_dequeue函数将其从队列中摘出（如果没有则选择idleproc），如果选择的进程不是刚刚入队的进程，就调用proc_run函数将其转为运行状态，取代之前入队的进程。最后开中断，完成调度过程。
```

### 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

```
根据优先级的不同设置多个队列，可以建立一个数组按优先级从高到低的顺序存放每一个队列的头地址。
优先级越低的队列，其内部的进程的时间片大小越大。
每个进程首次创建后，进入优先级最高的队列中等待调度。
当进行调度时，如果该进程的时间片用完后进程的处理仍然没有结束，则将其插入到优先级次一级的队列中等待，并将其时间片置为该队列对应的时间片大小，同时从优先级最高的队列中取出一个进程令其执行（选择时可采用FCFS的方法）。
在查找需要插入的队列的起始地址时，进程的优先级可以作为下标，直接从数组中获得。
```

## 练习2: 实现 Stride Scheduling 调度算法（需要编码）

### 设计思路

```
首先设置BIG_STRIDE大小为0x7fffffff,为最大的int。

stride_init：初始化rq中的run_list（调用list_init）、lab6_run_pool（设为NULL）以及proc_num（设为0）。

stride_enqueue：首先将传入的proc插入到rp的进程池中（调用函数skew_heap_insert），然后根据情况设置proc的时间片大小，将proc中的rq指针指向当前的rq，最后将rq中的proc_num参数加1，表示rq中加入了一个新的进程。

stride_dequeue：直接调用函数skew_heap_remove将进程proc从进程池中删除，同时减少rq中的进程计数（proc_num减1）

stride_pick_next：首先检查rq的进程池是否为空，如果为空则表明当前无进程可选择，返回NULL。如果不为空，则直接将进程池的首元素取出，调用函数le2proc将其转化为进程控制块结构，然后修改其stride参数，返回该元素。

stride_proc_tick：首先将传入的proc中的时间片参数减1，然后查看时间片是否被用完（为0），如果是则将need_resched参数置为1，表明该进程时间片已经被用完，需要调度一个新的进程。该函数配合时间中断处理来使用，由于是静态函数无法调用非静态变量，所以将static标志删除，以便在kern/trap/trap.c中调用该函数。
```

### 程序运行结果

```
badsegment:              (2.0s)
  -check result:                             OK
  -check output:                             OK
divzero:                 (1.4s)
  -check result:                             OK
  -check output:                             OK
softint:                 (1.4s)
  -check result:                             OK
  -check output:                             OK
faultread:               (1.4s)
  -check result:                             OK
  -check output:                             OK
faultreadkernel:         (1.4s)
  -check result:                             OK
  -check output:                             OK
hello:                   (1.4s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.4s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (1.4s)
  -check result:                             OK
  -check output:                             OK
yield:                   (1.3s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (1.4s)
  -check result:                             OK
  -check output:                             OK
exit:                    (1.4s)
  -check result:                             OK
  -check output:                             OK
spin:                    (1.6s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (1.9s)
  -check result:                             OK
  -check output:                             OK
forktest:                (1.4s)
  -check result:                             OK
  -check output:                             OK
forktree:                (1.4s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (11.6s)
  -check result:                             OK
  -check output:                             OK
priority:                (11.4s)
  -check result:                             OK
  -check output:                             OK
Total Score: 170/170
```

## 与标准答案的区别

```
区别在于kern/trap/trap.c中对时间中断的处理上，答案中直接断言current不为NULL后就完成了处理，并没有调用sched_class_proc_tick函数对时间片进行相应的处理，在我的实现中加入了这一个实现，同时为了能够正常运行，将该函数的static声明删除。
```

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点

```
RR调度算法的实现
stride scheduling调度算法的实现
进程的优先级
进程的调度
```

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

```
多级反馈队列调度算法
短进程优先调度算法
最高相应比优先调度算法
公平共享调度算法
实时调度、多处理器调度
优先级反置
```
