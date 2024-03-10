---
layout: post
title: Performance tuning for recomposition in Compose UI
tags:
  - code
  - Compose UI
  - Android
  - syntax highlighting
---

## Background

Recently, I conducted a performance evaluation of the Compose UI layout using the "Layout Inspector" in Android Studio. This evaluation was prompted by the addition of a clock countdown feature on the app's Home page. 

To my surprise, I noticed that the number of redrawn components kept increasing even when there was no countdown information displayed. I expected that only the parts of the UI affected by state changes would be recomposed. However, the current code did not behave as expected. Consequently, I decided to optimize this aspect of the code.

## Performance Measurement and Metrics Analysis

To begin the optimization process, I decided to profile the app and identify any potential bottlenecks. I discovered a way to gather "Composable metrics" by executing [a custom task](https://chrisbanes.me/posts/composable-metrics/#enabling-the-metrics) against the release build. Before diving into the optimization process, let's cover some important concepts in the world of Compose UI:

* Recomposition: Recomposition involves calling composable functions again when their inputs change. Compose only calls the functions or lambdas that might have changed and skips the rest.
 
* Skippability: Skippability refers to Compose's ability to skip a function during recomposition if all of its parameters remain equal.
  
In general, we can assume that all functions annotated with @Composable are restartable. For these functions, if the related parameters haven't changed since the last call, they should be skippable.

Identifying functions that are restartable but not skippable proved to be an efficient approach. By examining the generated files, I found parameters and fields marked as either stable or unstable. Let's take a look at some code examples to illustrate this:

for methods

```kotlin
restartable scheme("[androidx.compose.ui.UiComposable]") 
fun BuildLayout(
  unstable model: ProfileModel
  stable isTarget: Boolean = @static true
  stable continuedClicked: Function0<Unit>
  stable itemClicked: Function0<Unit>
)
```

and classes

```kotlin
unstable class ProfileModel {
  stable var name$delegate: MutableState<String>
  stable var topic$delegate: MutableState<String>
  unstable var pages: List<PageModel>
}
```

## Make it Stable

To ensure stability, a type must adhere to the following contract:

* The `equals` function should consistently return the same result for the same instances. If a public property of the type changes, Compose should be notified.
  
* All public property types should also be stable.
  
Certain common types are treated as stable by the Compose compiler, even without explicitly using the `@Stable` annotation. These types include primitive value types (e.g., `Boolean`, `Int`, `Long`), strings, and function types (lambdas). Immutable types inherently maintain stability because they never change.

However, the compiler cannot infer stability if any of the fields on a class are mutable (associated with a var property) or have a non-stable type. If you encounter a restartable but not skippable function, it's an opportunity to consider two options:

* Make the function skippable by ensuring that all of its parameters are stable.
  
* Make the function not restartable by marking it as `@NonRestartableComposable`.

### Compose annotations to rescue

* `@Immutable`: This annotation indicates that all publicly accessible properties and fields will not change after the instance is constructed. Compose can easily detect changes between two instances of an @Immutable class.

* `@Stable`: Unlike @Immutable, @Stable is less strict and allows a stable class to hold mutable data. However, all mutable data needs to notify Compose when changes occur for recomposition to happen.

While Compose attempts to automatically infer whether a class is immutable or stable, there are cases where it fails to make accurate inferences. In such cases, we can use the @Immutable and @Stable annotations on the class to explicitly indicate its stability.

### Special Case: List or Immutable List?
The compiler may struggle to correctly infer parameter types like `List<T>` due to the `MutableList<E>` interface also extending `List<E>`. To handle this situation, there are two approaches:

* Wrap the List into a Stable-annotated data class.

* Use `ImmutableList`, which ensures immutability. To use ImmutableList, you will need to import it from `org.jetbrains.kotlinx:kotlinx-collections-immutable`.


### Item Key: Adding Extra Information for Smart Recompositions

To optimize recompositions when working with lists, we can provide an item key using the key parameter in Compose's LazyColumn or LazyRow. The item key helps the runtime identify which parts of the tree have changed. By doing so, the runtime can reorder instances in the Composition tree instead of recomposing each ItemView composable with a different item instance. Here's an example:

```kotlin
LazyColumn {
    items(movies, key = { it -> it.id }) { item ->
        ItemView(item)
    }
}
```
When the list changes, such as by adding, removing, or reordering items, Compose can recompose only the affected ItemView composables based on the changes identified by the item.

## Ref

* [Official doc for Compose UI](https://developer.android.google.cn/jetpack/compose/documentation?hl=en)

* [Composable metrics](https://chrisbanes.me/posts/composable-metrics/)

* [Interpreting Compose Compiler Metrics](https://github.com/androidx/androidx/blob/androidx-main/compose/compiler/design/compiler-metrics.md)

* [ComposeInvestigator](https://github.com/jisungbin/ComposeInvestigator)

* [Performance Optimization with @Stable and @Immutable](https://www.youtube.com/watch?v=_FtKhWvHiTg)

* [Compose 类型稳定性注解：@Stable & @Immutable](https://zhuanlan.zhihu.com/p/620252416)







