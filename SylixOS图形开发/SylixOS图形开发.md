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

## LLVM

### LLVM编译器框架

LLVM编译器的大致组成如下：

![LLVM](./Image/LLVM_1.png)

- 不同语言编写的源文件被翻译成未被优化的LLVM IR
- LLVM IR经过优化生成优化后的LLVM IR
- 优化后的LLVM IR再翻译成具体的平台机器码

这里介绍一下IR是什么，IR全称为Intermediate Representation，指的是编译器对于源程序进行扫描后生成的内部表示，代表源程序的语义和语法结构，编译器的各个阶段都在IR上进行分析或优化变换，它对编译器的整体结构、效率和健壮性都有着极大的影响，并对编译器的可移植性以及代码生成起到关键作用。

不同的编译器的实现使用了不同的IR，并且可能使用不止一种IR。比如LLVM使用的是LLVM IR，GCC使用了AST/GENERIC、GIMPLE、RTL三种IR。

### LLVM编译器总结

我们这里不考虑交叉编译，仅讨论在x86平台的编译情况。以C语言为例，C语言程序的编译器运行的平台为x86 cpu，编译后的C语言程序的执行平台是x86 cpu，编译器本身的运行方式是作为另外的可执行程序运行。总结以上三点是为了方便和后面文章中的GLSL语言还有OpenCL C语言做对比。如下所示:

| 编译器名字         | LLVM CLANG                 |
| ------------------ | -------------------------- |
| 编译器作用         | 将C语言翻译成x86 cpu机器码 |
| 编译器运行平台     | x86 cpu                    |
| 编译后程序运行平台 | x86 cpu                    |
| 编译器运行方式     | 可执行程序                 |

## OpenGL

### OpenGL渲染管线

管线的英文名叫pipeline，其实翻译成流水线更贴切形象一点，渲染管线就是图形图像从数据一步一步形成最终输出的画面所要经历的各种操作过程。数据经过一个操作后，被处理成下一个步骤需要的数据，最终一步一步整合成拼凑最终画面的元素。通常来讲，以下两个大步骤是必要的：

- 顶点渲染：用于渲染出形状
- 像素渲染：在形状中填充色彩

所以可以简单地认为，渲染管线就是：

![浅谈编译(二)](./Image/OpenGL_1.png)

顶点渲染和像素渲染是图形渲染流水线中最重要的两个步骤，这两步发生在GPU硬件上，本质上就是GPU硬件在执行GPU硬件指令的操作。

类似于C语言的作用，为了对不同的GPU硬件提供统一的顶点渲染和像素渲染编程方式，OpenGL的维护组织Khronos Group制定了GLSL语言规范。GLSL全称为OpenGL Shading Language，针对顶点渲染步骤使用GLSL编写的源程序叫做Vertex Shader（顶点着色器），针对像素渲染步骤使用GLSL编写的源程序叫做Pixel Shader（像素着色器）。

### Shader

通过一个简单的示例来直观的感受下如何在OpenGL程序中使用Shader：

```
///
// Initialize the shader and program object
//
int Init ( void)
{
   char vShaderStr[] =
      "#version 300 es                          \n"
      "layout(location = 0) in vec4 vPosition;  \n"
      "void main()                              \n"
      "{                                        \n"
      "   gl_Position = vPosition;              \n"
      "}                                        \n";

   char fShaderStr[] =
      "#version 300 es                              \n"
      "precision mediump float;                     \n"
      "out vec4 fragColor;                          \n"
      "void main()                                  \n"
      "{                                            \n"
      "   fragColor = vec4 ( 1.0, 0.0, 0.0, 1.0 );  \n"
      "}                                            \n";

   GLuint vertexShader;
   GLuint fragmentShader;
   GLuint programObject;

   // Load the vertex/fragment shaders
   vertexShader = LoadShader ( GL_VERTEX_SHADER, vShaderStr );
   fragmentShader = LoadShader ( GL_FRAGMENT_SHADER, fShaderStr );

   // Create the program object
   programObject = glCreateProgram ( );
   
   glAttachShader ( programObject, vertexShader );
   glAttachShader ( programObject, fragmentShader );

   // Link the program
   glLinkProgram ( programObject );
   
   // Use the program object
   glUseProgram ( programObject );
}
```

假设以上的代码截取自demo.c源文件，对这个示例代码的说明如下:

- vShaderStr字符串表示的就是顶点着色器源码，fShaderStr字符串表示的就是像素着色器源码
- GLSL语言类似于C语言，每个着色器都必须包含main函数
- 通过自定义的函数LoadShader 加载顶点着色器和像素着色器，在本示例中，着色器是直接以字符串源码形式在demo.c中提供，也可以将着色器写在单独的文件中，然后在demo.c中通过打开文件->读取文件中内容->然后加载着色器来实现
- 着色器加载好之后需要进行编译，使用OpenGL接口glLinkProgram 来实现这一目的

