# SylixOS字符设备驱动开发

## SylixOS文件基本操作

文件的基本操作包括打开文件、关闭文件、读文件和写文件。打开文件通过调用open函数来实现：

```
int fd;
fd = open("demo.txt", O_RDWR | O_CREAT);
if (fd < 0) {
    printf("open file fail.\r\n");
    return -1;
}
```

O_RDWR表示以可读可写的方式打开文件，O_CREAT表示如果文件不存在则创建新文件。文件打开成功后会返回一个文件描述符，后续对文件的操作都是基于这个文件描述符来进行的。关闭文件通过调用close函数来实现：

```
close(fd);
```

写文件通过write函数来实现：

```
ssize_t size;
char buffer_w[10] = {'1', '2', '3', '4'};
    
size = write(fd, buffer_w, 4);
if (size < 0) {
    printf("write file fail.\r\n");
    return -1;
}
```

write的返回值表示写成功的字节数。读文件通过read函数来实现：

```
char buffer_r[10] = {0,};

size = read(fd, buffer_r, size);
if (size < 0) {
    printf("read file fail.\r\n");
    return -1;
} else if (!size) {
    printf("read file end.\r\n");
}
```

read的返回值表示读成功的字节数，为0表示未读到数据或者读到文件末尾。通过一个demo来测试文件的基本操作：

```
#include <stdio.h>

int main (int argc, char **argv)
{
    int fd;
    ssize_t size;
    char buffer_w[10] = {'1', '2', '3', '4'};
    char buffer_r[10] = {0,};

    fd = open("demo.txt", O_RDWR | O_CREAT);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    size = write(fd, buffer_w, 4);
    if (size < 0) {
        printf("write file fail.\r\n");
        return -1;
    }

    close(fd);

    fd = open("demo.txt", O_RDWR | O_CREAT);
    if (fd < 0) {
        printf("open file fail2.\r\n");
        return -1;
    }

    size = read(fd, buffer_r, size);
    if (size < 0) {
        printf("read file fail.\r\n");
        return -1;
    } else if (!size) {
        printf("read file end.\r\n");
    }

    printf("read : %s\r\n", buffer_r);

    close(fd);

    return  (0);
}
```

测试结果：

```
[root@sylixos:/apps/app_demo1]# ./app_demo1
read : 1234
[root@sylixos:/apps/app_demo1]#
```

## SylixOS驱动模块加载和卸载

​		通过IDE新建SylixOS Kernel Module工程，这种方式将驱动编译为xxx.ko模块的方式使用，ko就是kernel object的缩写，在系统启动后通过动态加载的形式加载驱动模块，使用方式类似linux下的模块。

​		由于驱动模块属于内核，在驱动中可能会调用到一些只能在内核中使用的接口，所以必须在源文件最开始定义 ***__SYLIXOS_KERNEL\*** 这个宏，这样就可以使用一些只能由内核使用的接口：

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
```

驱动的入口函数是 ***module_init\*** ：

```
int module_init (void)
{
    printk("hello_module init!\n");

    return 0;
}
```

驱动的退出函数是 ***module_exit\*** ：

```
void module_exit (void)
{
    printk("hello_module exit!\n");
}
```

驱动模块默认是放在系统中的 ***/lib/modules\*** 目录下，加载驱动使用 ***insmod\*** 命令：

```
[root@sylixos:/root]# insmod /lib/modules/driver_demo2.ko
hello_module init!
module /lib/modules/driver_demo2.ko register ok, handle: 0x365c7e0
[root@sylixos:/root]#
```

可以通过 ***lsmod\*** 命令查看当前系统中已经加载哪些驱动模块：

```
[root@sylixos:/root]# lsmod

            NAME           HANDLE   TYPE  GLB   BASE     SIZE   SYMCNT
------------------------- -------- ------ --- -------- -------- ------
VPROC: kernel             pid:   0 TOTAL MEM: 49152
+ xsiipc.ko               036688d0 KERNEL YES 10008000     49a4     14
+ xinput.ko               03668ea8 KERNEL YES 10006000     18b4      1
+ driver_demo2.ko         0365c7e0 KERNEL YES 10005000       68      1

total modules: 3
[root@sylixos:/root]#
```

使用 ***rmmod\*** 命令卸载驱动模块：

```
[root@sylixos:/root]# rmmod /lib/modules/driver_demo2.ko
hello_module exit!
module /lib/modules/driver_demo2.ko unregister ok.
[root@sylixos:/root]#
```

**附 driver_demo2源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>

int module_init (void)
{
    printk("hello_module init!\n");

    return 0;
}

void module_exit (void)
{
    printk("hello_module exit!\n");
}
```

## SylixOS字符设备操作之open和close

​		在第一节我们学习过在应用层如何操作普通的文件，就是通过open、close、read、write这些接口，普通文件是存放在具体的存储设备上的，所以从某种角度来说，我们是通过上面这四个接口操作了存储设备。

​		SylixOS采用Unix中万物皆文件的设计思想，将不同的设备都抽象成一个个文件，当然这个文件肯定不是普通的文件，这些文件叫做设备文件，在应用层同样通过open、close、read、write等接口来操作这些设备文件，这些接口在底层驱动中都有对应的接口。也就是说应用层调用open打开一个设备文件，最终会调用到这个设备底层驱动中的open函数，底层驱动这些所有操作函数构成了这个设备驱动所支持功能的集合，在SylixOS中，使用 ***struct file_operations\*** 数据结构来表示这样的操作集：

```
typedef struct file_operations {
    struct module              *owner;
    
    long                      (*fo_create)();                           /*  DEVENTRY_pfuncDevCreate     */
    int                       (*fo_release)();                          /*  DEVENTRY_pfuncDevDelete     */
    
    long                      (*fo_open)();                             /*  DEVENTRY_pfuncDevOpen       */
    int                       (*fo_close)();                            /*  DEVENTRY_pfuncDevClose      */
    
    ssize_t                   (*fo_read)();                             /*  DEVENTRY_pfuncDevRead       */
    ssize_t                   (*fo_read_ex)();                          /*  DEVENTRY_pfuncDevReadEx     */
    
    ssize_t                   (*fo_write)();                            /*  DEVENTRY_pfuncDevWrite      */
    ssize_t                   (*fo_write_ex)();                         /*  DEVENTRY_pfuncDevWriteEx    */
    
    int                       (*fo_ioctl)();                            /*  DEVENTRY_pfuncDevIoctl      */
    int                       (*fo_select)();                           /*  DEVENTRY_pfuncDevSelect     */
    
    int                       (*fo_lock)();                             /*  not support now             */
    off_t                     (*fo_lseek)();                            /*  DEVENTRY_pfuncDevLseek      */
    
    int                       (*fo_fstat)();                            /*  DEVENTRY_pfuncDevFstat      */
    int                       (*fo_lstat)();                            /*  DEVENTRY_pfuncDevLstat      */
    
    int                       (*fo_symlink)();                          /*  DEVENTRY_pfuncDevSymlink    */
    ssize_t                   (*fo_readlink)();                         /*  DEVENTRY_pfuncDevReadlink   */
    
    int                       (*fo_mmap)();                             /*  DEVENTRY_pfuncDevMmap       */
    int                       (*fo_unmap)();                            /*  DEVENTRY_pfuncDevUnmmap     */
    
    ULONG                       fo_pad[16];                             /*  reserve                     */
} FILE_OPERATIONS;
```

open和close函数是所有驱动都需要实现的，其他的操作函数根据驱动的具体功能来选择实现。

SylixOS驱动使用 ***LW_DEV_HDR\*** 数据结构来表示一个设备实例：

```
typedef struct {
    LW_LIST_LINE               DEVHDR_lineManage;                       /*  设备头管理链表              */
    UINT16                     DEVHDR_usDrvNum;                         /*  主设备号                    */
    UINT16                     DEVHDR_usDevNum;                         /*  子设备号                    */
    PCHAR                      DEVHDR_pcName;                           /*  设备名称                    */
    UCHAR                      DEVHDR_ucType;                           /*  设备 dirent d_type          */
    atomic_t                   DEVHDR_atomicOpenNum;                    /*  打开的次数                  */
    PVOID                      DEVHDR_pvReserve;                        /*  保留                        */
} LW_DEV_HDR;
```

一个设备实例就对应着一个具体的设备文件，SylixOS中的设备文件都创建在 ***/dev\*** 目录下：

```
[root@sylixos:/root]# cd /dev/
[root@sylixos:/dev]# ll
srwxrwxrwx root     root     Sat Jan 01 08:00:00 2000      0 B, log
crw------- root     root     Sat Jan 01 08:00:00 2000      0 B, netbd
crw------- root     root     Sat Jan 01 08:00:00 2000      0 B, netbr
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           net/
srw-rw-rw- root     root     Sat Jan 01 08:00:00 2000      0 B, socket
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000   4096 B, netevent
crw-rw-rw- root     root     Sat Jan 01 08:00:00 2000    750KB, fb0
crw-rw-rw- root     root     Sat Jan 01 08:00:00 2000      0 B, ttyS0
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000      0 B, urandom
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000      0 B, random
drw-rw-rw- root     root     Sat Jan 01 08:00:00 2000           shm/
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000   4096 B, hotplug
crw-rw-rw- root     root     Sat Jan 01 08:00:00 2000      0 B, epollfd
drw-rw-rw- root     root     Sat Jan 01 08:00:00 2000           gpiofd/
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000      0 B, signalfd
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000      0 B, hstimerfd
cr--r--r-- root     root     Sat Jan 01 08:00:00 2000      0 B, timerfd
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           semfd/
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           bmsg/
crw-rw-rw- root     root     Sat Jan 01 08:00:00 2000      0 B, eventfd
crw-rw-rw- root     root     Sat Jan 01 08:00:00 2000      0 B, zero
crw-rw-rw- root     root     Sat Jan 01 08:00:00 2000      0 B, null
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           blk/
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           input/
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           pipe/
drwxr-xr-- root     root     Sat Jan 01 08:00:00 2000           pty/
      total items: 26
[root@sylixos:/dev]#
```

一般的做法是在驱动中定义一个全局的设备实例变量：

```
LW_DEV_HDR demo_dev;
```

这个设备实例会作为驱动中open函数的第一个参数：

```
static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    return (long)dev;
}
```

> 注意：SylixOS中的IO系统有两种类型，分别是ORIG和NEW_1两种类型，对应的底层驱动也分为ORIG型驱动和NEW_1型驱动。ORIG型结构主要是为了兼容VxWorks系统的IO框架和驱动，这也是SylixOS早期使用的IO框架，而NEW_1型是为了兼容linux的驱动。本次教程先以ORIG型为例进行讲解，后面会介绍NEW_1型驱动和ORIG型驱动有哪些差异。

open函数的各参数意思：

- dev：设备实例数据结构地址。
- name：在ORIG型驱动中这个指针为设备名尾指针，无作用。
- flag：应用层打开文件的标志集合，比如只读方式打开的O_RDONLY或者只写方式打开的O_WRONLY等。
- mode：应用层打开文件的权限，这个权限意义和linux下文件的权限意义一样，如果打开文件时没有指定权限，则使用默认权限***0644\***（八进制表示）。
- open函数执行失败返回***PX_ERROR\***，**ORIG型驱动open函数可以返回自定义数据结构的地址，这个地址会作为其他操作函数比如close、read等的第一个入参**。

有了open函数相对应的有一个close函数：

```
static int demo_close(LW_DEV_HDR *dev)
{
    printk("%s.\r\n", __func__);

    return ERROR_NONE;
}
```

close函数成功返回ERROR_NONE，失败返回PX_ERROR。

有了open函数和close函数之后，就可以定义一个驱动操作集变量并初始化这个变量：

```
static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
};
```

有了设备实例变量和驱动操作集变量后，我们就可以进行驱动注册和设备文件创建，这个将在下一节讲解。

**附driver_demo3源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>

