---
layout: post
title: EventBus VS. OttoBus
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

# EventBus 笔记

APP中模块/层级间交互的逻辑处理往往也起着关键性的作用，一般较为常见的是通过自定义的接口即回调( `callback` )的方式实现，但是逻辑稍微复杂的情况下很容易发展为`callback hell`。本篇我们要看的是另外的一种方式，即采用Event bus来解耦发送端与接收端。实质上看 `EventBus` 是设计模式中 [`Observer`模式](https://en.wikipedia.org/wiki/Observer_pattern) 的典型代表。

先看到主流的Event bus 的对比 (Greenrobot EventBus VS. Otto Bus):

## 0. 性能对比

先摘录[官方的 Comparison 对比一览表](https://github.com/greenrobot/EventBus/blob/master/COMPARISON.md)

![](../../../assets/images/bus_feature_comp.png)

![](../../../assets/images/bus_time_comp.png)


## 1. 对DeadEvent的处理

OttoBus 对没有接收者的事件，保留了和 Guava eventbus 相同处理方式，即发现没有处理事件的handler时[将发出 `DeadEvent`](https://github.com/square/otto/blob/master/otto/src/main/java/com/squareup/otto/Bus.java#L334)，此时 `DeadEvent` 将事件的发送者和源事件包装起来了。

而 Greenrobot EventBus 中则附加定义了 `postSticky` 的方法，会在内存中缓存 ( `Sticky Events map` 默认以 `Event class`作为key保存 ) 最近的 Sticky Event 对象，在接收的方法中明确指定 `sticky` 为 `true` 时，则可以自动处理；但值得注意的是，**此时如未指定`sticky` 为 `true`，则不能收到 `event`**。同时在处理完了Sticky Event之后，需要手动移除，以避免重复处理该事件。

```
@Subscribe(sticky = true)
public void onHandle(YourEvent event) {
    // process your sticky event
    // ...
    EventBus.getDefault().removeStickyEvent(event);
}
```

相较之下，OttoBus 则可能需要自行搜集所有的`DeadEvent`，然后适时再次发送。

## 2. 事件处理的可继承性与订阅者索引

由于EventBus 对Event 以及 Subscriber 的继承性都进行了实现，简单来讲基类的订阅逻辑在子类中同样是有效的；而Otto bus则处于性能上的考虑未能给予支持。

再看 Greenrobot EventBus，为了优化性能，减少靠反射调用来查找事件的订阅方法，EventBus 推荐采用[对订阅者建立索引](https://greenrobot.org/eventbus/documentation/subscriber-index/)， 最终利用 `annotation processor` 生成索引类，使用 `builder` 模式构建最后的 `EventBus` 实例（将复杂度从运行时转到了编译期 😁）。

⚠️如果项目中引入了 `Kotlin` 或是已经是完全的 `Kotlin`开发，请使用
```
apply plugin: 'kotlin-kapt' // ensure kapt plugin is applied

dependencies {
    def eventbus_version = '3.2.0'
    implementation "org.greenrobot:eventbus:$eventbus_version"
    kapt "org.greenrobot:eventbus-annotation-processor:$eventbus_version"
}

kapt {
    arguments {
        arg('eventBusIndex', 'com.example.myapp.MyEventBusIndex')
    }
}
```
而无需同时指定
```
javaCompileOptions {
      annotationProcessorOptions {
              arguments = [ eventBusIndex : 'com.example.myapp.MyEventBusIndex' ]
      }
}     
```
否则，可能无法生成 `MyEventBusIndex` class.