可以看出，GLSL程序的编译类似于C语言程序的编译，只不过C语言程序的编译是通过可执行编译器程序实现，而GLSL程序的编译通过调用一系列的OpenGL接口来实现。

### OpenGL驱动

OpenGL规范其实包含两个部分：

- OpenGL API规范，使用OpenGL API编写的程序叫做OpenGL程序，比如上面的示例demo.c，是在CPU上运行的
- GLSL规范，符合GLSL规范的程序叫做Shader程序，分为顶点Shader和像素Shader，是在GPU上运行的

OpenGL的驱动主要就有两大功能：

- 实现OpenGL API
- 实现GLSL编译器，用于将GLSL程序编译成具体的GPU硬件指令

如下图所示：

![浅谈编译(二)](./Image/OpenGL_2.png)

GLSL编译器一般在OpenGL驱动中以动态库形式提供，比如对于vivante公司实现的OpenGL驱动中的libGLSLC.so就是GLSL编译器。可以看出，GLSL程序的编译是在OpenGL程序运行时编译，从某种角度来说，GLSL程序的编译属于跨平台编译，因为编译发生在CPU平台，运行发生在GPU平台。

### GLSL编译器

类似于LLVM编译器，GLSL编译器编译流程也大致分为三步，如下：

![浅谈编译(二)](./Image/OpenGL_3.png)

- GLSL程序被翻译成某种IR
- 对IR进行优化
- 将优化后的IR翻译成具体的GPU硬件指令

不同的GLSL编译器会使用不同的IR，我们这里介绍Mesa中的GLSL编译器的基本组成。

由于历史的原因，Mesa的GLSL针对不同的GPU使用不同的IR来优化，这里以AMD的GPU为例来说明，如下图所示：

![浅谈编译(二)](./Image/OpenGL_4.png)

- GLSL程序首先翻译成GLSL IR
- GLSL IR又翻译成TGSI IR
- TGSI IR翻译成LLVM IR
- LLVM IR最终翻译成AMD GPU的指令，AMD GPU指令使用的是GCN架构

SPIR-V是Vulkan和OpenCL中使用的IR，在后面会介绍。

## Vulkan

### Vulkan简介

Vulkan是Khronos组织制定的“下一代”开放的图形显示API，是与DirectX12能够匹敌的GPU API标准。

这里我们仅仅讨论Vulkan着色器相关内容，与OpenGL的着色器需要使用GLSL语言编写不同的是，Vulkan着色器必须以二进制字节码的格式使用，而不是像GLSL这样具有比较好的可读性的语法。此字节格式称为SPIR-V，它可以与Vulkan和OpenCL一同使用。这是一种可以编写图形和计算着色器的格式。

使用二进制字节码格式的优点之一是使得GPU厂商编写将着色器代码转换为本地代码的编译器复杂度减少了很多。经验表明使用可读性比较强的语法，比如GLSL，一些GPU厂商能相当灵活地理解这个标准。这会导致一种情况发生，比如编写好，并在一个厂商的GPU运行的不错的着色器程序，可能在其他的GPU厂商的GPU驱动程序上运行异常，可能是语法的问题，或者更糟的是不同GPU厂商编写的编译器差异，导致着色器运行错误。如果直接使用编译好的二进制字节码格式，可以避免这种情况。

但是，并不意味着我们要手写字节码。Khronos发布了与厂商无关的编译器，它将GLSL编译成SPIR-V。该编译器用于验证着色器代码是否符合标准，并生成与Vulkan功能运行的SPRIR-V二进制文件。除此之外还可以将此编译器作为库在运行时编译生成SPRI-V，安卓平台就是如此使用。

### Vulkan中编译框架

由于Vulkan规范中规定了Vulkan接口只能使用SPIR-V IR，所以Vulkan驱动省去了从GLSL语言编译成SPIR-V IR这一编译前端的工作，而只需要实现SPIR-V IR的优化和SPIR-V IR转换成具体的GPU硬件指令这两步骤，如下所示：

![浅谈编译(三)](./Image/Vulkan_1.png)

下面这张图能更详细的展示这种关系：

![浅谈编译(三)](./Image/Vulkan_2.png)

在实际的Vulkan驱动中，可能会将SPIR-V IR进一步转换成其他IR来进行优化，比如在Mesa的AMD GPU的Vulkan驱动中就将SPIR-V转换成LLVM IR进行优化，如下所示：

