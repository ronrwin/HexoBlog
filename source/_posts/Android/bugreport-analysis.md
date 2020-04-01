---
title: bugreport分析工具----ChkBugReport简介
date: 2016-05-17 22:12:25
description: ChkBugReport是一个开源工具，它可以把你得到的bugreprot解析成适合阅读的html文件。导出的html文件包含了根据bugreport数据得出的图表和分析结论。
tags:
- Android
---
原文链接：[New bugreport analysis tool released as open source](http://developer.sonymobile.com/2012/01/25/new-bugreport-analysis-tool-released-as-open-source/)

在安卓开发过程中，是否经常会遇到 `Application Not Responding (ANR)`信息和崩溃信息？这些信息都记录在bugreport文件中，但是信息繁杂，我们该如何准确筛选需要的信息呢？这需要使用到ChkBugReport来辅助分析。

下载地址：https://github.com/sonyxperiadev/ChkBugReport/downloads

## 如何使用
把jar文件下载下来。

从手机中导出bugreport文件
```
adb shell bugreport > bugreport.txt
```

之后运行ChkBugReport:
```
java –jar chkbugreport.jar bugreport.txt
```

执行之后将生成html文件。

## 了解ChkBugReport的输出
bugreport文件包括了几部分，每一部分都是一个文件的内容拷贝（例如`/data/system/packages.xml`）、一个应用的输出（例如`top`）或者是一个服务的堆栈（例如`window manager service`）。

ChkBugReport解析该文件并分成几个部分，方便用户快速找到想要分析的部分。这些部分会保存在`bugreport_out/raw`，以便用户方便找到。

作为第二步，工具执行一系列的build-in插件，负责运行一个或者几个分析的部分。这些插件的输出会保存到报告里分开的章节中，最终保存为一个html文件。在浏览器中打开`index.html`，就可以开始分析bugreport了。
![bugreport截图](/images/chkbugreport/chkbugreport_screenshot_001.png "bugreport截图")

插件还可以检测错误。例如，系统日志分析可以找到ANR报告。这些都被归纳到`Errors`部分中，可以方便找到。

每个插件会提供每个进程的片段，叫`Processes`。这样，你可以找到某个特定进程的所有信息（内存使用， 系统日志等等）。

现在，ChkBugReport有实现如下插件
+ BatteryInfoPlugin ---- 提取电量使用信息并画出消耗图表。
+ CpuFreqPlugin ---- 展示CPU支持的频率，以及它们的使用时间。
+ SystemLogPlugin, MainLogPlugin 以及 EventLogPlugin ---- 这些插件分别提取下面几部分的信息：
    + GC日志的内存使用
	+ 数据库和ContentProvider的时间数据
	+ 进程和服务生命周期图表
	+ 崩溃和ANR
	+ 配置修改
	+ Activitiy启动
+ FTracePlugin  ---- 如果[FTrace](http://www.mjmwired.net/kernel/Documentation/trace/ftrace.txt)功能在内核中可用，Android bugreport工具则支持保存追溯缓冲。插件可以分析追溯和建立统计并归纳成图表。例如，展示哪一个线程在运行以及等待时长。并且会保存VCD格式的信息，所以我们可以使用[GtkWave](http://gtkwave.sourceforge.net/)分析追溯信息
+ MemPlugin ---- 从不同的资源中提取内存使用
+ StackTracePlugin ---- 分析栈追溯并且侦测一些程序错误（例如，尝试在主线程建立网络连接）。
+ SurfaceFlingerPlugin ---- 层级可视化
+ WindowManagerPlugin ---- 重构窗口列表

这是ChkBugReport功能的一个例子，这个工具从bugreport提取了很多有价值的信息。我们会不断在[Github](https://github.com/sonyxperiadev/ChkBugReport/wiki)上增加新功能，会逐个解释功能。我们也会在[ChkBugReport forum thread](http://forum.xda-developers.com/showthread.php?p=21804810#post21804810)XDA论坛更新。

更多资料：
+ 从[ChkBugReport forum thread](http://forum.xda-developers.com/showthread.php?p=21804810#post21804810)获取最新更新
+ 下载[ChkBugReport](https://github.com/sonyericssondev/ChkBugReport)源代码
+ Wiki: https://github.com/sonyxperiadev/ChkBugReport/wiki/
+ [安装指导](https://github.com/sonyxperiadev/ChkBugReport/wiki/How-to-install-it)
+ 阅读[FTRACE](http://www.mjmwired.net/kernel/Documentation/trace/ftrace.txt)功能
+ [GtkWave](http://gtkwave.sourceforge.net/)
+ 学习更多关于GC--[Garbage Collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))









