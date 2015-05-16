#lab7 实验报告
计22 徐天宇 2012011275

## 练习0：填写已有实验

```
本实验中改进的代码包括trap.c中的trap_dispatch函数，将对时钟中断的处理更改为run_timer_list()，用于更新当前系统时间点，遍历当前所有处在系统管理内的计时器，找出其中所有应该激活的计时器，并将其激活。
```

## 练习1: 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题（不需要编码）

### 请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。

```
在ucore中，将其封装为数据结构semaphore_t，包含了一个整型变量value和一个进程等待队列wait_queue。
同时提供了4个函数sem_init、up、down和try_down。其中关键函数是up和down。
up函数模拟了信号量的V操作。首先关中断，如果信号量对应的wait_queue中没有进程在等待，则直接令value加一，然后开中断返回；如果有进程在等待且进程等待的原因是semophore设置的，则调用wakeup_wait函数将waitqueue中等待的第一个wait删除，且把此wait关联的进程唤醒，最后开中断返回。
down函数模拟了信号量的P操作。首先关中断，然后判断当前value是否大于0，如果是则让value减一并打开中断返回即可；如果不是则需要将当前的进程加入到等待队列中，然后开中断并调用schedule函数切换进程。如果被V操作唤醒，则关中断、把自身关联的wait从等待队列中删除并开中断。
```

### 请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

```
提供一个表示信号量的数据结构，该数据结构和内核中的信号量一一对应，当用户态中一个信号量被构建后，内核态中相应的生成一个信号量，两者建立映射关系，并关联相应进程，这些关系存储在内核的某个表中。然后提供相应于初始化、P操作和V操作的系统调用，对内核信号量进行操作处理。同时建立三个函数init、down和up作为上述三个系统调用的用户态接口，从而将用户态的信号量处理转移到内核态中。
同：从提供的功能上来看，都实现了相似的同步互斥机制。
异：用户态实现需要和内核态进行相应的交流和转换，而内核态实现则不需要这一点。用户态并不能直接操作信号量。
```

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）


### 实现思路

```
需要补全的函数包括kern/sync/monitor.c中的cond_signal和cond_wait以及kern/sync/check_sync.c中的phi_take_forks_condvar和phi_put_forks_condvar。
cond_signal函数首先判断count计数的大小，如果大于0则说明有进程需要唤醒，首先唤醒相应的进程（up），然后令自身进入睡眠状态（down）同时增加next_count计数。最后，当该进程被唤醒时，也就是down函数执行完毕后，将next_count减一。否则说明没有进程在等待，函数直接返回。
cond_wait函数首先对count计数加1，然后根据next_count计数进行相应的唤醒操作或者释放mutex（借助于信号量的up函数），然后让自身睡眠（借助信号量的down函数），当自身由于另一个进程的signal而被唤醒时，从这里继续执行下去。被唤醒后，令count计数减1。
phi_take_forks_condvar函数首先设置相应哲学家状态为饥饿状态（HUNGRY），然后调用phi_test_condvar函数检测该哲学家是否可以用餐并进行相应的操作，如果函数处理完成后该哲学家的状态不是EATING，也就是说该哲学家现在不能就餐（缺少某一边或两边的叉子），循环调用cond_wait函数令该哲学家进程进入等待状态。
phi_put_forks_condvar函数首先将该哲学家的状态更改为思考状态（THINKING），然后调用phi_test_condvar函数分别探测其左边和右边的哲学家能否就餐。
```

### 与标准答案的不同

```
首先在phi_take_forks_condvar函数中，与答案的不同之处在于循环cond_wait的时候缺少一句提示语句，已根据答案将其加入。
在phi_put_forks_condvar函数中，调用phi_test_condvar函数时，我原先传入的参数是(i + 1)%5和(i + 4)%5，然而答案是传入预先定义的宏LEFT和RIGHT，感觉这样的实现更好，便于修改，于是将其更改。
```

### 请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程。

