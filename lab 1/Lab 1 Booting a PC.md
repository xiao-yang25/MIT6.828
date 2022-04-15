# Lab 1 Booting a PC

## Part 1 PC Bootstrap

官网资料地址：https://pdos.csail.mit.edu/6.828/2018/labs/lab1/

###Exercise 1

#### Simulating the x86

要求从学习x86汇编语言开始，我这里选择了官网给的简洁版本的文档[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)。里面简要给出了lab使用的AT&T语法和Intel语法的对应关系。

#### The PC's Physical Address Space

```c
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

早期的8088处理器只能寻16位地址，但它的地址总线是20位)，那它是怎么用16位达到20位的寻址从而管理1M（2^20^=1M）物理内存的呢？

这里就要祭出它的寻址方式了：**physical address = 16*segment + offset**

也就是 **物理地址 = 16*段地址+偏移位置**，这就是所谓的（CS:IP）寻址方式了。其中CS为**代码段寄存器**，IP为**指令指针寄存器**。

####The ROM BIOS

开两个终端，分别先后在lab目录下输入以下命令：

```c
make qemu-gdb
make gdb
```

然后可以看到如下代码：

```c
Type "apropos word" to search for commands related to "word".
+ target remote localhost:26000
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

The target architecture is assumed to be i8086
[f000:fff0]    0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
```

可以看到BIOS第一条执行地址在`0xffff0`，是怎么计算出来的呢？用我上面给出的公式，PC通电后会设置CS为`0xf000`，IP为`0xfff0`，代入公式计算后即为`0xffff0`。

### Exercise 2

接下来查看BIOS开始阶段执行了哪些指令。

```c
0xffff0:	ljmp   $0xf000,$0xe05b
```

跳转到`0xf000`*16+`0xe05b`=`0xfe05b`处。

输入si：

```c
0xfe05b:	cmpl   $0x0,%cs:0x6ac8
```

将`0x0`这个立即数和%cs:0x6ac8地址处的值进行对比。

```c
0xfe062:	jne    0xfd2e1
```

ZF（zero flag）为0时，即%cs:0x6ac8不为0时跳转。

```c
0xfe066:	xor    %dx,%dx
0xfe068:	mov    %dx,%ss
0xfe06a:	mov    $0x7000,%esp
0xfe070:	mov    $0xf34c2,%edx
0xfe076:	jmp    0xfd15c
0xfd15c:	mov    %eax,%ecx
```

看地址可知没有跳转，将dx寄存器清零。

然后将栈的段地址`ss`清零，也即设为`0x0000`，偏移量设为`0x7000`，跳转到`0xfd15c`执行。

```c
0xfd15f:	cli
```

关闭中断指令。

```c
0xfd160:	cld 
```

设置方向标志位为0，表明串操作方向是从低地址到高地址。

```c
0xfd161:	mov    $0x8f,%eax
0xfd167:	out    %al,$0x70
0xfd169:	in     $0x71,%al
```

关闭不可屏蔽中断。

```c
0xfd16b:	in     $0x92,%al
0xfd16d:	or     $0x2,%al
0xfd16f:	out    %al,$0x92
```

激活A20地址线，用于测试可用的地址空间。

```c
0xfd171:	lidtw  %cs:0x6ab8
0xfd177:	lgdtw  %cs:0x6a74
```

加载`0x6ab8`开始的6个字节到中断描述符表寄存器，加载`0x6a74`开始的6个字节全局描述符表寄存器。

```c
0xfd17d:	mov    %cr0,%eax
0xfd180:	or     $0x1,%eax
0xfd184:	mov    %eax,%cr0
```

CR0最低位置为1，检测是否能工作在保护模式下。

```c
0xfd187:	ljmpl  $0x8,$0xfd18f
0xfd18f:	mov    $0x10,%eax
0xfd194:	mov    %eax,%ds
0xfd196:	mov    %eax,%es
0xfd198:	mov    %eax,%ss
0xfd19a:	mov    %eax,%fs
0xfd19c:	mov    %eax,%gs
0xfd19e:	mov    %ecx,%eax
```

重新加载所有段寄存器的值。

## Part 2 The Boot Loader

PC的软盘和硬盘氛围一个个扇区，1 sector = 512 byte。如果磁盘是可引导的，那第一个扇区叫引导扇区。

当BIOS找到可引导的软硬盘时，会将引导扇区加载到物理地址`0X7C00`到`0x7dff`的内存中， 然后将`CS:IP`设置为`0000:7c00`，把控制权交给Boot Loader。

而Boot Loader主要做了两件事：

一是将CPU从从16-bit的实模式转化到32-bit的保护模式；

二是使用特殊的 I/O指令，从硬盘中读取kernel。

Q1：At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

在`boot/boot.S`中，有这样一行代码：

```c
# Jump to next instruction, but in 32-bit code segment.
# Switches processor into 32-bit mode.
ljmp    $PROT_MODE_CSEG, $protcseg
```

看注释可以很清楚地直到，在这里开始执行32位代码。







