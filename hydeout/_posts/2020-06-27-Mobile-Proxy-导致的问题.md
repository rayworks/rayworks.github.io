---
layout: post
title: Mobile Proxy 导致的问题
excerpt_separator:  <!--more-->
tags:
  - http
---

## 问题
在最近的一次调试过程中，发现网络良好的状况下，Flutter 页面网络请求出现快速失败的问题。而比较奇怪的是，如果使用配置好的代理（proxy）后，网络错误却消失了。

## 解决过程
最直接的想法是看 `HttpClient` 中报错的异常信息到底是什么，在重现的设备上观察到，在进行 `GET` 请求时它报出了类似 `" Invalid proxy  :0 "` 的错误消息。此时问题似乎比较明朗了 ，某种条件下读取的设备proxy出现了无效值：host值变为" "，端口port值变为了 0 。

同时问题可以在部分手机上重现了：设备上正确配置好proxy之后，随即停用proxy；此时**新创建的**`HttpClient`使用了native端`System.getProperty("http.proxyHost")` 与 `System.getProperty("http.proxyPort")` 读取到的 `" :0"` 配置了proxy，进而导致后续请求出现网络错误 ❎。

## 小结
* 需要严格检测读取系统proxy信息的有效性，强化以及定义好 Flutter / Native channel API；
* 一般情况下只在开发阶段启用Proxy，不干扰生产环境代码逻辑。
