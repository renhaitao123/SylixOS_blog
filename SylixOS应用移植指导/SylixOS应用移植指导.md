[TOC]



# SylixOS 应用移植指导——以Redis5.0.3为例

> 本文章致力于指导有一定SylixOS开发基础的用户进行中大型应用软件移植。
>
> 没有SylixOS应用开发基础的建议先参考IDE自带的《RealEvo-IDE使用手册.pdf》，了解SylixOS IDE下的不同的工程类型，尝试过APP与动态库编译后再读本文，会更容易上手。
>
> 推荐书籍：《程序员的自我修养--链接、装载和库》

## 总述：

SylixOS应用移植遵循的四步原则是：**搬 改 测 查**。

###  搬：

既然是移植，首先我们要确定移植的目的。本文以Redis5.0.3为例，源码来自Redis官网，源码编译环境为Ubuntu20.0，移植目标环境是SylixOS X86平台，版本是2.0.1。移植的目标是将Redis-Server、Redis-Client运行在SylixOS环境下，编译出的其他文件，可移植的就一起加入工程。

移植不是盲目的，像Redis这样的，对第三方库依赖非常少的，可行性较高。如果是Linux下依赖Linux比较核心机制的库就很难移植，比如一些依赖X11的中间件。而那些根本不开源只提供Linux下动态库的就更不用说了。

言归正传，何谓搬，搬就是希望将Linux下的整个工程源码、编译流程、生成的文件，分毫不动的在SylixOS下实现一遍。这当然是移植的最好结果，然而事实是SylixOS并不是Linux，或多或少都存在差异性，差异体现在API实现的数量、进程创建的机制、系统参数的配置等各方面。

所以，原则上应用移植需要照搬Linux工程源码，并参考Linux编译流程，搭建基于SylixOS IDE的应用工程。

### 改：

然而照搬的工程编译一般没法一次通过，甚至编译通过后，运行时会提示“can not find symbol xxx“ 的提示，这是因为SylixOS是交叉编译的，编译器并不清楚实际运行的环境中是否存在某个符号（符号可以理解为函数、变量），只会有告警（有声明不会告警），而无法判断你的应用是否正确的实现了所有需要的符号。

所以在工程编译阶段，由于操作系统的不同，需要我们修改工程源码。大部分情况下是注释掉不支持的接口，移植性较好的工程一般有config.h供用户注释相关宏定义，以做到工程裁剪或替换功能。小部分情况需要我们自行修改相关实现，以适用于SylixOS的运行环境。

当工程修改到：编译不会报错，编译完成的可执行文件（ELF）可以上传到SylixOS运行环境下，运行不会提示找不到符号表了。别太早高兴，这只说明，移植的第一步终于完成了。

### 测

既然都可以运行了，那就开测吧。一般来说，Linux的工程源码内都会配有test目录，或其他测试脚本。测的目的就是通过这些测试用例。

当然，初步移植的应用大概率是 ./xxx 运行的时候，应用直接跑飞，系统报个栈回溯。无论是应用直接跑飞、测试用例跑飞、还是测试用例表现结果与预想不一致，这都说明我们的移植还存在问题，需要开始 查。

### 查

问题排查，移植中最头疼的一部，由于问题五花八门都有可能，在后续的移植指导中会带着提到。

当然，大概率问题是出在第二步 改 那里，可以重点关注一下自己改的是否存在问题。

测和查是循环的过程，当排完所有问题，所有测试用例通过之后，可以说自己的应用移植已经完成大部分了。那还有什么问题呢？那就涉及到更高级的一些玩法，比如跨平台配置，工程的美观性配置等等。

 

接下来，就以Redis5.0.3的移植过程为例，演示SylixOS的应用移植吧。

## 前期准备

首先需要有一台Linux作为参考，SylixOS工程最终的表现形式与Linux越接近，说明移植的效果越好。

其次需要找到需要移植的应用源码，搞清它库之间的依赖关系，并确保其所有的依赖库（SylixOS下的动态库，或源码）可以获得。举个例子，我需要移植A工程，其依赖libB.so，是A工程内自带并编译的，依赖zlib，依赖libssl。由于libB.so是工程自带并有源码编译的，不需要关心，libssl是SylixOS base工程内自带的，不需要关心，只需要链接，而zlib是没有的，这时候就需要先自行下载zlib源码并编译。

像zlib、libxml2、libjson这类的库，自身无依赖，只作为其他工程依赖的库，移植都比较简单，大部分是编译过就能用的。

然后需要有SylixOS的验证环境，这里推荐使用真机，如果没有就用IDE自带的模拟器验证吧。

## Linux下编译

移植不是乱移，不是把所有.c编译一下就可以的。首先我们需要在Linux下确定需要编译的.c文件。

获得工程源码，并解压：

![Linux下编译](./Image/Linux下编译_1.png)

进入Linux工程目录，Linux下的工程一般是三步走，即./configure、make、make install ：

进目录一看，没有configure，那就看一下Makefile，发现实际调用的是src目录下的Makefile：

![Linux下编译](./Image/Linux下编译_2.png)

那就敲个make看看，发现有报错提示：

![Linux下编译](./Image/Linux下编译_3.png)

这里是因为Redis可以依赖jemalloc，而我的环境下没有jemalloc，刚好SylixOS也不支持jemalloc，百度可知 make MALLOC=libc 可以使用普通的malloc。

敲make MALLOC=libc，发现编译的时候只提示编出哪些.o，而没有完整的编译过程，这点不好，不利于我们移植，通过V=1参数让gcc打印完整过程：

![Linux下编译](./Image/Linux下编译_4.png)

![Linux下编译](./Image/Linux下编译_5.png)

等待编译中。。。

发现链接的时候报了错，是没有liblua.a，且这个库应当在deps目录下。进入deps目录，发现有一些redis的依赖库，分别是lua、hiredis、jemalloc和linenoise。lua和linenoise是必要的，hiredis是redis client的依赖库，jemalloc不是必要的。那就先编译依赖库：

![Linux下编译](./Image/Linux下编译_6.png)

再切回上一级目录继续编译redis，发现已经可以编完：

![Linux下编译](./Image/Linux下编译_7.png)

那就make install一下，看看究竟编了哪些ELF，如果不喜欢install到/usr/bin之类的目录下，可以自行指定PREFIX:

![Linux下编译](./Image/Linux下编译_8.png)

看一下build目录，编出了redis-server、redis-client以及其他附带的ELF：

![Linux下编译](./Image/Linux下编译_9.png)

这时候我们明确这次移植的流程：

首先需要编译linenoise、liblua.a、libhiredis.so。再依赖其编译出redis-server、redis-client等其他工具。并且，我们有了所有工程的编译流程，可以获得以下信息：需要编译的.c文件名、编译的参数、链接参数、库依赖关系等。这为我们的移植铺平了道路。什么？你说不知道liblua要编哪些文件？make distclean一下，再到deps目录下make lua去。

把所有的编译流程贴到notepad，准备开始下一步的移植。

## 创建IDE工程

总结一下本阶段的工作：

### **已知：**

1. Redis源码
2. Redis编译流程
3. 需要编译的依赖库
4. 需要编译出的目标文件

### **目标：**

1. 在SylixOS IDE下构建工程
2. 配置与Linux下一致的编译流程
3. 编译出所有依赖的库
4. 编译出最终的可执行文件

### **SylixOS IDE 工程**

在把Linux工程代码贴入IDE工程之前，首先需要了解一下SylixOS的工程结构和Makefile规则。

首先我们创建一个专家模式的APP工程，关于专家模式与普通模式的区别，可以参考《RealEvo-IDE使用手册.pdf》，总结一下就是：专家模式所有的配置通过修改Makefile来实现，普通模式所有的配置通过图形化勾选来实现，这两种模式不要混用，否则可能出现在Makefile里改了内容但编译一下内容消失的“惨案”。

![创建IDE工程](./Image/创建IDE工程_1.png)

刚创建的工程一般有以下几个文件：

![创建IDE工程](./Image/创建IDE工程_2.png)

简单介绍一下几个文件：

- **src目录：**

一般工程源码直接贴入此目录，默认会创建一个和工程同名的.c文件，可删除。

- **config.mk：**

指定工程依赖的BASE路径，以及工程的debug level是Debug还是Release，一般不需要改动。

- **Makefile：**

SylixOS工程的Makefile文件，里面大部分都是include各种模板，其中用户唯一需要关心的就是“Include targets makefiles”部分：

![创建IDE工程](./Image/创建IDE工程_4.png)

这里include的就是实际工程需要编译的app或者库的makefile文件了，默认只include工程同名的mk文件。而随着移植的进展，可能会有libA.mk，libB.mk、C.mk、D.mk等等，需要在此位置分别include进来，否则编译器不会编译。

- **redis.mk：**

这里是redis.mk，是因为我工程名字叫redis，所以自动生成的。这个mk下的宏分别对应的内容为：

- **LOCAL_TARGET_NAME：**

生成的目标名称，一般生成动态库A的名称格式是“LOCAL_TARGET_NAME := libA.aso”，生成静态库A名称的格式是“LOCAL_TARGET_NAME := libA.a”，生成可执行文件A的名称格式就是 “LOCAL_TARGET_NAME := A”。需要注意的是，这里的名称并**不会**让编译器按照APP或动态库的模板去编译，所以不要认为你改了target_name编译器就直接帮你编动态库了，实际是有地方指定的，下面会说。

- **LOCAL_SRCS :**

这里指定编译的所有源文件。

- **LOCAL_INC_PATH :**

这里指定编译的头文件路径，注意，有时候头文件是有先后顺序之分的，最好的办法就是直接参考Linux编译流程。

- **LOCAL_DSYMBOL :**

这里指定的是编译的宏或者符号，也就是-Da=xxx，相当于.c里写了#define a xxx。属于GCC的语法，所以一样可以参考Linux下的编译流程。

- **LOCAL_CFLAGS : LOCAL_CXXFLAGS :**

分别是gcc和g++的编译参数。

- **LOCAL_DEPEND_LIB : LOCAL_DEPEND_LIB_PATH：**

