# 骨架

**等等！你读了[起步](Getting_Started.md)，[初学者的错误](Beginner_Mistakes.md)以及一些相关的[OS理论](Category:OS_theory.md)吗？**

在这个教程中你将会写一个简单的[32位x86](IA32_Architecture_Family.md)内核并且启动它。这是你创建你自己操作系统的第一步。这个教程通过提供一个例子来介绍如何创建一个最小的的操作系统，但是这并不能作为如何正确组织你的项目的例子。这些步骤由社区审核过并且按照当前的推荐顺序有足够的原因。警惕许多其他在线教程，它们并没有遵循现代的建议，是又缺乏经验的人编写的。

你现在正准备开发一个新的操作系统。也许有一天你的新系统能够在它自己上开发。这个过程被叫做自举或者由自己作为宿主。现在，你将简单的配置好一套系统来再现有的操作系统上编译你的新操作系统。这个过程叫做[交叉编译](Why_do_I_need_a_Cross_Compiler?.md)，它是你开发操作系统的第一步。

这个教程使用已经存在的技术来让你能够直接进入[内核](Kernel.md)开发，而不是开发你自己的[编程语言](Languages.md)、你自己的[编译器](Compiler.md)以及你自己的[引导器](Bootloader.md)。在本教程中，你会用到：

* 来自[Binutils](Binutils.md)中的[GNU链接器](LD.md)，用于链接你的[对象文件](Object_File.md)成为最终的内核。

* 来自[Binutils](Binutils.md)中的[GNU汇编器](GAS.md)（[NASM](NASM.md)也是可选项），用于将汇编代码[汇编](Assembly.md)成包含机器指令的对象文件。

* [GNU编译器套件](GCC.md)，用于将你高级语言的代码编译成汇编代码。

* C编程语言（C++也是可选项），用于编写[内核](Kernel.md)的上层部分。

* [GRUB](GRUB.md)引导器，用于加载使用了[Multiboot](Multiboot)协议的你的内核，并将你切换到禁用分页的32位保护模式。

* [可执行文件格式](Executable_Formats)ELF，使我们能够控制内核加载的位置以及方法。

本文章假设你正在使用如Linux的类Unix的操作系统，它们能很好的支持开发操作系统的需要。Windows用户可以使用[WSL](WSL.md)、[MinGW](MinGW.md)或[Cygwin](Cygwin.md)环境来实现。

