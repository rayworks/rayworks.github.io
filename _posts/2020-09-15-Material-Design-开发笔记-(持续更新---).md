---
layout: post
title: Material Design 开发笔记 (持续更新...)
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

## 1. FitSystemWindow

在 layout 布局中设置 `android:fitsSystemWindows="true"` 到底发生了什么？
默认的 View的行为是在该属性设置后，系统通过在布局中预留出padding，使得布局有个相对的偏移。特别的, 在顶层layout是 `CoordinatorLayout` ，`DrawerLayout`，或是`CollapsingToolbarLayout`时，这些layout (相比一般的布局，如`FrameLayout`) 已经有了自定义的行为。

### 1.1 以 CoordinatorLayout 为例

```
if (ViewCompat.getFitsSystemWindows(this)) {
    if (mApplyWindowInsetsListener == null) {
        mApplyWindowInsetsListener =
            new androidx.core.view.OnApplyWindowInsetsListener() {
                @Override
                public WindowInsetsCompat onApplyWindowInsets(View v,
                        WindowInsetsCompat insets) {
                    return setWindowInsets(insets);
                }
            };
    }
    // First apply the insets listener
    ViewCompat.setOnApplyWindowInsetsListener(this, mApplyWindowInsetsListener);

    // Now set the sys ui flags to enable us to lay out in the window insets
    setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE
            | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
}
```

在5.0+系统中， 其方法`setupForInsets()`会检测 `fitsSystemWindows`属性是否有设置过，如果有，则设置当前View的属性为`View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN` 然后进一步调用`dispatchApplyWindowInsetsToBehaviors`，将`WindowInsetsCompat` 传递个也设置过`fitsSystemWindows`子view的`Behavior` class,直至`WindowInsetsCompat`被最终消费掉(consumed)。

```
private WindowInsetsCompat dispatchApplyWindowInsetsToBehaviors(WindowInsetsCompat insets) {
    if (insets.isConsumed()) {
        return insets;
    }

    for (int i = 0, z = getChildCount(); i < z; i++) {
        final View child = getChildAt(i);
        if (ViewCompat.getFitsSystemWindows(child)) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior b = lp.getBehavior();

            if (b != null) {
                // If the view has a behavior, let it try first
                insets = b.onApplyWindowInsets(this, child, insets);
                if (insets.isConsumed()) {
                    // If it consumed the insets, break
                    break;
                }
            }
        }
    }

    return insets;
}   
```

### 1.2 Inset issue (e.g. Fragment transition)

在单Activity 多Fragment的UI结构下，可能出现多个Fragment的布局都需要处理 Window insets的情况。而实际上`ViewGroup#dispatchApplyWindowInsets()`默认的实现中是会遍历子View (`DFS`)，开始dispatch window insets 直至有子view消费了insets。这就意味着，一旦有子view消费了insets，后续的子view就不会再有机会处理insets。

所以对应的解决方法就是在Fragment的 Container中借助`OnApplyWindowInsetsListener` 改写处理insets的机制。

```
fragment_container.setOnApplyWindowInsetsListener { view, insets ->
  var consumed = false

  (view as ViewGroup).forEach { child ->
    // Dispatch the insets to the child
    val childResult = child.dispatchApplyWindowInsets(insets)
    // If the child consumed the insets, record it
    if (childResult.isConsumed) {
      consumed = true
    }
  }

  // If any of the children consumed the insets, return
  // an appropriate value
  if (consumed) insets.consumeSystemWindowInsets() else insets
}
```

### 1.3 代码实践
* 如果你使用了 `CoordinatorLayout` 或是 `DrawerLayout`，而且想要在system bars (包括status bar) 之下显示 View，可在其直接的子View中（递归地）指定`android:fitsSystemWindows="true"`。

* 对于当前status Bar颜色的设置，一般地为了能让自定义的view填充status bar的background区域，statusBarColor通常设置为transparent ： 
`Window#setStatusBarColor(int)` 
或在Activity Theme中指定为：
`<item name="android:statusBarColor">@android:color/transparent</item>`

* 如果想要获取status bar的高度，不必使用hardcoded value (24 dp) 或是读取系统resource，可通过监听系统 `WindowInsets` 实现：
```
myView.setOnApplyWindowInsetsListener { view, insets ->
    val statusBarSize = insets.systemWindowInsetTop
    return insets
}
```

## 2. Fragment中添加 Option Menu

一般来说，常见的对于ActionBar的处理是放在Host Activity中。但根据特定的的UI设计，Fragment中可能都有自定义的ActionBar，故将ActionBar的建立以及menu的响应处理分散到各自的Fragment中，这是更为内聚的做法。

常见的在Fragment中使用Options Menu 需要注意:

### 2.1 启用menu

在Fragment中设置setHasOptionsMenu(true)。

### 2.2 ActionBar 的对应更新

由于其它Fragment中也有设置 menu，在切换Fragment后，当前Activity关联的的ActionBar可能已经不是对应到当前的Fragment了，此时需要重新设置当前的ActionBar :

```
// toolbar in current Fragment
setSupportActionBar(toolbar)
```

## 4. References

* [Why would I want to fitsSystemWindows?](https://medium.com/androiddevelopers/why-would-i-want-to-fitssystemwindows-4e26d9ce1eec)

* [Why fitsSystemWindows doesn’t work sometimes?](https://proandroiddev.com/draw-under-status-bar-like-a-pro-db38cfff2870)

* [Windows Insets + Fragment Transitions](https://medium.com/androiddevelopers/windows-insets-fragment-transitions-9024b239a436)
  
* [Becoming a master window fitter](https://chris.banes.dev/talks/2017/becoming-a-master-window-fitter-lon/)

* [Intercepting everything with CoordinatorLayout Behaviors](https://medium.com/androiddevelopers/intercepting-everything-with-coordinatorlayout-behaviors-8c6adc140c26)

* [fitsSystemWindows effect gone for fragments added via FragmentTransaction](https://stackoverflow.com/questions/31190612/fitssystemwindows-effect-gone-for-fragments-added-via-fragmenttransaction)

