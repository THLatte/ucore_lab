#lab5 实验报告
计22 徐天宇 2012011275

## 练习0 填写已有实验

```
由于lab1～lab4中有部分代码需要补充，这里说明一下补充的地方
首先是kern/trap/trap.c里的idt_init函数，补充设置了系统调用中断门，使得用户进程能够使用系统调用来获取ucore提供的服务。同一文件中的rap_dispatch函数，lab1中补充的时钟中断处理中加入了对当前进程的need_resched设置为1的操作，从而能够实现RR调度方法。
然后是kern/process/proc.c中的alloc_proc函数，因为proc_struct结构为了实现用户进程管理而进行了扩充，所以在初始化的时候还要对新的两个参数进行初始化，分别是wait_state、cptr、yptr和optr，分别初始化为0、NULL、NULL、NULL。同一文件中的do_fork函数中，添加了对proc的父进程的设置（确保父进程没有在等待其他子进程）。
```

### 与标准答案的区别

```
标准答案中对于kern/mm/vmm.c中函数do_pgfault部分进行了扩充，为事先COW机制做准备。参考了标准答案后在自己的代码中加入了这一段。
```

## 练习1 加载应用程序并执行（需要编码）

### 实现思路

```
需要补充的函数为kern/process/proc.c里的load_icode。
需要完成对trapframe的设定，使得在中断结束后，能够让CPU转到用户态，并回到用户态内存空间，使用用户态的代码段、数据段和堆栈，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断。
实验基本思路同注释中所述，设置tf_cs为USER_CS，这时已经设置好了CS段寄存器的最低两位为3，表示cpu将在用户态运行。然后设置tf_ds、tf_es和tf_ss为USER_DS，说明将要使用用户态的数据段。之后将tf_esp设置为USTACKTOP，表示将要使用用户态的堆栈。设置tf_eip为elf->e_entry，这样便设置了用户进程第一条指令的地点，当完成中断处理后，就能够跳转到用户进程的开头来进行执行。最后设置 tf_eflags为FL_IF（定义于kern/mm/mmu.h，是eflags中控制中断使能的位），代表对用户态使能了中断机制，这样用户态便能够相应中断。至此完成了trapframe的设置，当中断结束使用iret指令后就能使CPU转到用户态来执行。
```

### 与参考答案的区别

```
按照注释中所述来写代码，由于是固定程式，变量名都是预先设定好的，所以和标准答案没有大的区别。
```

### 请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

```
在proc_run函数中有实现，代码如下：
	local_intr_save(intr_flag);
	{
		current = proc;
		load_esp0(next->kstack + KSTACKSIZE);
		lcr3(next->cr3);
		switch_to(&(prev->context), &(next->context));
	}
	local_intr_restore(intr_flag);
首先关中断，然后设置cr3为要运行的用户态进程的页目录表起始地址，设置esp为用户态进程内核堆栈的栈顶，之后调用switch_to函数将先前的进程上下文切换为要运行的用户态进程上下文，完后开启中断，这时在用户态进程的PCB的中断帧中有存储第一条指令的地址（tf_eip），中断结束后会直接跳转到相应位置去执行。
```

## 练习2 父进程复制自己的内存空间给子进程（需要编码）

### 实现思路

```
需要补充的函数为kern/mm/pmm.c中的copy_range函数。
实现思路同注释所述，首先获取源内存区间和目标内存区间的内核虚拟地址，然后调用memcpy函数将源内存区间中的内容拷贝到目的内存区间中，最后调用page_insert函数设置好目的内存区间的物理地址和线性地址的映射关系。
```

### 与标准答案的不同

```
因为是按照注释的描述来写代码，都是固定的程式，所以和标准答案的实现差别并不大，除了变量名称有所不同以外。
```

### 请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。

```
在do_fork产生子进程的时候，修改传入的参数clone_flags，使得在调用copy_mm函数的时候，由实现复制内存改为父子进程共享内存，并设置对应的页表项为只读。对应的物理页设置reference_counter来对映射到它上面的进程数进行计数。当这些进程只是读取这个页的时候，返回相应的指针；当有进程需要改写它的时候，产生page_fault异常，将该物理页复制到一段新的物理内存中，然后修改它的内容，并修改相应进程的页表项使得其指向这段新的物理内存。
```

## 练习3 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）

### fork/exec/wait/exit的实现和系统调用的实现

