# raspberry-pi-os-cn
这个是开源教学版树莓派操作系统raspberry-pi-os的中文版教程。  
英文原版教程 https://github.com/s-matyukevich/raspberry-pi-os  

## 教程简介
这个教程将手把手教你如何在树莓派上编写一个简单的操作系统内核。我将这个操作系统简称为RPi OS。这个操作系统很大程度上是基于Linux内核，但功能上比Linux少了很多，而且仅支持树莓派3。  
  
这里的每一节课程中，首先会解释一些RPi OS内核特性的实现，然后会展示Linux是如何实现这些特性的。每节课的内容在src文件夹下都有对应的代码实现，这有助于更加清晰地体现教程中的概念，让读者感受到RPi OS的演进过程。这份教程针对的是对操作系统开发感兴趣的读者，并不要求读者具备任何的操作系统开发基础。  
  
# 目录
+ [介绍](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/docs/Introduction.md)
+ [准备工作](https://github.com/Sword-holder/raspberry-pi-os-cn/blob/master/docs/Prerequisites.md)
+ **Lesson 1：内核初始化**
  - 1.1 [欢迎来到RPi OS的世界，裸机上的"Hello, world"](./docs/lesson01/rpi-os.md)
  - 1.2 [练习题](./docs/lesson01/exercises.md)
+ **Lesson 2：处理器初始化**
  - 2.1 [RPi OS](./docs/lesson02/rpi-os.md)
  - 2.2 练习题
+ **Lesson 3：中断处理**
  - 3.1 RPi OS
  - Linux
    * 3.2 底层异常处理
    * 3.3 中断控制
    * 3.4 计时器
  - 3.5 练习题
+ **Lesson 4：进程调度**
  - 4.1 RPi OS
  - Linux
    * 4.2 调度器基本结构
    * 4.3 Fork一个进程
    * 4.4 调度器
  - 4.5 练习题
+ **Lesson 5：用户进程和系统调用**
  - 5.1 RPi OS
  - 5.2 Linux
  - 5.3 练习题
+ **Lesson 6：虚拟内存管理**
  - 6.1 RPi OS
  - 6.2 Linux
  - 6.3 练习题
+ **Lesson 7：信号和中断等待（未完成）**
+ **Lesson 8：文件系统（未完成）**
+ **Lesson 9：ELF可执行文件（未完成）**
+ **Lesson 10：驱动（未完成）**
+ **Lesson 11：网络（未完成）**
