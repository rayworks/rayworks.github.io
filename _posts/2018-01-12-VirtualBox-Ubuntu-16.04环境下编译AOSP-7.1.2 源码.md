---
layout: post
title: VirtualBox Ubuntu 16.04环境下编译AOSP 7.1.2 源码
excerpt_separator:  <!--more-->
---

话说最初投奔Android平台就是看中其开源的特性，善于利用OS源码，对个人开发成长确实有不可替代的作用。[May the Force be with you](http://starwars.wikia.com/wiki/May_the_Force_be_with_you).

在经历若干次下载失败，编译错误之后，最后编译成功，看到了久违的 emulator (是的，几乎20mins 后才能启动的那个原生模拟器 = =! ).

主要步骤如下：
# 0 安装VirtualBox 以及 安装Ubuntu系统
请参考 [http://blog.csdn.net/u013553529/article/details/54838490](打造自己的Android源码学习环境之二：在虚拟机中安装Ubuntu)。
注意: 最好预留100GB+的磁盘空间 和 4GB+的内存，并优选OS 镜像
 Ubuntu(64-bit)。

# 1 下载Android源代码
由于众所周知的原因，官方的Google 站点 [https://source.android.com/source/downloading.html](https://source.android.com/source/downloading.html) 不能访问。建议选择国内[清华镜像](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)。

## 1.1 下载 repo 工具
同样通过镜像下载  https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/ 。要注意的一点是，该工具 repo 是一个**可执行文件** (出现过一次下载后发现是一个HTTP 404 response 的html 文件)。

## 1.2 下载特定的OS版本
按 传统初始化方法
建立工作目录:

`mkdir WORKING_DIRECTORY`
`cd WORKING_DIRECTORY`

初始化仓库:
`repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest`

如果需要某个特定的 Android 版本([列表](https://source.android.com/source/build-numbers.html#source-code-tags-and-builds))：

```
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-7.1.2_r36
```

同步源码树（以后只需执行这条命令来同步）：

```
repo sync
```
>由于网络原因，在使用repo sync同步代码的过程中会多次出错，总不能时时刻刻刻盯着，能不能在同步失败的情况下，自动重试呢？当然可以，我们可以写一个[简单的shell脚本](http://blog.csdn.net/dd864140130/article/details/51718187)
```
#!/bin/bash
#FileName source_asyn.sh
PATH=~/bin:$PATH
# 注意修改成你要编译的版本，比如这里我在mac上编译的是android-7.1.2_r36
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-7.1.2_r36
repo sync
while [$? = 1]: do
echo "=========download failed,again============"
sleep 5
repo sync
done
```
代码同步完成后，原来的 .repo 隐藏文件下的对应版本的源代码会出现在 `WORKING_DIRECTORY` 文件夹下。

# 2 编译代码
## 2.0 预设置参数

**鉴于极易出现`2.0.2`中描述的问题，建议提前检查修改配置，并启动server**

编译过程中主要碰到的问题包括：
### 2.0.1 Communication error with Jack server
> Communication error with Jack server (28). Try 'jack-diagnose'

解决方法：
`jack-admin start-server`

### 2.0.2 GC overhead limit exceeded (version 1.2-rc4 'Carnac' ...)
解决方法：
按出错的提示，可以对应修改JVM参数，修改最大可用heap内存。
* 1 停止运行中的server `./prebuilds/sdk/tools/jack-admin stop-server`

* 2 修改VM 启动参数：
`gedit ./prebuilds/sdk/tools/jack-admin`
>将 `JACK_SERVER_VM_ARGUMENTS="${JACK_SERVER_VM_ARGUMENTS:=-Dfile.encoding=UTF-8 -XX:+TieredCompilation}" ` 直接修改为 :
`JACK_SERVER_VM_ARGUMENTS="-Xmx4096m -Dfile.encoding=UTF-8 -XX:+TieredCompilation"。注意：这里不再使用${parameter:=default}的语法，直接给JACK_SERVER_VM_ARGUMENTS赋值。`

* 3 重新启动 server `./prebuilds/sdk/tools/jack-admin start-server`, 可以从log中看到刚刚设置的 `-Xmx` 参数

以上是本人碰到过的编译错误。如果遇到其它类型的错误，可以参考[ 这里 ](http://blog.csdn.net/u013553529/article/details/54869266) 解决。

## 2.1 设置环境
```
cd WORKING_DIRECTORY
```
```
source build/envsetup.sh
```
## 2.2 选择编译目标
执行 `lunch`，选择默认的`aosp_arm-eng`。
>aosp(Android Open Source Project)代表Android开源项目;arm表示系统是运行在arm架构的处理器上
>-eng:代表engineer,也就是所谓的开发工程师的版本,拥有最大的权限(root等),此外还附带了许多debug工具

更多关于build 以及 build type的说明，请移步[这里](http://blog.csdn.net/dd864140130/article/details/51718187).

## 2.3 编译
>通过make指令进行代码编译,该指令通过-j参数来设置参与编译的线程数量,以提高编译速度.比如这里我们设置8个线程同时编译:
```make -j8```
需要注意的是,参与编译的线程并不是越多越好,通常是根据你机器cup的核心来确定:core*2,即当前cpu的核心的2倍.比如,我现在的笔记本是双核四线程的,因此根据公式,最快速的编译可以make -j8. 
(通过cat /proc/cpuinfo查看相关cpu信息)

编译成功后，你会看到类似的信息：
`#### make completed successfully (hh:mm:ss) ####`

![](assets/images/aosp_build_complete.png)


# 3 启动模拟器
`emulator `

![](assets/images/diy_simulator.png)


# 4 参考
[[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/)
](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)

[打造自己的Android源码学习环境 系列 blog](http://blog.csdn.net/u013553529/article/details/54829345)

[自己动手编译最新Android源码及SDK(Ubuntu)](http://blog.csdn.net/dd864140130/article/details/51718187)
