---
layout: post
title: Boot complete 之前无法接收广播吗？
date: 2019-08-13
modified: 2019-11-24
tags: [Android,PackageManager,BroadcastReceiver]
categories: [Developer]
excerpt:  默认情况下，如果系统版本是 Android 8.0 以上，则收不到。如果是 Android 7.0，需要设置 directBootAware 为 true。如果想在 Android 8.0 以上版本中收到...
description: 默认情况下，如果系统版本是 Android 8.0 以上，则收不到。如果是 Android 7.0，需要设置 directBootAware 为 true。如果想在 Android 8.0 以上版本中收到...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.github.io](xianzhu21.github.io)。


# 0. 先说结论

默认情况下，如果系统版本是 Android 8.0 以上，则收不到。如果是 Android 7.0，需要设置 directBootAware 为 true。

如果想在 Android 8.0 以上版本中收到，需要先设置 directBootAware，然后广播发送时需要添加 `FLAG_RECEIVER_INCLUDE_BACKGROUND`。

不管是什么版本，如果 `BroadcastReceiver` 所在的应用从来没有启动过（停止状态），则广播发送者需要额外添加 `FLAG_INCLUDE_STOPPED_PACKAGES`。

# 1. 发送广播到查找相关 Receiver

发送广播是调用 `Context` 类的 `sendBroadcast()` 方法，最终会调用 `ActivityManagerService` 类的 `broadcastIntent()` 方法，然后它又会调用 `broadcastIntentLocked()` 方法。

```java
// com.android.server.am.ActivityManagerService
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
    synchronized(this) {
        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                requiredPermissions, appOp, bOptions, serialized, sticky,
                callingPid, callingUid, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
}}
```

`broadcastIntentLocked()` 方法中根据 `intent` 查找相应的 `BroadcastReceiver` 对象。方法中创建两个局部变量，`receivers` 存储静态注册的 `BroadcastReceiver` 对象，`registeredReceivers` 存储动态注册的 `BroadcastReceiver` 对象。

Boot compete 之前三方应用没有启动过，所以也没有动态注册的广播。我们关注 `receivers` 变量的赋值过程。
```java
// com.android.server.am.ActivityManagerService
final int broadcastIntentLocked(ProcessRecord callerApp,
        String callerPackage, Intent intent, String resolvedType,
        IIntentReceiver resultTo, int resultCode, String resultData,
        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
    // Figure out who all will receive this broadcast.
    List receivers = null;
    List<BroadcastFilter> registeredReceivers = null;
    // Need to resolve the intent to interested receivers...
    if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
        receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
    }
    if (intent.getComponent() == null) {
        if (userId == UserHandle.USER_ALL && callingUid == SHELL_UID) {
            ...
        } else {
            registeredReceivers = mReceiverResolver.queryIntent(intent,
                    resolvedType, false /*defaultOnly*/, userId);
}}}
```

`receivers` 的值有 `collectReceiverComponents()` 方法返回 。该方法会调用 `PackageManagerService` 类的 `queryIntentReceivers()` 方法，因为静态广播是由 `PackageManagerService` 类在系统启动的时候扫描解析 AndroidManifest 文件获取的。

```java
// com.android.server.am.ActivityManagerService
private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType,
        int callingUid, int[] users) {
    int pmFlags = STOCK_PM_FLAGS | MATCH_DEBUG_TRIAGED_MISSING;
    List<ResolveInfo> receivers = null;
    for (int user : users) {
        List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
        if (receivers == null) {
            receivers = newReceivers;
        } else if (newReceivers != null) {
            ...
    }}
    return receivers;
}
```

# 2. 获取 receiver 的三种情况

`ParceledListSlice` 类是在进程间通信过程中用于传输一堆元素组成 List 对象的。`queryIntentReceivers()` 方法再调用 `queryIntentReceiversInternal()` 方法。在这个方法分三种情况，一个是 `intent.getComponent()` 不是 `null`，另外两个是 `intent.getPackage()` 不是 `null` 和是 `null` 的情况。我们在 2.1 和 2.2 中分析这三种情况。

