# 1.1：欢迎来到RPi OS的世界，裸机上的"Hello, World"
我们即将完成一个在裸机上输出"Hello World"的程序，以此作为操作系统开发之旅的开端。在此之前，我默认你已经阅读了[准备工作](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/docs/Prerequisites.md)，准备好了这上面所说的工具，如果还没有，请先去完成。  
  
在我往下讲之前，我想先把教程的层次结构规范化。从README文件中你可以看到整个教程被分为多节**课程**，每节课程都包含了很多**章节**，一章课程根据标题又被分为很多**小节**。以后，我会分别使用课程、章节、小节来指代文章的引用位置。  
  
另一个我想说明的点就是，文章中会出现很多源代码，通常我会先展示整块源码，然后再对其进行逐行讲解。  
  
## 项目结构
课程的文件结构都是一样的，你可以在[这里]()找到课程对应的源代码。先简单介绍一下文件夹的主要组成部分：  
  
1.**Makefile**： 我们使用make工具来构建内核。`make`会根据makefile的配置来执行相应的动作，所以在makefile中，我们需要给出一些如何编译与链接源代码的指令。  
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
`ARMGNU ?= aarch64-linux-gnu`  
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
我们使用`OBJ_FILES`列表去构建`kernel8.elf`文件。我们用链接脚本`src/linker.ld`来定义将要生成的可执行镜像的基本布局（我们将在下一节讨论链接脚本）。  
```  
$(ARMGNU)-objcopy kernel8.elf -O binary kernel8.img  
```  
`kernel8.elf`是一个[ELF](https://baike.baidu.com/item/ELF/7120560?fr=aladdin)格式的文件。但现在我们面临的问题是，ELF是一种被操作系统执行的文件，而我们现在是在裸机上写程序，则需要将所有的可执行的指令和数据区域从ELF文件中提取出来，然后把它们放入到`kernel8.img`镜像文件中。结尾的那个`8`用于64位的ARMv8架构，这个文件名告诉我们树莓派的固件，以64位模式去启动处理器。你也可以通过使用`config.txt`文件中的`arm_control=0x200`标志去启动CPU的64位模式，RPi OS原先就是采用这种方法的，后面练习中的一些题目也会涉及到。然而，`arm_control`标志的使用并没有在官方文档中，所以使用`kernel8.img`这样的惯用名字是一种更好的做法。  
## 链接脚本
