#lab8 实验报告
计22 徐天宇 2012011275

## 练习0：填写已有实验

```
将lab7的代码merge到lab8后，进一步修改了alloc_proc函数和do_fork函数。
alloc_proc:增加对进程中文件系统的文件相关信息filesp的初始化，将其初始化为NULL。
do_fork:增加将父进程文件信息拷贝到子进程的流程。
```

## 练习1: 完成读文件操作的实现（需要编码）

### 实现思路

```
首先处理读取起始位置和第一块不对齐的情况。此时先获取不对齐部分的大小，然后由内存inode编号获取磁盘inode编号并根据此编号调用sfs_buf_op函数从磁盘中进行读取，之后更新已读取长度，然后判断剩余块数nblks是否为0，是的话就表明已经读取完毕，跳转至out处，否则更新buffer位置buf、读取块号blkno以及剩余块数nblks，继续读取。
然后读取中间部分对齐块的内容。读取流程同上，只是不需要获取每次读取的大小，且对nblks的判断改变为while的条件。
最后处理读取结束位置和最后一块不对齐的情况，流程也和读取第一块不对齐的流程相同，只是不需要判断nblks以及更新参数。
```

### 请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案，鼓励给出详细设计方案

```
PIPE机制与文件系统相关联，而且提供给用户的只是往管道内送入数据以及从管道中取出数据两个函数。通过系统调用来进行管道的初始化和读写操作。其中读写操作应保证为互斥操作，同一时刻只能有一个进程对管道进行操作。
```

## 练习2: 完成基于文件系统的执行程序机制的实现（需要编码）

### 实现思路

```
首先为当前进程创建一个内存区域并建立新的页表，然后将代码段、数据段、BSS段复制到进程的内存区域中（读取文件并解析elfhdr和proghdr，然后调用mm_map函数来建立代码段和数据段的VMA，之后调用pgdir_alloc_page来为代码段、数据段和BSS段分配页，其中代码段和数据段的内容从文件中读取，BSS段的内容置0）。之后，调用mm_map函数建立用户栈，设置进程的内存空间、CR3寄存器和页表项，将程序参数传入栈中，最后设置进程的trapframe。
```

### 请在实验报告中给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案，鼓励给出详细设计方案

```
建立硬链接时，把新创建文件的inode指向目标文件对应的inode，并将目标文件的inode的引用计数加1；在删除文件时，将所指向的目标文件inode的引用计数减1，当且仅当inode的引用次数降为0时，才真正删除并回收inode和原文件。
建立软链接时，先创建一个新文件及其对应的inode，文件中存储目标路径，inode指定为链接类型。在删除时直接删除该链接文件及其对应的inode即可。
```

## 运行结果

```
执行make qemu后，输入ls命令和hello命令，输出如下：

ls
 @ is  [directory] 2(hlinks) 23(blocks) 5888(bytes) : @'.'
   [d]   2(h)       23(b)     5888(s)   .
   [d]   2(h)       23(b)     5888(s)   ..
   [-]   1(h)       10(b)    40386(s)   badsegment
   [-]   1(h)       10(b)    40516(s)   waitkill
   [-]   1(h)       11(b)    44640(s)   ls
   [-]   1(h)       10(b)    40404(s)   divzero
   [-]   1(h)       10(b)    40391(s)   faultreadkernel
   [-]   1(h)       10(b)    40382(s)   badarg
   [-]   1(h)       10(b)    40381(s)   pgdir
   [-]   1(h)       10(b)    40408(s)   testbss
   [-]   1(h)       10(b)    40435(s)   forktree
   [-]   1(h)       11(b)    44694(s)   sh
   [-]   1(h)       11(b)    44584(s)   matrix
   [-]   1(h)       10(b)    40404(s)   sleep
   [-]   1(h)       10(b)    40381(s)   hello
   [-]   1(h)       10(b)    40410(s)   forktest
   [-]   1(h)       10(b)    40380(s)   spin
   [-]   1(h)       10(b)    40383(s)   softint
   [-]   1(h)       10(b)    40381(s)   yield
   [-]   1(h)       11(b)    44571(s)   priority
   [-]   1(h)       10(b)    40385(s)   faultread
   [-]   1(h)       10(b)    40385(s)   sleepkill
   [-]   1(h)       10(b)    40406(s)   exit
lsdir: step 4
$ hello
Hello world!!.
I am process 14.
hello pass.

之后运行make grade，输出结果如下：
badsegment:              (3.5s)
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
hello:                   (2.8s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.4s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (2.7s)
  -check result:                             OK
  -check output:                             OK
yield:                   (2.7s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (2.9s)
  -check result:                             OK
  -check output:                             OK
exit:                    (2.7s)
  -check result:                             OK
  -check output:                             OK
spin:                    (3.4s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (3.5s)
  -check result:                             OK
  -check output:                             OK
forktest:                (2.9s)
  -check result:                             OK
  -check output:                             OK
forktree:                (5.6s)
  -check result:                             OK
  -check output:                             OK
priority:                (15.5s)
  -check result:                             OK
  -check output:                             OK
sleep:                   (11.4s)
  -check result:                             OK
  -check output:                             OK
sleepkill:               (3.0s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (12.5s)
  -check result:                             OK
  -check output:                             OK
Total Score: 190/190

可见程序运行正确。
```

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点

```
文件、目录
文件系统、简单文件系统、虚拟文件系统
磁盘索引节点、内存索引节点
文件分配、索引分配
文件描述符
I/O设备
```

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

```
空闲空间管理
```
