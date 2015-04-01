#lab1 实验报告 
计22 徐天宇 2012011275

## 练习1 理解通过make生成执行文件的过程

1. 操作系统镜像文件ucore.img是如何一步一步生成的？

```
ucore.img的生成是通过以下命令来完成的：
{
	$(UCOREIMG): $(kernel) $(bootblock)
		$(V)dd if=/dev/zero of=$@ count=10000
		$(V)dd if=$(bootblock) of=$@ conv=notrunc
		$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
}
第一行说明，为了生成ucore.img，首先应该生成kernel和bootblock。
$(V)dd if=/dev/zero of=$@ count=10000：向ucore.img中写入10000个块，块大小默认为512字节，全部用0填充。
$(V)dd if=$(bootblock) of=$@ conv=notrunc：将bootblock中的内容写入第一个块中，不缩减输出文件（即若输出文件已存在则只改变指定字节然后退出）。
$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc：将kernel中的内容写入从第二个块开始的后续内容中，不缩减输出文件（即若输出文件已存在则只改变指定字节然后退出）。

为此，找到生成kernel的代码如下：
{
	$(kernel): tools/kernel.ld
	$(kernel): $(KOBJS)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
		@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
		@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
}
这些表明生成kernel需要kernel.ld和由$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))生成的.o文件。
实际的运行中，生成这些.o文件的命令为：
	gcc -I**/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/**/**.c -o obj/kern/**/**.o
包括的参数有：
	-I+dir(如-Ilibs/)  添加搜索头文件的路径
	-fno-builtin  除非引用__builtin_前缀，否则不进行builtin函数的优化，用于提升效率
	-Wall  开启编译器几乎所有常用的警告
	--ggdb  生成可供gdb使用的调试信息
	-m32  生成适用于32位环境的代码
	-gstabs  生成stabs格式的调试信息
 	-nostdinc  不使用标准库
	-fno-stack-protector  不生成用于检测缓冲区溢出的代码
 最终使用ld命令完成kernel的生成：
 	ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
这里面列举出了所有需要的.o文件。其中“-T SCRIPTFILE”的意思是让编译器使用指定的脚本来进行连接，-m elf_i386用处是模拟i386上的连接器。

然后找到生成bootblock的代码：
{
	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
		@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
		@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
		@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
		@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
}
实际运行的命令为：
	gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
	gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
	gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
	gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
	ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
上述两个命令依次生成了bootasm.o和bootmain.o。其中参数-Os的作用是对代码进行不缩减尺寸的优化（经查阅资料相当于-O2.5，优化级别位于-O2和-O3之间，采用了-O2所有的优化又不缩减代码尺寸）；第三、第四两个命令则完成了sign工具的生成，其中参数-O2代表次高级的优化（-O3为最高级）。最后一个命令完成对bootblock的生成，其中参数有：
	-nostdlib  仅搜索那些在命令行上显式指定的库路径
	-N  设置代码段和数据段均可读写
	-e ENTRY  指定程序的开始执行点
	-Ttext ORG 通过指定ORG, 指定节在输出文件中的绝对地址
```

2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

```
通过阅读sign.c的代码，发现其限制的主引导扇区的大小为512字节，其中第511个字节限制为0x55
，第512个字节被限制为0xaa，即被系统认为符合规范的硬盘主引导扇区的特征为：以0x55aa做结尾、大小512字节。
```

## 练习2 使用qemu执行并调试lab1中的软件

1. 从CPU加电后执行的第一条指令开始， 单步跟踪BIOS的执行

```
打开命令行，进入到labcodes/lab1中，然后输入命令make debug，进入gdb调试界面，通过命令“s”便能够单步跟踪BIOS的执行。
```

2. 在初始化位置0x7c00设置实地址断点,测试断点正常。

```
打开命令行，进入到labcodes/lab1中，然后输入命令make debug，进入gdb调试界面，输入命令“b *0x7c00”，然后输入命令“c”就可以使程序运行至断点0x7c00处。
输入命令“x /10i $pc”便能得到断点处开始的10条指令：
	0x7c00:      cli    
   	0x7c01:      cld    
   	0x7c02:      xor    %eax,%eax
   	0x7c04:      mov    %eax,%ds
   	0x7c06:      mov    %eax,%es
   	0x7c08:      mov    %eax,%ss
   	0x7c0a:      in     $0x64,%al
   	0x7c0c:      test   $0x2,%al
   	0x7c0e:      jne    0x7c0a
   	0x7c10:      mov    $0xd1,%al
与bootasm.S文件中第16行开始的汇编指令是基本相同的，除了%eax和%ax，但由于生成的指令是16位的，实际运行中%eax也只有%ax部分被占用，可以认为是一致的。说明断点正常
```

3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

```
在第二问中已经和bootasm.S做了比较，结果对应相同。
将得到的10行汇编指令和bootblock.asm进行比较，和从第12行开始的汇编指令也基本相同（同样是%eax和%ax的差别），地址也相同。
```

4. 自己找一个bootloader或内核中的代码位置， 设置断点并进行测试。

```
设置断点“b *0x7c32”，并获取到断点附近的指令如下：
 	0x7c32:      mov    $0x10,%ax
   	0x7c36:      mov    %eax,%ds
   	0x7c38:      mov    %eax,%es
   	0x7c3a:      mov    %eax,%fs
   	0x7c3c:      mov    %eax,%gs
   	0x7c3e:      mov    %eax,%ss
   	0x7c40:      mov    $0x0,%ebp
经对比和bootblock.asm以及bootasm.S中的对应指令相同。
```

## 练习3 分析bootloader进入保护模式的过程。