LW_DEV_HDR demo_dev;

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    return (long)dev;
}

static int demo_close(LW_DEV_HDR *dev)
{
    printk("%s.\r\n", __func__);

    return ERROR_NONE;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
};

int module_init (void)
{
    printk("hello_module init!\n");

    return 0;
}

void module_exit (void)
{
    printk("hello_module exit!\n");
}
```

## SylixOS注册驱动和创建设备文件

​		上一节我们完成了驱动中的open和close函数功能，并使用这两个函数初始化了***demo_fops\*** 这个数据结构，现在我们将驱动注册到系统中，在SylixOS中这是通过调用***iosDrvInstallEx\*** 接口来实现的，这个接口其实就是***API_IosDrvInstallEx\*** 接口的宏定义，用宏重定义为***iosDrvInstallEx\*** 是为了和VxWorks的驱动相兼容，在SylixOS中，所有原生接口都是***API_xxx\*** 这种格式。

***API_IosDrvInstallEx\***接口的函数原型：

```
INT  API_IosDrvInstallEx (struct file_operations  *pfileop);
```

成功返回一个驱动索引号，失败返回PX_ERROR。在SylixOS中，所有注册的驱动都是存放在一张全局的驱动表（数组）中，每次注册驱动时，都会在当前表中找到一个可用的位置给注册的驱动使用，并返回这个位置在表中的索引（数组下标）。 ***调用此接口默认创建的是ORIG型驱动\***。

一般在驱动中会定义一个全局变量用来保存驱动索引号：

```
int drv_index;

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    return 0;
}
```

这个驱动索引号会在驱动卸载的时候用到，卸载驱动所使用的接口是***iosDrvRemove\***，***iosDrvRemove\*** 是***API_IosDrvRemove\*** 的宏定义，函数原型：

```
ULONG  API_IosDrvRemove (INT  iDrvNum, BOOL  bForceClose);
```

iDrvNum就是驱动注册时的索引号，bForceClose表示是否强制删除驱动。成功返回ERROR_NONE，失败返回错误号。

在linux驱动中我们需要填写驱动所遵守的开源协议以及驱动的作者等信息，同样的，在SylixOS中也可以填入这些信息：

```
DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
DRIVER_AUTHOR(drv_index, "GeWenBin");
DRIVER_DESCRIPTION(drv_index, "demo driver.");
```

驱动注册完成之后，我们就可以创建设备文件，这是通过***iosDevAdd\*** 接口实现的，***iosDevAdd\*** 是***API_IosDevAdd\*** 的宏定义，函数原型：

```
ULONG  API_IosDevAdd (PLW_DEV_HDR    pdevhdrHdr,
                      CPCHAR         pcName,
                      INT            iDrvNum);
```

pdevhdrHdr就是设备实例变量的地址，pcName是设备文件的名称，比如“/dev/demo”，iDrvNum就是注册驱动时返回的索引号。成功返回ERROR_NONE，失败返回错误码。

删除设备文件使用***iosDevDelete\***接口，***iosDevDelete\***是***API_IosDevDelete\*** 的宏定义，函数原型：

```
VOID  API_IosDevDelete (PLW_DEV_HDR    pdevhdrHdr);
```

这个接口只有一个入参，就是设备实例变量的地址。

在驱动模块卸载时，一般是先删除设备文件再卸载设备驱动：

```
void module_exit (void)
{
    iosDevDelete(&demo_dev);
    iosDrvRemove(drv_index, TRUE);
}
```

**附driver_demo4源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>

int drv_index;
LW_DEV_HDR demo_dev;

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    return (long)dev;
}

static int demo_close(LW_DEV_HDR *dev)
{
    printk("%s.\r\n", __func__);

    return ERROR_NONE;
} 

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev, "/dev/demo", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev);
    iosDrvRemove(drv_index, TRUE);
}
```

## SylixOS驱动使用和信息查询

在上一节我们学习了如何注册驱动和创建设备文件，现在我们来学习驱动注册后如何在应用层使用驱动和如何查询驱动在系统中的相关信息。

应用层使用设备文件很简单，就是先使用open接口以读写方式打开，然后打印一句话再关闭：

```
#include <stdio.h>

int main (int argc, char **argv)
{
    int fd;

    fd =  open("/dev/demo", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    printf("Hello demo.\r\n");

    close(fd);

    return  (0);
}
```

> 通过IDE中的SylixOS App工程来创建一个应用程序。

首先通过insmod命令加载驱动，然后运行app，终端输出打印信息：

```
[root@sylixos:/root]# insmod /lib/modules/driver_demo4.ko
module /lib/modules/driver_demo4.ko register ok, handle: 0x365c7e0
[root@sylixos:/root]#
[root@sylixos:/root]# cd /apps/app_demo4
[root@sylixos:/apps/app_demo4]# ./app_demo4
demo_open .
open read/write.
file permission: 644.
Hello demo.
demo_close.
[root@sylixos:/apps/app_demo4]#
```

可以看出，运行程序后，输出的前3条信息都是驱动的open函数打印的，第4条信息是应用程序自身打印的，最后一条信息是驱动的close函数打印的。

上面这种方式是通过一个应用进程的方式打开和关闭一个设备文件，我们还可以直接在命令行使用***open\*** 打开一个设备文件：

```
[root@sylixos:/root]# open /dev/demo
demo_open .
open read only.
file permission: 644.
_IosFileIoctl() error: unknown request.
in thread "t_tshell" context.
open file return: 8 dev 250000 inode 0 size 0
[root@sylixos:/root]#
```

通过这种方式打开的文件是属于内核的，而不是属于某个进程的。通过上面的打印信息我们还知道内核是以只读的方式打开文件的，另外还报了一个ioctl相关的错误，因为我们驱动中还未实现ioctl函数，最后的***open file return:\*** 后面的8表示文件描述符id为8。我们可以通过***files\*** 命令来查看内核一共打开了哪些文件：

```
[root@sylixos:/root]# files
kernel filedes show (process filedes in /proc/${pid}/filedes) >>
 fd abn name                       type   drv
  3     /dev/ttyS0                 orig    18 GLB STD_IN GLB STD_OUT GLB STD_ERR
  4     /dev/socket                socket  32
  5     /dev/socket                socket  32
  6     /dev/hotplug               orig    12
  7     /dev/input/touch0          new_1   29
  8     /dev/demo                  orig    37
[root@sylixos:/root]#
```

通过***close\*** 命令关闭内核打开的文件，如下所示：

```
[root@sylixos:/root]# close 8
demo_close.
[root@sylixos:/root]#
```

通过***devs\*** 查看当前系统所有设备文件信息，如下所示：

```
[root@sylixos:/root]# devs
device show (minor device) >>
drv dev open name
 37   0    0 /dev/demo
 36   1    0 /dev/input/xmse
 36   0    0 /dev/input/xkbd
 35   0    0 /dev/netbd
 34   0    0 /dev/netbr
 33   0    0 /dev/net/vnd
 32   0    0 /dev/socket
 31   0    0 /dev/netevent
 24   0    0 /ram
 29   0    1 /dev/input/touch0
 30   0    0 /dev/fb0
 23   0    0 /media/sdcard0
 10   0    0 /dev/blk/sdcard-0
 27   0    0 /yaffs2
 18   0    1 /dev/ttyS0
 15   1    0 /dev/urandom
 15   0    0 /dev/random
 14   0    0 /dev/shm
 13   0    0 /proc
 12   0    1 /dev/hotplug
 11   0    0 /dev/epollfd
  9   0    0 /dev/gpiofd
  8   0    0 /dev/signalfd
  7   0    0 /dev/hstimerfd
  6   0    0 /dev/timerfd
  5   0    0 /dev/semfd
  4   0    0 /dev/bmsg
  3   0    0 /dev/eventfd
  1   0    0 /dev/zero
  0   0    0 /dev/null
  2   0    0 /
[root@sylixos:/root]#
```

drv表示驱动索引号，dev表示这是驱动对应的第几个设备实例，比如上面的36号驱动有两个设备文件，***/dev/input/xkbd\*** 和 ***/dev/input/xmse\*** 分别对应36号驱动的第0个和第1个设备。open表示当前设备被打开几次。

> 一个设备文件被打开的次数是在驱动在open和close时通过增减设备实例变量中的引用计数来实现的，这是留给驱动开发者来做的，而不是系统来做的。

通过***drvlics\*** 命令来查看所有驱动遵循的开源协议和开发者信息：

```
[root@sylixos:/root]# drvlics
driver License show (major device) >>

DRV          DESCRIPTION                 AUTHOR                 LICENSE
--- ------------------------------ -------------------- ------------------------
  0 null device driver.            Han.hui              GPL->Ver 2.0
  1 zero device driver.            Han.hui              GPL->Ver 2.0
  2 rootfs driver.                 Han.hui              GPL->Ver 2.0
  3 eventfd driver.                Han.hui              GPL->Ver 2.0
  4 block message driver.          Han.hui              GPL->Ver 2.0
  5 semaphore file driver.         Han.hui              GPL->Ver 2.0
  6 timerfd driver.                Han.hui              GPL->Ver 2.0
  7 hstimerfd driver.              Han.hui              GPL->Ver 2.0
  8 signalfd driver.               Han.hui              GPL->Ver 2.0
  9 gpiofd driver.                 Han.hui              GPL->Ver 2.0
 10 blk io driver.                 Han.hui              GPL->Ver 2.0
 11 epoll driver.                  Han.hui              GPL->Ver 2.0
 12 hotplug message driver.        Han.hui              GPL->Ver 2.0
 13 procfs driver.                 Han.hui              GPL->Ver 2.0
 14 share memory driver.           Han.hui              GPL->Ver 2.0
 15 random number generator.       Han.hui              GPL->Ver 2.0
 16 pty driver (device node).      Han.hui              GPL->Ver 2.0
 17 pty driver (host node).        Han.hui              GPL->Ver 2.0
 18 tty driver.                    Han.hui              GPL->Ver 2.0
 19 VxWorks memory device driver.  Han.hui              GPL->Ver 2.0
 20 VxWorks pipe driver.           Han.hui              GPL->Ver 2.0
 21 stream pipe driver.            Han.hui              GPL->Ver 2.0
 22 FAT12/16/32 driver.            Han.hui              Dual BSD/GPL->Ver 1.0
 23 tpsFs driver.                  Jiang.Taijin         GPL->Ver 2.0
 24 ramfs driver.                  Han.hui              GPL->Ver 2.0
 25 romfs driver.                  Han.hui              GPL->Ver 2.0
 26 NFSv3 driver.                  Han.hui              Dual BSD/GPL->Ver 1.0
 27 yaffs2 driver.                 Han.hui              GPL->Ver 2.0
 28 CAN Bus driver.                Wang.feng            GPL->Ver 2.0
 29 virt touch driver.             Zhang.Xu             Dual BSD/GPL->Ver 1.0
 30 graph frame buffer driver.     Han.hui              GPL->Ver 2.0
 31 net event message driver.      Han.hui              Dual BSD/GPL->Ver 1.0
 32 socket driver v2.0             Han.hui              GPL->Ver 2.0
 33 virtual net device driver.     Han.hui              Dual GPL->Ver 2.0
 34 net bridge management driver.  Han.hui              Dual BSD/GPL->Ver 1.0
 35 net bonding management driver. Han.hui              Dual BSD/GPL->Ver 1.0
 36 xinput driver.                 Han.hui              GPL->Ver 2.0
 37 demo driver.                   GeWenBin             GPL->Ver 2.0

[root@sylixos:/root]#
```

可以看到最后的37号驱动是我们加载的demo驱动。

## SylixOS驱动统计打开次数

在上一节我们说驱动被打开的次数是由驱动开发者维护的，其实在设备实例数据结构中就有一个成员表示设备被打开的次数：

