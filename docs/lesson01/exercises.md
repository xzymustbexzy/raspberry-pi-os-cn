## 1.2：练习

练习是一个可选项，但是我强烈建议你通过练习感受一下源代码。如果你完成了一些练习，请把你的代码分享给其他人。具体细节参见[贡献指南](../Contributions.md)

1. 引入常量`baud_rate`，用这个常数计算必要的Mini UART寄存器的值，使得程序能在除了115200的波特率下也能运行。
1. 在源码中使用UART设备而不是使用Mini UART。使用`BCM2837 ARM Peripherals` 和 [ARM PrimeCell UART (PL011)](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0183g/DDI0183G_uart_pl011_r1p5_trm.pdf)手动指明如何使用UART寄存器以及如何配置GPIO引脚。UART设备使用了48MHZ的时钟基准频率。
1. 尝试使用4个处理器核心。操作系统应该用每个核心打印出`Hello, from precessor <processor index>`，不要忘了为每个处理器核心设置独立的栈，同时确保Mini UART仅仅被初始化一次。你可以将全局变量和`delay`函数相结合。
1. 将课程1的内容运行在菜单上。参见[这里](https://github.com/s-matyukevich/raspberry-pi-os/issues/8)。

##### Previous Page

1.1 [欢迎来到RPi OS的世界，裸机上的"Hello, World"](rpi-os.md)

##### Next Page

