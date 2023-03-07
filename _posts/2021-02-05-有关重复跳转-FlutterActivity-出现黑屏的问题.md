---
layout: post
title: 有关重复跳转 FlutterActivity 出现黑屏的问题
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

## 1. 问题表现

最近的功能中有一部分需要集成Flutter到现有app中，特定的场景是登录页(`LoginActivity`)，也有入口可以跳转到Flutter相关的页面中。而开发中爆出的问题最明显的有：重复进出`FlutterActivity`后，出现所在Activity黑屏。

由于每次启动是复用在Application中早已预热过的engine，即启动的`FlutterActivity`会共用同一engine。

## 2. 观察与推断

### 2.1 Flutter相关的模块

最直接的想法是看Flutter相关的模块，而从log中发现有

```
E/FlutterEnginePluginRegistry: Attempted to notify ActivityAware plugins of onSaveInstanceState, but no Activity was attached.
```

进而从注册的`FlutterPlugin`中观察其与Activity 对象的关联情况，果然出现了 Host Activity 与 `FlutterPlugin` 关联异常的情况，快速切换时，由于某种原因`onDetachedFromActivity` 被触发了，即此时Plugin关联的`methodChannel` 也被置为`null`。因为跳转的Flutter中的对应的Route是在Channel方法中指定的，而此时 Channel 处于未初始化的状态，所以呈现的是黑屏的状态。

### 2.2 Android 中Activity的生命周期

从 `LoginActivity` 进入`FlutterActivity` 然后返回出来，观察到`FlutterActivity` 从Paused到Destroyed 竟然耗时10s ! 

```
2021-02-05 21:51:52.625 8473-8473/com.your.pkg.name W/DroidApp: >>> activity Paused EngageFlutterActivity
2021-02-05 21:51:52.626 8473-8553/com.your.pkg.name I/flutter: >>> AppState : AppLifecycleState.inactive
2021-02-05 21:51:52.633 8473-8473/com.your.pkg.name D/AudioManager: dispatching onAudioFocusChange(-2) to android.media.AudioManager@a8d9aaacom.google.android.exoplayer2.AudioFocusManager$AudioFocusListener@ce6d69b
2021-02-05 21:51:52.636 8473-8473/com.your.pkg.name I/AndroidRunningStateService: still foreground
2021-02-05 21:51:52.637 8473-8473/com.your.pkg.name W/DroidApp: >>> activity Resumed LoginActivity
2021-02-05 21:51:52.638 8473-8791/com.your.pkg.name D/AudioTrack: ClientUid 10563 AudioTrack::pause 
2021-02-05 21:51:52.662 8473-8496/com.your.pkg.name D/DecorView: onWindowFocusChangedFromViewRoot hasFocus: true, DecorView@38a23b1[LoginActivity]
2021-02-05 21:52:02.640 8473-8553/com.your.pkg.name I/flutter: >>> AppState : AppLifecycleState.paused
2021-02-05 21:52:02.641 8473-8473/com.your.pkg.name I/AndroidRunningStateService: Active Activities: 3
2021-02-05 21:52:02.642 8473-8473/com.your.pkg.name W/DroidApp: >>> activity Destroyed EngageFlutterActivity
2021-02-05 21:52:02.650 8473-8553/com.your.pkg.name I/flutter: >>> AppState : AppLifecycleState.detached
```

### 2.3 推断

从上面的线索可以初步推断，主线程中有性能的问题从而影响了`FlutterActivity`正常的生命周期。

## 3. 性能分析 Profile

在AS中开启Profiler，选择CPU一项，通过`record trace` 在Sample(Java method) 模式下发现原来在自定义的View中，onDraw()方法成为性能的瓶颈：

![image.png](../../../assets/images/ondraw_edittext.png)


## 4. 后记

问题表象和实际的原因往往并不一致，只有抽离表象看到问题的实质，才能有所突破。本例中有大半的时间是花费在了Flutter模块，尝试定位问题；实际上**高效的**性能剖析之后，答案才水落石出。

也许你也会关注 Android Studio 中 Profiler的应用，这里还有一篇UI性能问题的分析的文章，[Android App 性能优化实践](https://www.jianshu.com/p/416d2b9e08d6)。