```
typedef struct {
    LW_LIST_LINE               DEVHDR_lineManage;                       /*  设备头管理链表              */
    UINT16                     DEVHDR_usDrvNum;                         /*  主设备号                    */
    UINT16                     DEVHDR_usDevNum;                         /*  子设备号                    */
    PCHAR                      DEVHDR_pcName;                           /*  设备名称                    */
    UCHAR                      DEVHDR_ucType;                           /*  设备 dirent d_type          */
    atomic_t                   DEVHDR_atomicOpenNum;                    /*  打开的次数                  */
    PVOID                      DEVHDR_pvReserve;                        /*  保留                        */
} LW_DEV_HDR;
```

***DEVHDR_atomicOpenNum\*** 就表示设备打开的次数，在open的时候调用***LW_DEV_INC_USE_COUNT\*** 接口增加打开次数，这是一个inline函数，函数原型：

```
static LW_INLINE INT  LW_DEV_INC_USE_COUNT(PLW_DEV_HDR  pdevhdrHdr)
{
    return  ((pdevhdrHdr) ? (API_AtomicInc(&pdevhdrHdr->DEVHDR_atomicOpenNum)) : (PX_ERROR));
}
```

可以看出就是通过原子增加计数的接口***API_AtomicInc\*** 增加了打开次数。同样的，在close的时候调用***LW_DEV_DEC_USE_COUNT\*** 接口减少打开次数，函数原型：

```
static LW_INLINE INT  LW_DEV_DEC_USE_COUNT(PLW_DEV_HDR  pdevhdrHdr)
{
    return  ((pdevhdrHdr) ? (API_AtomicDec(&pdevhdrHdr->DEVHDR_atomicOpenNum)) : (PX_ERROR));
}
```

可以看出其调用了原子减少计数的接口***API_AtomicDec\*** 减少了打开次数。

在驱动的open和close函数中分别添加这两个接口后我们来看看设备的打开次数是不是确实会改变。加载驱动后在命令行open两次“/dev/demo”之后，使用***devs\*** 查看驱动的打开次数：

```
[root@sylixos:/root]# devs
device show (minor device) >>
drv dev open name
 37   0    2 /dev/demo
 36   1    0 /dev/input/xmse
 36   0    0 /dev/input/xkbd
 35   0    0 /dev/netbd
 34   0    0 /dev/netbr
 33   0    0 /dev/net/vnd
 32   0    0 /dev/socket
 31   0    0 /dev/netevent
 24   0    0 /ram
 29   0    1 /dev/input/touch0
 30   0    0 /dev/fb0
 23   0    0 /media/sdcard0
 10   0    0 /dev/blk/sdcard-0
 27   0    0 /yaffs2
 18   0    1 /dev/ttyS0
 15   1    0 /dev/urandom
 15   0    0 /dev/random
 14   0    0 /dev/shm
 13   0    0 /proc
 12   0    1 /dev/hotplug
 11   0    0 /dev/epollfd
  9   0    0 /dev/gpiofd
  8   0    0 /dev/signalfd
  7   0    0 /dev/hstimerfd
  6   0    0 /dev/timerfd
  5   0    0 /dev/semfd
  4   0    0 /dev/bmsg
  3   0    0 /dev/eventfd
  1   0    0 /dev/zero
  0   0    0 /dev/null
  2   0    0 /
[root@sylixos:/root]#
```

可以看到37号驱动的open次数确实是两次，这时close一个刚刚打开的设备文件后再次查看信息：

```
[root@sylixos:/root]# devs
device show (minor device) >>
drv dev open name
 37   0    1 /dev/demo
 36   1    0 /dev/input/xmse
 36   0    0 /dev/input/xkbd
 35   0    0 /dev/netbd
 34   0    0 /dev/netbr
 33   0    0 /dev/net/vnd
 32   0    0 /dev/socket
 31   0    0 /dev/netevent
 24   0    0 /ram
 29   0    1 /dev/input/touch0
 30   0    0 /dev/fb0
 23   0    0 /media/sdcard0
 10   0    0 /dev/blk/sdcard-0
 27   0    0 /yaffs2
 18   0    1 /dev/ttyS0
 15   1    0 /dev/urandom
 15   0    0 /dev/random
 14   0    0 /dev/shm
 13   0    0 /proc
 12   0    1 /dev/hotplug
 11   0    0 /dev/epollfd
  9   0    0 /dev/gpiofd
  8   0    0 /dev/signalfd
  7   0    0 /dev/hstimerfd
  6   0    0 /dev/timerfd
  5   0    0 /dev/semfd
  4   0    0 /dev/bmsg
  3   0    0 /dev/eventfd
  1   0    0 /dev/zero
  0   0    0 /dev/null
  2   0    0 /
[root@sylixos:/root]#
```

可以看到37号驱动的open次数变成了1。

**附driver_demo6源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>

int drv_index;
LW_DEV_HDR demo_dev;

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(dev);

    return (long)dev;
}

static int demo_close(LW_DEV_HDR *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(dev);

    return ERROR_NONE;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev, "/dev/demo", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev);
    iosDrvRemove(drv_index, TRUE);
}
```

## SylixOS驱动创建多个设备文件

在前面几节教程中我们在驱动中是直接定义一个设备实例的全局变量，但是实际项目中我们有时候需要定义自己的数据结构，用来将一些私有数据保存在其中，这时就可以按如下方式定义数据结构：

```
typedef struct _demo_dev {
    LW_DEV_HDR dev;
    void *priv;
} demo_dev_t;
```

这里将***LW_DEV_HDR\*** 设备实例作为第一个成员，后面我们会看到这样做带来的好处。

另外在实际项目中，有时候我们需要创建多个同类型的设备而使用同一个驱动。比如摄像头设备一般名为“/dev/video”，如果系统中有两个摄像头，我们希望创建两个设备文件分别为“/dev/video0”和“/dev/video1”，那么这种情况下驱动层该如何处理?

首先我们用上面的自定义数据结构定义一个有2个元素的数组：

```
demo_dev_t demo_dev[2];
```

然后创建设备文件的时候我们使用***iosDevAdd\*** 接口创建两个设备文件：

```
ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
if (ret != ERROR_NONE) {
    printk("device add fail.\r\n");
    iosDrvRemove(drv_index, TRUE);
    return -1;
}

ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
if (ret != ERROR_NONE) {
    printk("device1 add fail.\r\n");
    iosDevDelete(&demo_dev[0].dev);
    iosDrvRemove(drv_index, TRUE);
    return -1;
}
```

同样驱动卸载的时候我们需要删除这两个设备文件：

```
iosDevDelete(&demo_dev[0].dev);
iosDevDelete(&demo_dev[1].dev);
```

其次open函数原来返回的是设备实例变量的地址，现在要改成返回自定义数据的地址，由于我们是将设备实例变量放在自定义数据第一个成员，**所以这两个地址值是一样的**，这样我们直接将open函数的第一个参数转换成自定义数据类型指针即可：

```
static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;

    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}
```

如果自定义数据结构第一个成员不是***LW_DEV_HDR\*** 设备实例的话，就需要通过类似linux下的***container_of\*** 这类辅助宏获得自定义数据结构首地址。

注册驱动后，通过***devs\*** 查看驱动信息：

```
[root@sylixos:/root]# devs
device show (minor device) >>
drv dev open name
 37   1    0 /dev/demo1
 37   0    0 /dev/demo0
 36   1    0 /dev/input/xmse
 36   0    0 /dev/input/xkbd
 35   0    0 /dev/netbd
 34   0    0 /dev/netbr
 33   0    0 /dev/net/vnd
 32   0    0 /dev/socket
 31   0    0 /dev/netevent
 24   0    0 /ram
 29   0    1 /dev/input/touch0
 30   0    0 /dev/fb0
 23   0    0 /media/sdcard0
 10   0    0 /dev/blk/sdcard-0
 27   0    0 /yaffs2
 18   0    1 /dev/ttyS0
 15   1    0 /dev/urandom
 15   0    0 /dev/random
 14   0    0 /dev/shm
 13   0    0 /proc
 12   0    1 /dev/hotplug
 11   0    0 /dev/epollfd
  9   0    0 /dev/gpiofd
  8   0    0 /dev/signalfd
  7   0    0 /dev/hstimerfd
  6   0    0 /dev/timerfd
  5   0    0 /dev/semfd
  4   0    0 /dev/bmsg
  3   0    0 /dev/eventfd
  1   0    0 /dev/zero
  0   0    0 /dev/null
  2   0    0 /
[root@sylixos:/root]#
```

可以看到37号驱动下有两个设备文件。

**附driver_demo7源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;

    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);
}
```

## SylixOS设备操作之ioctl

前面我们学习过驱动open和close操作，open一般用于申请和初始化一些资源，close用于回收资源。如果设备还有一些其他功能，比如可以设置某些参数，获取某些属性等，应用层我们可以使用ioctl接口来实现相应的这些操作。

要想在应用层使用ioctl接口，驱动层必须先实现ioctl操作：

```
static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    printk("%s %x.\r\n", __func__, cmd);

    return ERROR_NONE;
}
```

***cmd\*** 表示是哪种命令，通过不同的命令来执行不同的功能，***arg\*** 表示一个命令对应的参数，可能表示一个具体的值或者是一个数据结构的地址，也可能不需要关心。命令可以分为两大类，一类是系统预留的，比如***FIOFSTATGET\***、***FIOTRUNC\*** 等等；一类是用户自定义的，起始命令从***FIOUSRFUNC\*** 开始。函数成功返回ERROR_NONE，失败返回PX_ERROR。

每个驱动都必须实现的一个ioctl系统命令是***FIOFSTATGET\***，这个命令用于获得设备文件的一些基本状态信息，比如文件权限、设备类型、大小等等：

```
static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    struct stat *pstat;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;
        
    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}
```

> 由于SylixOS使用的是类似VxWorks5.5那种大平板地址设计，并不存在硬件上的内核权限和用户权限之分，所以驱动层ioctl的arg参数如果表示的是指针，那么驱动可以直接访问而不需要像linux那样从用户空间复制到内核空间，反之亦然。另外没有实现的命令一定要返回PX_ERROR。

加载驱动后通过***ll\*** 命令来查看设备文件的基本状态信息：

```
[root@sylixos:/root]# ll /dev/demo0
demo_open .
open read only.
file permission: 644.
demo_ioctl 26.
demo_close.
crwxrwxrwx root     root     Sat Jan 01 08:00:00 2000      0 B, demo0
      total items: 1
[root@sylixos:/root]#
```

通过设计自定义的命令和数据结构可以实现例如获取驱动版本的功能：

```
/*
 * driver.h
 *
 *  Created on: Dec 7, 2020
 *      Author: Administrator
 */

#ifndef SRC_DRIVER_H_
#define SRC_DRIVER_H_

struct data {
    int version;
};

#define CMD_GET_VERSION (FIOUSRFUNC + 0)

#endif /* SRC_DRIVER_H_ */
```

在***driver.h\*** 中定义驱动所用到的命令和相应数据结构，并在驱动的ioctl中对***CMD_GET_VERSION\*** 做相应的处理：

```
static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}
```

驱动开发者将***driver.h\*** 头文件提供给应用开发人员，应用开发者就可以通过应用层ioctl接口来获取驱动版本：

```
ret = ioctl(fd, CMD_GET_VERSION, &data);
if (ret) {
    printf("CMD_GET_VERSION fail.\r\n");
    close(fd);
    return -1;
}

printf("driver version %x\r\n", data.version);
```

执行app的结果：

```
[root@sylixos:/apps/app_demo8]# ./app_demo8
demo_open .
open read/write.
file permission: 644.
demo_ioctl 2000.
driver version 3
demo_close.
[root@sylixos:/apps/app_demo8]#
```

可以看到应用层获取到了驱动版本号为3。

**附driver_demo8源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include "driver.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;

    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);
}
```

**附app_demo8源码:**

```
#include <stdio.h>
#include "driver.h"

