---
layout: post
title: Android App 性能优化实践
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

# 1. 问题发现
在最近的一次 feature 开发中，我发现单个页面上 button pressed 的状态在手指移开后1s 左右才生效，整个过程像是镜头回放的慢动作一样。在查看了[metrics tool](https://github.com/frogermcs/AndroidDevMetrics) 中对应 Activity 的性能指标后，我意识到应该是有比较严重的性能问题了：layout 耗费了2~3s的时间，同时出现严重的掉帧。
![before_opt-crunch.png](../../../assets/images/before_opt.png)


# 2. 首次尝试
首先想到的方案是查看当前Activity的布局，通过`adb shell dumpsys activity pkg-name` 看到当前的 View Hierarchy 层级深度已经达到了14层。整个布局深度包括了系统的id 为 `android:id/content`的`FrameLayout`至Root View 的4层，以及第三方某自定义的6层布局。

重新调整layout文件：
* 使用`<merge/>` 标签对运行过程中动态 inflate 的View 布局进行优化
* 将Activity的顶层 `ViewGroup` 设置为 `ConstraintLayout`，以便减少冗余的层级。

在调整完成后，再次开启监控模式中的`GPU条形图`发现性能有所提升，但是整体来看仍不明显。layout耗时以及掉帧的问题依旧存在。

# 3. Profile 瓶颈
完成2中的调整后，我发觉必须要能定位到瓶颈点才能从根本上解决这次的性能问题。对比官方文档 [profile-gpu](https://developer.android.com/topic/performance/rendering/profile-gpu) 中的GPU柱状图颜色的说明，可以初步判定问题极有可能出现在 `Animation` 和 `Measure` 的节点上。

更进一步的，我开始了 [profile CPU](https://developer.android.com/studio/profile/cpu-profiler), 通过 `record trace` 在Sample(Java) 模式下发现了指定的间隔时间段调用最频繁的方法，`measure` 方法的频繁调用果然成为了整个的瓶颈部分：
![Screen Shot 2018-12-04 at 23.43.15.png](../../../assets/images/method_trace.png)


切换到 Instrumented (Java) 模式后再次进行观察，发现调用最频繁的方法终于浮出水面：
![Screen Shot 2018-12-04 at 11.33.42.png](../../../assets/images/method_hotspot.png)

# 4. 真相大白
原来是Video播放过程中，进度更新的回调方法里面，有个逻辑是通过 `TextView#setText()` 直接更新当前的播放时间戳 MM:SS 。而播放进度的刷新频率被设置成了 33ms. ( ⊙ o ⊙ )！

将刷新时间戳的逻辑改成秒级刷新后，UI表现终于恢复正常了：

![after_opt-crunch.png](../../../assets/images/after_opt.png)

# 5. 后记
P.S. 
2018.12.05 经过与一位同事的讨论发现，如果将显示播放的时间戳的 `TextView` 布局中的 `layout_width` 设置为固定的值（之前的设置是`wrap_content`!），则也可以完全避免这个问题。