```java
// com.android.server.pm.PackageManagerService
public @NonNull ParceledListSlice<ResolveInfo> queryIntentReceivers(Intent intent,
        String resolvedType, int flags, int userId) {
    return new ParceledListSlice<>(
            queryIntentReceiversInternal(intent, resolvedType, flags, userId,
                    false /*allowDynamicSplits*/));
}

private @NonNull List<ResolveInfo> queryIntentReceiversInternal(Intent intent,
        String resolvedType, int flags, int userId, boolean allowDynamicSplits) {
    if (!sUserManager.exists(userId)) return Collections.emptyList();
    final int callingUid = Binder.getCallingUid();
    flags = updateFlagsForResolve(flags, userId, intent, callingUid,
            false /*includeInstantApps*/);
    ComponentName comp = intent.getComponent();
    // ---- 1 ----
    if (comp != null) {
        final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
        final ActivityInfo ai = getReceiverInfo(comp, flags, userId);
        if (ai != null) {
            ...
            // blockResolution is related to instant app
            if (!blockResolution) {
                ResolveInfo ri = new ResolveInfo();
                ri.activityInfo = ai;
                list.add(ri);
        }}
        return applyPostResolutionFilter(
                list, instantAppPkgName, allowDynamicSplits, callingUid, false, userId,
                intent);
    }

    // reader
    synchronized (mPackages) {
        String pkgName = intent.getPackage();
        // ---- 2 ----
        if (pkgName == null) {
            final List<ResolveInfo> result =
                    mReceivers.queryIntent(intent, resolvedType, flags, userId);
            return applyPostResolutionFilter(
                    result, instantAppPkgName, allowDynamicSplits, callingUid, false, userId,
                    intent);
        }
        // ---- 3 ----
        final PackageParser.Package pkg = mPackages.get(pkgName);
        if (pkg != null) {
            final List<ResolveInfo> result = mReceivers.queryIntentForPackage(
                    intent, resolvedType, flags, pkg.receivers, userId);
            return applyPostResolutionFilter(
                    result, instantAppPkgName, allowDynamicSplits, callingUid, false, userId,
                    intent);
        }
        return Collections.emptyList();
}}
```

## 2.0 MATCH_DIRECT_BOOT_UNAWARE

`MATCH_DIRECT_BOOT_UNAWARE` 在 `PackageManager` 中声明，在注释中说明了，如果 `MATCH_DIRECT_BOOT_AWARE` 和 `MATCH_DIRECT_BOOT_UNAWARE` 都没有指定，则只能匹配运行中的组件。当 user locked 状态下只能匹配 `MATCH_DIRECT_BOOT_AWARE` 的组件，当 user unlocked 的时候会匹配 `MATCH_DIRECT_BOOT_AWARE` 和 `MATCH_DIRECT_BOOT_UNAWARE` 的组件。

有点饶，对于我们的情况有什么用处，当系统开启的时候 boot complete 之前是 user locked 状态，boot complete 之后会转到 user unlocked 状态。调用 `UserManager` 类的 `isUserUnlocked()` 可以获取 unlocked 状态。

```java
/**
    * Querying flag: match components which are direct boot <em>unaware</em> in
    * the returned info, regardless of the current user state.
    * <p>
    * When neither {@link #MATCH_DIRECT_BOOT_AWARE} nor
    * {@link #MATCH_DIRECT_BOOT_UNAWARE} are specified, the default behavior is
    * to match only runnable components based on the user state. For example,
    * when a user is started but credentials have not been presented yet, the
    * user is running "locked" and only {@link #MATCH_DIRECT_BOOT_AWARE}
    * components are returned. Once the user credentials have been presented,
    * the user is running "unlocked" and both {@link #MATCH_DIRECT_BOOT_AWARE}
    * and {@link #MATCH_DIRECT_BOOT_UNAWARE} components are returned.
    *
    * @see UserManager#isUserUnlocked()
    */
```