int main (int argc, char **argv)
{
    int fd;
    int ret;
    struct data data;

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    ret = ioctl(fd, CMD_GET_VERSION, &data);
    if (ret) {
        printf("CMD_GET_VERSION fail.\r\n");
        close(fd);
        return -1;
    }

    printf("driver version %x\r\n", data.version);

    close(fd);

    return  (0);
}
```

## SylixOS设备操作之read和write

有些设备可能会和应用层有大量的数据进行传输，比如网卡、硬盘等等，这时候用ioctl来和应用层传输数据就显得不怎么适合了，因为ioctl一般是用来进行对设备进行控制的，而不是大量数据传输，传输大量数据就需要用到read和write接口。

驱动层的read接口函数原型：

```
ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size);
```

***buf\*** 是应用层buffer地址，用来存放从驱动中读取到的数据，***size\*** 表示应用层希望读取的数据大小。函数失败返回PX_ERROR，正常返回读取成功的数据大小，如果没有数据可读，返回0。

> 如果可读数据大小比用户需要读取的大，则函数返回用户需要读取的大小。

我们设计demo_read函数只是简单的复制一段字符串数据给应用层表示读取数据：

```
static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
#define DRIVER_READ_DATA "<data from demo driver>"
    char *rdata = DRIVER_READ_DATA;
    int rdata_len = strlen(DRIVER_READ_DATA);

    printk("%s.\r\n", __func__);

    rdata_len = (size > rdata_len) ? rdata_len : size;
    memcpy(buf, rdata, rdata_len);

    return rdata_len;
}
```

类似的，驱动层的write接口函数原型：

```
ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size);
```

***buf\*** 是应用层buffer地址，用来存放应用层要写的数据，***size\*** 表示应用层希望写入的数据大小。函数失败返回PX_ERROR，正常返回写入成功的数据大小，如果没有数据写入，返回0。

我们设计demo_write函数只是从应用层简单的复制一段字符串数据给驱动层表示写入数据：

```
static ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}
```

应用层首先需要定义分别用于读写的buffer：

```
char rbuffer[30]= {0};
char wbuffer[30] = "<data from app>";
```

写入的数据这里固定为一个字符串。读取数据的处理：

```
ret = read(fd, rbuffer, 30);
if (ret < 0) {
    printf("read data fail.\r\n");
    close(fd);
    return -1;
} else if (ret == 0) {
    printf("read data end.\r\n");
} else {
    printf("read %d data,data is: %s.\r\n", ret, rbuffer);
}
```

> 由于read和write的返回值表示实际读取或者写入的数据大小，这个值可能比你实际想要读取或者写入的大小要小，所以一定要判断返回值，如果不满足的话可能需要发起多次read和write直到满足。

写入数据的处理：

```
ret = write(fd, wbuffer, strlen(wbuffer));
if (ret < 0) {
    printf("write data fail.\r\n");
    close(fd);
    return -1;
} else if (ret == 0) {
    printf("write data end.\r\n");
} else {
    printf("write %d data success.\r\n", ret);
}
```

app执行的结果：

```
[root@sylixos:/apps/app_demo9]# ./app_demo9
demo_open .
open read/write.
file permission: 644.
demo_read.
read 23 data,data is: <data from demo driver>.
demo_write.
write data <data from app> success.
write 15 data success.
demo_close.
[root@sylixos:/apps/app_demo9]#
```

**附driver_demo9源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include <string.h>
#include "driver.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;

    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
#define DRIVER_READ_DATA "<data from demo driver>"
    char *rdata = DRIVER_READ_DATA;
    int rdata_len = strlen(DRIVER_READ_DATA);

    printk("%s.\r\n", __func__);

    rdata_len = (size > rdata_len) ? rdata_len : size;
    memcpy(buf, rdata, rdata_len);

    return rdata_len;
}

static ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
    .fo_read  = demo_read,
    .fo_write = demo_write,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);
}
```

**附app_demo9源码:**

```
#include <stdio.h>
#include <string.h>

int main (int argc, char **argv)
{
    int fd;
    int ret = 0;
    char rbuffer[30]= {0};
    char wbuffer[30] = "<data from app>";

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    ret = read(fd, rbuffer, 30);
    if (ret < 0) {
        printf("read data fail.\r\n");
        close(fd);
        return -1;
    } else if (ret == 0) {
        printf("read data end.\r\n");
    } else {
        printf("read %d data,data is: %s.\r\n", ret, rbuffer);
    }

    ret = write(fd, wbuffer, strlen(wbuffer));
    if (ret < 0) {
        printf("write data fail.\r\n");
        close(fd);
        return -1;
    } else if (ret == 0) {
        printf("write data end.\r\n");
    } else {
        printf("write %d data success.\r\n", ret);
    }

    close(fd);

    return  (0);
}
```

## SylixOS设备操作之阻塞read和write

在实际的外设驱动中，如果需要读取设备数据，设备都需要一定的准备时间，比如读硬盘扇区数据，硬盘需要一段时间准备好数据后，通过中断来通知cpu数据准备好。在数据没准备好之前，驱动主要有三种处理策略：

- 轮询方法：通过一直查询相关寄存器直到确认数据准备好。这种方式编程简单，但是由于是轮询方式，cpu占用率高，降低程序并行执行效率。
- 睡眠线程等待通知：将线程设置为阻塞状态，并在中断处理程序中唤醒阻塞的线程。这种方式编程略微复杂，因为要设置中断处理程序，但是好处是提高了cpu并行执行程序的能力。
- 不处理直接返回：这种方式不做任何处理，直接从驱动返回，这时应用层的代码就需要严格判断read返回值来确定到底有没有读取到数据。

一般驱动read和write操作默认都是阻塞的，如何进行非阻塞read和write我们将在下一节教程中介绍。本小节以read为例讲解阻塞操作，write操作类似不作讲解。另外本节我们在驱动里编写了一个虚拟的外设，同时我们为封装了主要的两个接口来操作这个外设，一个是初始化接口：

```
int vir_device_init(void);
```

成功返回ERROR_NONE，失败返回PX_ERROR，另外还提供了读取外设数据的接口：

```
int vir_device_read (BOOL non_block);
```

***non_block\*** 参数表示是否需要非阻塞读取数据，此虚拟外设每隔3秒钟产生一个新的数据（其实就是将一个变量从1开始每隔3S加一），如果读取的时候数据没有准备好，会根据传入的参数来决定是阻塞等待数据还是直接返回。函数正常返回一个正数数值，失败返回PX_ERROR，如果非阻塞模式没有读取到数据则返回0。

将上一节教程中的read函数进行修改：

```
static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
    int data = -1;

    printk("%s.\r\n", __func__);

    data = vir_device_read(FALSE);

    memcpy(buf, &data, sizeof(int));

    return sizeof(int);
}
```

我们本节学习的是阻塞读取数据，所以***vir_device_read\*** 参数传入的是FALSE，应用层读取数据的流程：

```
while (1) {
    ret = read(fd, &data, 4);
    if (ret < 0) {
        printf("read data fail.\r\n");
        close(fd);
        return -1;
    } else if (ret == 0) {
        printf("read data end.\r\n");
    } else {
        printf("read %d data,data is: %d.\r\n", ret, data);
    }
}
```

正常执行的结果就是每隔3S终端上打印读取到的数据：

```
[root@sylixos:/apps/app_demo10]# ./app_demo10
demo_open .
open read/write.
file permission: 644.
demo_read.
read 4 data,data is: 2.
demo_read.
read 4 data,data is: 3.
demo_read.
read 4 data,data is: 4.
demo_read.
read 4 data,data is: 5.
demo_read.
read 4 data,data is: 6.
demo_read.
```

如上所示，每次从驱动中读取的数据都是自增的。

**附driver_demo10.c源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include <string.h>
#include "driver.h"
#include "vir_peripheral_device.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;

    printk("%s %s.\r\n", __func__, name);

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
    int data = -1;

    printk("%s.\r\n", __func__);

    data = vir_device_read(FALSE);

    memcpy(buf, &data, sizeof(int));

    return sizeof(int);
}

static ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
    .fo_read  = demo_read,
    .fo_write = demo_write,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    vir_device_init();

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);
}
```

**附vir_peripheral_device.c源码：**

```
/*
 * vir_peripheral_device.c
 *
 *  Created on: Nov 28, 2020
 *      Author: Administrator
 */
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include "vir_peripheral_device.h"

#define READ_PERIOD (3)
#define MUTEX_CREATE_DEF_FLAG        \
        LW_OPTION_WAIT_PRIORITY    | \
        LW_OPTION_INHERIT_PRIORITY | \
        LW_OPTION_DELETE_SAFE      | \
        LW_OPTION_OBJECT_GLOBAL

static LW_HANDLE tid;
static LW_HANDLE read_sync;
static BOOL readable = FALSE;
static int data = 1;

static void *vdev_thread (void *sarg)
{
    while (1) {
        sleep(READ_PERIOD);

        data++;
        readable = TRUE;
        API_SemaphoreBPost(read_sync);
    }

    return NULL;
}

int vir_device_init (void)
{
    LW_CLASS_THREADATTR threadattr;

    API_ThreadAttrBuild(&threadattr,
                        4 * LW_CFG_KB_SIZE,
                        LW_PRIO_NORMAL,
                        LW_OPTION_THREAD_STK_CHK,
                        LW_NULL);
    tid = API_ThreadCreate("t_vdev", vdev_thread, &threadattr, LW_NULL);
    if (tid == LW_OBJECT_HANDLE_INVALID) {
        return (PX_ERROR);
    }

    read_sync = API_SemaphoreBCreate("count_lock",
                                0,
                                LW_OPTION_OBJECT_LOCAL,
                                LW_NULL);
    if (read_sync == LW_OBJECT_HANDLE_INVALID) {
        API_ThreadDelete(&tid, NULL);
        printk("semaphore create failed.\n");
        return (PX_ERROR);
    }

    return ERROR_NONE;
}

int vir_device_deinit (void)
{
    API_ThreadDelete(&tid, NULL);

    return ERROR_NONE;
}

int vir_device_read (BOOL non_block)
{
    if (non_block) {
        if (readable) {
            goto return_data;
        } else {
            return 0;
        }
    }

    API_SemaphoreBPend(read_sync, LW_OPTION_WAIT_INFINITE);

return_data:
    readable = FALSE;
    return data;
}
```

附vir_peripheral_device.h源码：

```
/*
 * vir_peripheral_device.h
 *
 *  Created on: Nov 28, 2020
 *      Author: Administrator
 */

#ifndef SRC_VIR_PERIPHERAL_DEVICE_H_
#define SRC_VIR_PERIPHERAL_DEVICE_H_

int vir_device_init(void);
int vir_device_deinit(void);
int vir_device_read (BOOL non_block);

#endif /* SRC_VIR_PERIPHERAL_DEVICE_H_ */
```

附app_demo10.c源码：

```
#include <stdio.h>
#include <string.h>

