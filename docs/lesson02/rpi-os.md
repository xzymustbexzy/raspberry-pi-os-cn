## 2,1：处理器初始化

本章节我们将深入探索ARM处理器，它给我们提供了一些必要的特性以便于操作系统使用。第一个特性叫做“异常等级”。

### 异常等级

每一个支持ARM .v8架构的ARM处理器都有4个异常等级，你可以你可以将异常等级（英文`Exception levels`，简称`EL`）看作处理器的执行模式，在这些执行模式下，仅有一部分的指令和寄存器是可用的。最小的特权异常等级是0级，当处理器运行在这个等级时，它几乎只能使用通用寄存器（X0 - X30）和栈指针寄存器（SP）。EL0上运行的用户程序也允许使用`STR`和`LDR`命令来从内存加载数据或将数据存储在内存，除此之外只能使用其它一小部分指令。

操作系统必须关注异常等级，因为它需要将进程隔离，也就是说一个用户进程不应该访问其它用户进程的数据。为了实现该功能，操作系统应该将用户程序始终运行在EL0上，在该异常等级上，进程只能使用虚拟内存并且不能使用任何修改虚拟内存设置的指令。因此，为了实现进程的隔离性，操作系统必须为每个进程进行虚拟内存映射，因而要在用户进程运行前将其放置于EL0模式上运行。

操作系统本身通常运行在EL1上，在这个异常等级下，处理器能够访问用于设置虚拟内存的寄存器和一些系统寄存器。所以树莓派OS也将运行于EL1上。

我们将不使用EL2和EL3，但是我想简单介绍一下他们，这样当你需要用到的时候就会想到他们。

EL2通常使用在被用于虚拟机监视器。在这种情景下，主机操作系统运行于EL2，访客账户运行于EL1，这就使得主机操作系统能够隔离不同的访客账户，正如同操作系统隔离用户进程一样。

EL3用于将ARM从“安全世界”转换到“不安全世界”。这一层抽象是为不同软件提供完全不同的“世界”。“不安全世界”中的应用无法访问、修改属于“安全世界”的信息，这种限制是在硬件层面上保证的。

### 内核中的调试