![浅谈编译(二)](./Image/OpenGL_4.png)

- 首先将SPIR-V IR转换成LLPC IR
- 接着将LLPC IR转换成LLVM IR
- 最后将LLVM IR转换成GCN指令

## OpenCL

### OpenCL简介

OpenCL 是由 Khronos Group 针对异构计算设备（heterogeneous device）进行并行运算所设计的标准 API 以及程式语言。

OpenCL 程序分成两部分：一部分是在设备上执行的（例如 GPU），另一部分是在主机上（CPU）运行的。在设备上执行的程序就是实现“异构”和“并行计算”的部分。 为了能在设备上执行代码，程序员需要写一个特殊的程序（kernel 程序）。这个程序需要使用 OpenCL 语言编写。 OpenCL 语言采用了 C 语言的一部分加上一些约束、关键字和数据类型。在主机上运行的程序用 OpenCL 的 API 管理设备上运行的程序。主机程序的 API 用 C 语言编写，也有 C++、 Java、 Python 等高级语言接口。

设计OpenCL的目的主要有两个：

- 并行计算
- 同一份代码在不同的平台（异构）都能执行

OpenCL的根本目的就是针对并行计算提供一个统一的编程方式，从而方便对使用不同类型的并行计算硬件进行编程，这些硬件设备有多核CPU、GPU、FPGA、DSP、其他形式硬件等。如下所示：

![浅谈编译(四)](./Image/OpenCL_1.png)

### Kernel程序

在OpenGL规范中，针对在GPU硬件上的编程提出了GLSL语言规范，同样的，在OpenCL规范中，针对在不同的并行计算设备上编程提出了“OpenCL C”语言规范。此语言设计参考了C语言，使用“OpenCL C”语言写的程序叫做kernel程序。通过一个简单的示例来直观的感受下如何在OpenCL程序中使用kernel程序。

```
void Init (void)
{
    char *kernelSource =
        "__kernel void test(__global int *pInOut)\n"
        "{\n"
        " int index = get_global_id(0);\n"
        " pInOut[index] += pInOut[index];\n"
        "}\n";
   
    cl_program program = NULL;
   
    // create program
    program = clCreateProgramWithSource(context, 1, (const char**)&kernelSource, NULL, &ret);

    // build program
    clBuildProgram(program, 1, &device_id, NULL, NULL, NULL);
}
```

假设以上的代码截取自demo.c源文件，对这个示例代码的说明如下:

- kernelSource字符串表示的就是OpenCL kernel程序源码
- 通过OpenCL接口clCreateProgramWithSource来加载kernel源码
- kernel源码加载好之后需要进行编译，使用OpenCL接口clBuildProgram来实现这一目的

### OpenCL驱动

OpenCL规范其实包含两个部分：

- OpenCL API规范，使用OpenCL API编写的程序叫做OpenCL程序，比如上面的示例demo.c，是在CPU上运行的
- OpenCL C规范，符合OpenCL C规范的程序叫做kernel程序，是在并行计算设备上运行的

OpenCL的驱动主要就有两大功能：

- 实现OpenCL API
- 实现OpenCL C编译器，用于将kernel程序编译成具体的并行计算设备硬件指令

如下图所示：

![浅谈编译(四)](./Image/OpenCL_2.png)

### OpenCL编译器

这里主要讨论OpenCL 2.1版本之后的编译框架。OpenCL 2.1相比于之前的版本增加了两大功能：

- 制定了OpenCL C++语言规范，之前的Kernel程序都是用OpenCL C语言编写的，现在可以使用OpenCL C++语言编写
- OpenCL接口可以直接识别加载SPIR-V IR，然后编译成kernel程序在并行计算设备上执行

详细的框架如下所示：

![浅谈编译(四)](./Image/OpenCL_3.png)

从上图中可以看出以下几点信息：

- 用OpenCL C++语言编写的kernel程序必须编译成SPIR-V格式的IR给OpenCL接口使用
- 用OpenCL C语言编写的kernel程序可以直接传给OpenCL接口使用（主要是为了兼容考虑），也可以编译成SPIR-V格式的IR给OpenCL接口使用
- OpenCL 2.1版本的驱动能处理OpenCL C和SPIR-V格式的kernel程序

从OpenCL 2.1版本的编译框架改动来看，Khronos组织希望使用SPIR-V这一IR来统一Vulkan和OpenCL的中间表示语言。Khronos提供了一个名为SPIR-V generator/Clang的基于LLVM的编译器，用于将用OpenCL C和OpenCL C++语言编写的kernel源码编译成SPIR-V格式的二进制文件。

## Wayland窗口系统