```
过程如下：
1.从%cs=0 $pc=0x7c00，进入实模式后，禁止中断的发生（cli）、清除方向标志（cld），然后将段寄存器置0。
2.开启A20：开始时地址线A20是被绑定为低电位，使得高于20位的地址全都为0，即高于1MB的地址不能使用。通过将A20线置于高电位，使得全部32条地址线可用，
可以访问4G的内存空间。
	seta20.1:
	    inb $0x64, %al              # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.1

	    movb $0xd1, %al             # 0xd1 -> port 0x64
	    outb %al, $0x64             # 0xd1 means: write data to 8042's P2 port

	seta20.2:
	    inb $0x64, %al              # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.2

	    movb $0xdf, %al             # 0xdf -> port 0x60
	    outb %al, $0x60             # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
经过这些步骤，便可以将A20地址线打开，从而能够访问4G内存空间。
3.初始化GDT表：通过命令lgdt gdtdesc，将已经存储好的GDT表和其描述符载入。
4.进入保护模式：将cr0寄存器的PE位置1便开启了保护模式
	movl %cr0, %eax
	orl $CR0_PE_ON, %eax
	movl %eax, %cr0
5.通过长跳转ljmp $PROT_MODE_CSEG, $protcseg，进入32位指令模式来进一步设置保护模式相关的参数
6.设置段寄存器
	movw $PROT_MODE_DSEG, %ax
	movw %ax, %ds
	movw %ax, %es
	movw %ax, %fs
	movw %ax, %gs
    movw %ax, %ss
同时建立堆栈    
    movl $0x0, %ebp
    movl $start, %esp

7.此时转到保护模式完成，执行call bootmain
```

## 练习4 分析bootloader加载ELF格式的OS的过程

```
分析bootmain.c文件，首先看readsect函数，它完成了对磁盘从secno处开始的扇区的读入，将其读入到dst位置。由于设置了0x1f2地址的内容为1，所以该函数每次读取一个扇区的内容。
其次是readseg函数，它扩充了readsect的机能，使得从offset处开始的大小为count的内容能够从磁盘中加载至虚拟地址va处。它首先确定了从磁盘中读取的内容存放的起始地址和结束地址，然后计算磁盘中所读取的内容的起始扇区号，利用这些条件作为限制，循环调用函数readsect来从磁盘中不断读取连续的内容。这样便能够获得ELF文件的内容。
获得了内容后，便交由bootmain函数做最后的处理。首先读取ELF的header以判断读取的文件是否是ELF格式的文件，如果不是的话跳至bad处做出错处理。然后，将program header表的位置偏移（ELFHDR->e_phoff）和入口数目（ELFHDR->e_phnum）从文件中提取出来保存，并以这两个参数作为限制条件，不断从文件中读取相应的内容（由函数readseg完成），这些内容是加载ELF格式OS的重要参数。然后，根据ELF header（ELFHDR）中的ELFHDR->e_entr参数找到内核的入口，便完成了ELF格式OS的加载。
```

## 练习5 实现函数调用堆栈跟踪函数

```
填充的代码如下：
	uint32_t ebp_value = read_ebp();
	uint32_t eip_value = read_eip();
	int i,j;
	for(i = 0; i < STACKFRAME_DEPTH; i++)
	{
		if(ebp_value == 0)
		{
			break;
		}

		cprintf("ebp:0x%08x eip:0x%08x ", ebp_value, eip_value);

		uint32_t *stack_args = (uint32_t *)ebp_value + 2;
		cprintf("args:");
		for (j = 0; j < 4; j++)
		{
			cprintf("0x%08x ", stack_args[j]);
		}
		cprintf("\n");

		print_debuginfo(eip_value - 1);
		eip_value = ((uint32_t *)ebp_value)[1];
		ebp_value = ((uint32_t *)ebp_value)[0];
	}
编写代码时，C++的代码习惯让我吃了一点亏……
实现的思路和代码中给出的注释基本一致，在经过询问同学后得知需要加入控制条件ebp_value != 0，即第一层for循环开始时的if条件。

获得的输出结果如下：
ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:295: print_stackframe+21
ebp:0x00007b18 eip:0x00100ca6 args:0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --

经过比对，输出结果大致一致，有个别的偏差。
最后一行意为：ebp是0x7bf8，eip是0x7d68，四个参数分别是0xc031fcfa、0xc08ed88e、0x64e4d08e、0xfa7502a8，这一层查不到调用者。
```

## 练习6 完善中断初始化和处理 

1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

```
中断描述符表表项定义如下：
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
可见其占用8个字节（64 bit），其中第3-4个字节是段选择子，第1-2字节是偏移量的低16位，第7-8个字节是偏移量的高16位，拼合起来便是段内偏移量，则该偏移量和段选择子联合便是中断处理代码的入口。
```

2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

```
填充代码如下：
	extern uintptr_t __vectors[];
	int i;
	for (i = 0; i < 256; i++)
	{
	    SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
	}
	
	//参考了答案后加上这一项，这一项的作用是设置用于处理用户态和内核态切换的表项。
	SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
	
	lidt(&idt_pd);
首先用__vectors[]获取每个ISR的入口地址，然后用SETGATE函数对IDT的256个表项进行初始化，其中包括对管理用户态和内核态切换的ID进行设置（参考答案后加入），然后调用lidt函数将IDT的地址告知CPU。
```

3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分， 使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

```
填充代码如下：
	ticks++;
    if(ticks % TICK_NUM == 0)
    {
		print_ticks();
   	}
基本实现思路同注释，利用kern/driver/clock.c中定义的全局变量ticks对时钟中断进行计数，每100次输出一次“100 ticks”。实验结果符合要求，每秒输出一次“100 ticks”，同时按键也能在屏幕上正常显示。
```
