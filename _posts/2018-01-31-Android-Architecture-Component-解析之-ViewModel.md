---
layout: post
title: Android Architecture Component 解析之 ViewModel
excerpt_separator:  <!--more-->
tags:
  - code
  - syntax highlighting
---

# 概览
在考虑到UI 元素生命周期变化的环境下，架构组件中的 [ViewModel](https://developer.android.google.cn/reference/android/arch/lifecycle/ViewModel.html) 主要用于存储以及管理UI相关的数据。ViewModel 可以在配置信息变化(configuration change) 时 (例如屏幕的旋转) 保持数据状态。

众所周知的是，Android系统框架管理着UI控制器（Activity / Fragment）的生命周期。出于对用户行为以及设备事件的响应，系统可能会销毁或重建UI控制器，而这些行为是完全不受控制的。如果系统销毁 或是 重建UI控制器时，任何UI相关的暂存数据都将会被清除。

另一个问题是，UI控制器通常需要频繁地调用异步接口，耗费一定的时间获取数据。除了维护相关的接口外，在UI控制器被重新创建时往往会发起重复的请求，这无疑会造成资源的浪费。

UI 控制器如Activity / Fragment主要用于展示UI数据，响应用户请求，以及处理系统的事件，如授权访问请求。在UI 控制器中加入请求加载数据库或是网络的数据的逻辑将使得类不断膨胀。将大量无关的职责分配给UI 控制器，而不是将各项职责分配出去则会导致出现 [God Object](https://en.wikipedia.org/wiki/God_object)，同时会降低整体项目的可测试性。

# ViewModel 的生命周期 ( outlive Fragment or Activity ?）
且看官方给出的ViewModel生命周期图示：

![viewmodel-lifecycle](../../../assets/images/vm_lifecycle.png)

对照Demo中的实际效果:

![vm_test](../../../assets/images/vm_test.gif)


利用ViewModel 在旋转屏幕后，可以看到ViewModel的instance 还是同一个，而且其中的score 数据依然保存了下来。（观察实际的log也可以印证这一点）

如果直接按back 键退出，可以看到几乎在`Activity`的`onDestroy()` 方法调用的同时，`ViewModel#onCleared()`方法被调用，此时ViewModel也随即销毁了。

显而易见的是，除了数据已经保存外，还有一个好处在于省去了需要手动关联/解除`ViewModel`与界面生命周期的调用。即不必在先前自实现的`ViewModel`|`Presenter`中添加类似`OnDestroy()`方法，并发起显示的调用了。

# 实现
以上的`ViewModel`中实现了‘自知’的生命周期，这究竟是如何做到的呢？让我们退一步回到在使用`Fragment`时，看如何在转屏条件下保存内存中的临时数据。

## 从Fragment的角度来看状态保存
在转屏时，按默认的情况，`FragmentManager`会负责销毁相关联的`Fragment`, `Fragment`的生命周期 `onPause`, `onStop`, `onDestroy`会被相继调用。然而Fragment有一个特性可以设置`Fragment#setRetainIntance(true)`保存当前的自身实例（instance）。当该选项设置后，Fragment 当前实例被保存下来了，然后随即被传给新的Activity。留存的Fragment实际利用了Fragment关联的View可以被销毁以及重新创建，而不需要销毁Fragment自身。

在发生configuration 改变的时候，`Fragment` 内部的View与`Activity` 包含的View 一样，都会因为这一原因被销毁以及重新创建。即当有一个新的configuration变化时，很有可能需要新的资源，为了使用更好适配的资源，View将会重新创建。

随后`FragmentManager`会检查每一个Fragment的`retainInstance`属性，如果值为默认的false，`FragmentManager`会将Fragment实例销毁，Fragment和它关联的View都将会在新的Activity中重新创建；而如果`retainInstance`值为true，当前`Fragment`关联的View将会销毁而保留下Fragment的实例，在新的Activity创建时，新的`FragmentManager`会找到留存的Fragment，并重建它的View。

留存的`Fragment`没有被销毁，而只是从快要销毁的`Activity`上解绑了。表示为`retain`状态的`Fragment`仍然存在，只是不被任何`Activity`持有。整个转屏过程，对应生命周期的变化可见下图：

![Fragment lifecycle](../../../assets/images/frag_lifecycle.png)

## 代码寻踪
按照上述Fragment实现的逻辑，架构组件中的实现又是怎样的呢？让我们从源码中寻找线索, 以Activity中的ViewModel创建为例。
```
ViewModelProviders.of(activity).get(YourViewModel.class)

ViewModelProviders.of(activity)
    -> new ViewModelProvider(ViewModelStores.of(activity), factory) // # factory 为默认的 AndroidViewModelFactory
            ViewModelStores.of(activity)
                -> ViewModelStore
                HolderFragment#holderFragmentFor(activity).getViewModelStore()
                    HolderFragmentManager#holderFragmentFor(activity)
                        FragmentManager fm = activity.getSupportFragmentManager();
                        HolderFragment holder = fm.findFragmentByTag(HOLDER_TAG)
                        if (holder != null) {
                            return holder;
                        }

                        holder = new HolderFragment();
                            public HolderFragment() {
                                setRetainInstance(true); // retained!
                            }
                            public void onDestroy() 
                                super.onDestroy();
                                mViewModelStore.clear();
                                    for (ViewModel vm : mMap.values()) {
                                        vm.onCleared(); // notifies ViewModels that they are no longer used.
                                    }

                        fm.beginTransaction().add(holder, HOLDER_TAG).commitAllowingStateLoss();
                holder.getViewModelStore()

viewModelProvider.get(modelClass) //YourViewModel.class
    get(key, modelClass)
        ViewModel viewModel = mViewModelStore.get(key); // 检查是否已经有缓存
        if (modelClass.isInstance(viewModel)) {
            return (T) viewModel
        }

        viewModel = mFactory.create(modelClass);
            AndroidViewModelFactory#create(modelClass)
                if(AndroidViewModel.class.isAssignableFrom(modelClass)) // 检查是否为AndroidViewModel类别
                    return modelClass.getConstructor(Application.class).newInstance(mApplication);
                super.create(modelClass)
                    NewInstanceFactory#create(modelClass)
                        return modelClass.newInstance(); // 调用默认构造方法构造实例

        mViewModelStore.put(key, viewModel); // 添加ViewModel到缓存中
        return (T) viewModel;
```
可以发现存储的`ViewModel`实际也是利用`Fragment` 设置为`retained`的特性实现的。它的生命周期与内置的不可见的`HolderFragment`是紧密联系的。在`Fragment`最终被销毁时，`ViewModel`的`onCleared`用于执行清理动作的回调方法同时被触发。

# 与 onSaveInstanceState() 的异同
ViewModel保持的实际是暂存的UI中的数据，但是它们并没有被持久化(persisted)。一旦相关的UI控制器被销毁或是App进程被暂停(由于系统资源的限制)，ViewModel和所包含的数据将被GC处理。

`onSaveInstanceState()` 该方法在以下2种情况下用来保留少量的UI相关数据。
* App在后台由于资源限制被暂停（stopped）
* 配置变化 (configuration change)

系统在UI层面已经利用`onSaveInstanceState()` 去保存View的状态信息（如 EditText中已输入的文本, ListView中的滑动位置等）。由于该方法在主线程中执行，并且序列化本身会有一定的内存开销，即要求此方法能较快执行，以免出现UI掉帧或是更严重的性能的问题。所以它的设计并不适合大数据的存储。

`Fragment#setRetainInstance(true)` 方法利用retained Fragment 实例，可以保留较大规模的数据 如 `bitmap` 或是像网络连接(network connections)这样复杂的对象。

值得注意的是，`ViewModel` 只在由配置信息变化改变(configuration change)引起的销毁过程中留存，当进程被暂停时，它也被销毁了。

# ViewModel 以及 Retained Fragment 测试源码
[Github: ArchComponentPlayground](https://github.com/rayworks/ArchComponentPlayground)

# 其它ViewMode实现
https://github.com/inloop/AndroidViewModel

# 参考
* [AndroidViewModel](https://github.com/inloop/AndroidViewModel)

* [官方 ViewModel 文档](https://developer.android.google.cn/topic/libraries/architecture/viewmodel.html)

* [Android Programming: The Big Nerd Ranch Guide](https://www.amazon.com/Android-Programming-Ranch-Guide-Guides/dp/0321804333/ref=sr_1_4?s=books&ie=UTF8&qid=1517329246&sr=1-4&keywords=Android+Programming%3A+The+Big+Nerd+Ranch+Guide)

* [Medium ViewModels: Persistence, onSaveInstanceState(), Restoring UI State and Loaders](https://medium.com/google-developers/viewmodels-persistence-onsaveinstancestate-restoring-ui-state-and-loaders-fc7cc4a6c090)


