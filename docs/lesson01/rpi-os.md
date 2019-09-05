# 1.1：欢迎来到RPi OS的世界，裸机上的"Hello, World"
我们即将完成一个在裸机上输出"Hello World"的程序，以此作为操作系统开发之旅的开端。在此之前，我默认你已经阅读了[准备工作](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/docs/Prerequisites.md)，准备好了这上面所说的工具，如果还没有，请先去完成。  
  
在我往下讲之前，我想先把教程的层次结构规范化。从README文件中你可以看到整个教程被分为多节**课程**，每节课程都包含了很多**章节**，一章课程根据标题又被分为很多**小节**。以后，我会分别使用课程、章节、小节来指代文章的引用位置。  
  
另一个我想说明的点就是，文章中会出现很多源代码，通常我会先展示整块源码，然后再对其进行逐行讲解。  
  
## 项目结构
课程的文件结构都是一样的，你可以在[这里]()找到课程对应的源代码。先简单介绍一下文件夹的主要组成部分：  
  
1.**Makefile**： 我们使用[make工具](http://www.math.tau.ac.il/~danha/courses/software1/make-intro.html)来构建内核。`make`会根据makefile的配置来执行相应的动作，所以在makefile中，我们需要给出一些如何编译与链接源代码的指令。  
2.**build.sh或build.bat**： 如果你打算用Docker来构建内核，你就会用到这两个文件的其中一个。这样你就不需要在电脑上安装make工具或编译工具链了。  
3.**src**： 该文件夹下包含了源代码。  
4.**include**： 所有的头文件都放在这里。  
  
## Makefile
现在让我们来进一步认识Makefile的使用。make工具最主要的功能，就是自动化地去判断项目中的哪些部分需要被重新编译，并调用命令把这些文件重新编译。如果你不熟悉make和Makefile，我推荐你先阅读[这篇部分内容](http://opensourceforu.com/2012/06/gnu-make-in-detail-for-beginners/)，第一节课用到的Makefile可以在[这里](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/src/lesson01/Makefile)找到。整个Makefile的内容如下：  
```  
ARMGNU ?= aarch64-linux-gnu  
  
COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only  
ASMOPS = -Iinclude  
  
BUILD_DIR = build  
SRC_DIR = src  
  
all : kernel8.img  
  
clean :  
    rm -rf $(BUILD_DIR) *.img  
  
$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c  
    mkdir -p $(@D)  
    $(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@  
  
$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S  
    $(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@  
  
C_FILES = $(wildcard $(SRC_DIR)/*.c)  
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)  
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)  
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)  
  
DEP_FILES = $(OBJ_FILES:%.o=%.d)  
-include $(DEP_FILES)  
  
kernel8.img: $(SRC_DIR)/linker.ld $(OBJ_FILES)  
    $(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o $(BUILD_DIR)/kernel8.elf  $(OBJ_FILES)  
    $(ARMGNU)-objcopy $(BUILD_DIR)/kernel8.elf -O binary kernel8.img  
```  
现在，让我们仔细看看每一行的内容：  
```
ARMGNU ?= aarch64-linux-gnu  
```
这个Makefile从一个变量定义开始，`ARMGNU`表示[交叉编译](https://baike.baidu.com/item/%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91/10916911?fr=aladdin)的前缀。我们需要使用一个交叉编译器，因为我们打算在`x86`的机器上生成`arm64`的可执行文件，所以我们用`aarch64-linux-gnu-gcc`来代替`gcc`。  
```  
COPS = -Wall -nostdlib -nostartfiles -ffreestanding -Iinclude -mgeneral-regs-only  
ASMOPS = -Iinclude 
```  
`COPS`和`ASMOPS`是我们将c语言和汇编语言各自独立编译的时候需要的编译器选项，有必要对这两个选项做一些解释：  
+ **-Wall** 输出所有的警告信息（wall是wallning的缩写）  
+ **-nostdlib** 不实用c语言的标准库。大多数的c语言标准库调用会转而去调用操作系统，但我们这个是将要运行在裸机上的程序，我们**没有任何底层操作系统**，所以标准库在这里是无法使用的。  
+ **-nostartfile** 不使用任何启动文件。启动文件负责设置程序初始化堆栈指针、初始化静态变量以及跳转到main函数入口，我们把所有这些事情都交给自己来完成。  
+ **-ffreestading** 一个freestading环境指的是一个不依赖于任何标准库的环境，而且程序不必要有一个main函数。`-ffreestading`选项告诉编译器不要假定标准函数都有常规的定义。  
+ **-include** 在include文件夹下寻找头文件  
+ **-mgeneral-regs-only** 只使用通用寄存器组。ARM处理器也有[NEON](https://developer.arm.com/architectures/instruction-sets/simd-isas/neon)寄存器组，我们不希望编译器使用它们，因为那会给程序带来不必要的复杂性（比如说，当程序上下文切换时，需要保存寄存器值，这么做就会带来麻烦）。  
```  
BUILD_DIR = build  
SRC_DIR = src
```  
`SRC_DIR`和`BUILD_DIR`分别指的是源代码目录和生成编译结果的目录。  
```
all : kernel8.img  
  
clean :  
    rm -rf $(BUILD_DIR) *.img
```  
下一步，我们定义了make的target。前两个target是非常简单的：`all`是一个默认的target，只要你输入`make`，不需要任何参数，它就会执行（make总是将第一个target设置为默认的），这个target仅仅将所有的任务都重定向到另一个target，`kernel8.img`。`clean`target负责删除所有编译过程的产物以及编译产生的内核映像。  
```
$(BUILD_DIR)/%_c.o: $(SRC_DIR)/%.c  
    mkdir -p $(@D)  
    $(ARMGNU)-gcc $(COPS) -MMD -c $< -o $@  
  
$(BUILD_DIR)/%_s.o: $(SRC_DIR)/%.S  
    $(ARMGNU)-gcc $(ASMOPS) -MMD -c $< -o $@
```  
接下来的两个target负责编译c和汇编文件。例如，如果在`src`文件夹下有`foo.c`和`foo.S`文件，它们分别被编译为`build/foo_c.o`和`build/foo_s.o`。`$<`和`$@`用于替代运行时的输入文件名和输出文件名（`foo.c`和`foo_c.o`）。在编译c文件之前，不要忘了创建`build`文件夹以免找不到这个文件夹。  
```
C_FILES = $(wildcard $(SRC_DIR)/*.c)  
ASM_FILES = $(wildcard $(SRC_DIR)/*.S)  
OBJ_FILES = $(C_FILES:$(SRC_DIR)/%.c=$(BUILD_DIR)/%_c.o)  
OBJ_FILES += $(ASM_FILES:$(SRC_DIR)/%.S=$(BUILD_DIR)/%_s.o)  
```  
这里我们构建了一个目标文件列表，这些文件是通过将c和汇编程序链接起来所产生的。  
```  
DEP_FILES = $(OBJ_FILES:%.o=%.d)  
-include $(DEP_FILES)  
```  
接下去的两行用到了一些小技巧。如果你再看一眼，我们是如何为c和汇编源码定义编译目标的，你就会我们使用了`-MMD`参数。这个参数命令`gcc`编译器要为每一个产生的目标文件创建一个依赖文件，依赖文件定义了一个特定源文件的所有依赖关系，这些依赖关系通常由一系列被包含进来的头文件组成。我们需要包含所有产生的依赖文件，这样make才能知道当一个头文件改变时，哪些东西需要被重新编译。  
```
$(ARMGNU)-ld -T $(SRC_DIR)/linker.ld -o kernel8.elf  $(OBJ_FILES)  
```  
我们使用`OBJ_FILES`列表去构建`kernel8.elf`文件。我们用链接器脚本`src/linker.ld`来定义将要生成的可执行镜像的基本布局（我们将在下一节讨论链接器脚本）。  
```  
$(ARMGNU)-objcopy kernel8.elf -O binary kernel8.img  
```  
`kernel8.elf`是一个[ELF](https://baike.baidu.com/item/ELF/7120560?fr=aladdin)格式的文件。但现在我们面临的问题是，ELF是一种被操作系统执行的文件，而我们现在是在裸机上写程序，则需要将所有的可执行的指令和数据区域从ELF文件中提取出来，然后把它们放入到`kernel8.img`镜像文件中。结尾的那个`8`用于64位的ARMv8架构，这个文件名告诉我们树莓派的固件，以64位模式去启动处理器。你也可以通过使用`config.txt`文件中的`arm_control=0x200`标志去启动CPU的64位模式，RPi OS原先就是采用这种方法的，后面练习中的一些题目也会涉及到。然而，`arm_control`标志的使用并没有在官方文档中，所以使用`kernel8.img`这样的惯用名字是一种更好的做法。  
  
## 链接器脚本
链接器脚本的最初目的是描述输入文件（`_c.o`和`_s.o`）中的各个节（section）是如何被映射到输出文件（`.elf`）的。想要了解更多关于链接器脚本的知识，可以看[这里](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)。现在，让我们来看看RPi OS的链接器脚本：  
```
SECTIONS
{
    .text.boot : { *(.text.boot) }
    .text :  { *(.text) }
    .rodata : { *(.rodata) }
    .data : { *(.data) }
    . = ALIGN(0x8);
    bss_begin = .;
    .bss : { *(.bss*) } 
    bss_end = .;
}
```
启动以后，树莓派就会把`kernel8.img`加载到内存中，然后从头开始执行文件，这也是`.text.boot`这个section必须被放在开头的原因。我们将要把操作系统的启动代码放到这个section中。`.text`，`.rodata`以及`.data`这几个section，分别表示内核编译后的指令序列、只读数据（read-only data）和常规数据，这些都是常规的部分，没有什么需要额外添加的。`.bss`section包含了那些应该被初始化为0的数据，将这些数据单独用一个section来存储，编译器就可以为二进制ELF节省更多的空间，因为这样只需要在ELF头部存储section的大小，而这个section本身就可以暂时不存了（因为里面都是0）。将镜像加载到内存以后，我们必须先将`.bss`section初始化为0，这也是我们必须要记录一个section的首地址和末地址的原因（即代码重的`bss_begin`和`bss_end`符号）。而且我们必须要让每个section的首地址以8的倍数对齐，如果一个section没有对齐，就没法用`str`指令在`bss`section的初始位置开始存0，因为`str`指令只能被用于按照8字节对齐的地址。  
  
## 启动内核
是时候来看看[boot.S](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/src/lesson01/src/boot.S)文件,该文件包含了内核启动代码：  
```
#include "mm.h"

.section ".text.boot"

.globl _start
_start:
    mrs    x0, mpidr_el1        
    and    x0, x0,#0xFF        // Check processor id
    cbz    x0, master        // Hang for all non-primary CPU
    b    proc_hang

proc_hang: 
    b proc_hang

master:
    adr    x0, bss_begin
    adr    x1, bss_end
    sub    x1, x1, x0
    bl     memzero

    mov    sp, #LOW_MEMORY
    bl    kernel_main
```
让我们仔细看看这个文件：  
```
.section ".text.boot"
```
首先，我们声明了`boot.S`文件中的所有内容都属于`.test..text.boot`section。之前，我们说过这个section被链接器脚本放在内核镜像的最开始位置，所以当内核被启动的时候，最先执行的是`start`函数：  
```
.globl _start
_start:
    mrs    x0, mpidr_el1        
    and    x0, x0,#0xFF        // Check processor id
    cbz    x0, master        // Hang for all non-primary CPU
    b    proc_hang
```
这个函数做的第一件事情就是检查处理器ID，树莓派3有四个核心处理器，启动开发版的时候，每个核心会开始执行同样的代码。然而，我们并不希望与四个处理器同时打交道，而是只想控制与第一个处理器，并使其它三个处理器处于无限循环中，这就是`_start`函数负责的工作，具体做法是从[mpidr_el1](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0500g/BABHBJCI.html)系统寄存器中获得处理器ID，如果当前的处理器ID是0，那么程序就转到`master`函数：  
```
master:
    adr    x0, bss_begin
    adr    x1, bss_end
    sub    x1, x1, x0
    bl     memzero
```
这里，我们通过调用`memzero`指令清空了`.bss`section，我们会在后面定义这个函数。通常，在ARMv8架构下，函数的前七个参数是通过寄存器x0-x6来传递的，这个`memzero`函数只接收两个参数：起始位置（`bss_begin`）和section的长度（`bss_end - bss_begin`）。  
```
    mov    sp, #LOW_MEMORY
    bl    kernel_main
```
将`.bss`section清零后，我们把堆栈指针初始化，然后转而执行`kernel_main`函数，树莓派将内核加载到0地址处。这也解释了为什么堆栈指针放置到足够高的地址处，堆栈无论怎么增长都保证不会覆盖内核区域。`LOW_MEMORY`被定义在mm.h文件中，且等于4MB。我们的内核堆栈不可能变得很大，而且镜像本身是非常小的，所以4MB的大小是完全足够的。  
  
对于一些不熟悉ARM汇编的人，我们来快速总结一下我们用到过的指令：  
+ [mrs](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289881374.htm)： 把值从系统寄存器移动到通用寄存器中。
+ [and](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289863017.htm)： 执行逻辑与运算。我们用这个指令获取`mpidr_el1`寄存器的最后一个字节。
+ [cbz](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289867296.htm)： 将上一个计算结果与0作比较，如果结果为真（也就是上一次运算结果为0），就跳转（ARM的专业术语叫做branch，分支）到对应的函数。  
+ [b](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289863797.htm)： 无条件跳转。
+ [adr](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289862147.htm)： 将变量的相对地址加载到目标寄存器。通过这种方式，我们加载了`.bss`区域的起始位置和结束位置。
+ [sub](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289908389.htm)： 让两个寄存器进行减法运算。  
+ [bl](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289865686.htm)： "Branch with a link"，执行无条件跳转，并将返回值的地址存储到x30（链接寄存器）。当函数执行完之后，用`ret`指令返回先前的运行地址。  
+ [mov](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289878994.htm)： 在寄存器之间移动值，或者将一个立即数移动到寄存器。  
  
这个是[ARMv8-A开发者指南](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/index.html)，是学习ARM指令集很好的资源。[这个页面](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/ch09s01s01.html)详细地列出了寄存器的惯用方法。  
  
## `kernel_main`函数
可以看到，启动代码最后还是将控制权交给了`kernel_main`函数，让我们来看看：  
```
#include "mini_uart.h"

void kernel_main(void)
{
    uart_init();
    uart_send_string("Hello, world!\r\n");

    while (1) {
        uart_send(uart_recv());
    }
}

```
这个函数是最简单的内核函数之一。它工作在`Mini UART`设备上，该设备可以在屏幕上打印内容，可以接收用户的键盘输入。内核仅仅打印了`Hello, World!`，然后进入了从用户那里读取字符输入的无限循环，并且会将读入的字符打印到屏幕上。  
  
## 树莓派设备
现在，让我们来深入了解一下树莓派。在开始之前，我推荐你们下载一份[BCM2837 ARM设备手册](https://github.com/raspberrypi/documentation/files/1888662/BCM2837-ARM-Peripherals.-.Revised.-.V2-1.pdf)。BCM2837是树莓派3B和3B+用的板子，后面我也会提到BCM2835和BCM2836，它们是更早期版本的树莓派的板子名字。  
  
在我们开始进一步的代码阅读之前，我想先分享一些关于内存映射型设备的基础概念。BCM2837是一种简单的[SOC（System on a chip）](https://en.wikipedia.org/wiki/System_on_a_chip)（中文：系统单芯片）板子，在这种板子上，所有的设备访问都是通过内存映射来完成的。树莓派3将高于`0x3F000000`的内存地址保留给外部设备，若要激活或者配置一个特定的设备，你需要在某个设备寄存器中写入数据。设备寄存器仅仅是一个32位的内存区域（这里说的寄存器跟CPU中的寄存器不是一个概念），其中每一个比特的含义请参见BCM2837 ARM设备手册。想知道为什么我们用`0x3F000000`作为起始地址的话，可以看看手册中1.2.3节关于物理地址的描述（即使整个手册中用的都是`0x7E000000`）。  
  
从`kernel_main`函数中，你可以猜到我们将要接触Mini UART设备了。UART代表[Universal asynchronous receiver-transmitter](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter)。这个设备有能力将内存映射寄存器中的值转化为一系列的高低电平，这个高低电平通过`TTL转串口线`传递到你的电脑上，然后被你的模拟器软件翻译。我们打算用Mini UART去与树莓派进行通信。如果你想了解Mini UART的细节，请翻阅`BCM2837 ARM设备手册`的第八页。  
  
一个树莓派有两个UART：Mini UART和PL011 UART。在这份教程中，我们仅仅使用前者，因此它更简单。然而，一个选做的[练习](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/docs/lesson01/exercises.md)中有关于如何使用PL011 UART的内容，如果你想了解更多关于树莓派的这些UART以及它们的区别，你可以参考[官方文档](https://www.raspberrypi.org/documentation/configuration/uart.md)  
  
另一个你需要熟悉的设备是GPIO（[General-purporse input/output](https://baike.baidu.com/item/gpio/4723219)）,GPIO
是负责控制`GPIO引脚`的，从下图可以清楚地看到它们：  
![GPIO引脚](https://github.com/Sword-holder/raspberry-pi-os-cn/tree/master/images/gpio-pins.jpg)  
可以通过对GPIO引脚的配置来使用GPIO。比如，为了能够使用Mini UART，我们可以将引脚14和引脚15设置为高电平，来激活该设备。下图阐述了GPIO引脚需要的分配：  
![GPIO引脚序号分配](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/images/gpio-numbers.png)  
  
## Mini UART的初始化
现在，让我们来看看怎么将mini UART初始化。这份代码写在[mini_uart.c](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/src/lesson01/src/mini_uart.c)中：  
```
void uart_init ( void )
{
    unsigned int selector;

    selector = get32(GPFSEL1);
    selector &= ~(7<<12);                   // clean gpio14
    selector |= 2<<12;                      // set alt5 for gpio14
    selector &= ~(7<<15);                   // clean gpio15
    selector |= 2<<15;                      // set alt5 for gpio 15
    put32(GPFSEL1,selector);

    put32(GPPUD,0);
    delay(150);
    put32(GPPUDCLK0,(1<<14)|(1<<15));
    delay(150);
    put32(GPPUDCLK0,0);

    put32(AUX_ENABLES,1);                   //Enable mini uart (this also enables access to it registers)
    put32(AUX_MU_CNTL_REG,0);               //Disable auto flow control and disable receiver and transmitter (for now)
    put32(AUX_MU_IER_REG,0);                //Disable receive and transmit interrupts
    put32(AUX_MU_LCR_REG,3);                //Enable 8 bit mode
    put32(AUX_MU_MCR_REG,0);                //Set RTS line to be always high
    put32(AUX_MU_BAUD_REG,270);             //Set baud rate to 115200

    put32(AUX_MU_CNTL_REG,3);               //Finally, enable transmitter and receiver
}
```
这里，我们使用了两个函数，分别是`put32`和`get32`。这两个函数非常简单，允许我们从一个32位寄存器去读取和写入数据。你可以在[utils.S](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/src/lesson01/src/utils.S)中看到它们的实现。`uart_init`这节课是最复杂也是最重要的一个函数，下面三节内容中，我们将围绕这个函数进行深入探索。  
  
### GPIO替代函数的选择