int main (int argc, char **argv)
{
    int fd;
    int ret = 0;
    int data = -1;

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    while (1) {
        ret = read(fd, &data, 4);
        if (ret < 0) {
            printf("read data fail.\r\n");
            close(fd);
            return -1;
        } else if (ret == 0) {
            printf("read data end.\r\n");
        } else {
            printf("read %d data,data is: %d.\r\n", ret, data);
        }
    }

    close(fd);

    return  (0);
}
```

## SylixOS设备操作之非阻塞read和write

上一节学习了阻塞方式对驱动进行读写，但是在某些情况下，我们需要非阻塞地读写驱动，如果数据没准备好，read或write调用需要立即返回以进行后续的业务处理。在SylixOS应用层，可以通过三种方法来进行非阻塞read或write调用。在本小节教程中，我们同样以read为例讲解非阻塞。

### **以非阻塞模式打开设备文件时**

应用层在以***open\*** 接口打开设备文件时，打开标志中添加***O_NONBLOCK\*** 即可以非阻塞的方式打开设备：

```
fd =  open("/dev/demo0", O_RDWR | O_NONBLOCK);
if (fd < 0) {
    printf("open file fail.\r\n");
    return -1;
}
```

驱动层也要做相应的修改，以对非阻塞调用进行处理，首先***demo_dev_t\*** 数据结构中需要添加一个成员用于记录当前是否需要进行非阻塞处理：

```
typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    void *priv;
} demo_dev_t;
```

驱动***open\*** 函数中需要对打开标志进行判断，如果带有非阻塞标志，则需要做相应的处理：

```
if (flag & O_NONBLOCK) {
    printk("open nonblock.\r\n");
    device->flag |= O_NONBLOCK;
}
```

驱动中的***read\*** 也要判断当前设备的阻塞类型来决定到底如何调用虚拟外设的读数据接口：

```
static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
    int ret = -1;
    BOOL non_block = FALSE;

    printk("%s.\r\n", __func__);

    if (dev->flag & O_NONBLOCK) {
        non_block = TRUE;
    } else {
        non_block = FALSE;
    }

    ret = vir_device_read(non_block);
    if (ret > 0)
        memcpy(buf, &ret, sizeof(int));

    return ret;
}
```

驱动中如果没有读取到数据，接口是返回0的，所以在应用层我们需要对read数据为0分支做小改动：

```
ret = read(fd, &data, 4);
if (ret < 0) {
    printf("read data fail.\r\n");
    close(fd);
    return -1;
} else if (ret == 0) {
    sleep(1);
    printf("read data end.\r\n");
} else {
    printf("read %d data,data is: %d.\r\n", ret, data);
}
```

与之前的处理代码相比，多了个sleep(1)处理，因为如果不加这条语句，终端上会一直打印***read data end.\*** 从而影响我们查看别的打印信息。加载驱动并运行程序后，终端上的打印：

```
[root@sylixos:/apps/app_demo11_2]# ./app_demo11_2
demo_open .
open nonblock.
open read/write.
file permission: 644.
demo_read.
read 10 data,data is: 10.
demo_read.
read data end.
demo_read.
read data end.
demo_read.
read 11 data,data is: 11.
demo_read.
read data end.
demo_read.
read data end.
demo_read.
read data end.
demo_read.
read 12 data,data is: 12.
demo_read.
```

可以看到程序确实走的是read为0的分支。

### 使用ioctl设置设备为非阻塞模式

这种方式打开设备时还是以默认的阻塞方式打开，打开成功后，通过调用***FIONBIO\*** ioctl命令来设置设备文件为非阻塞模式：

```
int non_block = 0;

non_block = 1;
ret = ioctl(fd, FIONBIO, &non_block);
if (ret) {
    printf("FIONBIO fail.\r\n");
    close(fd);
    return -1;
}
```

***FIONBIO\*** 命令的参数为一个int类型变量的地址，变量值为1表示需要设置为非阻塞模式，为0表示取消非阻塞模式。驱动层的ioctl函数中需要对***FIONBIO\*** 命令做相应的处理：

驱动层和应用层read函数还是和第一种方法一样，不需要修改，程序运行输出也一样，这里就不贴出来了。

### 使用fcntl设置设备为非阻塞模式

这种方式有点类似前两种方法的结合，首先通过***F_GETFL\*** 命令获取设备文件当前的打开标志，然后加上***O_NONBLOCK\*** 作为新的文件标志通过***F_SETFL\*** 命令再设置到设备文件中：

```
ret = fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
if (ret) {
    printf("F_SETFL fail.\r\n");
    close(fd);
    return -1;
}
```

驱动层的ioctl函数中需要对***F_SETFL\*** 命令做相应的处理，如下所示：

```
case FIOSETFL:
    if ((int)arg & O_NONBLOCK) {
        dev->flag |= O_NONBLOCK;
    } else {
        dev->flag &= ~O_NONBLOCK;
    }
    break;
```

驱动层和应用层read函数还是和前两种方法一样，不需要修改，程序运行输出也一样，这里也不贴出来了。

通过前面三种方法我们可以看出最终的目的都是改变驱动中的标志来告知驱动read或write函数使用何种方式进行操作，所以应用开发者想要使用阻塞或者非阻塞方式来操作设备，其实是需要驱动来提供相应的功能的。

**附driver_demo11.c源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include <string.h>
#include "driver.h"
#include "vir_peripheral_device.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;
    device->flag = 0;

    printk("%s %s.\r\n", __func__, name);

    if (flag & O_NONBLOCK) {
        printk("open nonblock.\r\n");
        device->flag |= O_NONBLOCK;
    }

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case FIONBIO:
        if (*(int *)arg) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSETFL:
        if ((int)arg & O_NONBLOCK) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
    int ret = -1;
    BOOL non_block = FALSE;

    printk("%s.\r\n", __func__);

    if (dev->flag & O_NONBLOCK) {
        non_block = TRUE;
    } else {
        non_block = FALSE;
    }

    ret = vir_device_read(non_block);
    if (ret > 0)
        memcpy(buf, &ret, sizeof(int));

    return ret;
}

static ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
    .fo_read  = demo_read,
    .fo_write = demo_write,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    vir_device_init();

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);
}
```

**附方法一应用层源码：**

```
#include <stdio.h>
#include <string.h>

int main (int argc, char **argv)
{
    int fd;
    int ret = 0;
    int data = -1;

    fd =  open("/dev/demo0", O_RDWR | O_NONBLOCK);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    while (1) {
        ret = read(fd, &data, 4);
        if (ret < 0) {
            printf("read data fail.\r\n");
            close(fd);
            return -1;
        } else if (ret == 0) {
            sleep(1);
            printf("read data end.\r\n");
        } else {
            printf("read %d data,data is: %d.\r\n", ret, data);
        }
    }

    close(fd);

    return  (0);
}
```

**附方法二应用层源码：**

```
#include <stdio.h>
#include <string.h>

int main (int argc, char **argv)
{
    int fd;
    int ret = 0;
    int data = -1;
    int non_block = 0;

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    non_block = 1;
    ret = ioctl(fd, FIONBIO, &non_block);
    if (ret) {
        printf("FIONBIO fail.\r\n");
        close(fd);
        return -1;
    }

    while (1) {
        ret = read(fd, &data, 4);
        if (ret < 0) {
            printf("read data fail.\r\n");
            close(fd);
            return -1;
        } else if (ret == 0) {
            sleep(1);
            printf("read data end.\r\n");
        } else {
            printf("read %d data,data is: %d.\r\n", ret, data);
        }
    }

    close(fd);

    return  (0);
}
```

**附方法三应用层源码：**

```
#include <stdio.h>
#include <string.h>

int main (int argc, char **argv)
{
    int fd;
    int ret = 0;
    int data = -1;

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    ret = fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
    if (ret) {
        printf("F_SETFL fail.\r\n");
        close(fd);
        return -1;
    }

    while (1) {
        ret = read(fd, &data, 4);
        if (ret < 0) {
            printf("read data fail.\r\n");
            close(fd);
            return -1;
        } else if (ret == 0) {
            sleep(1);
            printf("read data end.\r\n");
        } else {
            printf("read %d data,data is: %d.\r\n", ret, data);
        }
    }

    close(fd);

    return  (0);
}
```

## SylixOS设备操作之select

​		通过前面的学习我们知道了SylixOS下操作设备是通过设备文件的方式来进行的，先open一个文件获取到一个文件描述符，然后通过文件描述符进行操作文件。有时候我们在程序里可能需要对多个文件描述符进行操作，比如有一个程序我们需要读取设备数据，如果没有数据准备好程序需要阻塞，同时我们还要检测用户在命令行的输入，如果输入'quit'程序需要退出，程序该如何设计？大家首先想到的接受输入用scanf不就行了么，但是scanf这个函数是阻塞型的，如果用户没有输入东西程序就会一直阻塞，这样即使设备有数据准备好程序也不能及时去处理

### select应用层用法简介

解决这类问题的办法就是使用多路IO复用接口select，函数原型：

```
LW_API INT     select(INT               iWidth, 
                      fd_set           *pfdsetRead,
                      fd_set           *pfdsetWrite,
                      fd_set           *pfdsetExcept,
                      struct timeval   *ptmvalTO);
```

select可以检测一系列的文件描述符集合，检测的方向为三个，分别为可读，可写，有异常，并且还可以带有超时检测机制，这里我们以检测描述符可读为例进行讲解。

在本节demo中，我们设计检测两个描述符是否可读，一个是标准输入描述符，一个是demo设备文件描述符。我们使用***fd_set\*** 数据结构来记录描述符的集合，在使用select之前需要初始化这个描述符集合：

```
FD_ZERO (&fds);
FD_SET (STD_IN, &fds);
FD_SET (fd, &fds);
```

首先使用***FD_ZERO\*** 将数据结构清零，然后使用***FD_SET\*** 依次设置想要检测的描述符，***STD_IN\*** 表示标准输入文件描述符，每一个进程都会默认打开标准输入、标准输出、标准出错三个文件描述符，在SylixOS下，这三个文件描述符分别用***STD_IN\*** 、***STD_OUT\*** 、***STD_ERR\*** 来表示：

```
/*********************************************************************************************************
  low-level I/O input, output, error fd's
*********************************************************************************************************/

#define STD_IN                      0
#define STD_OUT                     1
#define STD_ERR                     2
```

初始化完描述符集合后就可以调用select来检测：

```
ret = select (fd + 1, &fds, NULL, NULL, NULL);
```

select第一个参数为要检测的文件描述符中最大的那个值再加一，当描述符可读后还需要调用***FD_ISSET\*** 来确定是哪个描述符准备好了：

```
if (FD_ISSET(fd, &fds)) {
    printf("data readable.\r\n");
}

if (FD_ISSET(0, &fds)) {
    printf("app demo quit.\r\n");
    break;
}
```

### 驱动层支持select

SylixOS中select是通过底层驱动ioctl来实现的，select实现伪代码表示的基本流程：

```
        select
           |
 ioctl调用FIOSELECT命令
           |
        线程睡眠
           |
ioctl调用FIOUNSELECT命令   
           |
          结束
```

驱动里select方法的阻塞和唤醒是通过select链表来实现的，首先在***demo_dev_t\*** 数据结构中添加select链表***wakeup_list\*** ：

```
typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    LW_SEL_WAKEUPLIST wakeup_list;
    void *priv;
} demo_dev_t;
```

然后在驱动加载时通过***SEL_WAKE_UP_LIST_INIT\*** 来初始化这个select链表：

```
SEL_WAKE_UP_LIST_INIT(&demo_dev[0].wakeup_list);
SEL_WAKE_UP_LIST_INIT(&demo_dev[1].wakeup_list);
```

如果驱动程序中数据还没准备好，就需要将线程睡眠，这是通过将当前驱动的等待结点加入到等待链表中来实现的：

```
PLW_SEL_WAKEUPNODE pselnode;

pselnode = (PLW_SEL_WAKEUPNODE)arg;
SEL_WAKE_NODE_ADD(&dev->wakeup_list, pselnode);
```

等待结点***pselnode\*** 是select在调用驱动的ioctl之前就准备好的，并作为ioctl的arg参数传进来，驱动层只需要转换成***PLW_SEL_WAKEUPNODE\*** 类型指针即可使用。

> 注意：SEL_WAKE_NODE_ADD只是将等待结点加入到等待链表中，此接口并不会阻塞，线程睡眠是通过线程TCB中的一个二进制信号量来实现的。

如果数据准备好了，就可以通过***SEL_WAKE_UP\*** 或者***SEL_WAKE_UP_ALL\*** 来唤醒线程，其中***SEL_WAKE_UP_ALL\*** 可以通过指定唤醒类型来进行更精确的控制。在实际的设备驱动中，一般是在中断处理中来调用此接口进行唤醒线程。在本例中，我们将唤醒调用放在虚拟设备的定时处理线程中：