## 2.1 ComponentName 不是 null 时

`intent.getComponent()` 不是 `null` 的情况下，调用 `getReceiverInfo()` 方法。该方法中首先调用 `updateFlagsForComponent()` 方法更新 flags，它又会调用 `updateFlags()` 方法。`updateFlags()` 方法中会根据当前的 user unlocked 状态给 flags 赋值。`isUserUnlockingOrUnlocked()` 返回 true 时设定 `MATCH_DIRECT_BOOT_AWARE` 和 `MATCH_DIRECT_BOOT_UNAWARE`，返回 false 时只设定 `MATCH_DIRECT_BOOT_AWARE`。因此，当 boot complete 之前 user 是 locked 状态，所以只设定 `MATCH_DIRECT_BOOT_AWARE`。

```java
// com.android.server.pm.PackageManagerService
public ActivityInfo getReceiverInfo(ComponentName component, int flags, int userId) {
    if (!sUserManager.exists(userId)) return null;
    final int callingUid = Binder.getCallingUid();
    flags = updateFlagsForComponent(flags, userId, component);
    synchronized (mPackages) {
        PackageParser.Activity a = mReceivers.mActivities.get(component);
        if (a != null && mSettings.isEnabledAndMatchLPr(a.info, flags, userId)) {
            PackageSetting ps = mSettings.mPackages.get(component.getPackageName());
            if (ps == null) return null;
            return PackageParser.generateActivityInfo(
                    a, flags, ps.readUserState(userId), userId);
    }}
    return null;
}

private int updateFlags(int flags, int userId) {
    if ((flags & (PackageManager.MATCH_DIRECT_BOOT_UNAWARE
            | PackageManager.MATCH_DIRECT_BOOT_AWARE)) != 0) {
        // Caller expressed an explicit opinion about what encryption
        // aware/unaware components they want to see, so fall through and
        // give them what they want
    } else {
        // Caller expressed no opinion, so match based on user state
        if (getUserManagerInternal().isUserUnlockingOrUnlocked(userId)) {
            flags |= PackageManager.MATCH_DIRECT_BOOT_AWARE | MATCH_DIRECT_BOOT_UNAWARE;
        } else {
            flags |= PackageManager.MATCH_DIRECT_BOOT_AWARE;
        }
    }
    return flags;
}
```

`getReceiverInfo()` 中有一个 `if` 语句，其中调用了 `Settings` 类的 `isEnabledAndMatchLPr()` 方法。该方法判断给定的 component 是否已经安装、可用且能匹配给定的 flags，最终调用 `UserState` 类的 `isMatch()` 方法。其中有两个变量，`matchesUnaware` 和 `matchesAware` 变量，分别表明该 component 能在 user unlocked 时匹配和能在 user locked 时匹配。

因为讨论的是 boot complete 之前的情况，所以在 `updateFlags()` 中只设定了 `MATCH_DIRECT_BOOT_AWARE`。所以下面我们只看 `matchesAware` 变量。

从代码中可以看到，当 `componentInfo` 的 `directBootAware` 字段是 true 时 `matchesAware` 才是 true。在 AndroidManifest 中声明 receiver 的时候 `directBootAware` 默认是 false，因此，如果没有指定 `directBootAware` 为 true，则 `isMatch()` 会返回 false，`isEnabledAndMatchLPr()` 也会返回 false，最后 `getReceiverInfo()` 会返回 `null`。最终，`queryIntentReceivers()` 的返回值中不包含没有指定 `directBootAware` 为 true 的 receiver，所以也收不到广播。

