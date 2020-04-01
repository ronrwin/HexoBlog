---
title: 电量监控----battery-historian
date: 2016-05-24 23:15:50
description: battery-historian是谷歌推荐使用的电量检测工具，我们看下应该如何使用 
tags:
- 性能
---
在Android项目中，较难监控应用的电量消耗，但是用户却非常关心手机的待机时间。 过度耗电的应用， 会遭到用户无情的卸载，不要存在侥幸心理。

battery-historian是一款由Google提供的Android系统电量分析工具。该工具支持Android 5.0 Lollipop（API 21）以上。也可以参照官方的文档：http://developer.android.com/intl/zh-cn/tools/performance/batterystats-battery-historian/index.html

## 准备工作
1、GO语言及安装
​    - 下载地址：http://golang.org/doc/install 。
​    - 建立工作路径，参考http://golang.org/doc/code.html#Organization.
​    - 设置环境变量。（GO安装执行完后自动设置了环境变量。如果没有的话，需要自己手动设置）
2、安装Git，链接https://git-scm.com/downloads 。
3、安装Python 2.7（不是Python 3.x），链接https://python.org/downloads 。
4、安装Java，http://www.oracle.com/technetwork/java/javase/downloads/index.html.
5、下载Battery Historian :
```go
$ go get -d -u github.com/google/battery-historian/
```
6、安装并运行Battery Historian 
```go
$ cd $GOPATH/src/github.com/google/battery-historian
 
# Compile Javascript files using the Closure compiler
$ go run setup.go
 
# Run Historian on your machine (make sure $PATH contains $GOBIN)
$ go run cmd/battery-historian/battery-historian.go [--port <default:9999>]
```
执行之后，访问http://localhost:9999/就可以开始分析过程了。

## 步骤
#### 重置数据
```
adb shell dumpsys batterystats --enable full-wake-history
shell dumpsys batterystats --reset
```
#### 输出bugreport文件
```
adb bugreport > bugreport.txt
```

#### 生成分析文件
我们有两种方式来获取分析结果文件
1、使用battery historian包里的python脚本来生成html文件。
```
> adb shell dumpsys batterystats > batterystats.txt
> python scripts/historian.py batterystats.txt > batterystats.html
```
打开batterystats.html，我们可以看到如下的分析界面了：
![监控](/images/battery-historian/battery-historian-local.png "battery-historian.html")

2、上传bugreport文件分析
我们进入到battery-historian目录，执行启动指令：

```
cd $GOPATH/src/github.com/google/battery-historian
go run cmd/battery-historian/battery-historian.go [--port <default:9999>]
```
访问http://localhost:9999，上传bugreport.txt文件，之后就可以得到下面的分析界面：
![联网分析1](/images/battery-historian/battery-historian-web1.png)
![联网分析2](/images/battery-historian/battery-historian-web2.png)
切换到App Stats，可以分析具体的app情况
![联网分析3](/images/battery-historian/battery-historian-web3.png)


## 指标说明
#### 横坐标
横坐标是表示时间段的分割，方便我们观察某个时间段或时间跨度的的数据。

#### 纵坐标
battery_level：当前电量值。
plugged：是否充电状态。
screen：屏幕是否点亮，这一点可以考虑到睡眠状态和点亮状态下电量的使用信息。
top：该栏显示当前时刻哪个app处于最上层，就是当前手机运行的app，用来判断某个app对手机电量的影响，这样也能判断出该app的耗电量信息。该栏记录了应用在某一个时刻启动，以及运行的时间，这对我们比对不同应用对性能的影响有很大的帮助。
wake_lock：记录wake_lock模块的工作时间。是否有停止的时候等。
sync：发生同步操作的对象。
running：界面的状态，主要判断是否处于idle的状态。用来判断无操作状态下电量的消耗。
wake_reason：唤醒的原因。
wake_lock_in：wake_lock有不同的组件，这个地方记录在某一个时刻，有哪些部件开始工作，以及工作的时间。
conn：数据连接方式。是无连接，还是wifi、2g/3g/4g。
status：电池状态信息，有充电，放电，未充电，已充满，未知等不同状态。 这一栏记录了电池状态的改变信息。
phone_signal_strength：手机信号强度变化
wifi_signal_strength：wifi信号强度。

## 总结
这个工具是谷歌官方使用的电量监测工具。数据来源是从手机的bugreport中获取，并绘制成图表以方便我们作分析。由于电量的记录是以电池状态为主，所以我们不能从该分析中准确知道哪个应用使用了多少电量。我们可以从电量的使用效率趋势上面获取到大概的信息，并从具体的app分析结果中得出大概的电量使用趋势。