```
static LW_SEL_WAKEUPLIST *wakeup_list;

static void *vdev_thread (void *sarg)
{
    while (1) {
        sleep(READ_PERIOD);

        data++;
        readable = TRUE;
        API_SemaphoreBPost(read_sync);
        if (wakeup_list) {
            SEL_WAKE_UP_ALL(wakeup_list, SELREAD);
        }
    }

    return NULL;
}
```

***wakeup_list\*** 这个全局变量在驱动调用***vir_device_wakeup_register\*** 接口时初始化：

```
int vir_device_wakeup_register (LW_SEL_WAKEUPLIST *list)
{
    wakeup_list = list;
    return 0;
}
```

线程唤醒后需要将之前的等待结点从等待链表里删除，这是通过***SEL_WAKE_NODE_DELETE\*** 来实现的：

```
PLW_SEL_WAKEUPNODE pselnode;

pselnode = (PLW_SEL_WAKEUPNODE)arg;
SEL_WAKE_NODE_DELETE(&dev->wakeup_list, pselnode);
```

通过前面的介绍，我们可以写出demo驱动ioctl中***FIOSELECT\*** 和***FIOUNSELECT\*** 命令相对应的处理：

```
case FIOSELECT:
    pselnode = (PLW_SEL_WAKEUPNODE)arg;

    if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
        if (vir_device_readable()) {
            SEL_WAKE_UP(pselnode);
        } else {
            SEL_WAKE_NODE_ADD(&dev->wakeup_list, pselnode);
            vir_device_wakeup_register(&dev->wakeup_list);
        }
    } else {
        return (PX_ERROR);
    }

    break;

case FIOUNSELECT:
    pselnode = (PLW_SEL_WAKEUPNODE)arg;

    if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
        SEL_WAKE_NODE_DELETE(&dev->wakeup_list, pselnode);
    } else {
        return (PX_ERROR);
    }

    break;
```

当调用select时驱动已经有准备好的数据时，需要调用***SEL_WAKE_UP\*** 告知系统已经有数据准备好，线程不需要阻塞。

### 实际运行效果

加载驱动后，运行程序，终端输出打印：

```
[root@sylixos:/apps/app_demo12]# ./app_demo12
demo_open .
open read/write.
file permission: 644.
demo_ioctl 1c.
demo_ioctl 1d.
data readable.
demo_read.
read data is: 3162.
demo_ioctl 1c.
demo_ioctl 1d.
data readable.
demo_read.
read data is: 3163.
demo_ioctl 1c.
demo_ioctl 1d.
data readable.
demo_read.
read data is: 3164.
demo_ioctl 1c.
quit
demo_ioctl 1d.
app demo quit.
demo_close.
[root@sylixos:/apps/app_demo12]#
```

可以看出在没有输入时，程序依然是每隔一段时间读取数据打印输出，如果命令行有输入，比如输入'quit'后，程序响应输入直接退出结束。

**附driver_demo12.c源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include <string.h>
#include "driver.h"
#include "vir_peripheral_device.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    LW_SEL_WAKEUPLIST wakeup_list;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;
    device->flag = 0;

    printk("%s %s.\r\n", __func__, name);

    if (flag & O_NONBLOCK) {
        printk("open nonblock.\r\n");
        device->flag |= O_NONBLOCK;
    }

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    PLW_SEL_WAKEUPNODE pselnode;
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case FIONBIO:
        if (*(int *)arg) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSETFL:
        if ((int)arg & O_NONBLOCK) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSELECT:
        pselnode = (PLW_SEL_WAKEUPNODE)arg;

        if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
            if (vir_device_readable()) {
                SEL_WAKE_UP(pselnode);
            } else {
                SEL_WAKE_NODE_ADD(&dev->wakeup_list, pselnode);
                vir_device_wakeup_register(&dev->wakeup_list);
            }
        } else {
            return (PX_ERROR);
        }

        break;

    case FIOUNSELECT:
        pselnode = (PLW_SEL_WAKEUPNODE)arg;

        if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
            SEL_WAKE_NODE_DELETE(&dev->wakeup_list, pselnode);
        } else {
            return (PX_ERROR);
        }

        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
    int ret = -1;
    BOOL non_block = FALSE;

    printk("%s.\r\n", __func__);

    if (dev->flag & O_NONBLOCK) {
        non_block = TRUE;
    } else {
        non_block = FALSE;
    }

    ret = vir_device_read(non_block);
    if (ret > 0)
        memcpy(buf, &ret, sizeof(int));

    return ret;
}

static ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
    .fo_read  = demo_read,
    .fo_write = demo_write,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    SEL_WAKE_UP_LIST_INIT(&demo_dev[0].wakeup_list);
    SEL_WAKE_UP_LIST_INIT(&demo_dev[1].wakeup_list);

    vir_device_init();

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);
}
```

**附vir_peripheral_device.c源码：**

```
/*
 * vir_peripheral_device.c
 *
 *  Created on: Nov 28, 2020
 *      Author: Administrator
 */
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include "vir_peripheral_device.h"

#define READ_PERIOD (3)
#define MUTEX_CREATE_DEF_FLAG        \
        LW_OPTION_WAIT_PRIORITY    | \
        LW_OPTION_INHERIT_PRIORITY | \
        LW_OPTION_DELETE_SAFE      | \
        LW_OPTION_OBJECT_GLOBAL

static LW_SEL_WAKEUPLIST *wakeup_list;
static LW_HANDLE tid;
static LW_HANDLE read_sync;
static BOOL readable = FALSE;
static int data = 1;

static void *vdev_thread (void *sarg)
{
    while (1) {
        sleep(READ_PERIOD);

        data++;
        readable = TRUE;
        API_SemaphoreBPost(read_sync);
        if (wakeup_list) {
            SEL_WAKE_UP_ALL(wakeup_list, SELREAD);
        }
    }

    return NULL;
}

int vir_device_init (void)
{
    LW_CLASS_THREADATTR threadattr;

    API_ThreadAttrBuild(&threadattr,
                        4 * LW_CFG_KB_SIZE,
                        LW_PRIO_NORMAL,
                        LW_OPTION_THREAD_STK_CHK,
                        LW_NULL);
    tid = API_ThreadCreate("t_vdev", vdev_thread, &threadattr, LW_NULL);
    if (tid == LW_OBJECT_HANDLE_INVALID) {
        return (PX_ERROR);
    }

    read_sync = API_SemaphoreBCreate("count_lock",
                                0,
                                LW_OPTION_OBJECT_LOCAL,
                                LW_NULL);
    if (read_sync == LW_OBJECT_HANDLE_INVALID) {
        API_ThreadDelete(&tid, NULL);
        printk("semaphore create failed.\n");
        return (PX_ERROR);
    }

    return ERROR_NONE;
}

int vir_device_deinit (void)
{
    API_ThreadDelete(&tid, NULL);

    return ERROR_NONE;
}

int vir_device_read (BOOL non_block)
{
    if (non_block) {
        if (readable) {
            goto return_data;
        } else {
            return 0;
        }
    }

    API_SemaphoreBPend(read_sync, LW_OPTION_WAIT_INFINITE);

return_data:
    readable = FALSE;
    return data;
}

BOOL vir_device_readable (void)
{
    return readable;
}

int vir_device_wakeup_register (LW_SEL_WAKEUPLIST *list)
{
    wakeup_list = list;
    return 0;
}
```

**附vir_peripheral_device.h源码：**

```
/*
 * vir_peripheral_device.h
 *
 *  Created on: Nov 28, 2020
 *      Author: Administrator
 */

#ifndef SRC_VIR_PERIPHERAL_DEVICE_H_
#define SRC_VIR_PERIPHERAL_DEVICE_H_

int vir_device_init(void);
int vir_device_deinit(void);
int vir_device_read(BOOL non_block);
BOOL vir_device_readable(void);
int vir_device_wakeup_register(LW_SEL_WAKEUPLIST *list);

#endif /* SRC_VIR_PERIPHERAL_DEVICE_H_ */
```

**附app_demo12源码：**

```
#include <stdio.h>
#include <string.h>

int main (int argc, char **argv)
{
    int fd;
    int ret = 0;
    int data = -1;
    fd_set fds;

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }


    while (1) {
        FD_ZERO (&fds);
        FD_SET (STD_IN, &fds);
        FD_SET (fd, &fds);

        ret = select (fd + 1, &fds, NULL, NULL, NULL);
        if (ret < 0) {
            printf("select fail.\r\n");
            close(fd);
            return -1;
        } else if (ret == 0){
            printf("select timeout.\r\n");
        } else {
            if (FD_ISSET(fd, &fds)) {
                printf("data readable.\r\n");
            }

            if (FD_ISSET(STD_IN, &fds)) {
                printf("app demo quit.\r\n");
                break;
            }
        }

        read(fd, &data, 4);
        printf("read data is: %d.\r\n", data);
    }

    close(fd);

    return  (0);
}
```

## SylixOS设备操作之mmap

有些外设控制器可能带有DMA功能，这类外设的驱动一般都需要使用物理连续的内存，而且只需要物理内存即可，在驱动层并不需要进行映射访问，这时可以通过下面接口申请物理内存：

```
PVOID API_VmmPhyAlloc(size_t stSize);
```

通过此接口返回的地址只表示物理地址，并没有经过映射，所以如果驱动直接访问会崩溃。像显示驱动、摄像头驱动都可能是通过此方式申请物理内存的，那么应用层如何访问驱动里的这些物理内存呢，因为应用层是通过虚拟地址访问的，所以必须先调用***mmap\*** 接口将驱动中的物理内存和虚拟内存进行映射，然后才能读写访问：

```
void *mmap(void  *pvAddr, size_t  stLen, int  iProt, int  iFlag, int  iFd, off_t  off);
```

***mmap\*** 的各个参数解释如下：

- pvAddr：指定固定的虚拟地址进行映射，一般为NULL表示让系统来分配虚拟地址。
- stLen：要映射的空间大小，单位为字节。（系统层面会转换为以页大小为基础单位大小来映射）
- iProt：映射为可读还是可写。
- iFlag：映射标志，比如是否共享等等。
- iFd：要进行映射的文件描述符。
- off：起始映射偏移，一般为0。

此接口成功返回虚拟地址，失败返回***MAP_FAILED\***。

当使用完后，通过***munmap\*** 来释放虚拟地址空间：

```
int munmap(void  *pvAddr, size_t  stLen);
```

### 驱动层mmap实现

首先***demo_dev_t\*** 数据结构中需要添加两个成员：***phy_addr\*** 、***map_flags\*** 分别用来记录驱动中申请的物理内存地址和要映射的属性：

```
typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    LW_SEL_WAKEUPLIST wakeup_list;
    void *phy_addr;
    int map_flags;
    void *priv;
} demo_dev_t;
```

在驱动加载时我们申请4KB大小的物理内存：

```
demo_dev[0].phy_addr = API_VmmPhyAlloc(4 * 1024);
demo_dev[1].phy_addr = API_VmmPhyAlloc(4 * 1024);
```

不同的设备驱动需要映射的属性不一样，比如显示驱动需要映射内存为非cache属性，摄像头驱动需要映射为cache属性。非cache属性使用***LW_VMM_FLAG_DMA\*** 来表示，cache属性用***LW_VMM_FLAG_RDWR\*** 来表示：

```
#if 1
    demo_dev[0].map_flags = LW_VMM_FLAG_DMA;
    demo_dev[1].map_flags = LW_VMM_FLAG_DMA;
#else
    demo_dev[0].map_flags = LW_VMM_FLAG_RDWR;
    demo_dev[1].map_flags = LW_VMM_FLAG_RDWR;
