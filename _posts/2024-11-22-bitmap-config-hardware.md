 ---
layout: post
title: '记一次 Bitmap.Config.HARDWARE 引发的事故'
tags:
  - code
  - Bitmap
  - Android
  - syntax highlighting
---

## 线上的问题

最近发现在新版本发布后App两周左右，在监控平台发现某些机型上出现大面积的 crash:

<details> <summary>Crash log</summary>
<div>
  <p>java.lang.IllegalArgumentException: Software rendering doesn't support hardware bitmaps
    at android.graphics.BaseCanvas.throwIfHwBitmapInSwMode(BaseCanvas.java:1031)
    at android.graphics.BaseCanvas.throwIfCannotDraw(BaseCanvas.java:146)
    at android.graphics.BaseCanvas.drawBitmap(BaseCanvas.java:204)
    at android.graphics.Canvas.drawBitmap(Canvas.java:1631)
    at android.graphics.drawable.BitmapDrawable.draw(BitmapDrawable.java:569)
    at android.widget.ImageView.onDraw(ImageView.java:1441)
    at android.view.View.draw(View.java:24736)
    at android.view.View.draw(View.java:24589)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24587)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24741)
    at android.widget.ScrollView.draw(ScrollView.java:1971)
    at android.view.View.draw(View.java:24589)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24587)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24741)
    at androidx.viewpager.widget.ViewPager.draw(ViewPager.java:2426)
    at android.view.View.draw(View.java:24589)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24741)
    at android.view.View.buildDrawingCacheImpl(View.java:23969)
    at android.view.View.buildDrawingCache(View.java:23829)
    at android.view.View.draw(View.java:24439)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.updateDisplayListIfDirty(View.java:23556)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.updateDisplayListIfDirty(View.java:23556)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24741)
    at android.view.View.updateDisplayListIfDirty(View.java:23569)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.updateDisplayListIfDirty(View.java:23556)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.updateDisplayListIfDirty(View.java:23556)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.updateDisplayListIfDirty(View.java:23556)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.updateDisplayListIfDirty(View.java:23556)
    at android.view.View.draw(View.java:24447)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4623)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4380)
    at android.view.View.draw(View.java:24741)
    at com.android.internal.policy.DecorView.draw(DecorView.java:896)
    at android.view.View.updateDisplayListIfDirty(View.java:23569)
    at android.view.ThreadedRenderer.updateViewTreeDisplayList(ThreadedRenderer.java:726)
    at android.view.ThreadedRenderer.updateRootDisplayList(ThreadedRenderer.java:735)
    at android.view.ThreadedRenderer.draw(ThreadedRenderer.java:845)
  </p>
</div>
</details>

直接看到的问题是设置了 `Bitmap.Config.HARDWARE` 的位图（Bitmap）无法进行软件渲染（Software rendering）。

## Config.HARDWARE 是什么？
   `Bitmap.Config.HARDWARE` 是从`Android SDK` 26 引入的一种特殊的Bitmap configuration 配置模式。它被设计成直接存储于GPU内存（显存）中，而不是CPU内存（堆）中。

### 优势

这种方法在以下几种场景有较优的性能表现：

   * 提升绘制（Drawing）的性能
当一张位图（Bitmap）存储在硬件内存（hardware memory）中时，它能由GPU直接绘制到屏幕上，而不需要经由CPU内存转到GPU中。这一过程消除了潜在的数据转移操作，从而使得渲染更加顺滑快速，尤其对于复杂或高频更新的images。

   * 减少内存开销（Memory Footprint）
硬件位图并不会消耗传统Android堆上的内存空间，而这一点对于在内存受限的设备以及操作大量图片的应用来说具有重要意义，同时它也可以在一定程度上避免内存耗尽（OOM）的错误。

### 何时使用

将位图绘制到屏幕时，使用Bitmap.Config.HARDWARE 配置是极为高效的。主要有以下用例：
* 在ImageView中显示图片：如果你只是使用ImageView显示图片，而不需要直接操作位图的像素，使用Bitmap.Config.HARDWARE 将显著地提升性能。

* 动画与游戏：对于频繁更新屏幕的应用，像动画或游戏的场景，使用硬件位图能维持顺滑的帧率。

* 使用Canvas绘制的自定义View：如果你创建自定义View时，涉及在canvas上绘制位图时，使用Bitmap.Config.HARDWARE 能优化绘制过程。

### 限制

* 不可变性：硬件位图是不可变的。你无法直接改变它的像素数据。如果你需要进行像素级别的操作，你需要创建一个软件位图（software bitmap, 配置为Bitmap.Config.ARGB_8888 或 Bitmap.Config.RGB_565）。

* API 级别限制：Bitmap.Config.HARDWARE 只在Android API 26及以上才可用。如果应用需要支持更旧的设备，需要寻找其它的替代方案。

* 受限制的操作：有些特定的操作，如getPixels() 和 setPixels()在硬件位图上是不支持的。

## View中被弃用的方法

观察stacktrace 后可发现，该设备上使用了[`View.buildDrawingCache()`](https://developer.android.com/reference/android/view/View.html#buildDrawingCache())的方法，而这个方法是在API 级别 28开始备注为`Deprecated` 。随着API 11+引入的硬件加速渲染，中间的缓存层（cache layers）已经不再成为必要，考虑到创建与更新layer操作对性能的影响，此方法已逐渐过期。

因而，基于软件渲染（software-rendered usages）的方案已经不推荐使用，它与仅限硬件渲染的功能，如配置了Config.HARDWARE的位图，实时阴影以及outline clipping 还有兼容性问题。

## 禁用优化？！

回到问题本身，最直接的解决办法可选择在ImageLoader中直接禁用对Bitmap的 Config.HARDWARE 设置。可以看到[Coil](https://github.com/coil-kt/coil)对[Palette 的处理](https://coil-kt.github.io/coil/recipes/#palette)：
```
imageView.load("https://example.com/image.jpg") {
    // Disable hardware bitmaps as Palette needs to read the image's pixels.
    allowHardware(false)
    listener(
        onSuccess = { _, result ->
            // Create the palette on a background thread.
            Palette.Builder(result.drawable.toBitmap()).generate { palette ->
                // Consume the palette.
            }
        }
    )
}
```
有选择地启用`Bitmap.Config.HARDWARE`，这也是`Coil`在代码中加入了[Blocked Device 名单](https://github.com/coil-kt/coil/blob/2.7.0/coil-base/src/main/java/coil/util/HardwareBitmaps.kt#L113)
来决定是否启用 Hardware Bitmap的原因。从碎片化严重的Android生态来看，显然这些名单是不完备的。App Crash相比App部分性能的下降是更为严重的问题，一旦线上App出现问题，代价往往是相对较大的。
Google推出的优化方案，各个OEM也需要积极配合跟进，切实形成标准。否则割裂的系统环境，不但会让开发者为难，也会降低用户体验。
