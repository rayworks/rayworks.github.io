---
layout: post
title: 获取 Android app 内部各个thread的信息
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

在App调试或面试✏️的过程中，一个常见的问题是：如何获取当前的thread的状态信息。这一方法对于app 性能分析，或是解决运行中的死锁问题，往往显得很有用处。

常见的场景是，线程A获取了Lock A， 线程B获取了Lock B；随后线程A进一步想获取Lock B，线程B进一步想获取Lock A，这样就出现了死锁。


```
thread A {
  lockA.lock();
  //...
  lockB.lock();
}
```

```
thread B {
  lockB.lock();
  //...
  lockA.lock();
}
```

这如果发生在主线程中，则会导致界面冻住，跳出ANR的弹框。
![ANR](../../../assets/images/test_anr.png)


为了便于调试和分析，让我们在genymotion模拟器上调试运行目标app，重现上述场景。
随后在命令行中执行：

1) `adb shell ps -ef | grep <pkg-name>`
查找到当前app的进程id  e.g. pid1

1) `adb shell run-as <pkg-name> kill -3 <pid1>`

|  action   | code       | comments  |  note |
| ----------- | ----------- | ----------- | ----------- |
|SIGQUIT |     3    |      /* Quit (POSIX).  */      |      建立CORE文件终止进程，并且生成core文件 |

向当前app发送 `QUIT` signal，此时 `console` 中会显示 `Wrote stack traces to tombstoned`，已将应用的stack信息记录下来了。

3) `adb bugreport ./bugreport.zip`
将出错信息导出到当前目录下的 `bugreport.zip` 文件中，解压以后就能看到详细的状态信息了
(更一般的，如果出现app已经出现了ANR，也可以直接从/data/anr/ 目录下使用 `adb pull` 把anr log 导出来)。

e.g.
```
"main" prio=5 tid=1 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x729291f0 self=0xe9c3ae00
  | sysTid=2742 nice=-10 cgrp=default sched=0/0 handle=0xea48bdc8
  | state=S schedstat=( 1000841099 167892657 575 ) utm=76 stm=23 core=1 HZ=100
  | stack=0xff23e000-0xff240000 stackSize=8192KB
  | held mutexes=
  at sun.misc.Unsafe.park(Native method)
  - waiting on an unknown object
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:902)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1227)
  at java.util.concurrent.locks.ReentrantReadWriteLock$WriteLock.lock(ReentrantReadWriteLock.java:950)
  at com.example.locktest.MainActivity.test(MainActivity.kt:65)
  at com.example.locktest.MainActivity.onCreate$lambda$2$lambda$1(MainActivity.kt:52)
  at com.example.locktest.MainActivity.$r8$lambda$97V3BXrElnLtyrlpwFsDN9nFJMY(MainActivity.kt:-1)
  at com.example.locktest.MainActivity$$ExternalSyntheticLambda1.onClick(D8$$SyntheticClass:-1)
  at com.google.android.material.snackbar.Snackbar$1.onClick(Snackbar.java:351)
  at android.view.View.performClick(View.java:7259)
  at com.google.android.material.button.MaterialButton.performClick(MaterialButton.java:1131)
  at android.view.View.performClickInternal(View.java:7236)
  at android.view.View.access$3600(View.java:801)
  at android.view.View$PerformClick.run(View.java:27892)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7356)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
```

Ref :
* [Default tombstones location in android](https://stackoverflow.com/questions/28105054/default-tombstones-location-in-android)

* [How to make Java Thread Dump in Android?](https://stackoverflow.com/questions/13589074/how-to-make-java-thread-dump-in-android)

* [Linux下signal信号汇总](https://www.cnblogs.com/frisk/p/11602973.html)



