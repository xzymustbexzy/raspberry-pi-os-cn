# 准备工作
## 1.[树莓派3B](https://uland.taobao.com/sem/tbsearch?refpid=mm_26632258_3504122_32538762&keyword=%E6%A0%91%E8%8E%93%E6%B4%BE3b&clk1=d3e729957468c523d3c0e60467960951&upsid=d3e729957468c523d3c0e60467960951)
这个操作系统无法在更早版本的树莓派上运行，因为之前的树莓派不支持64位系统和ARMv8架构，所以请准备好树莓派3B，至于3B+应该也可以，尽管我没试过。  
## 2.[USB转TTL串口线](https://s.taobao.com/search?q=usb%E8%BD%ACttl&imgfile=&commend=all&ssid=s5-e&search_type=item&sourceId=tb.index&spm=a21bo.2017.201856-taobao-item.1&ie=utf8&initiative_id=tbindexz_20170306)
拿到串口线之后，你需要先进行测试，看看它是否能连通。如果你之前从来没用过串口，我建议你看看这个[教程](https://cdn-learn.adafruit.com/downloads/pdf/adafruits-raspberry-pi-lesson-5-using-a-console-cable.pdf)，它非常详细地教你如何用串口连接到你的树莓派。  
  
这份教程会告诉你如何用串口线给你的树莓派供电，这样你的树莓派能够正常启动，在此之后你需要启动电脑上的终端。  
  
## 3.一张SD卡
SD卡可以是空的，也可以是安装了[Raspbian OS](https://www.raspberrypi.org/downloads/raspbian/)的。主要是需要SD卡上的启动程序来引导操作系统。  

## 4.Docker
严格地说，Docker并不是硬性要求，安装它仅仅是为了方便。Docker可以在编译时屏蔽操作系统的差异，尤其是对于Windows和Mac用户来说，可以非常轻松地编译源文件。每一节课都有`build.sh`脚本（在Windows上是build.bat），这个脚本用于在Docker编译源码。安装Docker的教程请参见[Docker官方文档](https://docs.docker.com/install/)。  
  
如果你不想用Docker，你可以安装[make工具](http://www.math.tau.ac.il/~danha/courses/software1/make-intro.html)以及`aarch64-linux-gnu`工具链，如果你用的是Ubuntu系统，你需要安装`gcc-aarch64-linux-gnu`和`build-essential packages`包。  
  
#### 上一章节
[介绍](Introduction.md)
#### 下一章节
[欢迎来到RPi OS的世界，裸机上的"Hello, World"](lesson01/rpi-os.md)