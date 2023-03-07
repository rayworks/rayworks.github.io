---
layout: post
title: Android Proguard 混淆规则
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

## 1 工具介绍 和 应用
[Proguard](https://www.guardsquare.com/en/products/proguard) 是一种对Java 字节码的优化工具，可以使得Java 或是 Android app 应用文件更小以及运行更快。通过混淆类名，字段名以及方法名，它也提供了一种对逆向工程的有限防护手段。Android的开发人员对它一定不陌生，它本身已是Google Android SDK的默认工具 🔧。

## 2 常见使用场景
混淆的本意是防止app被逆向工程破解，但特定的场景下我们往往并不希望启用混淆机制。主要常见于一些动态化或是有特殊约定的的场景。一般来看主要包括：
* 序列化/反序列化相关的对象(e.g. JSON data models, Serializables, Parcelables)
* 有按规则约定的方法名 (e.g. JavaScriptInterface )
* 动态加载的class
* 需要保留的平台相关的部分
* 特定Annotation修饰的方法

## 3 注意事项⚠️
* Android library project 需要特别的指定其自身相关的proguard文件
    ```
    consumerProguardFiles 'your-proguard-rules.pro'
    ```
* Support library 中有 `Keep` 注解的class
    ```
    -keep @android.support.annotation.Keep class * {*;}
    -keepclassmembers class * {
        @android.support.annotation.Keep *;
    }
    ```
* androidx 中有 `Keep`注解的class
    ```
    -keep @androidx.annotation.Keep class * {*;}
    -keepclassmembers class * {
         @androidx.annotation.Keep *;
     }
    ```

## 4 典型的 Proguard 文件
```
-repackageclasses

-dontwarn org.springframework.**
-dontwarn com.google.common.**
-dontwarn sun.misc.Unsafe

# Keep Options:

-keepattributes *Annotation*,EnclosingMethod
-keepattributes JavascriptInterface
-keepattributes Exceptions
-keepattributes Signature
-keepattributes SourceFile,LineNumberTable

-keep class sun.misc.Unsafe{*;}

-keep class com.google.gson.** {*;}
-keep class com.google.common.** {*;}
-keep class com.loopj.android.http.** {*;}
-keep class sun.misc.Unsafe{*;}

-keep public class * extends android.app.Activity
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.app.Service
-keep public class * extends android.view.View
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.Application
-keep public class * extends android.app.Dialog
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference
-keep public class * extends android.support.v4.app.Fragment
-keep public class com.android.vending.licensing.ILicensingService
-keep class * implements android.os.Parcelable { public static final android.os.Parcelable$Creator *; }

-keep public class * extends android.view.View {
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
    public void set*(...);
}

-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet);
}

-keepclasseswithmembers class * {
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

-keepclassmembers class * extends android.content.Context {
   public void *(android.view.View);
   public void *(android.view.MenuItem);
}

-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}

-keepclasseswithmembers class * { public <init>(android.content.Context, android.util.AttributeSet, int); }
-keepclasseswithmembers class * { public <init>(android.content.Context, android.util.AttributeSet); }
-keepclasseswithmembernames class * { native <methods>; }

-keepclassmembers class * extends android.app.Activity { public void *(android.view.View); }

-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# Android support libs
-dontwarn android.support.v4.**
-dontwarn android.support.design.**
-dontwarn android.support.v13.**

# JavaScript interface
-keepclassmembers class {your.javascript.interface.for.webview} {
   public *;
}

# Webview
-keepclassmembers class * extends android.webkit.WebViewClient {
    public void *(android.webkit.WebView, java.lang.String, android.graphics.Bitmap);
    public boolean *(android.webkit.WebView, java.lang.String);
}
-keepclassmembers class * extends android.webkit.WebViewClient {
    public void *(android.webkit.WebView, jav.lang.String);
}

# Butterknife
-keep class butterknife.** { *; }
-dontwarn butterknife.internal.**
-keep class **$$ViewBinder { *; }

-keepclasseswithmembernames class * {
    @butterknife.* <fields>;
}

-keepclasseswithmembernames class * {
    @butterknife.* <methods>;
}

# okhttp
-dontwarn com.squareup.okhttp.**
-dontwarn okio.**
-keep class com.squareup.okhttp.** { *; }
-keep interface com.squareup.okhttp.** { *; }

# GreenDao
-keep class de.greenrobot.dao.** {*;}
-keepclassmembers class * extends de.greenrobot.dao.AbstractDao {
    public static java.lang.String TABLENAME;
}
-keep class **$Properties

# Retrofit
-dontwarn retrofit.**
-keep class retrofit.** { *; }

# The 'Keep' annotation
-keep @android.support.annotation.Keep class * {*;}
-keep,allowobfuscation @interface android.support.annotation.Keep
-keepclassmembers class * {
    @android.support.annotation.Keep *;
}

# Data models in Your application
-keep class {your.data..model}.** { *;}

# EventBus from GreenRobot
-keepclassmembers class * {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }
```

## 4 其它可选项
[DexGuard](https://www.guardsquare.com/en/products/dexguard)

[R8](https://developer.android.com/studio/build/shrink-code?hl=zh-cn#configuration-files)