指定工程的依赖库和依赖库路径，这里的路径指的是IDE下的库路径，即Windows下的路径，不是SylixOS运行环境下的库路径。依赖SylixOS Base的libcextern库的写法为：

```
LOCAL_DEPEND_LIB      := \
-lcextern
LOCAL_DEPEND_LIB_PATH := \
-L"$(SYLIXOS_BASE_PATH)/libcextern/$(Output)"
```

- **LOCAL_USE_CXX :LOCAL_USE_CXX_EXCEPT :**

是否使能C++，是否使用C++异常库，如果是C++工程就=yes。

- **LOCAL_NO_UNDEF_SYM、LOCAL_USE_GCOV、LOCAL_USE_OMP：**

基本用不到，感兴趣可以自己研究。

- **LOCAL_PRE_LINK_CMD、LOCAL_POST_LINK_CMD、LOCAL_PRE_STRIP_CMD、LOCAL_POST_STRIP_CMD：**

对应link阶段或strip阶段执行的命令，一般用不到，巧的是redis还真用到了。

- **include $(APPLICATION_MK)**

这个很重要，一般位于.mk的最后面，指定本mk使用哪一套模板，APP就是APPLICATION_MK，动态库/静态库是LIBRARY_MK。这里指定了本工程的最终属性。

了解了IDE的工程属性以及Makefile模板规则，可以贴入代码了。先残忍了删了自带的redis.c，然后将linux下的工程源码拖到工程下，我一般推荐是configure过后的文件贴过来，而不是拿tar.gz直接解压的，因为configure一般会生成config.h，并做过第一遍的修改。当然由于redis没有configure，区别倒不是太大：

![创建IDE工程](./Image/创建IDE工程_3.png)

## 添加Makefile文件

前面我们知道了，要想编redis-server，需要先编译lua，要想编译redis-client，需要先编译linenoise、hiredis，什么，你说怎么知道的？不知道就看redis-server编译的link阶段打印，看看它链接了哪些库，比如-llua，-lhiredis。

简单分析一下，lua和hiredis是库的形式存在的，linenoise是单独编成.o链接的，那就不用管linenoise，到时候填到mk里就行，再仔细看看，lua是变成静态库，hiredis是编成动态库。

那就从lua开始，首先复制个mk，取个名叫liblua.mk。再回到Linux下，如果lua的编译流程没了，就make distclean一下，然后在deps目录下， make lua > ./lua.log，获得lua.log：

```
(cd hiredis && make clean) > /dev/null || true
(cd linenoise && make clean) > /dev/null || true
(cd lua && make clean) > /dev/null || true
(cd jemalloc && [ -f Makefile ] && make distclean) > /dev/null || true
(rm -f .make-*)
(echo "" > .make-cflags)
(echo "" > .make-ldflags)
[32;1mMAKE[0m [37;1mlua[0m
cd lua/src && make all CFLAGS="-O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC='' " MYLDFLAGS="" AR="ar rcu"
make[1]: 杩涘叆鐩綍鈥�/home/tc/workspace/redis/redis-5.0.3/deps/lua/src鈥�
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lapi.o lapi.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lcode.o lcode.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ldebug.o ldebug.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ldo.o ldo.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ldump.o ldump.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lfunc.o lfunc.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lgc.o lgc.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o llex.o llex.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lmem.o lmem.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lobject.o lobject.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lopcodes.o lopcodes.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lparser.o lparser.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lstate.o lstate.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lstring.o lstring.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ltable.o ltable.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ltm.o ltm.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lundump.o lundump.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lvm.o lvm.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lzio.o lzio.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o strbuf.o strbuf.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o fpconv.o fpconv.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lauxlib.o lauxlib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lbaselib.o lbaselib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ldblib.o ldblib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o liolib.o liolib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lmathlib.o lmathlib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o loslib.o loslib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o ltablib.o ltablib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lstrlib.o lstrlib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o loadlib.o loadlib.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o linit.o linit.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lua_cjson.o lua_cjson.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lua_struct.o lua_struct.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lua_cmsgpack.o lua_cmsgpack.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lua_bit.o lua_bit.c
ar rcu liblua.a lapi.o lcode.o ldebug.o ldo.o ldump.o lfunc.o lgc.o llex.o lmem.o lobject.o lopcodes.o lparser.o lstate.o lstring.o ltable.o ltm.o lundump.o lvm.o lzio.o strbuf.o fpconv.o lauxlib.o lbaselib.o ldblib.o liolib.o lmathlib.o loslib.o ltablib.o lstrlib.o loadlib.o linit.o lua_cjson.o lua_struct.o lua_cmsgpack.o lua_bit.o	# DLL needs all object files
ranlib liblua.a
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o lua.o lua.c
cc -o lua  lua.o liblua.a -lm 
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o luac.o luac.c
cc -O2 -Wall -DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''    -c -o print.o print.c
cc -o luac  luac.o print.o liblua.a -lm 
make[1]: 绂诲紑鐩綍鈥�/home/tc/workspace/redis/redis-5.0.3/deps/lua/src鈥�
```

多么清晰的流程，需要编译的文件就是lapi.c、lcode.c、ldebug.c。。。编译参数是-O2 -Wall，实际上SylixOS工程自带了相关参数，不重要。定义的符号是：-DLUA_ANSI -DENABLE_CJSON_GLOBAL -DREDIS_STATIC=''。

再往下看：

![添加Makefile文件](./Image/添加Makefile文件_1.png)

发现libua.a只需要ar 这些.o，而下面还编了个luac出来，对我们的redis没啥用，可以不管。

那么，我们的liblua.mk参考这些流程，应该很容易编写出来：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: liblua.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2019 年 03 月 06 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := liblua.a

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS := \
src/redis-5.0.3/deps/lua/src/lapi.c \
src/redis-5.0.3/deps/lua/src/lcode.c \
src/redis-5.0.3/deps/lua/src/ldebug.c \
src/redis-5.0.3/deps/lua/src/ldo.c \
src/redis-5.0.3/deps/lua/src/ldump.c \
src/redis-5.0.3/deps/lua/src/lfunc.c \
src/redis-5.0.3/deps/lua/src/lgc.c \
src/redis-5.0.3/deps/lua/src/llex.c \
src/redis-5.0.3/deps/lua/src/lmem.c \
src/redis-5.0.3/deps/lua/src/lobject.c \
src/redis-5.0.3/deps/lua/src/lopcodes.c \
src/redis-5.0.3/deps/lua/src/lparser.c \
src/redis-5.0.3/deps/lua/src/lstate.c \
src/redis-5.0.3/deps/lua/src/lstring.c \
src/redis-5.0.3/deps/lua/src/ltable.c \
src/redis-5.0.3/deps/lua/src/ltm.c \
src/redis-5.0.3/deps/lua/src/lundump.c \
src/redis-5.0.3/deps/lua/src/lvm.c \
src/redis-5.0.3/deps/lua/src/lzio.c \
src/redis-5.0.3/deps/lua/src/strbuf.c \
src/redis-5.0.3/deps/lua/src/fpconv.c \
src/redis-5.0.3/deps/lua/src/lauxlib.c \
src/redis-5.0.3/deps/lua/src/lbaselib.c \
src/redis-5.0.3/deps/lua/src/ldblib.c \
src/redis-5.0.3/deps/lua/src/liolib.c \
src/redis-5.0.3/deps/lua/src/lmathlib.c \
src/redis-5.0.3/deps/lua/src/loslib.c \
src/redis-5.0.3/deps/lua/src/ltablib.c \
src/redis-5.0.3/deps/lua/src/lstrlib.c \
src/redis-5.0.3/deps/lua/src/loadlib.c \
src/redis-5.0.3/deps/lua/src/linit.c \
src/redis-5.0.3/deps/lua/src/lua_cjson.c \
src/redis-5.0.3/deps/lua/src/lua_struct.c \
src/redis-5.0.3/deps/lua/src/lua_cmsgpack.c \
src/redis-5.0.3/deps/lua/src/lua_bit.c

