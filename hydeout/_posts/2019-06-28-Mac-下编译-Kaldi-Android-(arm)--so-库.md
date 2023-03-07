---
layout: post
title: Mac 下编译 Kaldi Android (arm) .so 库
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

话说之前公司里面一直用到的是 [PocketSphinx](https://cmusphinx.github.io/wiki/tutorial/) , 但是在新的项目中有一个应用的场景，这时候发现噪声影响以及识别的精确度方面都不是很理想。于是在 Telegram Channel 里面咨询了下，[@nshmyrev]([https://github.com/nshmyrev](https://github.com/nshmyrev)
) 回复建议可以利用 Kaldi DNN 模型，应该会有显著提升。于是考虑转向研究 [Kaldi](https://github.com/kaldi-asr/kaldi)。 

首先碰到的一个问题是跨平台编译。网上搜索后发现，被引用最多的一篇文章（下文简称“编译指南”）是 [compile-kaldi-android](http://jcsilva.github.io/2017/03/18/compile-kaldi-android/)，但它是基于 Ubuntu 环境来编译的，也看到了编译 Kaldi 可用的 [docker file](https://github.com/jcsilva/docker-kaldi-android)。但是在Mac环境下又可以怎样成功编译呢？让我们分解来看：

## 1. 配置Android NDK 以及 独立的编译 toolchain
这部分和编译指南中的大体一致，对不同的平台，没有NDK的需要相应的下载，并且配置ANDROID_NDK 路径。

### 安装 toolchain ：
```
$ANDROID_NDK/build/tools/make_standalone_toolchain.py --arch arm --api 21 --stl=libc++ --install-dir /tmp/my-android-toolchain
```
以上命令创建 `/tmp/my-android-toolchain/` 文件目录，并且已经包含了 android-21/arch-arm sysroot，以及32位 ARM 架构的工具链可执行文件等。

### 将 toolchain 加入到系统 PATH中：
```
export ANDROID_TOOLCHAIN_PATH=/tmp/my-android-toolchain
export PATH=${ANDROID_TOOLCHAIN_PATH}/bin:$PATH
```

## 2. 编译 Android 版本的 OpenBLAS
注：考虑到 gfortran 已经是GCC的一部分了，可以选择性安装 gfortran。

### 下载源码：
```
git clone https://github.com/xianyi/OpenBLAS
```

### 选择 ARMV7 进行编译：
```
make \
    TARGET=ARMV7 \
    ONLY_CBLAS=1 \
    CC=$ANDROID_TOOLCHAIN_PATH/bin/arm-linux-androideabi-clang \
    AR=$ANDROID_TOOLCHAIN_PATH/bin/arm-linux-androideabi-ar \
    HOSTCC=gcc \
    ARM_SOFTFP_ABI=1 \
    -j4
```
此处与编译指南中有所不同，如果按它上面的操作，会报出 找不到"crtbegin_so"之类的错误。

### 安装库文件
```
make install NO_SHARED=1 PREFIX=`pwd`/install
```
## 3. 编译 CLAPACK
```
git clone https://github.com/simonlynen/android_libs.git

cd android_libs/lapack
```
### 打开 jni/Android.mk, 注释掉测试相关的编译指令
```
# remove some compile instructions related to tests

# LOCAL_MODULE:= testlapack
# LOCAL_SRC_FILES:= testclapack.cpp
# LOCAL_STATIC_LIBRARIES := lapack
# include $(BUILD_SHARED_LIBRARY)
```

### 打开 jni/Application.mk
将 `APP_STL := gnustl_static` 替换为 `APP_STL := c++_shared` 
将 `APP_ABI := armeabi armeabi-v7a` 替换为 `APP_ABI := armeabi-v7a`。armeabi 已经不再支持了。
文件最后增加 `NDK_TOOLCHAIN_VERSION := clang`

### 编译
```
$ANDROID_NDK/ndk-build
```
编译完成后会在 `obj/local/armeabi-v7a/`生成库文件。将生成的库文件拷贝到前面你安装OpenBLAS库文件的目录下(e.g: OpenBlas/install/lib)。Kaldi 将会在这个目录下查找相关的依赖项。

## 4. 编译 Kaldi
下载源码
```
git clone https://github.com/kaldi-asr/kaldi.git
```
编译 OpenFST
查看当前的kaldi `tools/Makefile` 后发现使用的版本是OpenFST-1.6.7。
```
cd kaldi/tools
wget -T 10 -t 1 http://www.openfst.org/twiki/pub/FST/FstDownload/openfst-1.6.7.tar.gz
tar -zxvf openfst-1.6.7.tar.gz

cd openfst-1.6.7/

CXX=clang++ ./configure --prefix=`pwd` --enable-static --enable-shared --enable-far --enable-ngram-fsts --host=arm-linux-androideabi LIBS="-ldl"

make -j 4

make install

cd ..

ln -s openfst-1.6.5 openfst
```
编译源码
```
cd ../src
```
打开 `matrix/Makefile` 文件，将其中的测试文件注释掉（似乎与Clang8.0有关的bug）。
```
#TESTFILES = matrix-lib-test sparse-matrix-test #matrix-lib-speed-tes
```
```
CXX=clang++ ./configure --static --android-incdir=/tmp/my-android-toolchain/sysroot/usr/include/ --host=arm-linux-androideabi --openblas-root=/path/to/OpenBLAS/install

make clean -j

make depend -j

make -j 4
```
按上述配置已经可以生成所有的静态链接库 .a 文件了，它们分别位于 src 下的各个子目录中：
```
.//tree/kaldi-tree.a
.//gmm/kaldi-gmm.a
.//online2/kaldi-online2.a
.//util/kaldi-util.a
.//feat/kaldi-feat.a
.//lm/kaldi-lm.a
.//sgmm2/kaldi-sgmm2.a
.//rnnlm/kaldi-rnnlm.a
.//nnet/kaldi-nnet.a
.//decoder/kaldi-decoder.a
.//nnet2/kaldi-nnet2.a
.//chain/kaldi-chain.a
.//nnet3/kaldi-nnet3.a
.//cudamatrix/kaldi-cudamatrix.a
.//ivector/kaldi-ivector.a
.//kws/kaldi-kws.a
.//hmm/kaldi-hmm.a
.//lat/kaldi-lat.a
.//fstext/kaldi-fstext.a
.//transform/kaldi-transform.a
.//matrix/kaldi-matrix.a
.//base/kaldi-base.a
```

等等，说好的`.so`文件在哪呢 ？

P.S.
* 打开 kaldi/src/configure 文件，将
```
--android-incdir=*)
    android=true;
    threaded_math=false;
    static_math=true;
    static_fst=true;
    dynamic_kaldi=false;
    MATHLIB='OPENBLAS';
```
其中的 `dynamic_kaldi=false` 改为 `dynamic_kaldi=true`。

* 更新 configure，指明库类型为 `--shared`:
```
CXX=clang++ ./configure --shared --android-incdir=/tmp/my-android-toolchain/sysroot/usr/include/ --host=arm-linux-androideabi --openblas-root=/path/to/OpenBLAS/install
```

* 编译kaldi过程中除去 debugging symbols, 打开 `src/kaldi.mk` 修改其中的`CXXFLAGS` 为：
```
CXXFLAGS = -std=c++11 -I.. -I$(OPENFSTINC) -O1 $(EXTRA_CXXFLAGS) \
           -Wall -Wno-sign-compare -Wno-unused-local-typedefs \
           -Wno-deprecated-declarations -Winit-self -Wno-mismatched-tags \
           -DKALDI_DOUBLEPRECISION=$(DOUBLE_PRECISION) \
           -DHAVE_CXXABI_H -DHAVE_OPENBLAS -DANDROID_BUILD \
           -I$(OPENBLASINC) -I$(ANDROIDINC) -ftree-vectorize -mfloat-abi=softfp \
           -mfpu=neon -pthread \
           -O3 -DNDEBUG
        #    -g # -O0 -DKALDI_PARANOID
```
* (已提交PR，最新源码已修复) ~~打开 `src/makefiles/default_rules.mk`，将第4行起按平台类型进行配置的部分替换为：~~
```
ifeq ($(KALDI_FLAVOR), dynamic)
  ifdef LIBNAME
      LIBFILE = lib$(LIBNAME).so
  endif
  LDFLAGS += -Wl,-rpath -Wl,$(KALDILIBDIR)
  EXTRA_LDLIBS += $(foreach dep,$(ADDLIBS), $(dir $(dep))$(notdir $(basename $(dep))).a)

  XDEPENDS =
else
  ifdef LIBNAME
    LIBFILE = $(LIBNAME).a
  endif
  XDEPENDS = $(ADDLIBS)
endif
```
具体原因在于，src下各个部分编译动态链接库时需要区分不同的平台类型，而 Makefile 中直接根据 shell 环境下 `uname` 返回的值来判定的。而这在跨平台编译时是不够充分的，此时的 `host=arm-linux-androideabi `，[不能以Mac下的动态链接库的条件直接判定](https://groups.google.com/forum/#!topic/kaldi-help/5CRcoiOhFZo)，否则会出现动态链接库不匹配的问题 : 
```
clang80++: error: linker command failed with exit code 1 (use -v to see invocation)
make[1]: *** [libkaldi-matrix.dylib] Error 1
```
* 重新编译即可，生成的so文件可在 `src/lib/`下找到。
```
make -j clean depend; make -j 4
```

## 5. 后记
这次解决跨平台的编译问题将近花了4天的时间，故记录整个过程，希望对后来尝试编译的人有所启示。在这期间非常感谢 [compile-kaldi-android](http://jcsilva.github.io/2017/03/18/compile-kaldi-android/)的指引, [@funcwj](https://github.com/funcwj) 在微信上的交流， `google group kaldi-help`论坛上面大家的热心回复。

## 6. 引用
* [compile-kaldi-android](http://jcsilva.github.io/2017/03/18/compile-kaldi-android/)
* [Doesn't compile on mac os x 64 bit](https://github.com/raboof/sfArkLib/issues/1)
* [dynamic_kaldi](https://blog.csdn.net/qq_33335062/article/details/80918600)
* [Compile OpenBLAS](https://github.com/xianyi/OpenBLAS/issues/2005)
* [compile-kaldi.sh - Error: FAILED matrix-lib-test ](https://github.com/jcsilva/docker-kaldi-android/issues/11)