```
fork的实现：
由函数do_fork来实现。
1. 调用alloc_proc函数来分配并初始化进程控制块TCB；
2. 调用setup_stack函数来分配并初始化内核栈；
3. 调用copy_mm函数，根据clone_flag标志复制或共享进程内存管理结构；
4. 调用设置copy_thread函数进程正常运行和调度所需的中断帧和执行上下文；
5. 关中断，把设置好的进程控制块放入hash_list和proc_list两个全局进程链表中，并设置进程的关联信息，然后开中断；
6. 调用wakeup_proc函数将进程状态设置为“就绪”态；
7. 设置返回码为子进程的id号。

exec的实现：
实现该功能的主要函数为do_execve和load_icode。
1. 首先清空用户态空间以加载新的执行码。如果mm不为NULL，则设置页表为内核空间页表(lcr3(boot_cr3))，且进一步判断mm的引用计数减1后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此将根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的mm内存管理指针为空。
2. 然后加载应用程序执行码到当前进程的新创建的用户态虚拟空间中，主要由load_icode函数完成。
	(1) 调用mm_create函数来申请进程的内存管理数据结构mm所需内存空间，并对mm进行初始化；
	(2) 调用setup_pgdir来申请一个页目录表所需的一个页大小的内存空间，并把描述ucore内核虚空间映射的内核页表（boot_pgdir所指）的内容拷贝到此新目录表中，最后让mm->pgdir指向此页目录表;
	(3) 根据应用程序执行码的起始位置来解析此ELF格式的执行程序，并调用mm_map函数根据ELF格式的执行程序说明的各个段（代码段、数据段、BSS段等）的起始位置和大小建立对应的vma结构，并把vma插入到mm结构中，从而表明了用户进程的合法用户态虚拟地址空间；
	(4) 调用根据执行程序各个段的大小分配物理内存空间，并根据执行程序各个段的起始位置确定虚拟地址，并在页表中建立好物理地址和虚拟地址的映射关系，然后把执行程序各个段的内容拷贝到相应的内核虚拟地址中；
	(5) 调用mm_mmap函数建立用户栈的vma结构，明确用户栈的位置在用户虚空间的顶端，大小为256个页，即1MB，并分配一定数量的物理内存且建立好栈的虚地址与物理地址的映射关系；
	(6) 把mm->pgdir赋值到cr3寄存器中，即更新了用户进程的虚拟内存空间；
	(7) 先清空进程的中断帧，再重新设置进程的中断帧，使得在执行中断返回指令“iret”后，能够让CPU转到用户态特权级，并回到用户态内存空间，使用用户态的代码段、数据段和堆栈，且能够跳转到用户进程的第一条指令执行，并确保在用户态能够响应中断；
3. 执行中断返回指令“iret”后，将切换到用户进程的第一条语句位置处开始执行。

wait的实现：
由函数do_wait实现。
1. 如果pid!=0，表示只找一个进程id号为pid的退出状态的子进程，否则找任意一个处于退出状态的子进程；
2. 如果此子进程的执行状态不为PROC_ZOMBIE，表明此子进程还没有退出，则当前进程只好设置自己的执行状态为PROC_SLEEPING，睡眠原因为WT_CHILD（等待子进程退出），调用schedule()函数选择新的进程执行，自己睡眠等待，如果被唤醒，则重复跳回步骤1处执行；
3. 如果此子进程的执行状态为PROC_ZOMBIE，表明此子进程处于退出状态，需要当前进程完成对子进程的最终回收工作，即首先把子进程控制块从两个进程队列proc_list和hash_list中删除，并释放子进程的内核堆栈和进程控制块。

exit的实现：
由函数do_exit实现。
首先，exit函数会把一个退出码error_code传递给ucore，ucore通过执行内核函数do_exit来完成对当前进程的退出处理，主要工作简单地说就是回收当前进程所占的大部分内存资源，并通知父进程完成最后的回收工作，具体流程如下：
1. 如果current->mm != NULL，表示是用户进程，则开始回收此用户进程所占用的用户态虚拟内存空间；
	(1) 首先执行lcr3(boot_cr3)，切换到内核态的页表上；
	(2) 如果当前进程控制块的成员变量mm的成员变量mm_count减1后为0（表明这个mm没有再被其他进程共享），则开始回收用户进程所占的内存资源：
		i.调用exit_mmap函数释放current->mm->vma链表中每个vma描述的进程合法空间中实际分配的内存，然后把对应的页表项内容清空，最后还把页表所占用的空间释放并把对应的页目录表项清空；
		ii.调用put_pgdir函数释放当前进程的页目录所占的内存；
		iii.调用mm_destroy函数释放mm中的vma所占内存，最后释放mm所占内存；
	(3) 设置current->mm为NULL；
2. 设置当前进程的执行状态current->state=PROC_ZOMBIE，当前进程的退出码current->exit_code=error_code；
3. 如果当前进程的父进程current->parent处于等待子进程状态，则通过执行执行wakeup_proc(current->parent)唤醒父进程，让父进程帮助自己完成最后的资源回收；
4. 如果当前进程还有子进程，则需要把这些子进程的父进程指针设置为内核线程initproc，且各个子进程指针需要插入到initproc的子进程链表中。如果某个子进程的执行状态是PROC_ZOMBIE，则需要唤醒initproc来完成对此子进程的最后回收工作。
5. 执行schedule()函数，选择新的进程执行。

系统调用的实现：
1. 初始化系统调用对应的中断描述符
2. 建立系统调用的用户库准备
3. 进行系统调用的执行
	(1)通过运行指令INT T_SYSCALL，CPU根据操作系统建立的系统调用中断描述符，转入内核态，并跳转到vector128处开始了操作系统的系统调用执行过程。关系如下所示：
		vector128(vectors.S)-->__alltraps(trapentry.S)-->trap(trap.c)-->trap_dispatch(trap.c)-->syscall(syscall.c)-->sys_getpid(syscall.c)-->……-->__trapret(trapentry.S)
	(2)在执行trap函数前，软件还需进一步保存执行系统调用前的执行现场，即把与用户进程继续执行所需的相关寄存器等当前内容保存到当前进程的中断帧trapframe中。
	(3)保存完毕后，操作系统可开始完成具体的系统调用服务。
	(4)完成服务后，操作系统按调用关系的路径原路返回到__alltraps中，然后操作系统开始根据当前进程的中断帧内容做恢复执行现场操作。
	(5)执行“IRET”指令，CPU根据内核栈的情况回复到用户态，并把EIP指向tf_eip的值，即INT T_SYSCALL后的那条指令。这样整个系统调用就执行完毕了。
```

