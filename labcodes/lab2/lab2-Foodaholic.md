#lab2 实验报告 
计22 徐天宇 2012011275

## 练习1 实现 first-fit 连续物理内存分配算法

```
修改的函数包括default_init_memmap，default_alloc_pages，default_free_pages。

对于default_init_memmap函数，原本的实现中只完成了将Head Page添加到free\_list中，而没有将其他的页表连接起来，同时也没有设置其他页表的p\_flag bit1为可用状态，这样的实现是错误的。因此我的实现中，首先把所有页表建好，同时完成对p\_flag的bit1的设置（调用SetPageProperty函数），使该页为可分配状态，然后再按照注释中的说明设置各项参数，最后将每一页都链接起来加入到free\_list中。最后将base（也就是Head Page）的property参数设为n，表明这一个空闲块含有n个空闲页面。最后将空闲页面总数加n。

对于default_alloc_pages函数，原本的实现只是将Head Page从列表中移除并修改相关参数，而且也没有设置分配出去的page为reservsed。于是我的实现中，我首先按照注释的提示，找到第一个大小超过需求量n的空闲块p，然后将p内的所有空闲页面的相关参数修改，并从列表中删除。当处理完所有的页面后，如果发现找到的p的大小超过了n，就将这一块之后的一个页面的property参数设置为p->property-n，这一个页面就成为了新的空闲块的Head Page。最后将总的空闲页面数量减去n，返回p的地址，结束分配。

对于default_free_pages函数，首先找到地址大于base的第一个空闲块的Head Page le，然后在这个页面之前依次插入n个页面，然后将base的相关参数修改使得这个块重新成为空闲块。然后，考察le和base的地址，如果le地址比base大n，则说明两个空闲块是相邻的，通过修改base和le的property参数实现空闲块的合并。接下来，如果在free\_list中base的前趋和base的地址只差1，说明base代表的空闲块之前还有一个相邻的空闲块，通过循环找到这个空闲块的Head Page，然后修改它的property参数以实现空闲块的合并。至此完成了对释放空间的回收。
```

你的first fit算法是否有进一步的改进空间

```
之前通过和答案的对比也发现了我自己写的算法实际上还存在很多的漏洞，有一些特殊的情况没有考虑到、一些可能会引起错误的情况没有注意到。比如，在default_init_memmap函数中对每个需要初始化的页面p调用assert(PageReserved(p))来确保在参数初始化前内存是正常的，这一点在最初的时候就没有考虑到。
```

## 练习2 实现寻找虚拟地址对应的页表项

```
首先用PDX函数根据线性地址从页目录表中取出相应的表项，然后判断该表项的存在位是否是使得该表项对应的页表存在，如果根据该位判断对应的页表存在，则直接获取地址并返回；若对应的页表不存在，就要根据create参数来确定是否分配一页，如果为0则直接返回NULL，为1则新分配一页（后期对比答案后加入了防止分配的页为空的情况），然后进行初始化。根据实验引导书，新申请的页必须全部置零，然后设置PTE\_U、PTE\_W、PTE\_P三位，最后再返回地址。
```

1. 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

```
给一个页表项和一个页目录表项大小为4B即32位。根据mmu.h中的定义，
右起第一位是存在位，表明该表项对应的页是否存在；
右起第二位代表页内容是否可写；
右起第三页代表用户态的软件是否能读取页的内容；
右起第四位代表是否为写直达；
右起第五位代表映射的页面有没有被装入缓存，可以帮助实现快表；
右起第六位代表映射的页面是否在被访问；
右起第七位代表该页是否被修改过，可以帮助实现虚拟存储中页面在外存和内存中的移动问题；
右起第八位和第九位必须被置零；
右起第10、11、12位是留给软件使用的，可以留给软件拓展自己功能的余地。
剩下的20位便是代表被映射的页面的基地址，以配合线性地址中的偏移量实现对应内存的定位。
```

2. 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

```
硬件会把产生异常的线性地址储存在CR2寄存器中，并给出出错码，说明页访问异常的类型，然后保存因为异常被打断的现场，将相关的参数压入栈，然后把异常中断号0xe对应的中断服务例程的地址加载到CS和EIP寄存器中，开始由操作系统执行中断服务例程处理异常。操作系统处理完成后，恢复之前被打断的程序现场。
```

## 练习3 释放某虚地址所在的页并取消对应二级页表项的映射

```
实现思路与代码中的注释提示基本相同，首先用PTE\_P位判断该pte是否有效，如果有效则根据pte找到其映射的页，将它的ref参数减1，如果ref的值变为了0，则说明该页已经空闲，调用free\_page函数将其释放。最后将该pte清零，并释放TLB entry。
``` 

1. 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

```
有对应关系。页目录表项或页表项的前20位左移12位后便是某一个page的基地址，表项中的一些位也代表了对应的page的一些状态。
```

2. 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？

```
修改gdt和函数gdt\_init，使得gdt\_init函数对段映射的更新方式改变为使得虚拟地址、线性地址和物理地址相等。
```