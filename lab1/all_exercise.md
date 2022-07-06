##### 这个实验是基于MIT的2017的6.826课程，搭建环境的时候踩了几个坑，但是当时没有记录下来，可惜~


# Part 1: PC Bootstrap
介绍了如何安装qemu以及如同通过qemu来模拟操作系统的启动。

## The PC's Physical Address Space
* 8088处理器的内存地址

![image](https://user-images.githubusercontent.com/20594445/176725671-0b648302-9df4-453a-a867-01edae6dd475.png)

 0 -          640K     内存区域
 0xF0000 - 0xFFFFF BIOS区域
 0xB8000 - 0xBFFF  显卡区域
 
 ## The ROM BIOS
 ```
[f000:fff0] 0xffff0:    ljmp   $0xf000,$0xe05b
```
 
 ### mit 6.828 lab1 Exercise2
 这部分的代码是嵌入在ROM BIOS中的，因此在磁盘上并没有对应的代码。
 
使用si命令得到的前22条汇编指令如下。虽然能看懂每条指令的字面意思，但看不懂具体实现的功能，后来参考[myk的6.828 Lab1](https://zhuanlan.zhihu.com/p/36926462)大致理解了基本功能：设置ss和esp寄存器的值，打开A20门（为了后向兼容老芯片）、进入保护模式（需要设置cr0寄存器的PE标志）。

 ```
 1 [ f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b
2  [f000:e05b]    0xfe05b:	cmpl   $0x0,%cs:0x6ac8
3  [f000:e062]    0xfe062:	jne    0xfd2e1
4  [f000:e066]    0xfe066:	xor    %dx,%dx
5  [f000:e068]    0xfe068:	mov    %dx,%ss
6  [f000:e06a]    0xfe06a:	mov    $0x7000,%esp
7  [f000:e070]    0xfe070:	mov    $0xf34c2,%edx
8  [f000:e076]    0xfe076:	jmp    0xfd15c
9  [f000:d15c]    0xfd15c:	mov    %eax,%ecx
10 [f000:d15f]    0xfd15f:	cli    
11 [f000:d160]    0xfd160:	cld    
12 [f000:d161]    0xfd161:	mov    $0x8f,%eax
13 [f000:d167]    0xfd167:	out    %al,$0x70
14 [f000:d169]    0xfd169:	in     $0x71,%al
15 [f000:d16b]    0xfd16b:	in     $0x92,%al
16 [f000:d16d]    0xfd16d:	or     $0x2,%al
17 [f000:d16f]    0xfd16f:	out    %al,$0x92
18 [f000:d171]    0xfd171:	lidtw  %cs:0x6ab8
19 [f000:d177]    0xfd177:	lgdtw  %cs:0x6a74
20 [f000:d17d]    0xfd17d:	mov    %cr0,%eax
21 [f000:d180]    0xfd180:	or     $0x1,%eax
22 [f000:d184]    0xfd184:	mov    %eax,%cr0
23 [f000:d187]    0xfd187:	ljmpl  $0x8,$0xfd18f
```
 
 参考[MIT6.828——Lab 1. Part 2 启动qemu](https://www.jianshu.com/p/39da7aa0a0bc)
 ### 代码笔记
1. 第一条指令：`[f000:fff0] 0xffff0: ljmp $0xf000,$0xe05b`
因为8086加载或者复位的时候，`CS = 0xFFFF, IP = 0x0000`，所以它取的第一条指令位于`0xFFFF0`处。
通过`ljmp`指令来跳转到地址`0xfe05b`，并改变`CS`和`IP`的地址，`CS = 0xF000`，`IP = 0xe05b`。

2. 第二、三条指令：`[f000:e05b]    0xfe05b:	cmpl   $0x0,%cs:0x6ac8`
判断0xf6ac8是否等于0，具体意思不清楚。如果不为0，则跳转到0xfd2e1。

3. 第四、五、六、七条指令
置`ss`为0，`esp`为`0x7000`，`edx`为`0xf34e2`，之后再跳转到`0xfd15c`


4. CLI：Clear Interupt，禁止中断发生。STI：Set Interupt，允许中断发生。CLI和STI是用来屏蔽中断和恢复中断用的，如设置栈基址SS和偏移地址SP时，需要CLI，因为如果这两条指令被分开了，那么很有可能SS被修改了，但由于中断，而代码跳去其它地方执行了，SP还没来得及修改，就有可能出错。


5. CLD: Clear Director。STD：Set Director。在字行块传送时使用的，它们决定了块传送的方向。CLD使得传送方向从低地址到高地址，而STD则相反。这里要解释一下，要实现段之间的批量数据传送，比如movsb或者movsw，需要指定是正向传送还是反向传送，正向传送是指传送操作的方向是从内存区域的低地址端到高地址端。
6. 第十二到十四条指令：
处理器是通过端口来和外围设备打交道的。本质上，端口就是一些寄存器，端口是处理器和外围设备通过I/O接口交流的窗口，每一个I/O接口都可能拥有好几个端口，所有端口都是统一编号的，比如I/O接口A有3个端口，端口号分别为0x0021~0x0023。
这里引入in，out操作：
**out %al, PortAddress    向端口地址为PortAddress的端口写入值，值为al寄存器中的值**
**in PortAddres,%al    把端口地址为PortAddress的端口中的值读入寄存器al中**
0x70端口和0x71端口是用于控制系统中一个叫做CMOS的设备。
操作CMOS存储器中的内容需要两个端口，一个是0x70另一个就是0x71。其中0x70可以叫做索引寄存器，**这个8位寄存器的最高位是不可屏蔽中断(NMI)使能位。如果你把这个位置1，则NMI不会被响应。低7位用于指定CMOS存储器中的存储单元地址**。


7. 第十五到十七条指令
读端口`0x92`的值，并将`0x92`端口对应的寄存器的`1`号bit位置1。
它控制的是 PS/2系统控制端口A，可以查到这个端口的bit1的功能是
**bit 1= 1 indicates A20 active**
即A20位，即第21个地址线被使能。

8. lidt指令：加载中断向量表寄存器(IDTR)。这个指令会把从地址0xf6ab8起始的后面6个字节的数据读入到中断向量表寄存器(IDTR)中。中断是操作系统中非常重要的一部分，有了中断操作系统才能真正实现进程。每一种中断都有自己对应的中断处理程序，那么这个中断的处理程序的首地址就叫做这个中断的中断向量。中断向量表自然是存放所有中断向量的表了。

9. 把从0xf6a74为起始地址处的6个字节的值加载到全局描述符表格寄存器中GDTR中。这个表实现保护模式非常重要的一部分。


10. 第20-22指令
CR0是处理器内部的控制寄存器，它是32位寄存器，包含了一系列用于控制处理器操作模式和运行状态的标志位。它的第1位是保护模式允许位，是开启保护模式大门的门把手。如果把该位置"1"，则处理器进入保护模式。但是这里出现了问题，我们刚刚说过BIOS是工作在实模式之下，后面的boot loader开始的时候也是工作在实模式下，所以这里把它切换为保护模式，显然是自相矛盾。所以只能推测它在检测是否机器能工作在保护模式下。


11. 23条指令之后
https://en.wikibooks.org/wiki/X86_Assembly/Global_Descriptor_Table指出，如果刚刚加载完GDTR寄存器我们必须要重新加载所有的段寄存器的值，而其中CS段寄存器必须通过长跳转指令，即23号指令来进行加载。所以这些步骤是在第19步完成后必须要做的。这样才能是GDTR的值生效。


**后面的就看不懂了，应该是调用了BIOS中的各种函数来检测各种底层的设备**

 #  Part 2: The Boot Loader
 
* If the disk is bootable, the first sector is called the boot sector, since this is where the boot loader code resides。

* For 6.828, however, we will use the conventional hard drive boot mechanism, which means that our boot loader must fit into a measly 512 bytes. The boot loader must perform two main functions:
    * First, the boot loader switches the processor from real mode to 32-bit protected mode, because it is only in this mode that software can access all the memory above 1MB in the processor's physical address space. 
    * Second, the boot loader reads the kernel from the hard disk by directly accessing the IDE disk device registers via the x86's special I/O instructions.

* 分析`boot/boot.S`的代码

```
00007c00 <start>:
    7c00:	fa                   	cli    
    7c01:	fc                   	cld    
    7c02:	31 c0                	xor    %eax,%eax
    7c04:	8e d8                	mov    %eax,%ds
    7c06:	8e c0                	mov    %eax,%es
    7c08:	8e d0                	mov    %eax,%ss
00007c0a <seta20.1>:
    7c0a:	e4 64                	in     $0x64,%al
    7c0c:	a8 02                	test   $0x2,%al
    7c0e:	75 fa                	jne    7c0a <seta20.1>
    7c10:	b0 d1                	mov    $0xd1,%al
    7c12:	e6 64                	out    %al,$0x64
00007c14 <seta20.2>:
    7c14:	e4 64                	in     $0x64,%al
    7c16:	a8 02                	test   $0x2,%al
    7c18:	75 fa                	jne    7c14 <seta20.2>
    7c1a:	b0 df                	mov    $0xdf,%al
    7c1c:	e6 60                	out    %al,$0x60
    7c1e:	0f 01 16             	lgdtl  (%esi)
    7c21:	64 7c 0f             	fs jl  7c33 <protcseg+0x1>
    7c24:	20 c0                	and    %al,%al
    7c26:	66 83 c8 01          	or     $0x1,%ax
    7c2a:	0f 22 c0             	mov    %eax,%cr0
    7c2d:	ea                   	.byte 0xea
    7c2e:	32 7c 08 00          	xor    0x0(%eax,%ecx,1),%bh
00007c32 <protcseg>:
    7c32:	66 b8 10 00          	mov    $0x10,%ax
    7c36:	8e d8                	mov    %eax,%ds
    7c38:	8e c0                	mov    %eax,%es
    7c3a:	8e e0                	mov    %eax,%fs
    7c3c:	8e e8                	mov    %eax,%gs
    7c3e:	8e d0                	mov    %eax,%ss
    7c40:	bc 00 7c 00 00       	mov    $0x7c00,%esp
    7c45:	e8 cb 00 00 00       	call   7d15 <bootmain>

```

boot.S文件中主要做了以下事情：初始化段寄存器、打开A20门、从实模式跳到虚模式（需要设置GDT和cr0寄存器），最后调用bootmain函数。

* gdt表信息如下
GDT表有23个表项，基地址在gdt标号处。
共设置了3个表项：
表项1 -- 空，GDT表项第一个表项必须为空
表项2 -- 代码段，基地址为0x0，总长度为0xffffffff = 4GB
表项3 -- 数据段，基地址为0x0，总长度为0xffffffff = 4Gb
说明xv6并没有使用分段机制，用的是平坦模型，和linux是一样的
```

1 # Bootstrap GDT
 2 .p2align 2                               # force 4 byte alignment
 
 3   gdt: 
 4       SEG_NULL                               # null seg
 5       SEG(STA_X|STA_R, 0x0, 0xffffffff)      # code seg
 6       SEG(STA_W, 0x0, 0xffffffff)            # data seg
 7 
 8   gdtdesc: 
 9       .word   0x17                           # sizeof(gdt) - 1
 10     .long   gdt                            # address gdt
```

.byte在当前位置插入一个字节；.word在当前位置插入一个字

* 分析`boot/main.c`代码

```
(gdb) i reg
eax            0x10     16
ecx            0x0      0
edx            0x80     128
ebx            0x0      0
esp            0x7be4   0x7be4
ebp            0x7bf8   0x7bf8
esi            0x0      0
edi            0x0      0
eip            0x7d26   0x7d26
eflags         0x6      [ PF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x10     16
gs             0x10     16

//执行call指令之后, call 0x7cdc
//栈顶减少4个字节，并将下一条指令的地址入栈，eip指向call的地址
(gdb) i reg
eax            0x10     16
ecx            0x0      0
edx            0x80     128
ebx            0x0      0
esp            0x7be0   0x7be0
ebp            0x7bf8   0x7bf8
esi            0x0      0
edi            0x0      0
eip            0x7cdc   0x7cdc
eflags         0x6      [ PF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x10     16
gs             0x10     16


//因此函数的调用，在汇编看来是这样的，跳转到0x7cdc之后的第一件事就是保持上一个栈的栈底并更新栈底寄存器ebp，作为当前栈的栈底
 0x7cdc:      push   %ebp
0x7cdd:      mov    %esp,%ebp


//执行ret指令之后，会将栈顶esp的4个字节的地址取出到eip，然后esp-4，跳转到eip
```

bootmain()函数的功能是：它其实是引导工作的最后部分（引导的大部分工作都在 bootasm.S 中实现），它负责将内核从硬盘上加载到内存中，然后开始执行内核中的程序。

**在makefile编译的时候，GNUmakefile->kern/Makefrag->boot/Makefrag，生成了两个ELF文件，boot/boot和kern/kernel，之后通过dd if命令生成kernel.img文件，这个文件大小为5M，第一个扇区为boot/boot，第二个扇区开始为kern/kernel。**

## Exercise 4
要求准确说出`points.c`代码中的输出值。
**printf("%p\n", a)中%p的含义其实和%x类似，只是%p会根据操作系统是32位还是64位来自动调整自己的输出**


## Exercise 5

The program headers specify which parts of the ELF object to load into memory and the destination address each should occupy.
```
You can inspect the program headers by typing:
athena% objdump -x obj/kern/kernel
```
[MIT 6.828 JOS学习笔记9. Exercise 1.5](https://www.cnblogs.com/fatsheep9146/p/5220004.html)

1. 修改-Ttext 0x7C00，我们可以把它修改为0x10C00，然后保存退出。
编译的时候，报错了，'截断重寻址至相符'。

2. 修改-Ttext 0x7C00，我们可以把它修改为0x1C00，然后保存退出。
  在0x7C00和0x1C00这两个地址均设置了断点，然后敲c，发现仍然是在0x7C00停住，再敲一次c，会报异常：“Program received signal SIGTRAP, Trace/breakpoint trap.”我预期修改后boot loader的起始地址应该从0x1c00开始，而gdb调试显示并没跑到地址为0x1c00的地方，所以怀疑对链接地址的修改没生效。后来看了[MIT 6.828 JOS学习笔记9. Exercise 1.5](https://www.cnblogs.com/fatsheep9146/p/5220004.html)，才知道这是正常的：BIOS是默认把boot loader加载到0x7C00内存地址处，所以boot loader的起始地址仍然是0x7C00.修改链接地址后，会导致`lgdt gdtdesc`和`ljmp $PROT_MODE_CSEG $protcseg`两句指令出错，两者都需要计算地址，计算方法为链接地址加上偏移，因此将链接地址修改成与加载地址不一样后，会导致地址计算失败。比如这里的gdtdesc和$protcseg的正确地址为0x7c64和0x7c32，修改链接地址后两者的地址分别变为0x1c64和0x1c32。    --- 引用自[《MIT 6.828 Lab1: Booting a PC》实验报告](https://www.cnblogs.com/wuhualong/p/mit_6-828_lab1.html)
  
3. 标号2的意思是，BIOS会自动将boot代码加载到0x7C00处，但是如果在ld链接的时候，将加载地址设置为0x1C00时，boot.S代码中，其引用的地址就全部是相对于0x1C00的了，因此虽然代码加载到0x7C00的时候，引用的地址却是0x1C00，而0x1C00处的内容全为0。

## Exercise 6 
1. Besides the section information, there is one more field in the ELF header that is important to us, named e_entry. This field holds the link address of the entry point in the program: the memory address in the program's text section at which the program should begin executing.
2. Exercise6 解答 
0x00100000这个地址是内核加载到内存中的地址。当BIOS进入boot loader时，还没将内核加载到这块内存，其内容是随机的；而当boot loader进入内核时，内核已经加载完成，其内容就是内核文件内容。因此这两个阶段对应的0x00100000地址的内容是不相同的。可以通过gdb来验证：
    * 当BIOS刚进入boot loader时，地址0x00100000往后的8个word取值均为0.
     ![Image63.png](/lab1/graphs/Image63.png)
    * 当boot loader进入内核时，地址0x00100000往后的8个word已经出现非零值。我们还可以多加几个断点，以观察此内存地址内容最早是什么时候被修改的。实验证明是在bootmain函数中的for循环第一次结束后被修改的，而for循环做的事情就是将内核中的各个program segment加载到内存中。
    ![Image64.png](/lab1/graphs/Image64.png)
    
 # Part 3: The Kernel
## Using virtual memory to work around position dependence
Operating system kernels often like to be linked and run at very high virtual address, such as 0xf0100000, in order to leave the lower part of the processor's virtual address space for user programs to use. 

Many machines don't have any physical memory at address 0xf0100000, so we can't count on being able to store the kernel there. Instead, we will use the processor's memory management hardware to map virtual address 0xf0100000 (the link address at which the kernel code expects to run) to physical address 0x00100000 (where the boot loader loaded the kernel into physical memory). This way, although the kernel's virtual address is high enough to leave plenty of address space for user processes, it will be loaded in physical memory at the 1MB point in the PC's RAM, just above the BIOS ROM. 

For now, you don't have to understand the details of how this works, just the effect that it accomplishes. Up until kern/entry.S sets the CR0_PG flag, memory references are treated as physical addresses (strictly speaking, they're linear addresses, but boot/boot.S set up an identity mapping from linear addresses to physical addresses and we're never going to change that). Once CR0_PG is set, memory references are virtual addresses that get translated by the virtual memory hardware to physical addresses. entry_pgdir translates virtual addresses in the range 0xf0000000 through 0xf0400000 to physical addresses 0x00000000 through 0x00400000, as well as virtual addresses 0x00000000 through 0x00400000 to physical addresses 0x00000000 through 0x00400000.
## Exercise 7
题目包括两部分：一是观察内存地址映射瞬间的状态，二是分析内存地址映射失败的影响。
### 一、观察内存地址映射瞬间的状态
1. 查看kernel的elf头文件，发现内核的入口为0x1000c
 ![Image65.png](/lab1/graphs/Image65.png)

2. 启动qemu和gdb，在mov %eax, %cr0所在的地址0x1000c处加断点，运行至此，使用x/16xw查看两个地址往后16个word的内容，发现两者不同（后者为全0），说明地址映射尚未完成。
 ![Image66.png](/lab1/graphs/Image66.png)

3. 继续往下执行一步，再查看两个地址往后16个word，发现内容完全相同，说明地址映射成功。
 ![Image67.png](/lab1/graphs/Image67.png)

### 二、分析内存地址映射失败的影响
题目第二个问题是判断内存地址失败后哪些指令会运行失败，我判断是下面两条指令mov $relocated, %eax和jmp %eax就会失败，我的推理过程：relocated这个地址是由段地址加上偏移地址得到的，段地址是0xf0100008，如果地址映射失败，那些jmp %eax就会跳到0xf010008加上偏移量的物理地址，导致出错。gdb调试结果恰好验证了我的猜测是正确的。
1. 将kern/entry.S的movl %eax, %cr0注释掉，重新启动qemu和gdb，在jmp %eax加断点，使用c命令运行到这里，使用x/16xw查看0x00100000和0xf0100000两个地址往后16个word的内容，发现两者不同，后者依然是全0（内容与第一节第1步的相同，此处不再提供）。可见地址映射确实失败了。
2. 继续往下执行一步，发现gdb报错。应该是因为0xf010002c地址后面的数据全为0，导致把空指针赋给寄存器而报错。


# Formatted Printing to the Console

## Exercise 8
*  参考打印10进制和16进制整数的代码即可。
```
case 'o':
    num = getuint(&ap, lflag);
    base = 8;
    goto number;
```
### va_arg, va_copy, va_end, va_start
[Microsoft: va_arg, va_copy, va_end, va_start](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/va-arg-va-copy-va-end-va-start?view=msvc-170)
The va_arg, va_copy, va_end, and va_start macros provide a portable way to access the arguments to a function when the function takes a variable number of arguments.

#### Syntax
```
type va_arg( va_list arg_ptr, type ); 
void va_copy( va_list dest, va_list src ); // (ISO C99 and later) 
void va_end( va_list arg_ptr ); 
void va_start( va_list arg_ptr, prev_param ); // (ANSI C89 and later) 
void va_start( arg_ptr ); // (deprecated Pre-ANSI C89 standardization version)
```

#### Parameterstype
`type`
Type of argument to be retrieved.arg_ptr
`arg_ptr`
Pointer to the list of arguments.dest
`dest`
Pointer to the list of arguments to be initialized from srcsrc
`src`
Pointer to the initialized list of arguments to copy to dest.prev_param
`prev_param`
Parameter that precedes the first optional argument.


#### Return Value
va_arg returns the current argument. va_copy, va_start and va_end do not return values.


#### Remarks
* va_start **sets arg_ptr to the first optional argument in the list of arguments that's passed to the function**. The argument arg_ptr must have the va_list type. The argument prev_param is the name of the required parameter that immediately precedes the first optional argument in the argument list. If prev_param is declared with the register storage class, the macro's behavior is undefined. va_start must be used before va_arg is used for the first time.
* va_arg **retrieves a value of type from the location that's given by arg_ptr, and increments arg_ptr to point to the next argument in the list by using the size of type to determine where the next argument starts**. va_arg can be used any number of times in the function to retrieve arguments from the list.
* va_copy makes a copy of a list of arguments in its current state. The src parameter must already be initialized with va_start; it may have been updated with va_arg calls, but must not have been reset with va_end. The next argument that's retrieved by va_arg from dest is the same as the next argument that's retrieved from src.
* After all arguments have been retrieved, va_end resets the pointer to NULL. va_end must be called on each argument list that's initialized with va_start or va_copy before the function returns.

### Be able to answer the following questions:
1. Explain the interface between printf.c and console.c. Specifically, what function does console.c export? How is this function used by printf.c?
解答： kernel/printf.c中使用了`void cputchar(int c)`接口。
具体调用关系：`cprintf -> vcprintf -> putch -> cputchar`。

2. Explain the following from console.c:
 ![Image68.png](/lab1/graphs/Image68.png)
解答：联系代码上下文，可以理解这段代码的作用。首先，CRT(cathode ray tube)是阴极射线显示器。根据console.h文件中的定义，CRT_COLS是显示器每行的字长（1个字占2字节），取值为80；CRT_ROWS是显示器的行数，取值为25；而#define CRT_SIZE (CRT_ROWS * CRT_COLS)是显示器屏幕能够容纳的字数，即2000。当crt_pos大于等于CRT_SIZE时，说明显示器屏幕已写满，因此将屏幕的内容上移一行，即将第2行至最后1行（也就是第25行）的内容覆盖第1行至倒数第2行（也就是第24行）。接下来，将最后1行的内容用黑色的空格塞满。将空格字符、0x0700进行或操作的目的是让空格的颜色为黑色。最后更新crt_pos的值。总结：这段代码的作用是当屏幕写满内容时将其上移1行，并将最后一行用黑色空格塞满。

3.  For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
Trace the execution of the following code step-by-step:
```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```
* In the call to cprintf(), to what does fmt point? 
解答：
在函数`i386_init`中加入上述的代码，之后，在gdb的时候，断到该位置。发现fmt指向"x %d, y %x, z %d\n"
 ![Image69.png](/lab1/graphs/Image69.png)
 ![Image70.png](/lab1/graphs/Image70.png)
 
* To what does ap point?List (in order of execution) each call to cons_putc, va_arg, and vcprintf. For cons_putc, list its argument as well. For va_arg, list what ap points to before and after the call. For vcprintf list the values of its two arguments.
解答：
    * ap指向第一个要打印的参数的内存地址，也就是x的地址。
        * cprintf首先调用vcprintf，而vcprintf调用vprintfmt，这个是主要逻辑函数。vprintfmt的入参为一个回调函数加上格式字符串以及valist表示的参数列表，每次要将字符打印显示到屏幕上的时候，就调用这个回调函数，将想要显示的字符传入，该回调函数只有两个入参，`void (*putch)(int, void*)`，一个需要被打印的字符，一个是void入参，用来计数的，每次调用该回调函数就自增1.

    * vprintfmt流程图
 ![Image71.png](/lab1/graphs/Image71.png)


























4. Run the following code.
```
unsigned int i = 0x00646c72;
cprintf("H%x Wo%s", 57616, &i);
```
What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise. Here's an ASCII table that maps bytes to characters.The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?
解答：the output is "He110 World"。`putch("0123456789abcdef"[num % base], putdat);`注意双引号及其内部实际上定义了一个数组，其元素依次为16进制的16个字符。解释下：57616转换成16进制就是0xe110.根据ASCII码，i的4个字节从低到高依次为'r', 'l', 'd', '\0'.这里要求主机是小端序才能正常打印出“World”这个单词。
如果是大端字节序，i的值要修改为0x726c6400，而57616这个值不用修改。因为根据printnum函数实现，打印整数时，总是按照从高位到低位来打印的。

5. In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
```
cprintf("x=%d y=%d", 3);
```
解答：通过函数va_arg(*ap, int)将x的指针向栈底移动4个字节(int大小），此时的位置指向的就是y，然后将这个位置的值转为int。（函数参数的压栈是从右到左的）。



6. Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change cprintf or its interface so that it would still be possible to pass it a variable number of arguments?
解答：

# Stack
## Exercise 9
Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?
解答：
1. 这两条指令修改了%ebp，%esp两个寄存器的值，而这两个寄存器的值是和堆栈息息相关的。
```
　　movl    $0x0,%ebp            # nuke frame pointer
　　movl    $(bootstacktop),%esp
```

2. How does the kernel reserve space for its stack?
这两个指令分别设置了%ebp，%esp两个寄存器的值。其中%ebp被修改为0。%esp则被修改为bootstacktop的值。这个值为0xf0111000。另外在entry.S的末尾还定义了一个值，bootstack。注意，在数据段中定义栈顶bootstacktop之前，首先分配了KSTKSIZE这么多的存储空间，专门用于堆栈，这个KSTKSIZE = 8 * PGSIZE  = 8 * 4096 = 32KB。`所以用于堆栈的地址空间为 0xf0109000-0xf0111000`，`其中栈顶指针指向0xf0111000. 那么这个堆栈实际坐落在内存的 0x00109000-0x00111000物理地址空间中`。
 ![Image72.png](/lab1/graphs/Image72.png)


## Exercise 10
[Lab 1 Exercise 10](https://www.cnblogs.com/fatsheep9146/p/5079930.html)
实际上就是让我们了解了每次调用函数的时候，esp，ebp，eip中的值是如何变化的，以及栈顶0xf0111000到0xf0109000栈底中每个地址存储的字节是什么。


，当每一次进入test_backtrace后，刚刚开始时，它要完成之前介绍过的过程调用的通用操作，如下：
```
1     push %ebp
2     mov %esp, %ebp
3     push %ebx
4     sub $0x14, %esp
```

这四个操作将被用于存放调用这个子程序的父程序的栈帧信息，以及为当前子程序分配新的栈帧。在Exercise 1.9中，我们已经讨论过，entry.S文件中为整个内核设置了堆栈空间的地址范围，从0xf0108000-0xf0110000。由于堆栈是向下增长的，所以在运行init函数之前，esp寄存器的值就是0xf0110000，代表堆栈尚未使用。进入i386_init函数后，如果要调用某个子程序，就会把 i386_init 程序的栈帧信息压入到这个堆栈空间中。


## Exercise 11
**遇到了一个坑：之前为了将跟踪到的代码和磁盘中的代码对上，将编译的-O1选项删除，导致了调用read_ebp函数的时候，read_ebp函数不再是一个inline函数，读取的是read_ebp自己的栈基地址，从而导致整个儿调用栈乱序，重新加上-O1优化选项之后，能够正常输出了**
问题1：知道地址，怎么取出这个地址中的值。
将这个整数强转为指针就行了
```
uint32_t *ebp = (uint32_t *)read_ebp();
```
解答：
* 代码如下：
```
uint32_t* ebp = (uint32_t *)read_ebp();
while (ebp != 0x0) {
    uint32_t *eip  = (ebp + 1);
    uint32_t *arg0 = (ebp + 2);
    uint32_t *arg1 = (ebp + 3);
    uint32_t *arg2 = (ebp + 4);
    uint32_t *arg3 = (ebp + 5);
    uint32_t *arg4 = (ebp + 6);

    cprintf("ebp %08x eip %08x args %08x %08x %08x %08x %08x\n", ebp, *eip, *arg0, *arg1, *arg2, *arg3, *arg4);

    ebp =(uint32_t *) (*ebp);
}
```
* 最后输出如下
 ![Image73.png](/lab1/graphs/Image73.png)


## Excercise 12
题目描述：Modify your stack backtrace function to display, for each eip, the function name, source file name, and line number corresponding to that eip.

1. In `debuginfo_eip`, where do `_STAB_*`   come from? 
解答：回答上述问题得先了解.ld文件，查看这个链接即可[LD 文件：规则详解](https://blog.csdn.net/shenjin_s/article/details/88712249)。
由此可以知道，我们是在kern/kernel.ld输出.stab和.stabstr段的时候，设置了__STAB_* 指向这两个section的开始和结尾地址。
**PROVIDE关键字**该关键字用于定义这类符号：在目标文件内被引用，但没有在任何目标文件内被定义的符号。
```
 25     /* Include debugging information in kernel memory */
 26     .stab : {
 27         PROVIDE(__STAB_BEGIN__ = .);
 28         *(.stab);
 29         PROVIDE(__STAB_END__ = .);
 30         BYTE(0)     /* Force the linker to allocate space
 31                    for this section */
 32     }
 33
 34     .stabstr : {
 35         PROVIDE(__STABSTR_BEGIN__ = .);
 36         *(.stabstr);
 37         PROVIDE(__STABSTR_END__ = .);
 38         BYTE(0)     /* Force the linker to allocate space
 39                    for this section */
 40     }
```
执行`objdump -h obj/kern/kernel`可以查看这两个section的起始和结尾地址
```
test@iZuf6bogmr00a004vw0sbeZ:~/MIT/6.828/lab$ objdump -h obj/kern/kernel

obj/kern/kernel：     文件格式 elf32-i386

节：
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001a49  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       000006f0  f0101a60  00101a60  00002a60  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003c55  f0102150  00102150  00003150  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      00001963  f0105da5  00105da5  00006da5  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         00009300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .got          00000008  f0111300  00111300  00012300  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  6 .got.plt      0000000c  f0111308  00111308  00012308  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  7 .data.rel.local 00001000  f0112000  00112000  00013000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  8 .data.rel.ro.local 00000044  f0113000  00113000  00014000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  9 .bss          00000648  f0113060  00113060  00014060  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 10 .comment      00000029  00000000  00000000  000146a8  2**0
                  CONTENTS, READONLY
```
gdb验证，发现和elf中的地址是一样的。
 ![Image74.png](/lab1/graphs/Image74.png)

2.  Complete the implementation of debuginfo_eip by inserting the call to stab_binsearch to find the line number for an address.
解答：
一，需要明确函数stab_binsearch和debuginfo_eip的作用
stab_binsearch通过二分查找，从.stabs段中查找type类型的字段，其在.stabs中的地址，正好涵盖addr。
debuginfo_eip的作用，给定指令地址，查找该地址所属的文件，行号，函数等信息。
二，了解STABS格式
**http://sourceware.org/gdb/onlinedocs/stabs.html**
**With the ‘-g’ option, GCC puts in the .s file additional debugging information, which is slightly transformed by the assembler and linker, and carried through into the final executable. This debugging information describes features of the source file like line numbers, the types and scopes of variables, and function names, parameters, and scopes.**
**An N_SLINE symbol represents the start of a source line. The desc field contains the line number and the value contains the code address for the start of that source line.**
三，代码
```
~ 182     stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
+ 183     if (lline <= rline) {
+ 184            info->eip_line = stabs[lline].n_desc;
+ 185     }
+ 186     else {
+ 187             cprintf("line not find\n");
+ 188     }
```
3. Add a backtrace command to the kernel monitor, and extend your implementation of mon_backtrace to call debuginfo_eip and print a line for each stack frame.
参考`monitor.c`中的`help`和`kerninfo`这两条命令。代码如下：
```
 static struct Command commands[] = {
     { "help", "Display this list of commands", mon_help },
     { "kerninfo", "Display information about the kernel", mon_kerninfo },
+  { "backtrace", "Display backtrace info", mon_backtrace },
 };
 
 int
 mon_backtrace(int argc, char **argv, struct Trapframe *tf)
 {
-    uint32_t ebp, *p;
+    uint32_t ebp, eip, *p;
+    struct Eipdebuginfo info;
 
     ebp = read_ebp();
     while (ebp != 0)
     {
         p = (uint32_t *) ebp;
-        cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, p[1], p[2], p[3], p[4], p[5], p[6]);
+        eip = p[1];
+        cprintf("ebp %x eip %x args %08x %08x %08x %08x %08x\n", ebp, eip, p[2], p[3], p[4], p[5], p[6]);
+        if (debuginfo_eip(eip, &info) == 0)
+        {
+            int fn_offset = eip - info.eip_fn_addr;
+
+            cprintf("%s:%d: %.*s+%d\n", info.eip_file, info.eip_line, info.eip_fn_namelen, info.eip_fn_name, fn_offset);
+        }
         ebp = p[0];
     }
```
参考了github代码[mit-jos-2014/Lab1/Exercise12/](https://github.com/clpsz/mit-jos-2014/tree/master/Lab1/Exercise12)
