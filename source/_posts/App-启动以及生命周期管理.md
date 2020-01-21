---
title: App 启动以及生命周期管理
date: 2020-01-21 11:13:21
tags:
---

> 想总结一下iOS App启动过程以及生命周期管理，第一遍整理

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
结构如下
``` C
struct load_command {
    uint32_t cmd;        /* type of load command */
    uint32_t cmdsize;    /* total size of command in bytes */
};
```
