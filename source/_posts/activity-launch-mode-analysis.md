---
title: Activity 启动模式分析
date: 2016-10-06
tags:
  - Android
---

### 前言 
启动模式决定了 Activity 是怎样被启动的。启动模式一共有四种，分别是 standard，singleTop，singleTask，singleInstance，默认的启动类型是 standard。在文中，我们会涉及到两个概念，分别是任务和返回栈，我们看看谷歌官方是如何定义的。

> 任务是指在执行特定作业时与用户交互的一系列 Activity。 这些 Activity 按照各自的打开顺序排列在堆栈（即“返回栈”）中。

按我的理解，返回栈不过是任务的存储结构而已。

### 使用

```java
<activity
    android:name=".SecondActivity"
    android:launchMode="standard">
</activity>
```
通过`intent.addFlags(Intent.FLAG_ACTIVITY_*)`也能起到和`android:launchMode=""`相同的作用。经过测试，在两者都设置了的情况下，前者的优先级更高。
<!--more-->

 - Intent.FLAG_ACTIVITY_NEW_TASK
使用一个新的 Task 来启动一个 Activity，但启动的每个 Activity 都将在一个新的 Task 中。该 Flag 通常用于从 Service 启动 Activity 的场景，由于在 Service 中并不存在 Activity 栈，所以使用该 Flag 来创建一个新的 Activity 栈，并创建新的 Activity 实例。
 - Intent.FLAG_ACTIVITY_SINGLE_TOP
使用 singletop 模式来启动一个 Activity，与指定 `android:launchMode="singleTop"`效果相同。
 - Intent.FLAG_ACTIVITY_CLEAR_TOP
使用 singletask 模式来启动一个 Activity，与指定 `android:launchMode="singleTask"`效果相同。
 - Intent.FLAG_ACTIVITY_NO_HISTORY
使用这种模式启动 Activity，当该 Activity 启动其他 Activity 后，该 Activity就消失了，不会保留在栈中，例如 A-B，B 中以这种模式启动 C，C再启动 D，则当前 Activity 栈为 ABD。

### standard
该模式是启动模式中的默认模式，也是最简单的一种模式，在每次启动该模式的 Activity 时，均会创建该 Activity 的一个实例。看到网上有文章说，在该模式下，由程序 A->FirstActivity 启动 B->FirstActivity，在 Android 5.0 之前与之后版本中的任务管理器会有不能的表现，我在真机（Android 4.4.2）与模拟器（Android 5.1）得出的结论是任务管理器中的变现是一样的。

如下图，任务管理器中只有一个任务：
![在不同系统版本中任务管理器的表现 | center][1]

### singleTop
singleTop 与 standard 基本上一致，唯一的区别就是当创建该模式的 Activity 位于调用者任务栈的栈顶时，则不创建实例，直接使用该 Activity 实例，以`onNewIntent()`进行调用。由于 singleTop 与 standard 的行为太相似了，所以在谷歌的官方文档中，这两种启动方式被分为一组，singleTask 与 singleInstance 被分为另一组。
### singleTask
我们先看看官方文档对 singleTask 的描述：

> The system creates the activity at the root of a new task and routes the intent to it. However, if an instance of the activity already exists, the system routes the intent to existing instance through a call to its onNewIntent() method, rather than creating a new one.

译：系统将会在一个新的任务中创建该 Activity，并把他放置该任务的根部，然而如果该 Activity 的实例已经存在，那么则不会创建他，而会通过`onNewIntent()`来调用他。

文档的其他地方也有这么一句话：
> A "singleTask" activity allows other activities to be part of its task. It's always at the root of its task, but other activities (necessarily "standard" and "singleTop" activities) can be launched into that task. 

文档说，singleTask 的 Activity 总是位于任务的根部。可是通过下面的描述，会发现，事实并不如此。

在这里，我们用 FirstActivity 来启动 singleTask：SecondActivity ，两个 Activity 在同一应用中。程序运行完之后，我们通过`adb shell dumpsys activity`查看返回栈的详细情况。此时，我要安利一个在 Windows 下替代 cmd 和 PowerShell 的软件 ———— [Cmder][2]，界面优美，自定义功能多，用户体验好。