#endif
```

本例中我们使用非cache属性来演示。

驱动中mmap的原型：

```
int demo_mmap(demo_dev_t *dev, LW_DEV_MMAP_AREA *vm);
```

SylixOS系统使用***LW_DEV_MMAP_AREA\*** 数据结构来描述要映射的虚拟地址空间信息：

```
typedef struct {
    PVOID                       DMAP_pvAddr;                            /*  起始地址                    */
    size_t                      DMAP_stLen;                             /*  内存长度 (单位:字节)        */
    off_t                       DMAP_offPages;                          /*  文件映射偏移量 (单位:页面)  */
    ULONG                       DMAP_ulFlag;                            /*  当前虚拟地址 VMM 属性       */
} LW_DEV_MMAP_AREA;
```

通过此结构可以获得要映射的虚拟空间的起始地址和大小这两个主要信息。然后在mmap中就可以使用***API_VmmRemapArea\*** 来建立映射了：

```
ULONG        API_VmmRemapArea(PVOID  pvVirtualAddr, PVOID  pvPhysicalAddr, 
                                     size_t  stSize, ULONG  ulFlag,
                                     FUNCPTR  pfuncFiller, PVOID  pvArg);
```

此接口的各个参数含义如下：

- pvVirtualAddr：要映射的虚拟起始地址。
- pvPhysicalAddr：要映射的物理起始地址。
- stSize：要映射的大小。
- ulFlag：映射属性。
- pfuncFiller：缺页中断回调函数。
- pvArg：缺页中断回调函数参数。

在此demo驱动中，mmap函数的实现：

```
static int demo_mmap(demo_dev_t *dev, LW_DEV_MMAP_AREA *vm)
{
    printk("%s.\r\n", __func__);

    if (API_VmmRemapArea(vm->DMAP_pvAddr,
                         dev->phy_addr,
                         vm->DMAP_stLen,
                         dev->map_flags,
                         LW_NULL,
                         LW_NULL)) {
        return  (PX_ERROR);
    }

    return  (ERROR_NONE);
}
```

### 应用层使用mmap

应用层程序中我们先使用mmap映射4KB大小空间，然后通过memset来将这段空间清零，最后使用munmap取消映射：

```
addr = mmap (NULL, 4 * 1024, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
if (addr == MAP_FAILED) {
    printf("mmap file fail.\r\n");
    close(fd);
    return -1;
}

memset(addr, 0, 4 * 1024);
printf("memset ok.\r\n");

sleep(100);

munmap(addr, 4 * 1024);
```

映射标志我们设置为可读写，延时100s是因为我们等会需要在命令行查看一些信息。

### 实际运行效果

加载驱动，然后不着急运行app，首先在命令行使用***virtuals\*** 命令查看当前系统虚拟地址空间使用情况：

```
[root@sylixos:/apps/app_demo13]# virtuals
vmm virtual area show >>
vmm virtual program from: 0x10000000, size: 0xc0000000, used: 0%
vmm virtual ioremap from: 0xd0001000, size: 0x04000000, used: 0%
vmm virtual area usage as follow:

VIRTUAL    SIZE   WRITE CACHE
-------- -------- ----- -----
d0001000     1000 true  false
d0002000     1000 true  false
d0003000     1000 true  false
d0004000     1000 true  false
10006000     2000 true  true
10008000     5000 true  true
1000d000     2000 true  true
1000f000     1000 true  true
10010000     1000 true  true
10011000     1000 true  true
```

可以看出虚拟地址空间信息主要包含四部分：起始地址、大小、是否可写、是否为cache属性。这时以后台方式运行app，也就是在app后面跟上 ***&\*** 这个符号：

```
[root@sylixos:/apps/app_demo13]# ./app_demo13 &
[root@sylixos:/apps/app_demo13]# demo_open .
open read/write.
file permission: 644.
demo_ioctl 2f.
demo_ioctl 26.
demo_mmap.
memset ok.
```

这时候再使用***virtuals\*** 命令查看当前系统虚拟地址空间使用情况：

```
[root@sylixos:/apps/app_demo13]# virtuals
vmm virtual area show >>
vmm virtual program from: 0x10000000, size: 0xc0000000, used: 1%
vmm virtual ioremap from: 0xd0001000, size: 0x04000000, used: 0%
vmm virtual area usage as follow:

VIRTUAL    SIZE   WRITE CACHE
-------- -------- ----- -----
d0001000     1000 true  false
d0002000     1000 true  false
d0003000     1000 true  false
d0004000     1000 true  false
10001000     1000 true  true
10006000     2000 true  true
10008000     5000 true  true
1000d000     2000 true  true
1000f000     1000 true  true
10010000     1000 true  true
10011000     1000 true  true
10012000    1f000 true  true
10038000     9000 true  true
10041000     5000 true  true
10046000     1000 true  false
10048000    20000 true  true
10068000  2000000 true  true
12068000    10000 true  true
```

可以发现最后一行信息就是我们使用mmap时系统分配的虚拟地址空间信息。

**附driver_demo13.c源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include <string.h>
#include "driver.h"
#include "vir_peripheral_device.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    LW_SEL_WAKEUPLIST wakeup_list;
    void *phy_addr;
    int map_flags;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    device->priv = NULL;
    device->flag = 0;

    printk("%s %s.\r\n", __func__, name);

    if (flag & O_NONBLOCK) {
        printk("open nonblock.\r\n");
        device->flag |= O_NONBLOCK;
    }

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    LW_DEV_INC_USE_COUNT(&device->dev);

    return (long)device;
}

static int demo_close(demo_dev_t *dev)
{
    printk("%s.\r\n", __func__);

    LW_DEV_DEC_USE_COUNT(&dev->dev);

    return ERROR_NONE;
}

static int demo_ioctl(demo_dev_t *dev, int cmd, long arg)
{
    PLW_SEL_WAKEUPNODE pselnode;
    struct stat *pstat;
    struct data *data;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case FIONBIO:
        if (*(int *)arg) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSETFL:
        if ((int)arg & O_NONBLOCK) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSELECT:
        pselnode = (PLW_SEL_WAKEUPNODE)arg;

        if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
            if (vir_device_readable()) {
                SEL_WAKE_UP(pselnode);
            } else {
                SEL_WAKE_NODE_ADD(&dev->wakeup_list, pselnode);
                vir_device_wakeup_register(&dev->wakeup_list);
            }
        } else {
            return (PX_ERROR);
        }

        break;

    case FIOUNSELECT:
        pselnode = (PLW_SEL_WAKEUPNODE)arg;

        if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
            SEL_WAKE_NODE_DELETE(&dev->wakeup_list, pselnode);
        } else {
            return (PX_ERROR);
        }

        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static ssize_t demo_read(demo_dev_t *dev, char *buf, size_t size)
{
    int ret = -1;
    BOOL non_block = FALSE;

    printk("%s.\r\n", __func__);

    if (dev->flag & O_NONBLOCK) {
        non_block = TRUE;
    } else {
        non_block = FALSE;
    }

    ret = vir_device_read(non_block);
    if (ret > 0)
        memcpy(buf, &ret, sizeof(int));

    return ret;
}

static ssize_t demo_write(demo_dev_t *dev, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}

static int demo_mmap(demo_dev_t *dev, LW_DEV_MMAP_AREA *vm)
{
    printk("%s.\r\n", __func__);

    if (API_VmmRemapArea(vm->DMAP_pvAddr,
                         dev->phy_addr,
                         vm->DMAP_stLen,
                         dev->map_flags,
                         LW_NULL,
                         LW_NULL)) {
        return  (PX_ERROR);
    }

    return  (ERROR_NONE);
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
    .fo_read  = demo_read,
    .fo_write = demo_write,
    .fo_mmap  = demo_mmap,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx(&demo_fops);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    SEL_WAKE_UP_LIST_INIT(&demo_dev[0].wakeup_list);
    SEL_WAKE_UP_LIST_INIT(&demo_dev[1].wakeup_list);

    vir_device_init();

    demo_dev[0].phy_addr = API_VmmPhyAlloc(4 * 1024);
    demo_dev[1].phy_addr = API_VmmPhyAlloc(4 * 1024);

#if 1
    demo_dev[0].map_flags = LW_VMM_FLAG_DMA;
    demo_dev[1].map_flags = LW_VMM_FLAG_DMA;
#else
    demo_dev[0].map_flags = LW_VMM_FLAG_RDWR;
    demo_dev[1].map_flags = LW_VMM_FLAG_RDWR;
#endif

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);

    if (demo_dev[0].phy_addr)
        API_VmmPhyFree(demo_dev[0].phy_addr);

    if (demo_dev[1].phy_addr)
        API_VmmPhyFree(demo_dev[1].phy_addr);
}
```

**附app_demo13.c源码：**

```
#include <stdio.h>
#include <string.h>
#include <sys/mman.h>

int main (int argc, char **argv)
{
    int fd;
    void *addr = NULL;

    fd =  open("/dev/demo0", O_RDWR);
    if (fd < 0) {
        printf("open file fail.\r\n");
        return -1;
    }

    addr = mmap (NULL, 4 * 1024, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        printf("mmap file fail.\r\n");
        close(fd);
        return -1;
    }

    memset(addr, 0, 4 * 1024);
    printf("memset ok.\r\n");

    sleep(100);

    munmap(addr, 4 * 1024);

    close(fd);

    return  (0);
}
```

## SylixOS设备驱动之NEW_1型驱动

在第三节教程中说过SylixOS下的驱动分为***ORIG\*** 和 ***NEW_1\*** 两种类型，通过前面十几节教程的学习，我们已经知道***ORIG\*** 型驱动如何开发了，但是***NEW_1\*** 型驱动有哪些不一样的地方呢？

> 关于ORIG和NEW_1型这两种IO系统结构框架上有什么具体的区别，请参见《SylixOS应用开发手册》5.1.3章节，本节只关注驱动层的差异。

**区别1：非open函数首个入参类型改变**

除了open函数的首个入参依然是***LW_DEV_HDR\*** 类型的指针外，其他的接口比如close、read等函数的首个入参变成了***LW_FD_ENTRY\*** 类型指针：

```
int demo_close(LW_FD_ENTRY *file);
```

SylixOS中的***LW_FD_ENTRY\*** 数据结构有点类似于linux下的***struct file\*** 数据结构，由于其他非open函数的首个入参并不是open函数的返回值了，所以如果想要使用***demo_dev_t\*** 数据结构可以通过以下方法获取：

```
demo_dev_t *dev = (demo_dev_t *)file->FDENTRY_pdevhdrHdr;
```

**区别2：open和close函数处理流程改变**

首先open函数的返回值并不是用户自定义的值了，而是一个称之为文件结点的数据结构地址：

```
/*********************************************************************************************************
  文件节点 
  
  只有 NEW_1 或更高级的设备驱动类型会用到此结构
  一个 dev_t 和 一个 ino_t 对应唯一一个实体文件, 操作系统统多次打开同一个实体文件时, 只有一个文件节点
  多个 fd_entry 同时指向这个节点.
  需要说明的是 FDNODE_oftSize 字段需要 NEW_1 驱动程序自己来维护.
  
  fd node 被锁定时, 将不允许写, 也不允许删除. 当关闭文件后此文件的 lock 将自动被释放.
*********************************************************************************************************/

typedef struct {
    LW_LIST_LINE               FDNODE_lineManage;                       /*  同一设备 fd_node 链表       */
    
    LW_OBJECT_HANDLE           FDNODE_ulSem;                            /*  内部操作锁                  */
    dev_t                      FDNODE_dev;                              /*  设备                        */
    ino64_t                    FDNODE_inode64;                          /*  inode (64bit 为了兼容性)    */
    mode_t                     FDNODE_mode;                             /*  文件 mode                   */
    uid_t                      FDNODE_uid;                              /*  文件所属用户信息            */
    gid_t                      FDNODE_gid;
    
    off_t                      FDNODE_oftSize;                          /*  当前文件大小                */
    
    struct  __fd_lockf        *FDNODE_pfdlockHead;                      /*  第一个锁                    */
    LW_LIST_LINE_HEADER        FDNODE_plineBlockQ;                      /*  当前有阻塞的记录锁队列      */
    
    BOOL                       FDNODE_bRemove;                          /*  是否在文件未关闭时有 unlink */
    ULONG                      FDNODE_ulLock;                           /*  锁定, 不允许写, 不允许删除  */
    ULONG                      FDNODE_ulRef;                            /*  fd_entry 引用此 fd_node 数量*/
    PVOID                      FDNODE_pvFile;                           /*  驱动使用此变量标示文件      */
    PVOID                      FDNODE_pvFsExtern;                       /*  文件系统扩展使用            */
} LW_FD_NODE;
```

为了使用文件结点还需要在***demo_dev_t\*** 中添加一个***fdnode_header\*** 文件结点链表：

```
typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    LW_LIST_LINE_HEADER fdnode_header;
    LW_SEL_WAKEUPLIST wakeup_list;
    void *phy_addr;
    int map_flags;
    void *priv;
} demo_dev_t;
```

在调用open函数时通过***API_IosFdNodeAdd\*** 来获得一个新的文件结点：

```
pfdnode = API_IosFdNodeAdd(&device->fdnode_header,
                           (dev_t)device, 0,
                           flag, mode, 0, 0, 0, NULL, &is_new);
if (pfdnode == NULL) {
    return  (PX_ERROR);
}

LW_DEV_INC_USE_COUNT(&device->dev);

return ((long)pfdnode);
```

如果申请成功则open函数返回文件结点的地址。

在关闭文件时我们还需要通过***API_IosFdNodeDec\*** 减少文件结点的引用值：

```
static int demo_close(LW_FD_ENTRY *file)
{
    demo_dev_t *dev = (demo_dev_t *)file->FDENTRY_pdevhdrHdr;
    PLW_FD_NODE pfdnode = (PLW_FD_NODE)file->FDENTRY_pfdnode;

    printk("%s.\r\n", __func__);

    if (file && pfdnode) {
        API_IosFdNodeDec(&dev->fdnode_header, pfdnode, NULL);
        LW_DEV_DEC_USE_COUNT(&dev->dev);

        return  (ERROR_NONE);
    }

    return  (PX_ERROR);
}
```

**区别3：驱动注册函数改变**

***NEW_1\*** 型驱动使用***iosDrvInstallEx2\*** 接口来注册驱动：

```
drv_index = iosDrvInstallEx2(&demo_fops, LW_DRV_TYPE_NEW_1);
if (drv_index < 0) {
    printk("driver install fail.\r\n");
    return -1;
}
```

***LW_DRV_TYPE_NEW_1\*** 表示将驱动注册为***NEW_1\*** 类型。

**实际运行效果**

在命令行加载驱动后，使用***open\*** 命令打开设备文件，接着使用***file\*** 命令查看内核打开的设备文件：

```
[root@sylixos:/root]# files
kernel filedes show (process filedes in /proc/${pid}/filedes) >>
 fd abn name                       type   drv
  3     /dev/ttyS0                 orig    18 GLB STD_IN GLB STD_OUT GLB STD_ERR
  4     /dev/socket                socket  32
  5     /dev/socket                socket  32
  6     /dev/hotplug               orig    12
  7     /dev/input/touch0          new_1   29
  8     /dev/demo0                 new_1   37
[root@sylixos:/root]#
```

可以看出驱动的类型变成了new_1类型。

> 一般如果移植的驱动原来是VxWorks下的建议使用ORIG型驱动，如果是Linux下的建议使用NEW_1型。

**附driver_demo14.c源码：**

```
#define  __SYLIXOS_KERNEL
#include <SylixOS.h>
#include <module.h>
#include <string.h>
#include "driver.h"
#include "vir_peripheral_device.h"

typedef struct _demo_dev {
    LW_DEV_HDR dev;
    int flag;
    LW_LIST_LINE_HEADER fdnode_header;
    LW_SEL_WAKEUPLIST wakeup_list;
    void *phy_addr;
    int map_flags;
    void *priv;
} demo_dev_t;

int drv_index;
demo_dev_t demo_dev[2];

static long demo_open(LW_DEV_HDR *dev, char *name, int flag, int mode)
{
    demo_dev_t *device = (demo_dev_t *)dev;
    PLW_FD_NODE pfdnode;
    BOOL is_new;
    device->priv = NULL;
    device->flag = 0;

    printk("%s %s.\r\n", __func__, name);

    if (flag & O_NONBLOCK) {
        printk("open nonblock.\r\n");
        device->flag |= O_NONBLOCK;
    }

    if ((flag & O_ACCMODE) == O_RDONLY) {
        printk("open read only.\r\n");
    } else if ((flag & O_ACCMODE) == O_WRONLY) {
        printk("open write only.\r\n");
    } else {
        printk("open read/write.\r\n");
    }

    printk("file permission: %o.\r\n", mode);

    pfdnode = API_IosFdNodeAdd(&device->fdnode_header,
                               (dev_t)device, 0,
                               flag, mode, 0, 0, 0, NULL, &is_new);
    if (pfdnode == NULL) {
        return  (PX_ERROR);
    }

    LW_DEV_INC_USE_COUNT(&device->dev);

    return ((long)pfdnode);
}

static int demo_close(LW_FD_ENTRY *file)
{
    demo_dev_t *dev = (demo_dev_t *)file->FDENTRY_pdevhdrHdr;
    PLW_FD_NODE pfdnode = (PLW_FD_NODE)file->FDENTRY_pfdnode;

    printk("%s.\r\n", __func__);

    if (file && pfdnode) {
        API_IosFdNodeDec(&dev->fdnode_header, pfdnode, NULL);
        LW_DEV_DEC_USE_COUNT(&dev->dev);

        return  (ERROR_NONE);
    }

    return  (PX_ERROR);
}

static int demo_ioctl(LW_FD_ENTRY *file, int cmd, long arg)
{
    PLW_SEL_WAKEUPNODE pselnode;
    struct stat *pstat;
    struct data *data;
    demo_dev_t *dev = (demo_dev_t *)file->FDENTRY_pdevhdrHdr;

    printk("%s %x.\r\n", __func__, cmd);

    switch (cmd) {
    case FIOFSTATGET:
        pstat = (struct stat *)arg;
        if (!pstat) {
            return  (PX_ERROR);
        }

        pstat->st_dev     = LW_DEV_MAKE_STDEV(&dev->dev);
        pstat->st_ino     = (ino_t)0;
        pstat->st_mode    = (S_IRWXU | S_IRWXG | S_IRWXO | S_IFCHR);
        pstat->st_nlink   = 1;
        pstat->st_uid     = 0;
        pstat->st_gid     = 0;
        pstat->st_rdev    = 0;
        pstat->st_size    = 0;
        pstat->st_blksize = 0;
        pstat->st_blocks  = 0;
        pstat->st_atime   = API_RootFsTime(LW_NULL);
        pstat->st_mtime   = API_RootFsTime(LW_NULL);
        pstat->st_ctime   = API_RootFsTime(LW_NULL);
        break;

    case FIONBIO:
        if (*(int *)arg) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSETFL:
        if ((int)arg & O_NONBLOCK) {
            dev->flag |= O_NONBLOCK;
        } else {
            dev->flag &= ~O_NONBLOCK;
        }
        break;

    case FIOSELECT:
        pselnode = (PLW_SEL_WAKEUPNODE)arg;

        if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
            if (vir_device_readable()) {
                SEL_WAKE_UP(pselnode);
            } else {
                SEL_WAKE_NODE_ADD(&dev->wakeup_list, pselnode);
                vir_device_wakeup_register(&dev->wakeup_list);
            }
        } else {
            return (PX_ERROR);
        }

        break;

    case FIOUNSELECT:
        pselnode = (PLW_SEL_WAKEUPNODE)arg;

        if (SEL_WAKE_UP_TYPE(pselnode) == SELREAD) {
            SEL_WAKE_NODE_DELETE(&dev->wakeup_list, pselnode);
        } else {
            return (PX_ERROR);
        }

        break;

    case CMD_GET_VERSION:
        data = (struct data *)arg;
        data->version = 3;
        break;

    default:
        return  (PX_ERROR);
    }

    return ERROR_NONE;
}

static ssize_t demo_read(LW_FD_ENTRY *file, char *buf, size_t size)
{
    int ret = -1;
    BOOL non_block = FALSE;
    demo_dev_t *dev = (demo_dev_t *)file->FDENTRY_pdevhdrHdr;

    printk("%s.\r\n", __func__);

    if (dev->flag & O_NONBLOCK) {
        non_block = TRUE;
    } else {
        non_block = FALSE;
    }

    ret = vir_device_read(non_block);
    if (ret > 0)
        memcpy(buf, &ret, sizeof(int));

    return ret;
}

static ssize_t demo_write(LW_FD_ENTRY *file, char *buf, size_t size)
{
    char wdata[30] = {0};
    int wdata_len = 30;

    printk("%s.\r\n", __func__);

    wdata_len = (size < wdata_len) ? size : wdata_len;
    memcpy(wdata, buf, size);

    printk("write data %s success.\r\n", wdata);

    return wdata_len;
}

static int demo_mmap(LW_FD_ENTRY *file, LW_DEV_MMAP_AREA *vm)
{
    demo_dev_t *dev = (demo_dev_t *)file->FDENTRY_pdevhdrHdr;

    printk("%s.\r\n", __func__);

    if (API_VmmRemapArea(vm->DMAP_pvAddr,
                         dev->phy_addr,
                         vm->DMAP_stLen,
                         dev->map_flags,
                         LW_NULL,
                         LW_NULL)) {
        return  (PX_ERROR);
    }

    return  (ERROR_NONE);
}

static struct file_operations demo_fops = {
    .owner    = THIS_MODULE,
    .fo_open  = demo_open,
    .fo_close = demo_close,
    .fo_ioctl = demo_ioctl,
    .fo_read  = demo_read,
    .fo_write = demo_write,
    .fo_mmap  = demo_mmap,
};

int module_init (void)
{
    int ret = 0;

    drv_index = iosDrvInstallEx2(&demo_fops, LW_DRV_TYPE_NEW_1);
    if (drv_index < 0) {
        printk("driver install fail.\r\n");
        return -1;
    }

    DRIVER_LICENSE(drv_index, "GPL->Ver 2.0");
    DRIVER_AUTHOR(drv_index, "GeWenBin");
    DRIVER_DESCRIPTION(drv_index, "demo driver.");

    ret = iosDevAdd(&demo_dev[0].dev, "/dev/demo0", drv_index);
    if (ret != ERROR_NONE) {
        printk("device add fail.\r\n");
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    ret = iosDevAdd(&demo_dev[1].dev, "/dev/demo1", drv_index);
    if (ret != ERROR_NONE) {
        printk("device1 add fail.\r\n");
        iosDevDelete(&demo_dev[0].dev);
        iosDrvRemove(drv_index, TRUE);
        return -1;
    }

    SEL_WAKE_UP_LIST_INIT(&demo_dev[0].wakeup_list);
    SEL_WAKE_UP_LIST_INIT(&demo_dev[1].wakeup_list);

    vir_device_init();

    demo_dev[0].phy_addr = API_VmmPhyAlloc(4 * 1024);
    demo_dev[1].phy_addr = API_VmmPhyAlloc(4 * 1024);

#if 1
    demo_dev[0].map_flags = LW_VMM_FLAG_DMA;
    demo_dev[1].map_flags = LW_VMM_FLAG_DMA;
#else
    demo_dev[0].map_flags = LW_VMM_FLAG_RDWR;
    demo_dev[1].map_flags = LW_VMM_FLAG_RDWR;
#endif

    return 0;
}

void module_exit (void)
{
    iosDevDelete(&demo_dev[0].dev);
    iosDevDelete(&demo_dev[1].dev);
    iosDrvRemove(drv_index, TRUE);

    if (demo_dev[0].phy_addr)
        API_VmmPhyFree(demo_dev[0].phy_addr);

    if (demo_dev[1].phy_addr)
        API_VmmPhyFree(demo_dev[1].phy_addr);
}
```