### 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？

```
fork过程创建了新进程后调用wakeup_proc函数将该进程设置为RUNNABLE状态。
exec过程并不影响执行状态。
wait过程可能使得父进程的状态变成了SLEEPING，从而能够等待子进程的完成。
exit过程将相应的进程的执行状态设置成ZOMBIE，表示该进程已经结束。同时，也有可能将其父进程或initproc的状态从SLEEPING转变成RUNNABLE，从而协助完成资源的回收。
```

### 请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）

```                                            
  alloc_proc                                 RUNNING
      |                                   +--<----<--+
      |                                   | proc_run |
      V                                   +-->---->--+ 
PROC_UNINIT --(proc_init/wakeup_proc)-->  PROC_RUNNABLE --(try_free_pages/do_wait/do_sleep)--> PROC_SLEEPING --+
                                            A      |                                                            |
                                            |      +--- do_exit --> PROC_ZOMBIE                                 |
                                            |                                                                   | 
                                            +----------------------wakeup_proc----------------------------------+
```

## 程序运行结果

```
执行make qemu和bash tools/grade.sh命令后，获得的输出结果如下（只列出bash tools/grade.sh的结果）：
badsegment:              (1.4s)
  -check result:                             OK
  -check output:                             OK
divzero:                 (1.5s)
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
hello:                   (1.3s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.5s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (1.5s)
  -check result:                             OK
  -check output:                             OK
yield:                   (1.3s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (1.3s)
  -check result:                             OK
  -check output:                             OK
exit:                    (1.4s)
  -check result:                             OK
  -check output:                             OK
spin:                    (4.4s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (13.6s)
  -check result:                             OK
  -check output:                             OK
forktest:                (1.5s)
  -check result:                             OK
  -check output:                             OK
forktree:                (1.4s)
  -check result:                             OK
  -check output:                             OK
Total Score: 150/150
```

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点。

```
1. fork/exec/wait/exit的实现，对应OS原理中进程的复制和进程的生命周期（状态及状态之间的转换）
2. 内核态到用户态的切换
3. 系统调用的实现
```

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

```
1. 进程的挂起和激活
2. 进程的生命周期比较简化，比如就绪和运行合并为RUNNABLE，而缺少了调度过程（虽然只是简单的FCFS）。
```