```java
 Stack #2 [Normal] id=6:
    Task id #10
      TaskRecord{450fad00 #10 A=me.zhengken.launchmodedemo U=0 sz=2}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=me.zhengken.launchmodedemo/.FirstActivity }
        Hist #1: ActivityRecord{44d808f0 u0 me.zhengken.launchmodedemo/.SecondActivity t10}
          Intent { cmp=me.zhengken.launchmodedemo/.SecondActivity }
          ProcessRecord{43bbaff8 26560:me.zhengken.launchmodedemo/u0a210}
        Hist #0: ActivityRecord{45a508b0 u0 me.zhengken.launchmodedemo/.FirstActivity t10}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=me.zhengken.launchmodedemo/.FirstActivity }
          ProcessRecord{43bbaff8 26560:me.zhengken.launchmodedemo/u0a210}
```

我们发现`Hist #1`的 SecondActivity 与 First Activity 在同一任务中，且前者并不位于返回栈的根部。那么有没有办法让 SecondActivity 位于新建任务的根部呢？

> The affinity of an activity is defined by the taskAffinity attribute. The affinity of a task is determined by reading the affinity of its root activity.
> To specify that the activity does not have an affinity for any task, set it to an empty string.

我们先来看一个属性：taskAffinity。Activity 和任务都有这个属性，如果 Activity 没有指定该属性，那么默认是包名，任务的 taskAffinity 由返回栈中栈底 Activity 的 taskAffinity 所决定的。Activity 会放在与之 taskAffinity 的值相等的任务中。当 Activity 中 taskAffinity 设为空的时候，那么该 Activity 不属于任何任务。

所以，我们在指定 singleTask 的同时，指定 taskAffinity，我们就可以在新的任务中启动活动了。

那么如果我们 FirstActivity-> singleTask & taskAffinity = ""：SecondActivity - > singleTask & taskAffinity = ""：ThirdActivity时，会怎样呢？

结果是三个活动会位于不同的任务中，且在后台也显示三个任务。
![任务管理器中的三个任务][3]

```java
Running activities (most recent first):
      TaskRecord{456f0c70 #18 I=me.zhengken.launchmodedemo/.ThirdActivity U=0 sz=1}
        Run #2: ActivityRecord{43aff050 u0 me.zhengken.launchmodedemo/.ThirdActivity t18}
      TaskRecord{4510bbd0 #17 I=me.zhengken.launchmodedemo/.SecondActivity U=0 sz=1}
        Run #1: ActivityRecord{431d9950 u0 me.zhengken.launchmodedemo/.SecondActivity t17}
      TaskRecord{44d40e28 #16 A=me.zhengken.launchmodedemo U=0 sz=1}
        Run #0: ActivityRecord{455fed68 u0 me.zhengken.launchmodedemo/.FirstActivity t16}
```

这里，我们对 singleTask 做一个小结：首先查找是否存在与 taskAffinity 值一样的任务；如果存在，那么继续查找在该任务中是否存在该 Activity，若存在，则清除其顶部所有的 Activity，通过`onNewIntent()`调用使其位于栈顶，若不存在，则创建一个该 Activity 的实例；如果不存在这样的任务，则在新的任务中创建该 Activity。

### singleInstance

singleInstance 与 singleTask 的行为类似，唯一不同的地方就是存放 singleInstance Activity A 的任务有且只能有 A 。如果 A 被任意其他的 Activity 启动，A 会放置在不同的任务中，或者 A 启动其他任意的 Activity 时，也会把 Activity 放置在不同的任务中。

### 使用建议

> As shown in the table above, standard is the default mode and is appropriate for most types of activities. SingleTop is also a common and useful launch mode for many types of activities. The other modes — singleTask and singleInstance — are **not appropriate for most applications**, since they result in an interaction model that is likely to be unfamiliar to users and is very different from most other applications.

上面总结起来就是一句话：要慎重使用 singleTask 和 singleInstance。

### 参考文献
1. https://developer.android.com/guide/topics/manifest/activity-element.html#lmode
2. http://droidyue.com/blog/2015/08/16/dive-into-android-activity-launchmode/
3. http://blog.csdn.net/luoshengyang/article/details/6714543
4. https://developer.android.com/guide/topics/manifest/application-element.html
5. [Android群英传-徐宜生][4]


  [1]: http://odc50o546.bkt.clouddn.com/android-launch-mode-analytise-1.png
  [2]: http://cmder.net/
  [3]: http://odc50o546.bkt.clouddn.com/android-launch-mode-analytise-2.png
  [4]: https://item.jd.com/11758334.html