#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := \
-I"src/lua/src"

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL := -DLUA_USE_LINUX=1

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := 
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB      := 
LOCAL_DEPEND_LIB_PATH := 

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD   := 
LOCAL_POST_LINK_CMD  := 
LOCAL_PRE_STRIP_CMD  := 
LOCAL_POST_STRIP_CMD := 

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(LIBRARY_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

同样的，我们很容易编出libhiredis.mk：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: libhiredis.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2019 年 03 月 06 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := libhiredis.so

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS := \
src/redis-5.0.3/deps/hiredis/net.c \
src/redis-5.0.3/deps/hiredis/hiredis.c \
src/redis-5.0.3/deps/hiredis/sds.c \
src/redis-5.0.3/deps/hiredis/async.c \
src/redis-5.0.3/deps/hiredis/read.c

#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := \
-I"src/redis-5.0.3/deps/hiredis"

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL := 

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := -std=c99 -pedantic -Wstrict-prototypes -Wwrite-strings -g -ggdb
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB      := 
LOCAL_DEPEND_LIB_PATH := 

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD   := 
LOCAL_POST_LINK_CMD  := 
LOCAL_PRE_STRIP_CMD  := 
LOCAL_POST_STRIP_CMD := 

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(LIBRARY_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

一定要注意.c文件的**路径**问题。

这时候，我们工程应该是这样的：

![添加Makefile文件](./Image/添加Makefile文件_2.png)

注意，别忘了把自己创建的mk加入Makfile：

![添加Makefile文件](./Image/添加Makefile文件_3.png)

尝试编译一下，两个库都编出来了：

![添加Makefile文件](./Image/添加Makefile文件_4.png)

下面要开始编redis-server和redis-client了。

## 完善Makefile

既然liblua和libhiredis都有了，那就照葫芦画瓢，把redis-cli和redis-server的Makefile构造出来。

根据make流程的log，找到redis-server依赖的所有.o：

![完善Makefile](./Image/完善Makefile_1.png)

根据log，分别获得：所有需要编译的.c文件、gcc编译参数、头文件路径、依赖的库。

这里说一下依赖的库，这里看出需要的链接的库分别是libhiredis.a、liblua.a、libm.so、libdl.so、libpthread.so、librt.so。其中hiredis和lua是我们之前本工程编译的，dl、rt是Linux特有的。m就是math库，SylixOS是默认自带，pthread在SylixOS下是内核自带的，不需要额外依赖。所以正常情况下只需要链接hiredis和lua（这里为啥说是正常情况下，后面排错的时候会发现问题）。

完成redis-server.mk的编写：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: redis-server.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2021 年 07 月 26 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := redis-server

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS := \
src/redis-5.0.3/src/adlist.c \
src/redis-5.0.3/src/quicklist.c \
src/redis-5.0.3/src/ae.c \
src/redis-5.0.3/src/anet.c \
src/redis-5.0.3/src/dict.c \
src/redis-5.0.3/src/server.c \
src/redis-5.0.3/src/sds.c \
src/redis-5.0.3/src/zmalloc.c \
src/redis-5.0.3/src/lzf_c.c \
src/redis-5.0.3/src/lzf_d.c \
src/redis-5.0.3/src/pqsort.c \
src/redis-5.0.3/src/zipmap.c \
src/redis-5.0.3/src/sha1.c \
src/redis-5.0.3/src/ziplist.c \
src/redis-5.0.3/src/release.c \
src/redis-5.0.3/src/networking.c \
src/redis-5.0.3/src/util.c \
src/redis-5.0.3/src/object.c \
src/redis-5.0.3/src/db.c \
src/redis-5.0.3/src/replication.c \
src/redis-5.0.3/src/rdb.c \
src/redis-5.0.3/src/t_string.c \
src/redis-5.0.3/src/t_list.c \
src/redis-5.0.3/src/t_set.c \
src/redis-5.0.3/src/t_zset.c \
src/redis-5.0.3/src/t_hash.c \
src/redis-5.0.3/src/config.c \
src/redis-5.0.3/src/aof.c \
src/redis-5.0.3/src/pubsub.c \
src/redis-5.0.3/src/multi.c \
src/redis-5.0.3/src/debug.c \
src/redis-5.0.3/src/sort.c \
src/redis-5.0.3/src/intset.c \
src/redis-5.0.3/src/syncio.c \
src/redis-5.0.3/src/cluster.c \
src/redis-5.0.3/src/crc16.c \
src/redis-5.0.3/src/endianconv.c \
src/redis-5.0.3/src/slowlog.c \
src/redis-5.0.3/src/scripting.c \
src/redis-5.0.3/src/bio.c \
src/redis-5.0.3/src/rio.c \
src/redis-5.0.3/src/rand.c \
src/redis-5.0.3/src/memtest.c \
src/redis-5.0.3/src/crc64.c \
src/redis-5.0.3/src/bitops.c \
src/redis-5.0.3/src/sentinel.c \
src/redis-5.0.3/src/notify.c \
src/redis-5.0.3/src/setproctitle.c \
src/redis-5.0.3/src/blocked.c \
src/redis-5.0.3/src/hyperloglog.c \
src/redis-5.0.3/src/latency.c \
src/redis-5.0.3/src/sparkline.c \
src/redis-5.0.3/src/redis-check-rdb.c \
src/redis-5.0.3/src/redis-check-aof.c \
src/redis-5.0.3/src/geo.c \
src/redis-5.0.3/src/lazyfree.c \
src/redis-5.0.3/src/module.c \
src/redis-5.0.3/src/evict.c \
src/redis-5.0.3/src/expire.c \
src/redis-5.0.3/src/geohash.c \
src/redis-5.0.3/src/geohash_helper.c \
src/redis-5.0.3/src/childinfo.c \
src/redis-5.0.3/src/defrag.c \
src/redis-5.0.3/src/siphash.c \
src/redis-5.0.3/src/rax.c \
src/redis-5.0.3/src/t_stream.c \
src/redis-5.0.3/src/listpack.c \
src/redis-5.0.3/src/localtime.c \
src/redis-5.0.3/src/lolwut.c \
src/redis-5.0.3/src/lolwut5.c


#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := \
-I"src/redis-5.0.3/src" \
-I"src/redis-5.0.3/deps/hiredis" \
-I"src/redis-5.0.3/deps/linenoise" \
-I"src/redis-5.0.3/deps/lua/src"

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL :=
LOCAL_DSYMBOL += -DREDIS_STATIC=""

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := -std=c99 -pedantic -Wno-missing-field-initializers -g -ggdb
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB      := \
-lhiredis \
-llua
LOCAL_DEPEND_LIB_PATH := \
-L"$(Output)"

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD   := 
LOCAL_POST_LINK_CMD  := 
LOCAL_PRE_STRIP_CMD  := 
LOCAL_POST_STRIP_CMD := \
cp $(Output)/redis-server $(Output)/redis-check-rdb; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-check-rdb; \
cp $(Output)/redis-server $(Output)/redis-check-aof; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-check-aof; \
cp $(Output)/redis-server $(Output)/redis-sentinel; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-sentinel

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(APPLICATION_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

这里注意一下，一般我们知道，链接库时，-l后面跟库名称，-L后面跟库路径。然而libhiredis和liblua是自己工程编译出来的，路径咋填呢，我这里是用的是$(Output)，这表示从编译的输出目录里找，如果工程是debug模式就从Debug目录下找，release模式就从Release目录下找。

![完善Makefile](./Image/完善Makefile_2.png)

将redis-server.mk加入Makefile，并放在libhiredis.mk和liblua.mk之下。尝试编译一下，发现报了些错误：

![完善Makefile](./Image/完善Makefile_3.png)

一个个看看，首先是没找着M_PI这个宏，追一下发现是在编译器的math.h里实现的，没事，我们在geohash_helper.c和server.h里定义一下：

![完善Makefile](./Image/完善Makefile_4.png)

另一个问题，发现是有个函数名称和内核的宏定义冲突了：

![完善Makefile](./Image/完善Makefile_5.png)

这个也很简单，我们把moduleLoad undef掉就行：

![完善Makefile](./Image/完善Makefile_6.png)

所有的修改都用#ifdef SYLIXOS包一下，方便以后排查，也不会影响原应用编译逻辑。

编译一下，发现编译通过了。先不着急测试，继续编redis-cli：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: redis-cli.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2021 年 07 月 26 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := redis-cli

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS := \
src/redis-5.0.3/src/anet.c \
src/redis-5.0.3/src/adlist.c \
src/redis-5.0.3/src/dict.c \
src/redis-5.0.3/src/redis-cli.c \
src/redis-5.0.3/src/zmalloc.c \
src/redis-5.0.3/src/release.c \
src/redis-5.0.3/src/ae.c \
src/redis-5.0.3/src/crc64.c \
src/redis-5.0.3/src/siphash.c \
src/redis-5.0.3/src/crc16.c \
src/redis-5.0.3/deps/linenoise/linenoise.c


#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := \
-I"src/redis-5.0.3/src" \
-I"src/redis-5.0.3/deps/hiredis" \
-I"src/redis-5.0.3/deps/linenoise" \
-I"src/redis-5.0.3/deps/lua/src"

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL :=
LOCAL_DSYMBOL += -DREDIS_STATIC=""

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := -std=c99 -pedantic -Wno-missing-field-initializers -g -ggdb
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB      := \
-lhiredis
LOCAL_DEPEND_LIB_PATH := \
-L"$(Output)"

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD   := 
LOCAL_POST_LINK_CMD  := 
LOCAL_PRE_STRIP_CMD  := 
LOCAL_POST_STRIP_CMD := \
cp $(Output)/redis-server $(Output)/redis-check-rdb; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-check-rdb; \
cp $(Output)/redis-server $(Output)/redis-check-aof; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-check-aof; \
cp $(Output)/redis-server $(Output)/redis-sentinel; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-sentinel

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(APPLICATION_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

尝试编译一下，也过了。

现在我们已经有了最重要的redis-server和redis-cli了，其他的先不用管，开始尝试运行应用吧。

## 排查符号表问题

我们现在编译已经可以通过了，但实际上APP未必能运行。这是因为SylixOS作为嵌入式操作系统，APP采用的是交叉编译的方式，即编译环境与运行环境是完全两套场景。这意味着，编译器在编译的时候，并不能确定某些符号，在运行环境中是否存在，所以不能报错，最多只会产生编译告警。而实际上只要声明了那些函数或变量，连告警都不会产生。

实际能不能运行，只能上传到SylixOS环境中验证。先把要上传的文件添加到Device Settings里：

![排查符号表问题](./Image/排查符号表问题_1.png)

因为我们的liblua都是以静态库的形式链接进app的，所以不需要再上传。将libhiredis.so上传到/lib目录下，其他传到/apps/redis下。这里我是用模拟器作为运行环境，模拟器的使用和配置方式，参考《RealEvo-Simulator使用手册.pdf》，这里就不再赘述了。

先从简单的来，尝试 ./redis-cli 运行client，发现报了一个找不到的符号表：

![排查符号表问题](./Image/排查符号表问题_2.png)

遇到这种问题，一般先搜索本工程（redis），确认一下这个符号是工程内实现的，还是base里实现的。找到使用的位置，按住ctrl+鼠标左键追踪一下，发现是base里实现的，实现的位置在libcextern的目录里，说明此符号是通过SylixOS的额外C库实现的，即函数实现在libcextern.so中，所以需要在APP的makefile里指定链接libcextern.so。

实际上redis-server里一样有这个问题，修改一下makefile：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: redis-cli.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2021 年 07 月 26 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := redis-cli

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS := \
src/redis-5.0.3/src/anet.c \
src/redis-5.0.3/src/adlist.c \
src/redis-5.0.3/src/dict.c \
src/redis-5.0.3/src/redis-cli.c \
src/redis-5.0.3/src/zmalloc.c \
src/redis-5.0.3/src/release.c \
src/redis-5.0.3/src/ae.c \
src/redis-5.0.3/src/crc64.c \
src/redis-5.0.3/src/siphash.c \
src/redis-5.0.3/src/crc16.c \
src/redis-5.0.3/deps/linenoise/linenoise.c


#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := \
-I"src/redis-5.0.3/src" \
-I"src/redis-5.0.3/deps/hiredis" \
-I"src/redis-5.0.3/deps/linenoise" \
-I"src/redis-5.0.3/deps/lua/src"

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL :=
LOCAL_DSYMBOL += -DREDIS_STATIC=""

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := -std=c99 -pedantic -Wno-missing-field-initializers -g -ggdb
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB      := \
-lhiredis \
-lcextern
LOCAL_DEPEND_LIB_PATH := \
-L"$(Output)" \
-L"$(SYLIXOS_BASE_PATH)/libcextern/$(Output)"

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD   := 
LOCAL_POST_LINK_CMD  := 
LOCAL_PRE_STRIP_CMD  := 
LOCAL_POST_STRIP_CMD :=

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(APPLICATION_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: redis-server.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2021 年 07 月 26 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := redis-server

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS := \
src/redis-5.0.3/src/adlist.c \
src/redis-5.0.3/src/quicklist.c \
src/redis-5.0.3/src/ae.c \
src/redis-5.0.3/src/anet.c \
src/redis-5.0.3/src/dict.c \
src/redis-5.0.3/src/server.c \
src/redis-5.0.3/src/sds.c \
src/redis-5.0.3/src/zmalloc.c \
src/redis-5.0.3/src/lzf_c.c \
src/redis-5.0.3/src/lzf_d.c \
src/redis-5.0.3/src/pqsort.c \
src/redis-5.0.3/src/zipmap.c \
src/redis-5.0.3/src/sha1.c \
src/redis-5.0.3/src/ziplist.c \
src/redis-5.0.3/src/release.c \
src/redis-5.0.3/src/networking.c \
src/redis-5.0.3/src/util.c \
src/redis-5.0.3/src/object.c \
src/redis-5.0.3/src/db.c \
src/redis-5.0.3/src/replication.c \
src/redis-5.0.3/src/rdb.c \
src/redis-5.0.3/src/t_string.c \
src/redis-5.0.3/src/t_list.c \
src/redis-5.0.3/src/t_set.c \
src/redis-5.0.3/src/t_zset.c \
src/redis-5.0.3/src/t_hash.c \
src/redis-5.0.3/src/config.c \
src/redis-5.0.3/src/aof.c \
src/redis-5.0.3/src/pubsub.c \
src/redis-5.0.3/src/multi.c \
src/redis-5.0.3/src/debug.c \
src/redis-5.0.3/src/sort.c \
src/redis-5.0.3/src/intset.c \
src/redis-5.0.3/src/syncio.c \
src/redis-5.0.3/src/cluster.c \
src/redis-5.0.3/src/crc16.c \
src/redis-5.0.3/src/endianconv.c \
src/redis-5.0.3/src/slowlog.c \
src/redis-5.0.3/src/scripting.c \
src/redis-5.0.3/src/bio.c \
src/redis-5.0.3/src/rio.c \
src/redis-5.0.3/src/rand.c \
src/redis-5.0.3/src/memtest.c \
src/redis-5.0.3/src/crc64.c \
src/redis-5.0.3/src/bitops.c \
src/redis-5.0.3/src/sentinel.c \
src/redis-5.0.3/src/notify.c \
src/redis-5.0.3/src/setproctitle.c \
src/redis-5.0.3/src/blocked.c \
src/redis-5.0.3/src/hyperloglog.c \
src/redis-5.0.3/src/latency.c \
src/redis-5.0.3/src/sparkline.c \
src/redis-5.0.3/src/redis-check-rdb.c \
src/redis-5.0.3/src/redis-check-aof.c \
src/redis-5.0.3/src/geo.c \
src/redis-5.0.3/src/lazyfree.c \
src/redis-5.0.3/src/module.c \
src/redis-5.0.3/src/evict.c \
src/redis-5.0.3/src/expire.c \
src/redis-5.0.3/src/geohash.c \
src/redis-5.0.3/src/geohash_helper.c \
src/redis-5.0.3/src/childinfo.c \
src/redis-5.0.3/src/defrag.c \
src/redis-5.0.3/src/siphash.c \
src/redis-5.0.3/src/rax.c \
src/redis-5.0.3/src/t_stream.c \
src/redis-5.0.3/src/listpack.c \
src/redis-5.0.3/src/localtime.c \
src/redis-5.0.3/src/lolwut.c \
src/redis-5.0.3/src/lolwut5.c


#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := \
-I"src/redis-5.0.3/src" \
-I"src/redis-5.0.3/deps/hiredis" \
-I"src/redis-5.0.3/deps/linenoise" \
-I"src/redis-5.0.3/deps/lua/src"

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL :=
LOCAL_DSYMBOL += -DREDIS_STATIC=""

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := -std=c99 -pedantic -Wno-missing-field-initializers -g -ggdb
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB      := \
-lhiredis \
-llua \
-lcextern
LOCAL_DEPEND_LIB_PATH := \
-L"$(Output)" \
-L"$(SYLIXOS_BASE_PATH)/libcextern/$(Output)"

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD   := 
LOCAL_POST_LINK_CMD  := 
LOCAL_PRE_STRIP_CMD  := 
LOCAL_POST_STRIP_CMD := \
cp $(Output)/redis-server $(Output)/redis-check-rdb; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-check-rdb; \
cp $(Output)/redis-server $(Output)/redis-check-aof; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-check-aof; \
cp $(Output)/redis-server $(Output)/redis-sentinel; \
cp $(Output)/strip/redis-server $(Output)/strip/redis-sentinel

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(APPLICATION_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

![排查符号表问题](./Image/排查符号表问题_3.png)

这样改完需要rebuild工程才有效，rebuild以后再上传。尝试运行一下redis-cli：

![排查符号表问题](./Image/排查符号表问题_4.png)

这说明APP已经可以运行了，先放一边，继续验证redis-server：

![排查符号表问题](./Image/排查符号表问题_5.png)

发现redis-server也已经可以运行了（server打印的时间还有问题），那运行着server，看看redis-cli能不能连接上server：

![排查符号表问题](./Image/排查符号表问题_6.png)

发现cli可以连上server，但貌似存在什么莫名其妙的问题，并没有响应我的命令，而且换行也很奇怪。

没事，起码移植的前两步已经完成了，开始慢慢的测和查吧。

## 排错

### redis-cli 换行问题

既然换行有问题，那就找redis-cli里换行的实现，即处理换行的函数，这里忽略怎么找的过程，全凭各自经验和排错习惯找，定位到linenoise.c里：

![排错](./Image/排错_1.png)

这里是通过while 1获取输入字符的，处理的ENTER的值是13，这里我们加个打印排查一下这个问题：

![排错](./Image/排错_2.png)

保存，编译，上传，运行，发现敲回车时的打印是：

![排错](./Image/排错_3.png)

发现SylixOS下的ENTER对应的值是10，找到ENTER宏定义的位置，修改一下：

![排错](./Image/排错_4.png)

保存编译上传，运行发现cli功能正常了，可以正确响应命令，别忘了把加的打印去掉。

### INFO命令崩溃问题

继续验证，发现redis-cli里敲了info命令后，server崩了：

![排错](./Image/排错_5.png)

![排错](./Image/排错_6.png)

这个打印就是SylixOS下程序崩溃出现的栈回溯，辅助我们定位问题的。栈回溯是由下而上一级级调用的，仔细观察，发现实际是崩溃在sdscatvprintf函数里，但实际这个函数并不能查出啥问题。只能对genRedisInfoString函数进行解剖，一段段加打印或者注释，定位问题。

最终定位问题在于%jd：

![排错](./Image/排错_7.png)

redis希望通过%jd打印64位值，然而SylixOS编译器不认这玩意，导致后续的打印产生偏移，产生越界问题而崩溃。

全局搜索%jd，使用%lld替换即可：

![排错](./Image/排错_8.png)

其他还有一些地方，自行替换。

保存，编译，上传，运行验证，发现INFO命令不会导致崩溃。

![排错](./Image/排错_9.png)

## SylixOS移植常见问题——编译过多文件导致报错

### 问题复现：

由于RealEvo IDE在Windows下使用的sh.exe支持的命令长度有限（命令要小于16K），如果需要编译的源文件过多，可能会报溢出错误或"make (e=87): 参数错误。"等，典型的表现型式有：

![SylixOS移植常见问题](./Image/SylixOS移植常见问题_1.png)

![SylixOS移植常见问题](./Image/SylixOS移植常见问题_2.png)

我这里是创建了0.c、1.c~999.c，编译为一个动态库，来复现这个错误。系统自动生成的mk文件为：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: test_big_range_so.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2021 年 08 月 09 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := libtest_big_range_so.so

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
LOCAL_SRCS :=  \
src/0.c \
src/1.c \
src/10.c \
src/100.c \
src/101.c \
src/102.c \
src/103.c \
src/104.c \
src/105.c \
src/106.c \
src/107.c \
src/108.c \
src/109.c \
src/11.c \
src/110.c \
src/111.c \
src/112.c \
src/113.c \
src/114.c \
src/115.c \
src/116.c \
src/117.c \
src/118.c \
src/119.c \
src/12.c \
src/120.c \
src/121.c \
src/122.c \
src/123.c \
src/124.c \
src/125.c \
src/126.c \
src/127.c \
src/128.c \
src/129.c \
src/13.c \
src/130.c \
src/131.c \
src/132.c \
src/133.c \
src/134.c \
src/135.c \
src/136.c \
src/137.c \
src/138.c \
src/139.c \
src/14.c \
src/140.c \
src/141.c \
src/142.c \
src/143.c \
src/144.c \
src/145.c \
src/146.c \
src/147.c \
src/148.c \
src/149.c \
src/15.c \
src/150.c \
src/151.c \
src/152.c \
src/153.c \
src/154.c \
src/155.c \
src/156.c \
src/157.c \
src/158.c \
src/159.c \
src/16.c \
src/160.c \
src/161.c \
src/162.c \
src/163.c \
src/164.c \
src/165.c \
src/166.c \
src/167.c \
src/168.c \
src/169.c \
src/17.c \
src/170.c \
src/171.c \
src/172.c \
src/173.c \
src/174.c \
src/175.c \
src/176.c \
src/177.c \
src/178.c \
src/179.c \
src/18.c \
src/180.c \
src/181.c \
src/182.c \
src/183.c \
src/184.c \
src/185.c \
src/186.c \
src/187.c \
src/188.c \
src/189.c \
src/19.c \
src/190.c \
src/191.c \
src/192.c \
src/193.c \
src/194.c \
src/195.c \
src/196.c \
src/197.c \
src/198.c \
src/199.c \
src/2.c \
src/20.c \
src/200.c \
src/201.c \
src/202.c \
src/203.c \
src/204.c \
src/205.c \
src/206.c \
src/207.c \
src/208.c \
src/209.c \
src/21.c \
src/210.c \
src/211.c \
src/212.c \
src/213.c \
src/214.c \
src/215.c \
src/216.c \
src/217.c \
src/218.c \
src/219.c \
src/22.c \
src/220.c \
src/221.c \
src/222.c \
src/223.c \
src/224.c \
src/225.c \
src/226.c \
src/227.c \
src/228.c \
src/229.c \
src/23.c \
src/230.c \
src/231.c \
src/232.c \
src/233.c \
src/234.c \
src/235.c \
src/236.c \
src/237.c \
src/238.c \
src/239.c \
src/24.c \
src/240.c \
src/241.c \
src/242.c \
src/243.c \
src/244.c \
src/245.c \
src/246.c \
src/247.c \
src/248.c \
src/249.c \
src/25.c \
src/250.c \
src/251.c \
src/252.c \
src/253.c \
src/254.c \
src/255.c \
src/256.c \
src/257.c \
src/258.c \
src/259.c \
src/26.c \
src/260.c \
src/261.c \
src/262.c \
src/263.c \
src/264.c \
src/265.c \
src/266.c \
src/267.c \
src/268.c \
src/269.c \
src/27.c \
src/270.c \
src/271.c \
src/272.c \
src/273.c \
src/274.c \
src/275.c \
src/276.c \
src/277.c \
src/278.c \
src/279.c \
src/28.c \
src/280.c \
src/281.c \
src/282.c \
src/283.c \
src/284.c \
src/285.c \
src/286.c \
src/287.c \
src/288.c \
src/289.c \
src/29.c \
src/290.c \
src/291.c \
src/292.c \
src/293.c \
src/294.c \
src/295.c \
src/296.c \
src/297.c \
src/298.c \
src/299.c \
src/3.c \
src/30.c \
src/300.c \
src/301.c \
src/302.c \
src/303.c \
src/304.c \
src/305.c \
src/306.c \
src/307.c \
src/308.c \
src/309.c \
src/31.c \
src/310.c \
src/311.c \
src/312.c \
src/313.c \
src/314.c \
src/315.c \
src/316.c \
src/317.c \
src/318.c \
src/319.c \
src/32.c \
src/320.c \
src/321.c \
src/322.c \
src/323.c \
src/324.c \
src/325.c \
src/326.c \
src/327.c \
src/328.c \
src/329.c \
src/33.c \
src/330.c \
src/331.c \
src/332.c \
src/333.c \
src/334.c \
src/335.c \
src/336.c \
src/337.c \
src/338.c \
src/339.c \
src/34.c \
src/340.c \
src/341.c \
src/342.c \
src/343.c \
src/344.c \
src/345.c \
src/346.c \
src/347.c \
src/348.c \
src/349.c \
src/35.c \
src/350.c \
src/351.c \
src/352.c \
src/353.c \
src/354.c \
src/355.c \
src/356.c \
src/357.c \
src/358.c \
src/359.c \
src/36.c \
src/360.c \
src/361.c \
src/362.c \
src/363.c \
src/364.c \
src/365.c \
src/366.c \
src/367.c \
src/368.c \
src/369.c \
src/37.c \
src/370.c \
src/371.c \
src/372.c \
src/373.c \
src/374.c \
src/375.c \
src/376.c \
src/377.c \
src/378.c \
src/379.c \
src/38.c \
src/380.c \
src/381.c \
src/382.c \
src/383.c \
src/384.c \
src/385.c \
src/386.c \
src/387.c \
src/388.c \
src/389.c \
src/39.c \
src/390.c \
src/391.c \
src/392.c \
src/393.c \
src/394.c \
src/395.c \
src/396.c \
src/397.c \
src/398.c \
src/399.c \
src/4.c \
src/40.c \
src/400.c \
src/401.c \
src/402.c \
src/403.c \
src/404.c \
src/405.c \
src/406.c \
src/407.c \
src/408.c \
src/409.c \
src/41.c \
src/410.c \
src/411.c \
src/412.c \
src/413.c \
src/414.c \
src/415.c \
src/416.c \
src/417.c \
src/418.c \
src/419.c \
src/42.c \
src/420.c \
src/421.c \
src/422.c \
src/423.c \
src/424.c \
src/425.c \
src/426.c \
src/427.c \
src/428.c \
src/429.c \
src/43.c \
src/430.c \
src/431.c \
src/432.c \
src/433.c \
src/434.c \
src/435.c \
src/436.c \
src/437.c \
src/438.c \
src/439.c \
src/44.c \
src/440.c \
src/441.c \
src/442.c \
src/443.c \
src/444.c \
src/445.c \
src/446.c \
src/447.c \
src/448.c \
src/449.c \
src/45.c \
src/450.c \
src/451.c \
src/452.c \
src/453.c \
src/454.c \
src/455.c \
src/456.c \
src/457.c \
src/458.c \
src/459.c \
src/46.c \
src/460.c \
src/461.c \
src/462.c \
src/463.c \
src/464.c \
src/465.c \
src/466.c \
src/467.c \
src/468.c \
src/469.c \
src/47.c \
src/470.c \
src/471.c \
src/472.c \
src/473.c \
src/474.c \
src/475.c \
src/476.c \
src/477.c \
src/478.c \
src/479.c \
src/48.c \
src/480.c \
src/481.c \
src/482.c \
src/483.c \
src/484.c \
src/485.c \
src/486.c \
src/487.c \
src/488.c \
src/489.c \
src/49.c \
src/490.c \
src/491.c \
src/492.c \
src/493.c \
src/494.c \
src/495.c \
src/496.c \
src/497.c \
src/498.c \
src/499.c \
src/5.c \
src/50.c \
src/500.c \
src/501.c \
src/502.c \
src/503.c \
src/504.c \
src/505.c \
src/506.c \
src/507.c \
src/508.c \
src/509.c \
src/51.c \
src/510.c \
src/511.c \
src/512.c \
src/513.c \
src/514.c \
src/515.c \
src/516.c \
src/517.c \
src/518.c \
src/519.c \
src/52.c \
src/520.c \
src/521.c \
src/522.c \
src/523.c \
src/524.c \
src/525.c \
src/526.c \
src/527.c \
src/528.c \
src/529.c \
src/53.c \
src/530.c \
src/531.c \
src/532.c \
src/533.c \
src/534.c \
src/535.c \
src/536.c \
src/537.c \
src/538.c \
src/539.c \
src/54.c \
src/540.c \
src/541.c \
src/542.c \
src/543.c \
src/544.c \
src/545.c \
src/546.c \
src/547.c \
src/548.c \
src/549.c \
src/55.c \
src/550.c \
src/551.c \
src/552.c \
src/553.c \
src/554.c \
src/555.c \
src/556.c \
src/557.c \
src/558.c \
src/559.c \
src/56.c \
src/560.c \
src/561.c \
src/562.c \
src/563.c \
src/564.c \
src/565.c \
src/566.c \
src/567.c \
src/568.c \
src/569.c \
src/57.c \
src/570.c \
src/571.c \
src/572.c \
src/573.c \
src/574.c \
src/575.c \
src/576.c \
src/577.c \
src/578.c \
src/579.c \
src/58.c \
src/580.c \
src/581.c \
src/582.c \
src/583.c \
src/584.c \
src/585.c \
src/586.c \
src/587.c \
src/588.c \
src/589.c \
src/59.c \
src/590.c \
src/591.c \
src/592.c \
src/593.c \
src/594.c \
src/595.c \
src/596.c \
src/597.c \
src/598.c \
src/599.c \
src/6.c \
src/60.c \
src/600.c \
src/601.c \
src/602.c \
src/603.c \
src/604.c \
src/605.c \
src/606.c \
src/607.c \
src/608.c \
src/609.c \
src/61.c \
src/610.c \
src/611.c \
src/612.c \
src/613.c \
src/614.c \
src/615.c \
src/616.c \
src/617.c \
src/618.c \
src/619.c \
src/62.c \
src/620.c \
src/621.c \
src/622.c \
src/623.c \
src/624.c \
src/625.c \
src/626.c \
src/627.c \
src/628.c \
src/629.c \
src/63.c \
src/630.c \
src/631.c \
src/632.c \
src/633.c \
src/634.c \
src/635.c \
src/636.c \
src/637.c \
src/638.c \
src/639.c \
src/64.c \
src/640.c \
src/641.c \
src/642.c \
src/643.c \
src/644.c \
src/645.c \
src/646.c \
src/647.c \
src/648.c \
src/649.c \
src/65.c \
src/650.c \
src/651.c \
src/652.c \
src/653.c \
src/654.c \
src/655.c \
src/656.c \
src/657.c \
src/658.c \
src/659.c \
src/66.c \
src/660.c \
src/661.c \
src/662.c \
src/663.c \
src/664.c \
src/665.c \
src/666.c \
src/667.c \
src/668.c \
src/669.c \
src/67.c \
src/670.c \
src/671.c \
src/672.c \
src/673.c \
src/674.c \
src/675.c \
src/676.c \
src/677.c \
src/678.c \
src/679.c \
src/68.c \
src/680.c \
src/681.c \
src/682.c \
src/683.c \
src/684.c \
src/685.c \
src/686.c \
src/687.c \
src/688.c \
src/689.c \
src/69.c \
src/690.c \
src/691.c \
src/692.c \
src/693.c \
src/694.c \
src/695.c \
src/696.c \
src/697.c \
src/698.c \
src/699.c \
src/7.c \
src/70.c \
src/700.c \
src/701.c \
src/702.c \
src/703.c \
src/704.c \
src/705.c \
src/706.c \
src/707.c \
src/708.c \
src/709.c \
src/71.c \
src/710.c \
src/711.c \
src/712.c \
src/713.c \
src/714.c \
src/715.c \
src/716.c \
src/717.c \
src/718.c \
src/719.c \
src/72.c \
src/720.c \
src/721.c \
src/722.c \
src/723.c \
src/724.c \
src/725.c \
src/726.c \
src/727.c \
src/728.c \
src/729.c \
src/73.c \
src/730.c \
src/731.c \
src/732.c \
src/733.c \
src/734.c \
src/735.c \
src/736.c \
src/737.c \
src/738.c \
src/739.c \
src/74.c \
src/740.c \
src/741.c \
src/742.c \
src/743.c \
src/744.c \
src/745.c \
src/746.c \
src/747.c \
src/748.c \
src/749.c \
src/75.c \
src/750.c \
src/751.c \
src/752.c \
src/753.c \
src/754.c \
src/755.c \
src/756.c \
src/757.c \
src/758.c \
src/759.c \
src/76.c \
src/760.c \
src/761.c \
src/762.c \
src/763.c \
src/764.c \
src/765.c \
src/766.c \
src/767.c \
src/768.c \
src/769.c \
src/77.c \
src/770.c \
src/771.c \
src/772.c \
src/773.c \
src/774.c \
src/775.c \
src/776.c \
src/777.c \
src/778.c \
src/779.c \
src/78.c \
src/780.c \
src/781.c \
src/782.c \
src/783.c \
src/784.c \
src/785.c \
src/786.c \
src/787.c \
src/788.c \
src/789.c \
src/79.c \
src/790.c \
src/791.c \
src/792.c \
src/793.c \
src/794.c \
src/795.c \
src/796.c \
src/797.c \
src/798.c \
src/799.c \
src/8.c \
src/80.c \
src/800.c \
src/801.c \
src/802.c \
src/803.c \
src/804.c \
src/805.c \
src/806.c \
src/807.c \
src/808.c \
src/809.c \
src/81.c \
src/810.c \
src/811.c \
src/812.c \
src/813.c \
src/814.c \
src/815.c \
src/816.c \
src/817.c \
src/818.c \
src/819.c \
src/82.c \
src/820.c \
src/821.c \
src/822.c \
src/823.c \
src/824.c \
src/825.c \
src/826.c \
src/827.c \
src/828.c \
src/829.c \
src/83.c \
src/830.c \
src/831.c \
src/832.c \
src/833.c \
src/834.c \
src/835.c \
src/836.c \
src/837.c \
src/838.c \
src/839.c \
src/84.c \
src/840.c \
src/841.c \
src/842.c \
src/843.c \
src/844.c \
src/845.c \
src/846.c \
src/847.c \
src/848.c \
src/849.c \
src/85.c \
src/850.c \
src/851.c \
src/852.c \
src/853.c \
src/854.c \
src/855.c \
src/856.c \
src/857.c \
src/858.c \
src/859.c \
src/86.c \
src/860.c \
src/861.c \
src/862.c \
src/863.c \
src/864.c \
src/865.c \
src/866.c \
src/867.c \
src/868.c \
src/869.c \
src/87.c \
src/870.c \
src/871.c \
src/872.c \
src/873.c \
src/874.c \
src/875.c \
src/876.c \
src/877.c \
src/878.c \
src/879.c \
src/88.c \
src/880.c \
src/881.c \
src/882.c \
src/883.c \
src/884.c \
src/885.c \
src/886.c \
src/887.c \
src/888.c \
src/889.c \
src/89.c \
src/890.c \
src/891.c \
src/892.c \
src/893.c \
src/894.c \
src/895.c \
src/896.c \
src/897.c \
src/898.c \
src/899.c \
src/9.c \
src/90.c \
src/900.c \
src/901.c \
src/902.c \
src/903.c \
src/904.c \
src/905.c \
src/906.c \
src/907.c \
src/908.c \
src/909.c \
src/91.c \
src/910.c \
src/911.c \
src/912.c \
src/913.c \
src/914.c \
src/915.c \
src/916.c \
src/917.c \
src/918.c \
src/919.c \
src/92.c \
src/920.c \
src/921.c \
src/922.c \
src/923.c \
src/924.c \
src/925.c \
src/926.c \
src/927.c \
src/928.c \
src/929.c \
src/93.c \
src/930.c \
src/931.c \
src/932.c \
src/933.c \
src/934.c \
src/935.c \
src/936.c \
src/937.c \
src/938.c \
src/939.c \
src/94.c \
src/940.c \
src/941.c \
src/942.c \
src/943.c \
src/944.c \
src/945.c \
src/946.c \
src/947.c \
src/948.c \
src/949.c \
src/95.c \
src/950.c \
src/951.c \
src/952.c \
src/953.c \
src/954.c \
src/955.c \
src/956.c \
src/957.c \
src/958.c \
src/959.c \
src/96.c \
src/960.c \
src/961.c \
src/962.c \
src/963.c \
src/964.c \
src/965.c \
src/966.c \
src/967.c \
src/968.c \
src/969.c \
src/97.c \
src/970.c \
src/971.c \
src/972.c \
src/973.c \
src/974.c \
src/975.c \
src/976.c \
src/977.c \
src/978.c \
src/979.c \
src/98.c \
src/980.c \
src/981.c \
src/982.c \
src/983.c \
src/984.c \
src/985.c \
src/986.c \
src/987.c \
src/988.c \
src/989.c \
src/99.c \
src/990.c \
src/991.c \
src/992.c \
src/993.c \
src/994.c \
src/995.c \
src/996.c \
src/997.c \
src/998.c \
src/999.c

#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := 

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL := 

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := 
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB := 
LOCAL_DEPEND_LIB_PATH := 

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD := 
LOCAL_POST_LINK_CMD := 
LOCAL_PRE_STRIP_CMD := 
LOCAL_POST_STRIP_CMD := 

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(LIBRARY_MK)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

### 解决方案：

由于问题出现的原因显而易见，就是在编译过程中，执行到链接阶段时，由于需要链接的.c文件过多，导致链接的那一条命令长度超出16K，从而导致报错。

这里我提供两种解决方案。

#### 人为收束

既然是因为命令过长导致错误，那在不能削减编译文件数量的前提下，可以考虑将降低源文件的目录深度。即将一些无用的上级目录删去，或减短目录名称，以达到降低链接命令长度的目的。

本方案适用于工程文件数量并没有超出太多，且不想打开专家模式的用户。好处是如果能通过削减目录的方式达到成功执行链接指令，则不需要人为修改makefile，避免由于不熟悉专家模式而引入的其他问题。

#### 修改makfile

修改makefile适用于任何场景，原理就是将大量的源文件分段处理。以我这个工程为例，可以将0.c~249.c作为part1，250.c~499.c作为part2，500.c~749.c作为part3，750.c~999.c作为part4，怎么分完全是个人爱好。接着编译阶段保持不变，手动加入链接命令，分别AR每个PART的.o文件，则每次链接命令和ar命令都不会超出16K。

修改后的mk文件：

```
#*********************************************************************************************************
#
#                                    中国软件开源组织
#
#                                   嵌入式实时操作系统
#
#                                SylixOS(TM)  LW : long wing
#
#                               Copyright All Rights Reserved
#
#--------------文件信息--------------------------------------------------------------------------------
#
# 文   件   名: test_big_range_so.mk
#
# 创   建   人: RealEvo-IDE
#
# 文件创建日期: 2021 年 08 月 09 日
#
# 描        述: 本文件由 RealEvo-IDE 生成，用于配置 Makefile 功能，请勿手动修改
#*********************************************************************************************************

#*********************************************************************************************************
# Clear setting
#*********************************************************************************************************
include $(CLEAR_VARS_MK)

#*********************************************************************************************************
# Target
#*********************************************************************************************************
LOCAL_TARGET_NAME := libtest_big_range_so.so

#*********************************************************************************************************
# Source list
#*********************************************************************************************************
MYLIB_PART1_SRCS := \
src/0.c \
src/1.c \
src/10.c \
src/100.c \
src/101.c \
src/102.c \
src/103.c \
src/104.c \
src/105.c \
src/106.c \
src/107.c \
src/108.c \
src/109.c \
src/11.c \
src/110.c \
src/111.c \
src/112.c \
src/113.c \
src/114.c \
src/115.c \
src/116.c \
src/117.c \
src/118.c \
src/119.c \
src/12.c \
src/120.c \
src/121.c \
src/122.c \
src/123.c \
src/124.c \
src/125.c \
src/126.c \
src/127.c \
src/128.c \
src/129.c \
src/13.c \
src/130.c \
src/131.c \
src/132.c \
src/133.c \
src/134.c \
src/135.c \
src/136.c \
src/137.c \
src/138.c \
src/139.c \
src/14.c \
src/140.c \
src/141.c \
src/142.c \
src/143.c \
src/144.c \
src/145.c \
src/146.c \
src/147.c \
src/148.c \
src/149.c \
src/15.c \
src/150.c \
src/151.c \
src/152.c \
src/153.c \
src/154.c \
src/155.c \
src/156.c \
src/157.c \
src/158.c \
src/159.c \
src/16.c \
src/160.c \
src/161.c \
src/162.c \
src/163.c \
src/164.c \
src/165.c \
src/166.c \
src/167.c \
src/168.c \
src/169.c \
src/17.c \
src/170.c \
src/171.c \
src/172.c \
src/173.c \
src/174.c \
src/175.c \
src/176.c \
src/177.c \
src/178.c \
src/179.c \
src/18.c \
src/180.c \
src/181.c \
src/182.c \
src/183.c \
src/184.c \
src/185.c \
src/186.c \
src/187.c \
src/188.c \
src/189.c \
src/19.c \
src/190.c \
src/191.c \
src/192.c \
src/193.c \
src/194.c \
src/195.c \
src/196.c \
src/197.c \
src/198.c \
src/199.c \
src/2.c \
src/20.c \
src/200.c \
src/201.c \
src/202.c \
src/203.c \
src/204.c \
src/205.c \
src/206.c \
src/207.c \
src/208.c \
src/209.c \
src/21.c \
src/210.c \
src/211.c \
src/212.c \
src/213.c \
src/214.c \
src/215.c \
src/216.c \
src/217.c \
src/218.c \
src/219.c \
src/22.c \
src/220.c \
src/221.c \
src/222.c \
src/223.c \
src/224.c \
src/225.c \
src/226.c \
src/227.c \
src/228.c \
src/229.c \
src/23.c \
src/230.c \
src/231.c \
src/232.c \
src/233.c \
src/234.c \
src/235.c \
src/236.c \
src/237.c \
src/238.c \
src/239.c \
src/24.c \
src/240.c \
src/241.c \
src/242.c \
src/243.c \
src/244.c \
src/245.c \
src/246.c \
src/247.c \
src/248.c \
src/249.c
MYLIB_PART2_SRCS := \
src/25.c \
src/250.c \
src/251.c \
src/252.c \
src/253.c \
src/254.c \
src/255.c \
src/256.c \
src/257.c \
src/258.c \
src/259.c \
src/26.c \
src/260.c \
src/261.c \
src/262.c \
src/263.c \
src/264.c \
src/265.c \
src/266.c \
src/267.c \
src/268.c \
src/269.c \
src/27.c \
src/270.c \
src/271.c \
src/272.c \
src/273.c \
src/274.c \
src/275.c \
src/276.c \
src/277.c \
src/278.c \
src/279.c \
src/28.c \
src/280.c \
src/281.c \
src/282.c \
src/283.c \
src/284.c \
src/285.c \
src/286.c \
src/287.c \
src/288.c \
src/289.c \
src/29.c \
src/290.c \
src/291.c \
src/292.c \
src/293.c \
src/294.c \
src/295.c \
src/296.c \
src/297.c \
src/298.c \
src/299.c \
src/3.c \
src/30.c \
src/300.c \
src/301.c \
src/302.c \
src/303.c \
src/304.c \
src/305.c \
src/306.c \
src/307.c \
src/308.c \
src/309.c \
src/31.c \
src/310.c \
src/311.c \
src/312.c \
src/313.c \
src/314.c \
src/315.c \
src/316.c \
src/317.c \
src/318.c \
src/319.c \
src/32.c \
src/320.c \
src/321.c \
src/322.c \
src/323.c \
src/324.c \
src/325.c \
src/326.c \
src/327.c \
src/328.c \
src/329.c \
src/33.c \
src/330.c \
src/331.c \
src/332.c \
src/333.c \
src/334.c \
src/335.c \
src/336.c \
src/337.c \
src/338.c \
src/339.c \
src/34.c \
src/340.c \
src/341.c \
src/342.c \
src/343.c \
src/344.c \
src/345.c \
src/346.c \
src/347.c \
src/348.c \
src/349.c \
src/35.c \
src/350.c \
src/351.c \
src/352.c \
src/353.c \
src/354.c \
src/355.c \
src/356.c \
src/357.c \
src/358.c \
src/359.c \
src/36.c \
src/360.c \
src/361.c \
src/362.c \
src/363.c \
src/364.c \
src/365.c \
src/366.c \
src/367.c \
src/368.c \
src/369.c \
src/37.c \
src/370.c \
src/371.c \
src/372.c \
src/373.c \
src/374.c \
src/375.c \
src/376.c \
src/377.c \
src/378.c \
src/379.c \
src/38.c \
src/380.c \
src/381.c \
src/382.c \
src/383.c \
src/384.c \
src/385.c \
src/386.c \
src/387.c \
src/388.c \
src/389.c \
src/39.c \
src/390.c \
src/391.c \
src/392.c \
src/393.c \
src/394.c \
src/395.c \
src/396.c \
src/397.c \
src/398.c \
src/399.c \
src/4.c \
src/40.c \
src/400.c \
src/401.c \
src/402.c \
src/403.c \
src/404.c \
src/405.c \
src/406.c \
src/407.c \
src/408.c \
src/409.c \
src/41.c \
src/410.c \
src/411.c \
src/412.c \
src/413.c \
src/414.c \
src/415.c \
src/416.c \
src/417.c \
src/418.c \
src/419.c \
src/42.c \
src/420.c \
src/421.c \
src/422.c \
src/423.c \
src/424.c \
src/425.c \
src/426.c \
src/427.c \
src/428.c \
src/429.c \
src/43.c \
src/430.c \
src/431.c \
src/432.c \
src/433.c \
src/434.c \
src/435.c \
src/436.c \
src/437.c \
src/438.c \
src/439.c \
src/44.c \
src/440.c \
src/441.c \
src/442.c \
src/443.c \
src/444.c \
src/445.c \
src/446.c \
src/447.c \
src/448.c \
src/449.c \
src/45.c \
src/450.c \
src/451.c \
src/452.c \
src/453.c \
src/454.c \
src/455.c \
src/456.c \
src/457.c \
src/458.c \
src/459.c \
src/46.c \
src/460.c \
src/461.c \
src/462.c \
src/463.c \
src/464.c \
src/465.c \
src/466.c \
src/467.c \
src/468.c \
src/469.c \
src/47.c \
src/470.c \
src/471.c \
src/472.c \
src/473.c \
src/474.c \
src/475.c \
src/476.c \
src/477.c \
src/478.c \
src/479.c \
src/48.c \
src/480.c \
src/481.c \
src/482.c \
src/483.c \
src/484.c \
src/485.c \
src/486.c \
src/487.c \
src/488.c \
src/489.c \
src/49.c \
src/490.c \
src/491.c \
src/492.c \
src/493.c \
src/494.c \
src/495.c \
src/496.c \
src/497.c \
src/498.c \
src/499.c
MYLIB_PART3_SRCS := \
src/5.c \
src/50.c \
src/500.c \
src/501.c \
src/502.c \
src/503.c \
src/504.c \
src/505.c \
src/506.c \
src/507.c \
src/508.c \
src/509.c \
src/51.c \
src/510.c \
src/511.c \
src/512.c \
src/513.c \
src/514.c \
src/515.c \
src/516.c \
src/517.c \
src/518.c \
src/519.c \
src/52.c \
src/520.c \
src/521.c \
src/522.c \
src/523.c \
src/524.c \
src/525.c \
src/526.c \
src/527.c \
src/528.c \
src/529.c \
src/53.c \
src/530.c \
src/531.c \
src/532.c \
src/533.c \
src/534.c \
src/535.c \
src/536.c \
src/537.c \
src/538.c \
src/539.c \
src/54.c \
src/540.c \
src/541.c \
src/542.c \
src/543.c \
src/544.c \
src/545.c \
src/546.c \
src/547.c \
src/548.c \
src/549.c \
src/55.c \
src/550.c \
src/551.c \
src/552.c \
src/553.c \
src/554.c \
src/555.c \
src/556.c \
src/557.c \
src/558.c \
src/559.c \
src/56.c \
src/560.c \
src/561.c \
src/562.c \
src/563.c \
src/564.c \
src/565.c \
src/566.c \
src/567.c \
src/568.c \
src/569.c \
src/57.c \
src/570.c \
src/571.c \
src/572.c \
src/573.c \
src/574.c \
src/575.c \
src/576.c \
src/577.c \
src/578.c \
src/579.c \
src/58.c \
src/580.c \
src/581.c \
src/582.c \
src/583.c \
src/584.c \
src/585.c \
src/586.c \
src/587.c \
src/588.c \
src/589.c \
src/59.c \
src/590.c \
src/591.c \
src/592.c \
src/593.c \
src/594.c \
src/595.c \
src/596.c \
src/597.c \
src/598.c \
src/599.c \
src/6.c \
src/60.c \
src/600.c \
src/601.c \
src/602.c \
src/603.c \
src/604.c \
src/605.c \
src/606.c \
src/607.c \
src/608.c \
src/609.c \
src/61.c \
src/610.c \
src/611.c \
src/612.c \
src/613.c \
src/614.c \
src/615.c \
src/616.c \
src/617.c \
src/618.c \
src/619.c \
src/62.c \
src/620.c \
src/621.c \
src/622.c \
src/623.c \
src/624.c \
src/625.c \
src/626.c \
src/627.c \
src/628.c \
src/629.c \
src/63.c \
src/630.c \
src/631.c \
src/632.c \
src/633.c \
src/634.c \
src/635.c \
src/636.c \
src/637.c \
src/638.c \
src/639.c \
src/64.c \
src/640.c \
src/641.c \
src/642.c \
src/643.c \
src/644.c \
src/645.c \
src/646.c \
src/647.c \
src/648.c \
src/649.c \
src/65.c \
src/650.c \
src/651.c \
src/652.c \
src/653.c \
src/654.c \
src/655.c \
src/656.c \
src/657.c \
src/658.c \
src/659.c \
src/66.c \
src/660.c \
src/661.c \
src/662.c \
src/663.c \
src/664.c \
src/665.c \
src/666.c \
src/667.c \
src/668.c \
src/669.c \
src/67.c \
src/670.c \
src/671.c \
src/672.c \
src/673.c \
src/674.c \
src/675.c \
src/676.c \
src/677.c \
src/678.c \
src/679.c \
src/68.c \
src/680.c \
src/681.c \
src/682.c \
src/683.c \
src/684.c \
src/685.c \
src/686.c \
src/687.c \
src/688.c \
src/689.c \
src/69.c \
src/690.c \
src/691.c \
src/692.c \
src/693.c \
src/694.c \
src/695.c \
src/696.c \
src/697.c \
src/698.c \
src/699.c \
src/7.c \
src/70.c \
src/700.c \
src/701.c \
src/702.c \
src/703.c \
src/704.c \
src/705.c \
src/706.c \
src/707.c \
src/708.c \
src/709.c \
src/71.c \
src/710.c \
src/711.c \
src/712.c \
src/713.c \
src/714.c \
src/715.c \
src/716.c \
src/717.c \
src/718.c \
src/719.c \
src/72.c \
src/720.c \
src/721.c \
src/722.c \
src/723.c \
src/724.c \
src/725.c \
src/726.c \
src/727.c \
src/728.c \
src/729.c \
src/73.c \
src/730.c \
src/731.c \
src/732.c \
src/733.c \
src/734.c \
src/735.c \
src/736.c \
src/737.c \
src/738.c \
src/739.c \
src/74.c \
src/740.c \
src/741.c \
src/742.c \
src/743.c \
src/744.c \
src/745.c \
src/746.c \
src/747.c \
src/748.c \
src/749.c
MYLIB_PART4_SRCS := \
src/75.c \
src/750.c \
src/751.c \
src/752.c \
src/753.c \
src/754.c \
src/755.c \
src/756.c \
src/757.c \
src/758.c \
src/759.c \
src/76.c \
src/760.c \
src/761.c \
src/762.c \
src/763.c \
src/764.c \
src/765.c \
src/766.c \
src/767.c \
src/768.c \
src/769.c \
src/77.c \
src/770.c \
src/771.c \
src/772.c \
src/773.c \
src/774.c \
src/775.c \
src/776.c \
src/777.c \
src/778.c \
src/779.c \
src/78.c \
src/780.c \
src/781.c \
src/782.c \
src/783.c \
src/784.c \
src/785.c \
src/786.c \
src/787.c \
src/788.c \
src/789.c \
src/79.c \
src/790.c \
src/791.c \
src/792.c \
src/793.c \
src/794.c \
src/795.c \
src/796.c \
src/797.c \
src/798.c \
src/799.c \
src/8.c \
src/80.c \
src/800.c \
src/801.c \
src/802.c \
src/803.c \
src/804.c \
src/805.c \
src/806.c \
src/807.c \
src/808.c \
src/809.c \
src/81.c \
src/810.c \
src/811.c \
src/812.c \
src/813.c \
src/814.c \
src/815.c \
src/816.c \
src/817.c \
src/818.c \
src/819.c \
src/82.c \
src/820.c \
src/821.c \
src/822.c \
src/823.c \
src/824.c \
src/825.c \
src/826.c \
src/827.c \
src/828.c \
src/829.c \
src/83.c \
src/830.c \
src/831.c \
src/832.c \
src/833.c \
src/834.c \
src/835.c \
src/836.c \
src/837.c \
src/838.c \
src/839.c \
src/84.c \
src/840.c \
src/841.c \
src/842.c \
src/843.c \
src/844.c \
src/845.c \
src/846.c \
src/847.c \
src/848.c \
src/849.c \
src/85.c \
src/850.c \
src/851.c \
src/852.c \
src/853.c \
src/854.c \
src/855.c \
src/856.c \
src/857.c \
src/858.c \
src/859.c \
src/86.c \
src/860.c \
src/861.c \
src/862.c \
src/863.c \
src/864.c \
src/865.c \
src/866.c \
src/867.c \
src/868.c \
src/869.c \
src/87.c \
src/870.c \
src/871.c \
src/872.c \
src/873.c \
src/874.c \
src/875.c \
src/876.c \
src/877.c \
src/878.c \
src/879.c \
src/88.c \
src/880.c \
src/881.c \
src/882.c \
src/883.c \
src/884.c \
src/885.c \
src/886.c \
src/887.c \
src/888.c \
src/889.c \
src/89.c \
src/890.c \
src/891.c \
src/892.c \
src/893.c \
src/894.c \
src/895.c \
src/896.c \
src/897.c \
src/898.c \
src/899.c \
src/9.c \
src/90.c \
src/900.c \
src/901.c \
src/902.c \
src/903.c \
src/904.c \
src/905.c \
src/906.c \
src/907.c \
src/908.c \
src/909.c \
src/91.c \
src/910.c \
src/911.c \
src/912.c \
src/913.c \
src/914.c \
src/915.c \
src/916.c \
src/917.c \
src/918.c \
src/919.c \
src/92.c \
src/920.c \
src/921.c \
src/922.c \
src/923.c \
src/924.c \
src/925.c \
src/926.c \
src/927.c \
src/928.c \
src/929.c \
src/93.c \
src/930.c \
src/931.c \
src/932.c \
src/933.c \
src/934.c \
src/935.c \
src/936.c \
src/937.c \
src/938.c \
src/939.c \
src/94.c \
src/940.c \
src/941.c \
src/942.c \
src/943.c \
src/944.c \
src/945.c \
src/946.c \
src/947.c \
src/948.c \
src/949.c \
src/95.c \
src/950.c \
src/951.c \
src/952.c \
src/953.c \
src/954.c \
src/955.c \
src/956.c \
src/957.c \
src/958.c \
src/959.c \
src/96.c \
src/960.c \
src/961.c \
src/962.c \
src/963.c \
src/964.c \
src/965.c \
src/966.c \
src/967.c \
src/968.c \
src/969.c \
src/97.c \
src/970.c \
src/971.c \
src/972.c \
src/973.c \
src/974.c \
src/975.c \
src/976.c \
src/977.c \
src/978.c \
src/979.c \
src/98.c \
src/980.c \
src/981.c \
src/982.c \
src/983.c \
src/984.c \
src/985.c \
src/986.c \
src/987.c \
src/988.c \
src/989.c \
src/99.c \
src/990.c \
src/991.c \
src/992.c \
src/993.c \
src/994.c \
src/995.c \
src/996.c \
src/997.c \
src/998.c \
src/999.c

LOCAL_SRCS := \
$(MYLIB_PART1_SRCS) \
$(MYLIB_PART2_SRCS) \
$(MYLIB_PART3_SRCS) \
$(MYLIB_PART4_SRCS) \
#*********************************************************************************************************
# Header file search path (eg. LOCAL_INC_PATH := -I"Your header files search path")
#*********************************************************************************************************
LOCAL_INC_PATH := 

#*********************************************************************************************************
# Pre-defined macro (eg. -DYOUR_MARCO=1)
#*********************************************************************************************************
LOCAL_DSYMBOL := 

#*********************************************************************************************************
# Compiler flags
#*********************************************************************************************************
LOCAL_CFLAGS   := 
LOCAL_CXXFLAGS := 

#*********************************************************************************************************
# Depend library (eg. LOCAL_DEPEND_LIB := -la LOCAL_DEPEND_LIB_PATH := -L"Your library search path")
#*********************************************************************************************************
LOCAL_DEPEND_LIB := 
LOCAL_DEPEND_LIB_PATH := 

#*********************************************************************************************************
# C++ config
#*********************************************************************************************************
LOCAL_USE_CXX        := no
LOCAL_USE_CXX_EXCEPT := no

#*********************************************************************************************************
# Code coverage config
#*********************************************************************************************************
LOCAL_USE_GCOV := no

#*********************************************************************************************************
# OpenMP config
#*********************************************************************************************************
LOCAL_USE_OMP := no

#*********************************************************************************************************
# User link command
#*********************************************************************************************************
LOCAL_PRE_LINK_CMD := 
LOCAL_POST_LINK_CMD := 
LOCAL_PRE_STRIP_CMD := 
LOCAL_POST_STRIP_CMD := 

#*********************************************************************************************************
# Depend target
#*********************************************************************************************************
LOCAL_DEPEND_TARGET := 

include $(LIBRARY_MK)
#*********************************************************************************************************
# Objects
#*********************************************************************************************************
MYLIB_PART1_OBJS := $(addprefix $(OBJPATH)/$(LOCAL_TARGET_NAME)/, $(addsuffix .o, $(basename $(MYLIB_PART1_SRCS))))
MYLIB_PART2_OBJS := $(addprefix $(OBJPATH)/$(LOCAL_TARGET_NAME)/, $(addsuffix .o, $(basename $(MYLIB_PART2_SRCS))))
MYLIB_PART3_OBJS := $(addprefix $(OBJPATH)/$(LOCAL_TARGET_NAME)/, $(addsuffix .o, $(basename $(MYLIB_PART3_SRCS))))
MYLIB_PART4_OBJS := $(addprefix $(OBJPATH)/$(LOCAL_TARGET_NAME)/, $(addsuffix .o, $(basename $(MYLIB_PART4_SRCS))))

#*********************************************************************************************************
# Override link command
#*********************************************************************************************************
$($(target)_SO): $($(target)_OBJS) $($(target)_DEPEND_TARGET) $($(target)_A)
		@rm -f $@
		$(__PRE_LINK_CMD)
		$(__LD) $(__CPUFLAGS) $(ARCH_PIC_LDFLAGS) -Wl,--whole-archive $(addsuffix .a, $(basename $@ )) -Wl,--no-whole-archive $(__LIBRARIES) -o $@ 
		$(__POST_LINK_CMD)

#*********************************************************************************************************
# Override make archive object files
#*********************************************************************************************************
$($(target)_A): $($(target)_OBJS)
		@rm -f $@
		$(__PRE_LINK_CMD)
		$(AR) $(TOOLCHAIN_AR_FLAGS) $@ $(MYLIB_PART1_OBJS)   
		$(AR) $(TOOLCHAIN_AR_FLAGS) $@ $(MYLIB_PART2_OBJS)   
		$(AR) $(TOOLCHAIN_AR_FLAGS) $@ $(MYLIB_PART3_OBJS)
		$(AR) $(TOOLCHAIN_AR_FLAGS) $@ $(MYLIB_PART4_OBJS)
		$(__POST_LINK_CMD)

#*********************************************************************************************************
# End
#*********************************************************************************************************
```

修改部分：

![SylixOS移植常见问题](./Image/SylixOS移植常见问题_5.png)

![SylixOS移植常见问题](./Image/SylixOS移植常见问题_3.png)

![SylixOS移植常见问题](./Image/SylixOS移植常见问题_4.png)

![SylixOS移植常见问题](./Image/SylixOS移植常见问题_6.png)

同理，如果是APP编译，将$($(target)_SO)改为$($(target)_EXE)，再对应修改$(__LD)后的参数即可。