---
layout: post
title: API Design in the LaunchDarkly Android SDK
tags:
  - code
  - SDK
  - Android
  - syntax highlighting
---

# API Design in the LaunchDarkly Android SDK

## Introduction
In my recent role, I found myself working on a legacy Android project, a common scenario for many developers. The project involved enabling a specific feature for a group of users, for which we had implemented the LaunchDarkly solution (v4.2.3). But we encountered an intriguing issue.

QA reported a peculiar inconsistency: 

>  It works well for iOS but for Android we have noticed some inconsistencies, where the popup is not displayed at all times.

Why would we encounter inconsistencies between the two sets of SDKs—one for Android and the other for iOS? Why would the same user encounter different feature flag values on different platforms?

## Delving Deeper
> We initially observed that the method invoked to retrieve a feature flag with a specific key always returned false. However, after re-logging in, we could fetch the value true. 
> 
This was further corroborated by another QA team member.

We noticed an error log when reading an invalid feature flag:

> LDClient: Unknown feature flag "xxx"; returning default value

> LaunchDarkly: W Client did not successfully initialize within 0 seconds. It could be taking longer than expected to start up

This log entry suggested there might be a duration parameter for the initialization phase.

Upon inspecting the LDClient constructor:

```kotlin
LDClient.init(app, ldConfig, id, 0)
```

And its signature:

```java
/**
 * Initializes the singleton instance and blocks for up to <code>startWaitSeconds</code> seconds
 * until the client has been initialized. If the client does not initialize within
 * <code>startWaitSeconds</code> seconds, it is returned anyway and can be used, but may not
 * have fetched the most recent feature flag values.
 * ...
 * /
public static LDClient init(Application application, LDConfig config, LDUser user, int startWaitSeconds) {
        return init(application, config, LDContext.fromUser(user), startWaitSeconds);
    }
```
We realized the problem lay in setting `startWaitSeconds` to 0, which resulted in a partially-initialized LDClient instance. This instance, when queried for values, returned outdated ones. As a consumer of the SDK, we had no way of knowing when the initialization would be complete, which led us to...

## Identifying a Workaround
After exploring the SDK sample, we found flag listeners, including `registerAllFlagsListener(LDAllFlagsListener allFlagsListener)`.

The workaround, therefore, was to observe when the flags were updated in the listener. This would be the appropriate time to retrieve the latest state of a specific feature flag value.

## Lessons Learned
This experience shed light on the ease with which the API could be misused. As an SDK provider, it's essential to avoid that.

In this case, if the initialization isn't complete, it would be more helpful to return a real-time state rather than a default value. An alternative solution could be to make the exposed method asynchronous and require an additional `FeatureResultListener` argument.