---
title: App启动过程以及优化
date: 2020-01-26 04:43:32
tags:
---

> **App launch is a user experience interruption.**
> *想总结一下iOS App启动过程以及生命周期管理，第一遍整理*

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
> 了解Mach-O [Mach-O](../../../../2020/01/09/了解Mach-O文件/)

1. ReadyToLoad Mach-o 系统准备读取Mach-O文件
2. Read Mach-o Header 读取Mach-O文件头部
3. Kernel Load Dyld  加载动态连接器
4. Dyld Load  dylibs  动态链接动态库
5. Rebase/Bind rebase修复指向当前镜像内部的资源指针；bind修复镜像外部的资源指针

> [iOS 13中dyld 3的改进和优化](https://easeapi.com/blog/blog/83-ios13-dyld3.html)

<!--more-->

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

## 如何测量App启动耗时
在任何时间下我们的设备可能有不同的状态，这可能在启动测量的时候导致不同的结果差异，所以我们应该屏蔽掉这些变量引起的差异。

### 测试环境准备

1. 重启设备，并放置2-3分钟
2. 开启飞行模式或Mock网络数据，排除网络对启动阶段的影响
3. 关闭iCloud
4. 使用Release Build进行测试
5. 使用温启动进行测量

### 测试设备选择
我们通常会选择比较旧一点的设备来进行测试，旧的设备性能相对较差，所以执行速度要比新设备慢一些，需要大部分用户对启动速度的要求我们需要用比较慢的设备进行测试。

### 使用Instruments定位问题

1. Xcode11在Instruments中加入了AppLaunch模板 用于我们App测量启动过程，并记录分析。
在Xcode中选择Profiling的方式进行编译
![App Lauch](applaunch1.png)
2. 编译成功后会自动打开Instruments界面，选择 App Launch 模板。
![App Launch](applaunch2.png)
3. 点击红色按钮启动App，Instruments会自动记录并分析启动过程以及各个阶段的耗时。
![App Launch](applaunch4.png)
4. 选择Time Profile选项，就可以看到各个阶段的耗时，方便定位问题，用鼠标点击三下，可以选中当前阶段进行过滤数据。
![App Launch](applaunch5.png)
选择 App Life Cycle 可以看到各个阶段启动的时间，由于为了测试效果项目中在+load 以及 didFinishLaunch方法中添加了两秒的耗时操作。
```
Start	Narrative	
00:00.000.000	The system took 40.15 ms to create the process.	
00:00.040.147	The system frameworks took 258.11 ms to initialize.	
00:00.298.261	Initializing (Static Runtime Initialization)	
00:02.304.522	Launching (UIKit Initialization)	
00:02.347.344	Launching (UIKit Scene Creation)	
00:02.348.240	Launching (didFinishLaunchingWithOptions())	
00:04.349.327	Launching (UIKit Scene Creation)	
00:04.426.623	Launching (sceneWillConnectTo())	
00:04.426.632	Launching (UIKit Scene Creation)	
00:04.445.663	Launching (sceneWillEnterForeground())	
00:04.445.671	Launching (UIKit Scene Creation)	
00:04.446.794	Launching (Initial Frame Rendering)	
00:04.452.063	Currently running in the foreground...	

```
5. 切换线程显示到 Thread State模式 可以看到当前线程执行状态，线程执行状态可以通过颜色进行区分
 **灰色** 阻塞状态，线程没有执行任何操作
 **红色** 可执行状态，线程缺乏CPU资源
 **橙色** 被抢占状态，线程正在执行过程中被高优先级的线程获取到CPU资源而停止
 **蓝色** 执行状态 线程正在被CPU执行
![App Launch](applaunch6.png)
可以看到当前主线程正在被阻塞，因为在AppDelegate中的AppDidLaunch方法中添加了sleep(2)的操作。

通过以上方法可以方便定位到具体哪个阶段耗时操作比较多，可以具体分析项目中的逻辑或者业务代码并进行优化。

### XCTest
在UITest内Xcode自动帮我们生成好了用于测试App启动的测试用例
![App Launch](applaunch8.png)
执行此用例我们可以获取到App温启动5次的时间统计，方便我们进行测试统计。

## 启动优化
启动优化主要可以分为以下三个方向进行优化。
- **Minimize Work**
1. 推迟与第一帧显示无关的任何内容
2. 避免阻塞主线程，比如网络I/O，文件I/O等
3. 减少内存的使用量，分配和操作

- **Prioritize Work**
1. 使用正确的队列执行任务
2. 合理的处理跨线程并发操作
3. 使用正确的优先级执行任务，避免优先级翻转带来的主线程阻塞，执行任务的线程因为优先级太低获取不到CPU
   
- **Optimize Work**
1. 简化跟限制已有的工作，比如加载数据我们可以只加载第一屏显示的数据
2. 优化算法跟数据结构
3. 缓存资源以及计算结果，可以适当的使用空间换取运行时间。


## 总结
App启动优化要根据基础知识，结合工具针对App进行分析，定位问题，发现问题所在，问题就已经解决了一半。