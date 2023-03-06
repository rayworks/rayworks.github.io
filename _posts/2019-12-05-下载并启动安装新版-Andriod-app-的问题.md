---
layout: post
title: 下载并启动安装新版 Andriod apk 的问题
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

## 1. 应用场景

对已发布的App如果出现重大升级，常常会启用 force update；或是检测到有新版本的时候提示用户主动下载安装。

## 2. 常见实现策略

而对于大文件下载，常见的实现策略之一就是使用系统的 `DownloadManager` 开启下载，监听下载完成的 action，依照 status 做后续的操作。

## 3. xiaomi 10.0上碰到的问题

按步骤2来看一般是不大容易出现问题，但是在特定的厂商设备（以MI9 以及 HUAWEI Mate30 OS 10.0为例）却出现了。网络条件良好的情况下，下载可以正常完成，但是对于随后自动发出的请求安装新apk 的 intent， 系统却没有任何响应也没有任何的报错信息。

## 4. 对应的解决过程

### 尝试 1. 检查权限的问题

由于7.0文件分享的权限限定，提请安装的URL是必须通过 `FileProvider` 来指定的，并且要对应有相关的配置(xml, manifest)等。

8.0后对于未知来源的 APP 安装，有一个隐含的权限。在部分机型上可能需要提前获取到这一权限。
然尔上述解法都没有能解决出现的问题。

### 观察系统的Intent传递

对比看到豌豆荚下载指定的APP确实可以跳转到系统的安装的页面的，表明请求安装的intent是得到了系统响应的。于是想查看下它相关的intent的设置，看是否能有所收货。
```
adb logcat | fgrep -i intent
```
### 尝试 2. 主动设置 Component

但是通过对比发现，WDJ发起的intent里面额外包含有 `Component` 的信息，这会是问题的根源吗？随后按 `resolveAppInfo` 过滤出 installer的信息，并指定了响应的 Component, 但问题依旧存在。

### 尝试 3. 使用 Foreground service
使用自定义前台的 `IntentService` 实现下载 ⏬功能，并且触发安装过程。从 log 看到，系统还是拦截了Intent。这中方案也不奏效。

😓 无意间从官方文档中也发现了这样一句话：
> Note: For the purposes of starting activities, an app running a **foreground service** is still considered to be **"in the background"**.

### 尝试 4. 在前台发送请求安装 apk 的 intent

此时再次对比发起 intent 的结果时，留意到不同之处在于我们发的 intent 被标记为了 `background`，并有 log 明确显示为它是从后台触发的。同时我也注意到，对于 force update的情况，在点击 `下载` 按钮开启下载任务的同时，代码中把 `Activity affinity` 也完全清除了。此时 APP 确实处于后台运行的状态中。

```W ActivityTaskManager: Background activity start [callingPackage: com.your-pkg-name; callingUid: 10181; isCallingUidForeground: false; isCallingUidPersistentSystemProcess: false; realCallingUid: 10181; isRealCallingUidForeground: false; isRealCallingUidPersistentSystemProcess: false; originatingPendingIntent: null; isBgStartWhitelisted: false; intent: Intent { act=android.intent.action.VIEW dat=content://com.your-pkg-name.fileprovider/files/storage/emulated/0/Download/latest-release.apk typ=application/vnd.android.package-archive flg=0x10000001 cmp=com.android.packageinstaller/.InstallStart }; callerApp: ProcessRecord{16aec51 11234:com.your-pkg-name/u0a181}]
```

据此修改，果然在保留Activity stack后，跳转系统安装新的apk被成功触发了。

```
I ActivityTaskManager: START u0 {act=android.intent.action.VIEW dat=content://com.your-pkg-name.fileprovider/files/storage/emulated/0/Download/latest-release.apk typ=application/vnd.android.package-archive flg=0x10000001 cmp=com.miui.packageinstaller/com.android.packageinstaller.PackageInstallerActivity} from uid 10309
```


## 5. 备注

* [adb shell](https://developer.android.google.cn/studio/command-line/adb#am)
* [🚫 OS 10.0 从后台启动Activity的限制](https://developer.android.google.cn/guide/components/activities/background-starts?hl=zh-CN)
* [你的 App 还能在后台启动 Activity 吗（非 AndroidQ 适配）](https://www.jianshu.com/p/e3676e8cae0e)
