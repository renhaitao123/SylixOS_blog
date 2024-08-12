[TOC]



# SylixOS图形开发

## Vivante GCXX系列GPU驱动工作原理简介

### FrameBuffer简介

SylixOS下将显示设备抽象为一个文件，一般为/dev/fb0、/dev/fb1等，通过对fb设备文件的操作就可以直接操作显示设备。本质就是读写显示设备的“显示区域”，也就是一块内存区域，只要将要在屏幕上显示的数据准备好，然后填入这块区域，显示控制器就能在屏幕上显示图像，这块“显示区域”就叫做FramBuffer。如下图所示：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_1.png)

假设屏幕的分辨率为1024*600，单个像素的颜色数据用RGBA表示，也就是一个像素颜色用4字节表示，那么这个屏幕的FramBuffer大小为1024*600*4字节，也就是2400KB。

### Vivante GPU渲染原理简介

- 将GPU需要计算的原始数据准备好
- 启动GPU计算
- 得到计算结果，这个结果就是一帧屏幕图像数据
- 将第3步的得到的数据拷贝到FramBuffer，这样GPU渲染的一帧图像就在显示设备上显示出来了

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_2.png)

### Vivante GPU驱动工作原理

#### Contex Buffer

GPU在启动渲染前需要设置其状态，也就是渲染环境，这些设置命令专门存放buffer中，这个buffer叫做contex buffer。如果GPU的渲染环境需要改变，就需要重新填写contex buffer中的内容。

#### Command Buffer

在设置好contex buffer后，就可以让GPU做渲染（计算）工作了，这些渲染指令也被放在专门的buffer中，这个buffer叫做command buffer。

#### Command queue

GPU除了设置状态和渲染指令外，还有一些控制类指令，类似于CPU的CALL、JMP等，通过这些控制指令来控制contex buffer和command buffer，这些指令同样被放在专门的buffer中，这个buffer叫做command queue。

#### GPU的event同步机制

当GPU的渲染工作完成后，会执行一条event指令，这条event指令是事先被填在command queue中的。event指令可以表示0~29共30个event，当执行event指令后，GPU会产生中断，中断程序通过读取GPU的中断状态寄存器，就可以知道发生的是哪个event的中断，从而执行相应的处理程序。

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_3.png)

如上图所示，GPU从command queue开始执行指令，link指令相当于一条跳转指令，跳到contex buffer中执行设置GPU环境的指令，设置好环境后接着又通过link指令跳转到command buffer中执行渲染指令，渲染指令执行完后再次通过link指令跳回command queue，然后执行event命令，GPU产生中断，CPU处理中断相关内容。最后的link/wait指令让GPU处于等待状态，直到有新的指令可以执行。

### 驱动中event管理简介

#### 空闲event控制块链表

每个event信息都是存放在一个event控制块中的，驱动初始化的时候会生成固定个数的event控制块，然后将这些event控制块构造成一个单向链表。如果有event信息产生，就从链表中取出一个控制块，将event信息填写进去。如下图所示，一个这样的控制块就叫做一个record。

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_4.png)

#### Event queue

每个record都有一个类型，相同类型的record会被链在一起，然后通过一个event queue来管理。驱动里共维护了30个这样的event queue，正好和GPU的event指令表示的范围相吻合。如下图：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_5.png)

### Command queue执行流程分析

#### command queue初始化

初始化很简单，就是在command queue中添加了wait/link指令。wait指令的作用是让GPU空闲n个周期，link指令指向的是command queue中wait指令的地址，这样就形成了一个“闭环”，GPU就不会跑飞了，如下图所示：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_6.png)

#### 在context buffer中添加link指令

context buffer中的link指令指向command buffer首地址：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_7.png)

#### 再次添加wait/link指令

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_8.png)

#### 在command buffer中添加link指令

command buffer中的link指向command queue中的wait指令地址：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_9.png)

#### 将command queue中此时第一个wait指令改成link指令

command queue中改完后的link指令指向context buffer首地址，从此刻开始，GPU就跳转到context buffer中运行了：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_10.png)

#### 在command queue中添加event指令

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_11.png)

#### 再次添加wait/link指令

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_12.png)

#### 将此时command queue中的第一个wait指令改成link指令

command queue中改完后的link指向command queue中的event指令地址：

![Vivante GCXX系列GPU驱动工作原理简介](./Image/GPU驱动工作原理简介_13.png)