```java
// com.android.server.pm.Settings
boolean isEnabledAndMatchLPr(ComponentInfo componentInfo, int flags, int userId) {
    final PackageSetting ps = mPackages.get(componentInfo.packageName);
    if (ps == null) return false;

    final PackageUserState userState = ps.readUserState(userId)
    return userState.isMatch(componentInfo, flags);
}

public boolean isMatch(ComponentInfo componentInfo, int flags) {
    final boolean isSystemApp = componentInfo.applicationInfo.isSystemApp();
    final boolean matchUninstalled = (flags & PackageManager.MATCH_KNOWN_PACKAGES) != 0;
    if (!isAvailable(flags) && !(isSystemApp && matchUninstalled)) return false;
    if (!isEnabled(componentInfo, flags)) return false;
    if ((flags & MATCH_SYSTEM_ONLY) != 0) {
        if (!isSystemApp) {
            return false;
    }}
    final boolean matchesUnaware = ((flags & MATCH_DIRECT_BOOT_UNAWARE) != 0)
            && !componentInfo.directBootAware;
    final boolean matchesAware = ((flags & MATCH_DIRECT_BOOT_AWARE) != 0)
            && componentInfo.directBootAware;
    return matchesUnaware || matchesAware;
}
```

## 2.2 ComponentName 是 null 时

根据 `intent.getPackage()` 是不是 `null`，分别调用 `ActivityIntentResolver` 类（继承自 `IntentResolver`）的 `queryIntent()` 方法和 `queryIntentForPackage()` 方法。两种情况我们一起分析，因为这两个方法当中调用 `IntentResolver` 类的 `buildResolveList()` 来查找相应 receiver。

`buildResolveList()` 的参数中有 `F[] src`，这个算是初选出来的候选，在 `ActivityIntentResolver` 类中 `F` 的实际类是 `ActivityIntentInfo`。

```java
// com.android.server.IntentResolver
private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
        boolean debug, boolean defaultOnly, String resolvedType, String scheme,
        F[] src, List<R> dest, int userId)
```

当 intent 中没有指定 package 但指定了 action 时是与 action 匹配的 receiver，而 intent 中指定了 package 时 src 是该应用中的所有 receiver（实际是 `PackageParser.Package` 对象的 `receivers` 字段）。

在 `buildResolveList()` 方法中会遍历 src ，然后调用 `match()` 再次匹配，参数有 `action`、`resolveType`、`scheme`、`data`、`categories`。如果匹配成功，则会调用 `newResult()` 创建一个 R 对象（`ActivityIntentResolver` 中的实际类是 `ResolveInfo`）并添加到返回结果中。

```java
// com.android.server.IntentResolver
private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
            boolean debug, boolean defaultOnly, String resolvedType, String scheme,
            F[] src, List<R> dest, int userId) {
    final int N = src != null ? src.length : 0;
    int i;
    F filter;
    for (i=0; i<N && (filter=src[i]) != null; i++) {
        int match;
        match = filter.match(action, resolvedType, scheme, data, categories, TAG);
        if (match >= 0) {
            if (!defaultOnly || filter.hasCategory(Intent.CATEGORY_DEFAULT)) {
                final R oneResult = newResult(filter, match, userId);
                if (oneResult != null) {
                    dest.add(oneResult);
}}}}}}
```

`ActivityIntentResolver` 类重写了 `newResult()` 方法，该方法中还是会调用 `Settings` 类的 `isEnabledAndMatchLPr()` 判断是否可匹配，否则会返回 null。在 2.1 中已经说明了，在 boot complete 之前，如果 `BroadcastReceiver` 没有设定 `directBootAware` 为 true，则 `isEnabledAndMatchLPr()` 会返回 false。因此，`newResult()` 会返回 null 导致无法接收广播。

```java
// com.android.server.pm.PackageManagerService.ActivityIntentResolver
protected ResolveInfo newResult(PackageParser.ActivityIntentInfo info,
        int match, int userId) {
    if (!sUserManager.exists(userId)) return null;
    if (!mSettings.isEnabledAndMatchLPr(info.activity.info, mFlags, userId)) {
        return null;
    }
    ...
}
```