下一步，我想了解当前运行的异常等级。但当我试图做这一步的时候，我意识到内核只能把字符串常量打印在屏幕上，而我想要的却是类似于[printf](https://en.wikipedia.org/wiki/Printf_format_string)函数的东西。通过`printf`我能轻松地在屏幕上显示一些寄存器和变量的值。这样的功能在内核开发的时候很有用，因为你没有任何调试器的支持，你只能通过`printf`来了解到程序内部发生了什么。

对于树莓派OS，我决定不再重复造轮子，所以我用了一个[现有的实现](http://www.sparetimelabs.com/tinyprintf/tinyprintf.php)。这个函数内部几乎都在做字符串操作，从内核开发者的视角看不是那么有趣。我使用的这份实现是简单版的，没有其它的外部依赖，它能够轻松地集成到内核中来。我要做的唯一的事情，就是定义`putc`函数，它是用来向屏幕中发送一个字符。这个函数的实现在[这里](../../src/lesson02/mini_uart.c#L59)可以看到，它复用了之前写的`uart_send`函数。同时，我们需要初始化`printf`库，并且指定`putc`函数的地址，这对应的代码是[这一行](../../src/lesson02/src.kernel.c#L8)。


### 获取当前异常等级

现在有了`printf`函数，我们就能实现我们一开始的目标了：搞清楚操作系统启动之后是运行在哪个异常等级上。[一个简短的函数](../../src/lesson02/src/utils.S#L1)就能回答我们的问题。

```
.globl get_el
get_el:
    mrs x0, CurrentEL
    lsr x0, x0, #2
    ret
```


这里，我们使用了`mrs`指令把系统寄存器`CurrentEL`的数据读取到`x0`寄存器，接着将这个值左移两个字节（因为`CurrentEL`寄存器的左边两位是保留的，始终是0），最后`x0`寄存器上的值就是我们想要得到的异常等级。剩下要做的，就是像[这样](../../src/lesson02/src/kernel.c#L10)显示这个值。

```
    int el = get_el();
    printf("Exception level: %d \r\n", el);
```

如果你成功复现了这个实验，你应该看到屏幕上显示了`Exception level: 3`的字样。

### 改变当前的异常等级

在ARM架构中，没有任何程序能够主动提高异常等级，除非有高等级的程序予以帮助。这是非常好的特性，否则，任何程序都可以逃离现在的异常等级，从而访问其它进程的数据。当前异常等级只能通过触发异常来改变，这种情况发生在当一个程序执行了一些非法指令时（比如试图访问一个不存在的地址或除法运算时除数为0），也可能发生在执行`svc`指令来故意制造异常时。由硬件产生的异常也被处理为一种特殊类型的异常。不论何时，当一个异常产生时，一系列动作将被执行。（在下面的描述中，我假定处理的异常等级为`n`，`n`可以是1、2或3）

1. 当前指令的地址将被存储在`ELR_ELn`寄存器中。（该寄存器又称`异常链接寄存器`）
1. 当前处理器的状态被存储在`SPSR_ELn`寄存器中。（`程序状态寄存器`）
1. 一个专门的异常处理程序会被用于处理异常，它会做它该做的事。
1. 异常处理程序会调用`eret`指令。该指令将根据`SPSR_ELn`恢复处理器状态，并且从`ELR_ELn`寄存器保存的地址开始执行程序。

但实际情况会更加复杂一些，因为异常处理程序也要保存所有通用寄存器的状态，并且在最后将它们还原，我们将在下一个章节仔细讨论。现在，我们只需要理解和记住`ELR_ELm`和`SPSR_ELn`这两个寄存器就行了。

一个重要的事情是异常处理程序不负责返回到异常的触发点，`ELR_ELm`和`SPSR_ELn`都是可写的，如果需要的话，异常处理程序可以修改它们的值。我们将使用该技术以便于将程序的异常等级从EL3切换到EL1。

### 切换到EL1

严格地讲，我们的操作系统不一定要在EL1上运行，但是EL1是一个很自然的选择，因为这个等级可以满足操作系统上所有任务所需要的特权级。了解异常等级到底是怎么切换的是一个很有趣的练习，让我们看一下[这部分](../../src/lesson02/src/boot.S#L17)代码。

```
master:
    ldr    x0, =SCTLR_VALUE_MMU_DISABLED
    msr    sctlr_el1, x0        

    ldr    x0, =HCR_VALUE
    msr    hcr_el2, x0

    ldr    x0, =SCR_VALUE
    msr    scr_el3, x0

    ldr    x0, =SPSR_VALUE
    msr    spsr_el3, x0

    adr    x0, el1_entry        
    msr    elr_el3, x0

    eret                
```

可以看到，这段代码绝大部分内容就是在设置一些系统寄存器。现在我们逐个看一下这些寄存器是用来干什么的。首先，我们需要下载[Arm64位架构参考手册](https://developer.arm.com/docs/ddi0487/ca/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile)。这份文档包含了`ARM.v8`的详细规范。

#### SCTLR_EL1, System Control Register (EL1), Page 2654 of AArch64-Reference-Manual.

```
    ldr    x0, =SCTLR_VALUE_MMU_DISABLED
    msr    sctlr_el1, x0        
```

Here we set the value of the `sctlr_el1` system register. `sctlr_el1` is responsible for configuring different parameters of the processor, when it operates at EL1. For example, it controls whether the cache is enabled and, what is most important for us, whether the MMU (Memory Management Unit) is turned on. `sctlr_el1` is accessible from all exception levels higher or equal than EL1 (you can infer this from `_el1` postfix) 

`SCTLR_VALUE_MMU_DISABLED` constant is defined [here](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L16) Individual bits of this value are defined like this:

* `#define SCTLR_RESERVED                  (3 << 28) | (3 << 22) | (1 << 20) | (1 << 11)` Some bits in the description of `sctlr_el1` register are marked as `RES1`. Those bits are reserved for future usage and should be initialized with `1`.
* `#define SCTLR_EE_LITTLE_ENDIAN          (0 << 25)` Exception [Endianness](https://en.wikipedia.org/wiki/Endianness). This field controls endianess of explicit data access at EL1. We are going to configure the processor to work only with `little-endian` format.
* `#define SCTLR_EOE_LITTLE_ENDIAN         (0 << 24)` Similar to previous field but this one controls endianess of explicit data access at EL0, instead of EL1. 
* `#define SCTLR_I_CACHE_DISABLED          (0 << 12)` Disable instruction cache. We are going to disable all caches for simplicity. You can find more information about data and instruction caches [here](https://stackoverflow.com/questions/22394750/what-is-meant-by-data-cache-and-instruction-cache).
* `#define SCTLR_D_CACHE_DISABLED          (0 << 2)` Disable data cache.
* `#define SCTLR_MMU_DISABLED              (0 << 0)` Disable MMU. MMU must be disabled until the lesson 6, where we are going to prepare page tables and start working with virtual memory.

#### HCR_EL2, Hypervisor Configuration Register (EL2), Page 2487 of AArch64-Reference-Manual. 

```
    ldr    x0, =HCR_VALUE
    msr    hcr_el2, x0
```

We are not going to implement our own [hypervisor](https://en.wikipedia.org/wiki/Hypervisor). Stil we need to use this register because, among other settings, it controls the execution state at EL1. Execution state must be `AArch64` and not `AArch32`. This is configured [here](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L22).

#### SCR_EL3, Secure Configuration Register (EL3), Page 2648 of AArch64-Reference-Manual.

```
    ldr    x0, =SCR_VALUE
    msr    scr_el3, x0
```

This register is responsible for configuring security settings. For example, it controls whether all lower levels are executed in "secure" or "nonsecure" state. It also controls execution state at EL2. [here](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L26) we set that EL2  will execute at `AArch64` state, and all lower exception levels will be "non secure". 

#### SPSR_EL3, Saved Program Status Register (EL3), Page 389 of AArch64-Reference-Manual.

```
    ldr    x0, =SPSR_VALUE
    msr    spsr_el3, x0
```

This register should be already familiar to you - we mentioned it when discussed the process of changing exception levels. `spsr_el3` contains processor state, that will be restored after we execute `eret` instruction.
It is worth saying a few words explaining what processor state is. Processor state includes the following information:

* **Condition Flags** Those flags contains information about previously executed operation: whether the result was negative (N flag), zero (A flag), has unsigned overflow (C flag) or has signed overflow (V flag). Values of those flags can be used in conditional branch instructions. For example, `b.eq` instruction will jump to the provided label only if the result of the last comparison operation is equal to 0. The processor checks this by testing whether Z flag is set to 1.

* **Interrupt disable bits** Those bits allows to enable/disable different types of interrupts.

* Some other information, required to fully restore the processor execution state after an exception is handled.

Usually `spsr_el3` is saved automatically when an exception is taken to EL3. However this register is writable, so we take advantage of this fact and manually prepare processor state. `SPSR_VALUE` is prepared [here](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/src/lesson02/include/arm/sysregs.h#L35) and we initialize the following fields:

* `#define SPSR_MASK_ALL        (7 << 6)` After we change EL to EL1 all types of interrupts will be masked (or disabled, which is the same).
* `#define SPSR_EL1h        (5 << 0)` At EL1 we can either use our own dedicated stack pointer or use EL0 stack pointer. `EL1h` mode means that we are using EL1 dedicated stack pointer. 

#### ELR_EL3, Exception Link Register (EL3), Page 351 of AArch64-Reference-Manual.

```
    adr    x0, el1_entry        
    msr    elr_el3, x0

    eret                
```

`elr_el3` holds the address, to which we are going to return after `eret` instruction will be executed. Here we set this address to the location of `el1_entry` label.

### Conclusion

That is pretty much it: when we enter `el1_entry` function the execution should be already at EL1 mode. Go ahead and try it out! 

##### Previous Page

1.5 [Kernel Initialization: Exercises](../../docs/lesson01/exercises.md)

##### Next Page

2.2 [Processor initialization: Linux](../../docs/lesson02/linux.md)
