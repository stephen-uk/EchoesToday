---
title: App启动以及生命周期管理（一）
date: 2020-01-9 11:13:21
tags:
---

> **App launch is a user experience interruption.**
> *想总结一下iOS App启动过程以及生命周期管理，第一遍整理*

在Windows下我们的可执行文件一般是.exe文件，iOS以及macOS系统中是我们常说的Mach-O文件，常见的文件有我们的可执行文件，动态库，以及动态链接库等。所以先简单了解一下可执行文件。

## 了解Mach-O
Mach-O,是Mach Object的文件格式的缩写，是一种用于记录可执行文件，对象代码，共享库，动态加载代码和内存转储的文件格式，是macOS/iOS上程序以及库的标准格式。 Mach-O文件可以通过MachOView打开查看。

### Mach-O 常见应用
Mach-O，在苹果头文件中定义的类型有11种, 常用的有`MH_OBJECT`，`MH_EXECUTE`，`MH_DYLIB`，`MH_DYLINKER`，以及我们常见的DSYM文件都是Mach-O文件

``` C
#define MH_OBJECT   0x1     /* 目标文件*/
#define MH_EXECUTE  0x2     /* 可执行文件 */
#define MH_FVMLIB   0x3     /* fixed VM shared library file */
#define MH_CORE     0x4     /*核心转储文件 */
#define MH_PRELOAD  0x5     /* preloaded executable file */
#define MH_DYLIB    0x6     /* 动态库 */
#define MH_DYLINKER 0x7     /* 动态链接器 */
#define MH_BUNDLE   0x8     /* dynamically bound bundle file */
#define MH_DYLIB_STUB   0x9     /* shared library stub for static */
                    /*  linking only, no section contents */
#define MH_DSYM     0xa     /* companion file with only debug */
                    /*  sections */
#define MH_KEXT_BUNDLE  0xb     /* x86_64 kexts */
```
[源码](https://opensource.apple.com/tarballs/xnu/)

<!--more-->

### Mach-O文件结构
Mach-O文件主要包含三个区域
+ Header ： 记录文件类型，目标CPU架构，Magic Number等。
+ Load Command ：记录描述文件在虚拟内存中的逻辑与布局
+ Raw Segment Data ： 原始数据

结构图
![结构图](mach-o.png)
  
#### Mach-O Header
下面👇是Mach-O 文件header的定义

``` c
struct mach_header_64 {
    uint32_t    magic;        /* mach magic number identifier 魔数*/
    cpu_type_t    cputype;    /* cpu specifier 支持的CPU架构类型*/
    cpu_subtype_t    cpusubtype;    /* machine specifier */
    uint32_t    filetype;    /* type of file  文件类型*/
    uint32_t    ncmds;        /* number of load commands */
    uint32_t    sizeofcmds;    /* the size of all the load commands */
    uint32_t    flags;        /* flags */
    uint32_t    reserved;    /* reserved */
};

```
`magic`: 魔数，用来标识Mach-O平台的属性，确认文件的类型，操作系统在加载时会进行校验，如果校验不过会拒绝加载。
`cputype`: 支持的CPU类型 比如 Arm
`cpu_subtype_t` : 具体支持的CPU类型
`filetype` : 文件类型
`ncmds` : 加载命令条数
`sizeofcmds` : 加载命令的大小

具体可以看图:

![Mach-O Header](mach-o_header.png)

#### Mach-O Load Commands

Mach-O Header之后就是Load commands,也就是加载命令。加载命令的作用是在Mach-O文件被加载到内存时，加载命令告诉内核加载器或者动态链接器如何调用。还是先试用MachOview看一下Mach-O文件中的加载命令：

![Mach-O Load Commands](mach-o_load_commands.png)

命令结构定义如下
``` C
struct load_command {
    uint32_t cmd;        /* type of load command */
    uint32_t cmdsize;    /* total size of command in bytes */
};
```

`cmdsize` ：具体的load_command结构所占内存的大小。
`cmd` : 具体的加载类型，比如：

```
#define LC_SEGMENT  0x1 /* segment of this file to be mapped */
#define LC_THREAD   0x4 /* thread */
#define LC_UNIXTHREAD   0x5 /* unix thread (includes a stack) */
#define LC_PREPAGE      0xa     /* prepage command (internal use) */
#define LC_DYSYMTAB 0xb /* dynamic link-edit symbol table info */
#define LC_LOAD_DYLIB   0xc /* load a dynamically linked shared library */
#define LC_CODE_SIGNATURE 0x1d   /* local of code signature */
……
```
LC_SEGMENT是一个段加载命令；LC_LOAD_DYLIB表示这事一个需要动态加载的链接库；LC_CODE_SIGNATURE和签名、加密有关。
上图中看到加载命令是LC_SEGMENT_64，表示的是将64位的段映射到进程的地址空间。可以看一下段加载命令的数据结构：

```
struct segment_command_64 { /* for 64-bit architectures */
    uint32_t    cmd;        /* LC_SEGMENT_64 */
    uint32_t    cmdsize;    /* includes sizeof section_64 structs */
    char        segname[16];    /* segment name */
    uint64_t    vmaddr;     /* memory address of this segment */
    uint64_t    vmsize;     /* memory size of this segment */
    uint64_t    fileoff;    /* file offset of this segment */
    uint64_t    filesize;   /* amount to map from the file */
    vm_prot_t   maxprot;    /* maximum VM protection */
    vm_prot_t   initprot;   /* initial VM protection */
    uint32_t    nsects;     /* number of sections in segment */
    uint32_t    flags;      /* flags */
};

```
sectname字段表示该section的name。
segname字段表示该section所属的segment的segmentName。
addr字段表示该section的内存起始地址。
size字段表示该section的大小。
offset字段表示该section相对文件的偏移量。
align字段表示字节区的内存对齐边界。
reloff表示重定位信息的文件偏移。
nreloc表示重定位条目的数目。
flags是section的一些标志属性。

#### Mach-O Data
Mach-O中Load Commands之后的就是Data数据。每个段的数据都保存在这里，这里存放了具体的数据与代码。由于Mach-O Data中的内容更多的与具体的数据有关，而与格式无关。


## App启动类型
 App启动过程有冷启动，温启动，恢复。 具体什么情况下是那种启动方式可以看下面摘自WWDC表格。

Cold | Warm |  Resume  
-|-|-
After reboot | Recently terminated | App is suspended |
App is not in memory | App is partially in memory | App is fully in memory |
No process exists | No process exists | Process exists |

## App 启动过程

**System Interface -> Runtime -> UIKit Init -> Init Application -> Init Initial Frame Render -> Extended**

### System Interface
该阶段是系统准备阶段，主要有读取Mach-O文件，加载动态连接器并连接动态库以及重新绑定符号表。
1. ReadyToLoad Mach-o 系统准备读取Mach-O文件
2. Read Mach-o Header 读取Mach-O文件头部
3. Kernel Load Dyld  加载动态连接器
4. Dyld Load  dylibs  动态链接动态库
5. Rebase/Bind rebase修复指向当前镜像内部的资源指针；bind修复镜像外部的资源指针

> [iOS 13中dyld 3的改进和优化](https://easeapi.com/blog/blog/83-ios13-dyld3.html)

### Runtime
初始化Objc运行环境。[Objc4 源码](https://github.com/opensource-apple/objc4)
主要过程为初始化 所有类对象以及元类对象，加载分类并合并方法列表，以及调用+load方法。
至此app已经有了最基本的运行环境。

### UIKit Init
系统调用程序中的Main方法，Main方法中创建 `UIApplication`,并设置`UIApplicationDelegate`

### Init Application
该过程主要就是我们在appdelegate中的代理方法。

![AppDelegate](app_delegate.png)

主要调用顺序为
``` c
- application:willFinishLaunchingWithOptions: 
Tells the delegate that the launch process has begun but that state restoration has not yet occurred.

- application:didFinishLaunchingWithOptions:
Tells the delegate that the launch process is almost done and the app is almost ready to run.

- applicationWillEnterForeground:
Tells the delegate that the app is about to enter the foreground.

- applicationDidBecomeActive:
Tells the delegate that the app has become active.

- applicationWillResignActive:
Tells the delegate that the app is about to become inactive.

- applicationDidEnterBackground:
Tells the delegate that the app is now in the background.

- applicationWillTerminate:
Tells the delegate when the app is about to terminate.

```

> iOS 13中 SenceDelegate 适配可以看这个 [iOS13 UISenceSession](../../../../2020/01/01/iOS13-UISceneSession/)

### Init Initial Frame Render 
Creates, performs layout for, and draws views.
调用第一个viewContoller的loadView，以及viewDidLoad跟layoutSubviews等方法。

### Extended
- App-specific period after first frame.
- Displays asynchronously loaded data.
- App should be interactive and responsive.