```
ucore中的管程的数据结构monitor_t定义如下：

typedef struct monitor
{
    semaphore_t mutex;
    semaphore_t next;
    int next_count;
    condvar_t *cv;
} monitor_t;
其中mutex是实现每次只允许一个进程进入管程的关键元素，确保了互斥访问性质。条件变量cv通过执行wait_cv，会使得等待某个条件C为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件C为真并执行signal_cv时，能够让等待某个条件C为真的睡眠进程被唤醒，从而继续进入管程中执行。管程中的成员变量信号量next和整形变量next_count是配合进程对条件变量cv的操作而设置的，这是由于发出signal_cv的进程A会唤醒睡眠进程B，进程B执行会导致进程A睡眠，直到进程B离开管程，进程A才能继续执行，这个同步过程是通过信号量next完成的；而next_count表示了由于发出singal_cv而睡眠的进程个数。

管程中的条件变量的数据结构condvar_t定义如下：

typedef struct condvar{
    semaphore_t sem;
    int count;
    monitor_t * owner;
} condvar_t;
条件变量的定义中也包含了一系列的成员变量，信号量sem用于让发出wait_cv操作的等待某个条件C为真的进程睡眠，而让发出signal_cv操作的进程通过这个sem来唤醒睡眠的进程。count表示等在这个条件变量上的睡眠进程的个数。owner表示此条件变量的宿主是哪个管程。

cond_wait函数实现了wait_cv的操作，而cond_signal实现了signal_cv的操作，其执行流程在“实现思路”一节中由有说明。
```

### 请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

```
设计方案和用户态信号量基本相似，提供一个表示条件变量的数据结构，该数据结构和内核中的条件变量一一对应，当用户态中一个条件变量被构建后，内核态中相应的生成一个条件变量，两者建立映射关系，并关联相应进程，这些关系存储在内核的某个表中。然后提供相应于初始化、wait操作和signal操作的系统调用，对内核信号量进行操作处理。同时建立三个函数作为上述三个系统调用的用户态接口，从而将用户态的信号量处理转移到内核态中。
同：从提供的功能上来看，都实现了相似的同步互斥机制。
异：用户态实现需要和内核态进行相应的交流和转换，而内核态实现则不需要这一点。用户态并不能直接操作条件变量，而是通过系统调用接口从内核态来实现。
```

## 程序执行结果

```
执行make qemu和make grade后，获得的输出结果如下：
badsegment:              (2.8s)
  -check result:                             OK
  -check output:                             OK
divzero:                 (3.0s)
  -check result:                             OK
  -check output:                             OK
softint:                 (2.9s)
  -check result:                             OK
  -check output:                             OK
faultread:               (1.4s)
  -check result:                             OK
  -check output:                             OK
faultreadkernel:         (1.4s)
  -check result:                             OK
  -check output:                             OK
hello:                   (2.9s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.4s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (2.9s)
  -check result:                             OK
  -check output:                             OK
yield:                   (2.7s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (3.0s)
  -check result:                             OK
  -check output:                             OK
exit:                    (3.0s)
  -check result:                             OK
  -check output:                             OK
spin:                    (2.9s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (3.5s)
  -check result:                             OK
  -check output:                             OK
forktest:                (2.6s)
  -check result:                             OK
  -check output:                             OK
forktree:                (2.7s)
  -check result:                             OK
  -check output:                             OK
priority:                (15.4s)
  -check result:                             OK
  -check output:                             OK
sleep:                   (11.4s)
  -check result:                             OK
  -check output:                             OK
sleepkill:               (3.0s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (12.0s)
  -check result:                             OK
  -check output:                             OK
Total Score: 190/190

```

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点

```
禁用硬件中断的同步方法(local_intr_save和local_intr_restore)
信号量的概念、P操作、V操作
管程和条件变量、wait和signal
哲学家就餐问题
```

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

```
原子操作的应用
死锁和饥饿的避免（同步互斥方法的优化）
```