要想成功开发操作系统，需要你成为一位有耐心、仔细阅读所有指令的专家。你需要在实际操作前阅读所有这篇文章中的内容。如果你遇到问题，你应该更加仔细的阅读并且三倍仔细的操作。如果你仍然遇到问题，OSDev社区很有经验并且愿意在[论坛](http://forum.osdev.org/)或[讨论](https://wiki.osdev.org/Chat)上帮助你。

---

[TOC]

---

## 构建交叉编译器

> 主文章：[GCC交叉编译器](GCC_Cross-Compiler.md)，[为什么我需要交叉编译器](Why_do_I_need_a_Cross_Compiler?.md)

你需要做的第一件事是配置好适用于**i686-elf**的[GCC交叉编译器](GCC_Cross-Compiler.md)。你还没有修改你自己的编译器来让它直达你操作系统的存在，因此你将会使用一个通用的目标架构，叫做i686-elf，它给你提供了一个适用于System V ABI的工具链。这项配置已经经过了完善的测试，osdev社区了解，以及允许你能通过GRUB和Multiboot轻易地配置好一个可引导的内核。（注意，如果你正在使用一个ELF平台，如Linux，你也许已经有一个生成GCC程序的GCC。这不适用于开发操作系统，因为这个编译器会生成用于Linux的代码，而你的系统**不是**Linux，不管它有多像。如果不用交叉编译器，你一定会遇到问题的。）

你*不*能不用交叉编译器正确的编译你的内核。

你*不*能使用适用于x86_64-elf的交叉编译器来完成这个教程，因为GRUB之你那个加载32位的Multiboot内核。如果这是你的第一个操作系统项目，你应当先开发一个32位的内核。如果你使用了x86_64的编译器并且想办法通过了之后的完整性检查，你最终会遇到GRUB不知道如何引导的问题。

## 简介

现在，你应该已经配置好了适用于i686-elf的交叉编译器（在上面介绍的）。这个教程提供了一个开发基于x86的操作系统的最小的解决方案。这个项目并不推荐作为项目组织的骨架，而是一个最小的内核的例子。在这简单的情况下，你只需要三个文件：

* boot.s - 配置处理器环境的内核入口

* kernel.c - 你实际的内核例程

* linker.ld - 用于链接上面的文件

## 引导操作系统

为了启动操作系统，需要加载一块存在的程序。这叫做引导程序，在本教程中你将使用[GRUB](GRUB.md)。编写你自己的引导器是一个复杂的项目，但通常会这么做。我们之后会配置引导器，但操作系统需要在引导器交出控制权时接管。接管时内核处于一个最小化的环境，即没有配置堆栈、没有启用虚拟内存、没有初始化硬件等等。

你要解决的第一件事是如何让引导器启动内核。OS的开发者是幸运的，因为已经有一个Multiboot标准，它在引导器和系统内核间定义了一个简单的接口。它通过在全局变量中设置的一些魔数（叫做Multiboot头）来让引导器搜索。当看见这些数值后，它识别到这个内核兼容Multiboot、知道如何加载，甚至可以我们传递重要的信息，比如内存分布表，尽管你并不需要用到。

由于还没有堆栈，并且需要确保全局变量被正确地设置，你将通过汇编来做这些。

### 引导汇编

> 你也可以使用[NASM](Bare_Bones_with_NASM.md)或[EPLOS](EPLOS)来作为汇编器。

你现在将创建文件boot.s，并且了解它的内容。在这个例子中，你将使用GNU汇编器，它是你之前构建的交叉编译器的一部分。这个汇编器和其它GNU工具配合的非常好。

最重要的部分是创建Multiboot头，它需要放在内核二进制的较前的位置，否则引导器无法识别到我们。

```gas
/* 定义multiboot头的常量 */
.set ALIGN,    1 << 10          /* 将加载的模块按页对其 */
.set MEMINFO,  1 << 1           /* 提供内存分布表 */
.set FLAGS,    ALIGN | MEMINFO  /* 这是Muiltiboot的标志项 */
.set MAGIC,    9x1BADB002       /* “魔数”让引导器能够找到头 */
.set CHECKSUM, -(MAGIC + FLAGS) /* 上述内容的校验和，来证明我们是multiboot */

/*
定义multiboot头来标志一个程序是内核。这些魔数记录在multiboot标准。
引导器会在内核文件的前8KiB中搜索按照32位对其的标志。
这个标志在它自己的节中，因此我们可以强制将它放在内核文件的前8KiB中。
*/
.section .multiboot
.align 4
.long MAGIC
.long FLAGS
.long CHECKSUM

/*
Multiboot标准没有定义堆栈指针寄存器ESP的值，而时由内核自己提供堆栈。
此处通过在堆栈底创建一个符号、分配16384字节、在堆栈顶创建一个符号来分配了一个小堆栈的空间。堆栈在x86上向低增长。
堆栈在它自己的节中，因此可以被标记为无数据，这样由于不需要存储一个未初始化的堆栈，内核文件可以更小。
由于System V ABI标准以及事实上的扩展，x86的堆栈必须按照16字节对齐。
编译器会假设堆栈被正确的对齐，对其堆栈失败会导致未定义行为。
*/
.section .bss
.align 16
stack_bottom:
.skip 16384   # 16 KiB
stack_top:

/*
链接脚本指定_start作为内核的入口，引导器会在内核加载完毕后跳转到此处。从该函数返回没有意义，因为引导器已经没有了。
*/
.section .text
.global _start
.type _start, @function
_start:
    /*
    引导器在x86机器上将我们载入32位保护模式。中断被禁用。分页被禁用。
    处理器的状态和定义在multiboot标准中的一样。内核拥有CPU完全的控制权。
    内核只能使用硬件特性或是任何它自己的代码。这里没有printf，除非内核提供它自己的<stdio.h>头文件并且printf的实现。
    这里没有安全限制，没有保障措施，没有调试机制，只有内核自己的提供的内容。
    它拥有机器绝对和完全的控制权。
    */

    /*
    为了配置一个堆栈，我们把ESP寄存器指向堆栈的栈顶（因为它在x86系统上向低增长）。这里必须由汇编来完成，因为C等的语言需要堆栈才能工作。
    */
    mov $stack_top, %esp

    /*
    这里，在进入上层内核之前，是初始化处理器关键状态的好地方。最好最小化关键功能不在线的环境。
    注意这里的处理器没有完全初始化完毕。如浮点数指令、扩展指令集等特性还没有初始化。
    应当在这里加载GDT。分页也应当在这里启用。如全局构造函数以及异常等C++的特性需要运行时支持来正常工作。
    */

    /*
    进入上层内核。ABI要求在call指令（这个指令会压入4字节的返回地址）调用时堆栈是16字节对齐。堆栈原来就是16字节对齐的，并且我们目前压入了16字节的整数倍（到目前为止压入了0字节），因此保持了对齐状态并且call指令现在是定义明确的。
    */
    call kernel_main

    /*
    如果系统没有任何事情需要做，使电脑死循环。要做到的话：
    1) 通过cli指令（清除eflags中的中断功能标志）。这已经由引导器帮你做了，所以这并不是必须的。
       但是注意你有可能之后启用了中断，然后从kernel_main中返回（虽然这很荒谬）
    2) 使用hlt指令（停指令）等待下一次中断。由于中断已经关闭了，这会导致电脑锁死。
    3) 如果由于不可屏蔽中断或系统管理模式被唤醒，跳转到hlt指令。
    */
    cli
1:  hlt
    jmp 1b

/*
设置_start符号的大小为当前位置'.'减去它开始的地方。这在调试或你自己实现调用追踪时会有用。
*/
.size _start, . - _start
```

你现在可以通过以下命令汇编boot.s：

```shell
i686-elf-as boot.s -o boot.o
```

## 实现内核

到目前为止你已经编写了为如C等的高级语言初始化处理器的汇编根基。使用其他如C++等的语言也是可行的。

### 独立的和有宿主的环境

如果你曾在用户空间编写C或C++程序，你已经用过所谓的有宿主的环境。有宿主是指有一个C标准库以及其它有用的特性。另外，也有独立的环境，就是你现在在这里使用的。独立是指这里没有标准C库，只有你自己提供的的东西。然而，部分头文件实际上并不是C标准库的一部分，而是编译器的。这些文件即使在独立环境的C代码中也是可用的。在这种情况下，你可以用<stdbool.h>来得到布尔类型，用<stddef.h>来得到size_t和NULL，用<stdint.h>来得到intx_t和uintx_t。这些类型在操作系统开发中是非常重要的，因为你会需要确保变量是固定大小（比如，如果你使用short而不是uint16_t，而short的长度发生了变化，那你的VGA驱动会崩溃！）另外你还可以使用<float.h>、<iso646.h>、<limits.h>和<stdarg.h>这些头文件，它们也是独立的。GCC实际上还会提供一些额外的头文件，但是这些是有特殊用途的。

### 使用C编写内核

下面的代码演示了如何用C创建一个简单的内核。这个内核使用VGA文本模式的缓冲区（位于0xB8000）作为输出的设备。它初始化了一个简单的驱动，这个驱动可以记住缓冲区中下一个字符的位置，以及提供了输出字符的原语。需要注意的是，这里没有支持换行符（'\n'）（并且写这个字符会显示特定于VGA的一些字符）和屏幕满的时候的滚动。添加这些功能可以作为你的第一个任务。请花几分钟理解这些代码。

**重要注意**：VGA文本模式（或者BIOS）对新机器来说是已弃用的，而UEFI只支持像素缓冲。为了向前兼容性，你也许会想直接从这里开始。用合适的Multiboot标志让GRUB设置好帧缓冲区或者你自己调用[VESA VBE](Vesa.md)。不同于VGA文本模式，一个帧缓冲区只有像素，因此你不得不自行绘制每一个字形。这意味着你需要一个不同的`terminal_putchar`，并且你需要一个字体（对于每一个字符的位图）。所有Linux发行版都提供了你可以使用的[PC屏幕字体](PC_Screen_Font.md)，而且维基上的文章内有一个简单的putchar()示例。否则这里描述的其它内容仍然可用（你需要自己记录光标位置，实现换行以及滚动等）

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
 
/* 检测是否编译器认为你针对的是错误的目标操作系统 */
#if defined(__linux__)
/* 你现在并没有在使用一个交叉编译器，你大概是遇到问题了 */
#error "You are not using a cross-compiler, you will most certainly run into trouble"
#endif
 
/* 这个教程只会在32位ix86的目标上工作 */
#if !defined(__i386__)
/* 这个教程需要由ix86-elf的编译器编译 */
#error "This tutorial needs to be compiled with a ix86-elf compiler"
#endif
 
/* 硬件中文本模式的颜色常量 */
enum vga_color {
	VGA_COLOR_BLACK = 0,
	VGA_COLOR_BLUE = 1,
	VGA_COLOR_GREEN = 2,
	VGA_COLOR_CYAN = 3,
	VGA_COLOR_RED = 4,
	VGA_COLOR_MAGENTA = 5,
	VGA_COLOR_BROWN = 6,
	VGA_COLOR_LIGHT_GREY = 7,
	VGA_COLOR_DARK_GREY = 8,
	VGA_COLOR_LIGHT_BLUE = 9,
	VGA_COLOR_LIGHT_GREEN = 10,
	VGA_COLOR_LIGHT_CYAN = 11,
	VGA_COLOR_LIGHT_RED = 12,
	VGA_COLOR_LIGHT_MAGENTA = 13,
	VGA_COLOR_LIGHT_BROWN = 14,
	VGA_COLOR_WHITE = 15,
};
 
static inline uint8_t vga_entry_color(enum vga_color fg, enum vga_color bg) 
{
	return fg | bg << 4;
}
 
static inline uint16_t vga_entry(unsigned char uc, uint8_t color) 
{
	return (uint16_t) uc | (uint16_t) color << 8;
}
 
size_t strlen(const char* str) 
{
	size_t len = 0;
	while (str[len])
		len++;
	return len;
}
 
static const size_t VGA_WIDTH = 80;
static const size_t VGA_HEIGHT = 25;
 
size_t terminal_row;
size_t terminal_column;
uint8_t terminal_color;
uint16_t* terminal_buffer;
 
void terminal_initialize(void) 
{
	terminal_row = 0;
	terminal_column = 0;
	terminal_color = vga_entry_color(VGA_COLOR_LIGHT_GREY, VGA_COLOR_BLACK);
	terminal_buffer = (uint16_t*) 0xB8000;
	for (size_t y = 0; y < VGA_HEIGHT; y++) {
		for (size_t x = 0; x < VGA_WIDTH; x++) {
			const size_t index = y * VGA_WIDTH + x;
			terminal_buffer[index] = vga_entry(' ', terminal_color);
		}
	}
}
 
void terminal_setcolor(uint8_t color) 
{
	terminal_color = color;
}
 
void terminal_putentryat(char c, uint8_t color, size_t x, size_t y) 
{
	const size_t index = y * VGA_WIDTH + x;
	terminal_buffer[index] = vga_entry(c, color);
}
 
void terminal_putchar(char c) 
{
	terminal_putentryat(c, terminal_color, terminal_column, terminal_row);
	if (++terminal_column == VGA_WIDTH) {
		terminal_column = 0;
		if (++terminal_row == VGA_HEIGHT)
			terminal_row = 0;
	}
}
 
void terminal_write(const char* data, size_t size) 
{
	for (size_t i = 0; i < size; i++)
		terminal_putchar(data[i]);
}
 
void terminal_writestring(const char* data) 
{
	terminal_write(data, strlen(data));
}
 
void kernel_main(void) 
{
	/* 初始化终端接口 */
	terminal_initialize();
 
	/* 换行支持留作练习 */
	terminal_writestring("Hello, kernel World!\n");
}
```

注意你在代码中很想使用一般的C函数`strlen`，但这个函数是C标准库的一部分因此你并没有一个可用的。相反，你可以依赖独立头文件<stddef.h>来提供`size_t`并且自己简单的提供你自己的`strlen`实现。你需要为每个你想用的函数这样做（因为独立的头文件只提供宏和数据类型）。

使用以下指令编译：

```shell
i686-elf-gcc -c kernel.c -o kernel.o -std=gnu999 -ffreestanding -O2 -Wall -Wextra
```

注意以上代码使用了一些扩展并且因此你以GNU的C99版本来构建。

### 使用C++编写内核

用C++写一个内核是容易的。需要注意不是所有的语言特性可用。具体来说，异常支持需要特殊的运行时支持，内存分配也是如此。为了用C++编写内核，简单的采用上述代码：给主函数添加一个`extern "C"`声明。需要注意kernel_main函数需要定义成C的链接模式，否则编译器会在它的汇编名称中添加类型信息（名称修饰）。这会复杂化从汇编中调用函数的过程，然而如果你使用C的链接模式，符号的名称和函数的名称一样的（没有额外的类型信息）。将代码保存为kernel.c++（或任何你喜欢的C++文件后缀名）。

你可以用以下命令编译：

```shell
i686-elf-g++ -c kernel.c++ -o kernel.o -ffreestanding -O2 -Wall -Wextra -fno-exception -fno-rtti
```

注意你必须已经为这个工作构建了C++的交叉编译器。

## 链接内核

你现在可以汇编boot.s和编译kernel.c。这会产生两个目标文件，其中各自包含内核的一部分。为了创建最终完整的内核你需要将这些目标文件链接成最终引导器可用内核程序。当开发用户空间开发程序时，你的工具链为链接这类程序提供了默认的链接脚本。然而这在内核开发中是不合适的，你需要提供你自己调整的链接脚本。保存以下内容到linker.ld：

```c
/* 链接器会查看这个镜像并且从被指定位入口的符号处开始执行 */
ENTRY(_start)
 
/* 声明目标文件中不同节在最终内核镜像中的位置 */
SECTIONS
{
	/* 从1MiB处开始放置，一个引导器加载内核的传统位置 */
	. = 1M;
 
	/* 首先放置multiboot头，因为它需要在镜像的靠前处，否则引导器无法识别文件格式。
	   接着我们放置.text节 */
	.text BLOCK(4K) : ALIGN(4K)
	{
		*(.multiboot)
		*(.text)
	}
 
	/* 只读数据 */
	.rodata BLOCK(4K) : ALIGN(4K)
	{
		*(.rodata)
	}
 
	/* 初始化后的可读写数据 */
	.data BLOCK(4K) : ALIGN(4K)
	{
		*(.data)
	}
 
	/* 未初始化的可读写数据以及堆栈 */
	.bss BLOCK(4K) : ALIGN(4K)
	{
		*(COMMON)
		*(.bss)
	}
 
	/* 编译器也许会产生其它节，默认它们会被放置在同名的节内。如果需要其它内容，简单加在这里即可。 */
}
```

有了这几部分，你现在可以真正构建最终的内核了。我们使用编译器来作为链接器，因为它允许对链接过程有更强的控制。注意如果你的内核是由C++编写，你应该使用C++的编译器。

你可以用一下命令链接内核：

```shell
i686-elf-gcc -T linker.ld -o myos.bin -ffreestanding -O2 -nostdlib boot.o kernel.o -lgcc
```

注意：有些教程建议使用i686-elf-ld而不是编译器来链接，但是这会导致编译器无法在链接中执行许多工作。

文件myos.bin就是你的内核（所有其它文件都不再需要了）。注意我们链接了libgcc，它实现了许多你的交叉编译器依赖的运行时的功能。如果不使用它的话会在将来给你带来问题。如果你没有构建并安装libgcc作为交叉编译器的一部分，你应该立刻回去重新构建一个带有libgcc的交叉编译器。编译器依赖它，并且无论你是否提供它都会依赖。

## 验证Multiboot

如果你已经安装了GRUB，你可以检查一个文件是否包含一个有效的1版本Multiboot头，就和你的内核一样。令Multiboot头位于程序文件的前8KiB、按照4字节对齐是非常重要的。可能会因为你在启动汇编、链接脚本或其它事情中犯错而不再满足要求。如果头不合法，GRUB会在你尝试引导它时提示找不到multiboot头错误。这段代码可以帮助你诊断这类情况：

```shell
grub-file --is-x86-multiboot myos.bin
```

grub-file本身不会输出，但是会在确实是一个multiboot内核（成功）时返回0，反之返回1.你可以在你的shell中紧跟着输入`echo $?`来查看退出状态。你可以在你的构建脚本中添加这个检测来作为一个完整性测试，据此可以觉察到编译时的问题。版本2的multiboot可以用`--is-x86-multiboot2`选项来检测。如果你要在shell中手动执行grub-file命令，将其包装到条件语句中可以方便的查看状态。以下命令应该能够工作：

```shell
if grub-file --is-x86-multiboot myos.bin; then
  echo multiboot confirmed
else
  echo the file is not multiboot
fi
```

## 引导内核

再过一会儿，你就能看到你的内核运行了。

### 构建一个可引导的CDROM镜像

你可以通过grub-mkrescue程序简单的创建一个装有GRUB引导器和你的内核的可引导CDROM镜像。你也许会需要安装GRUB使用程序以及程序xorriso（0.5.6版或更新）。首先，你应该创建一个包含如下内容的文件grub.cfg。

```c
menuentry "myos" {
	multiboot /boot/myos.bin
}
```

注意大括号必须放置在上面显示的地方。你现在可以用以下命令为你的操作系统创建一个可引导的镜像：

```shell
mkdir -p isodir/boot/grub
cp myos.bin isodir/boot/myos.bin
cp grub.cfg isodir/boot/grub/grub.cfg
grub-mkrescue -o myos.iso isodir
```

恭喜！你现在已经创建了文件myos.iso，它包含了你的Hello World操作系统。如果你没有安装程序grub-mkrescue，现在正是安装GRUB的时候。在Linux系统上它应该已经被安装了。对于Windows用户，如果没有可用的本地grub-mkrescue程序，可能会想使用一个Cygwin上变体。

**警告**：grub-mkrescue使用的引导器GNU GRUB是基于GPL协议的。你的ISO文件含有受该协议保护的内容，违背GPL协议分发它将会构成侵权。GPL要求你发布与引导器相对应的源代码。你应当在执行grub-mkrescue时（因为发行包偶尔会更新）获取与你安装的GRUB包对应的代码包。之后你应当将代码和你的ISO一同发布以满足GPL协议。另外，你可以自行从源代码构建GRUB。从savannah处克隆最新的GRUB的git仓库（不要使用它们2012年最后的发行，这严重过时了）。执行autogen.sh、./configure和make dist。这构建了一个GRUB压缩包。将其在某处解压，然后从里面构建GRUB，之后将其安装在某个独立的prefix下。将其加入到你的PATH中并保证是在用它的grub-mkrescue来生成你的ISO。之后将你自己构建的这个GRUB压缩包附带在你OS的发行版中。你并没有被要求发布你自己OS的代码，只有你ISO中的引导器的代码需要。

### 测试你的操作系统（QEMU）

虚拟机在开发操作系统中非常有用，它们允许你迅速地测试你的代码并且在执行时能够访问到源代码。否则，你将会处于一个无尽的重启循环，只会惹恼你。它们启动地很快，尤其是配合小型的操作系统，就像你自己的。

在这个教程中，我们将会使用QEMU。你也可以使用你中意的其他虚拟机。简单地将ISO加入到一个空虚拟机的CD驱动中就能成功。

从你的仓库中安装QMEU，然后使用以下命令启动你新的操作系统。

```shell
qemu-system-i386 -cdrom myos.iso
```

这应该能启动一个只含有被当做CDROM的你的ISO的虚拟机。如果一切顺利，你将会遇到一个由引导器提供的菜单。简单地选择myos，如果一切顺利的话你应该就能看见"Hello, Kernel World!"以及一些奇怪的字符紧跟在后面。

另外，QEMU支持不用可引导介质直接启动multiboot内核：

```shell
qemu-system-i386 -kernel myos.bin
```

### 测试你的操作系统（真实硬件）

grub—mkrescue程序很好，它制作的可引导ISO在实体机和虚拟机上都能工作。你可以构建一个ISO并且在任何地方使用它。要用你自己的机器启动你的内核，你可以将myos.bin安装你的/boot目录并且正确配置你的引导器。

又或者，你可以将它烧录到一个U盘上（会删除上面的所有数据！）。要想这样做的话，只需要找到你U盘的块设备名称，在我这是/dev/sdb但这并不固定，而且使用错误的块设备（比如你的硬盘，噫！）可能会损失惨重。如果你正在使用Linux而且你的块设备名称是/dev/sdx，只需要：

```shell
sudo dd if=myos.bin of=/dev/sdx && sync
```

你的操作系统就会被安装到你的U盘上。如果你配置了你的BIOS让从USB优先启动，你可以插入U盘然后你的电脑就应该会启动你的操作系统了。

另外， .iso文件就是普通的CDROM镜像。将一个几KiB的“大”内核烧录到一个CD或DVD，如果你愿意浪费的话。

## 下一步

现在你已经可以运行你崭新的内核了，恭喜！当然，基于这对你来说有多有趣，这可能只是一个开始。下面有一些你可以继续做的事。

### 给终端驱动添加换行支持

现在的终端驱动没有处理换行符。由于它是一个逻辑实体，它永远不会被实际渲染，因此VGA文字模式的字体在那存储了一个其它的字符。更确切地说，在terminal_putchar里面检查，如果c=='\n'则增加terminal_row并且重置terminal_column。

### 实现终端滚动

当遇到终端填满的情况，它只是简单的回到屏幕的开头。这对于通常的使用是不可接受的。相反，它应当将所有行上移并且丢弃最上面的，以及在最下面预留一个将会被填充字符的空白行。实现它。

### 渲染彩色的文字艺术

使用已有的终端驱动渲染一些使用全部16种可用颜色的漂亮东西。注意背景可能只有8种可用颜色，因为默认情况下实体的最高位是用于其他含义而非背景色。你会需要一个真正的VGA驱动来修复它。

### 调用全局构造函数

> 主文章：[调用全局构造函数](Calling_Global_Constructors.md)

这个教程展示了如何开发C、C++内核的最小化环境的一个小例子。不幸的是，你还没有将所有的事情处理好。比如，C++中的全局对象的构造函数不会被调用，因为你没有做。通过crt*.o，编译器使用了一个特殊的机制来实现程序初始化期间的任务，这对C程序员可能很重要。如果你正确的组合了crt*.o，你应该会创建一个执行所有程序初始化任务的_init函数。你的boot.o对象文件之后可以在执行kernel_main之前调用_init。

### 肉骨架

> 主文章：[肉骨架](Meaty_Skeleton.md)

这个教程是作为一个最小化例子，给没耐心的初学者一个快速的hello world操作系统。它故意最小化并且没有展示组织你的操作系统的最佳实践。肉骨架教程展示了如何组织一个带有内核、标准库有增长空间以及为用户空间出现做准备的一个例子。

### 走的更远

> 主文章：[在x86上走的更远](Going_Further_on_x86.md)

这个指南是一个做什么的概览，这样当你准备为内核实现更多特性时，不用重新设计你的内核就能添加它们。

### 骨架2

让你的操作系统能够以自己作为宿主，然后在你自己的系统上跟随所有步骤完成骨架。这是一个5星级的锻炼，你可能需要几年来完成它。

## 常问的问题

#### 为什么要有multiboot头？GRUB不能加载纯ELF文件吗？

GRUB有能力加载许多格式。然而，在本教程中我们创建了一个符合multiboot的内核，它能够被其它任何合规的引导器引导。要实现这个，multiboot头是不可或缺的。

#### 我的内核需要AOUT补丁吗？

AOUT补丁对于ELF格式的内核不是必须的：一个符合multiboot的引导器能识别ELF可执行文件，因此会使用程序头来将其加载到合适的位置。你可以为你的ELF内核提供AOUT补丁，在这种情况下ELF头会被忽略。对于其他格式，如AOUT、COFF或者PE的内核，AOUT补丁则是必须的。

#### multiboot头能够位于内核的任何地方吗？又或者它必须位于一个特定的偏移处吗？

multiboot头必须位于内核文件的前8kb并且按照32位（4字节）对齐来让GRUB找到它。你可以通过将头放在单独的代码中然后将其作为第一个对象文件传给LD来确保是这种情况。

#### GRUB会在加载内核前清空BSS节吗？

是的。对于ELF内核，.bss节是自动识别并且清空的（尽管 Multiboot 规范对此有点模糊）。对于其它格式，如果你礼貌地让它做，指使用multiboot头中的“地址覆盖”信息（标志#16）并且给bss_end_addr域提供一个非0值，的话，会的。注意，对ELF格式使用“地址覆盖”的话会禁用默认的行为，只做在“地址覆盖”头中描述的内容。

#### 当GRUB调用我的内核时，寄存器、内存等是什么状态？

GRUB是一个multiboot规范的实现。任何它未指定的都是未定义行为，这应该（但不仅）为C、C++程序员敲响警钟……最好查阅multiboot文档中机器状态节的内容，并且假设没有其它任何东西。

#### 我仍然从GRUB处遇到“Error 13: Invalid or unsupported executable format”……

有可能最后的可执行文件中缺少了multiboot头，或者它不在正确的位置。

如果你正在使用其他格式而非ELF（比如PE），你应当在multiboot头中给出AOUT补丁。上面提到的grub-file程序以及`objdump -h`应该能够给你更多关于发生了什么的提示。

这也有可能会在你使用了一个ELF对象文件（例如，一个含有未解决符号或不确定位置的ELF）而非可执行文件时发生。尝试将你的ELF文件连接成二进制可执行文件来获取更准确的错误信息。

一个常见的问题是随着你内核大小增加，multiboot头不再位于输出的二进制的头部了。通常的解决方法是将multiboot头放在一个独立的节内并且确保它是输出二进制的第一个节，或者在链接脚本中包含multiboot头本身。

#### 当我尝试用QEMU启动ISO镜像时遇到了“Boot failed: Could not read from CD-ROM (code 0009)”

如果你的开发系统是通过EFI引导的，你可能并没有安装PC-BIOS版本的GRUB二进制。如果你安装了它们，那grub-mkrescue会默认创建一个能在QEMU上工作的混合镜像。在Ubuntu上，你可以通过这来实现：`apt-get install grub-pc-bin`。

## 参考

### 文章

* [书籍](Books.md)

* [Stivale](Stivale.md)

* [BOOTBOOT](BOOTBOOT.md)

### 外部链接

* [Stivale和Stivale2规范](https://github.com/stivale/stivale)

* [Multiboot规范](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)

* [Multiboot2规范](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)

* [BOOTBOOT规范](https://gitlab.com/bztsrc/bootboot/raw/master/bootboot_spec_1st_ed.pdf)

* [POSIX标准](https://pubs.opengroup.org/onlinepubs/9699919799/)

> [分类](Special:Categories.md)：[一级教程](Category:Level_1_Tutorials.md) | [骨架教程](Category:Bare_bones_tutorials.md) | [C](Category:C.md) | [C++](Category:C++.md)