# 3. 如果设置了 directBootAware 为 true

Direct Boot mode 是在 Android 7.0 加入的，Android 官网中有关于此的文章「[Support Direct Boot mode](https://developer.android.com/training/articles/direct-boot)」，还有官方博客的一篇文章「[Developing for Direct Boot](https://android-developers.googleblog.com/2016/04/developing-for-direct-boot.html?utm_campaign=android_series_blogpost_062116&utm_source=anddev&utm_medium=yt-desc)」。

如果我们在 AndroidManifest 中给 `BroadcastReceiver` 设置了 `directBootAware` 为 true，是不是就可以在 boot complete 之前收到广播了？这个需要分两种情况，Persistent 应用和其他应用。这个与 Android 8.0 加入的「[Broadcast Limitations](https://developer.android.com/about/versions/oreo/background#broadcasts)」有关。

在 `BroadcastQueue` 类的 `processNextBroadcastLocked()` 中会调用 `ActivityManagerService` 类的 `getAppStartModeLocked()` 方法判断应用的 start mode，只要 packageTargetSdk 大于 26，则会返回 `APP_START_MODE_DELAYED_RIGID`。

```java
// com.android.server.am.BroadcastQueue
final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
    if (!skip) {
        final int allowed = mService.getAppStartModeLocked(
                info.activityInfo.applicationInfo.uid, info.activityInfo.packageName,
                info.activityInfo.applicationInfo.targetSdkVersion, -1, true, false, false);
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            if (allowed == ActivityManager.APP_START_MODE_DISABLED) {
                skip = true;
            } else if (((r.intent.getFlags()&Intent.FLAG_RECEIVER_EXCLUDE_BACKGROUND) != 0)
                    || (r.intent.getComponent() == null
                        && r.intent.getPackage() == null
                        && ((r.intent.getFlags()
                                & Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND) == 0)
                        && !isSignaturePerm(r.requiredPermissions))) {
                mService.addBackgroundCheckViolationLocked(r.intent.getAction(),
                        component.getPackageName());
                Slog.w(TAG, "Background execution not allowed: receiving "
                        + r.intent + " to "
                        + component.flattenToShortString());
                skip = true;
}}}}
```

接下来就需要判断在「[Broadcast Limitations](https://developer.android.com/about/versions/oreo/background#broadcasts)」中说明的情况。第一种情况是，当 flags 中包含 `FLAG_RECEIVER_EXCLUDE_BACKGROUND` 时直接忽略，不会发送广播。

第二种情况其实说的就是隐式广播不可接收的情况。当 `getComponent()` 和 `getPackage()` 都返回 null 说明是隐式广播。在隐式广播的情况下，没有设定 `FLAG_RECEIVER_INCLUDE_BACKGROUND`，且该广播要求的 permission 不是系统签名，则会忽略该广播发送。

动态注册不受影响。

# 4. 怎样才能在 boot complete 之前接收广播

首先需要在 AndroidManifest 中声明 `BroadcastReceiver` 的时候需要设定 `directBootAware` 为 true。

如果发送的是显示广播，那就可以接收，不受「[Broadcast Limitations](https://developer.android.com/about/versions/oreo/background#broadcasts)」中说的限制。但如果是隐式广播，需添加 `FLAG_RECEIVER_INCLUDE_BACKGROUND`。`FLAG_RECEIVER_INCLUDE_BACKGROUND` 是 hide 的，但是我们可以硬编码，它的值是 0x01000000。如果应用从来没有启动过，它的状态是停止状态，则需要再加 `FLAG_INCLUDE_STOPPED_PACKAGES`。

如果是想接收系统发出的隐式广播怎么办？没办法，但是系统的隐式广播中有一些是不受限制的，在「[Implicit Broadcast Exceptions](https://developer.android.com/guide/components/broadcast-exceptions.html)」中有说明。