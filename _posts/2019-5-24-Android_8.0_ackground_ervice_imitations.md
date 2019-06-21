---
layout: post
title: Android 8.0 加入的「Background Service Limitations」
date: 2019-05-24
modified: 2019-06-21
tags: [Android,PackageManager,ActivityManager]
categories: [I'm a developer]
excerpt:  为了减少系统资源的使用，Android 8 引入了「Background Execution Limits」，其中对于 Service 的限制是，不能调用 `startService()` 方法启动一个后台 Service，`bindService()` 无影响...
description: 为了减少系统资源的使用，Android 8 引入了「Background Execution Limits」，其中对于 Service 的限制是，不能调用 `startService()` 方法启动一个后台 Service，`bindService()` 无影响...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.github.io](xianzhu21.github.io)。

# 0. Background Service Limitations 介绍

为了减少系统资源的使用，Android 8 引入了「[Background Execution Limits](https://developer.android.com/about/versions/oreo/background)」，其中对于 Service 的限制是，不能调用 `startService()` 方法启动一个后台 Service，`bindService()` 无影响。

满足下面条件中的一项，则被定义为前台应用：

- 有可见的 Activity。
- 拥有一个前台 Service。调用 `startForegroundService()` 方法启动前台 Service。
- 其他的前台应用绑定到该应用，不管是绑定服务或是使用 ContentProvider。

一个应用进入后台后，会有几分钟的时间窗口期，之后会进入 idle 状态。进入 idle 状态后，后台 Service 被受限，而且系统会停止已经启动的 Service，就像调用 `stopSelf()` 一样。

当应用在处理用户可见任务情况下，会临时加入到白名单中，并持续数分钟。例如：

- 处理高优先级的 Firebase Cloud Messaging 消息。
- 接收广播，例如短信或彩信。
- 从 Notification 执行 PendingIntent。
- 启动 VpnService。

注意，`IntentService` 也是 Service，所以也被受限。因此，[Android Support Library 26.0.0](https://developer.android.com/topic/libraries/support-library/revisions.html#26-0-0) 中介绍了 `JobIntentService` 类，它提供与 `IntentService` 类似的功能，但是使用 `JobSchedule` 实现。

大部分情况下， 可以用 `JobSchedule` 替换后台 Service。

启动前台 Service 的方法是 `startForegroundService()`，Service 启动后需要在 5 秒内调用 `startForeground()` 方法启动用户可见的 Notification，否则系统会停止该 Service，并报出 ANR。

# 1. 应用后台执行限制的源码分析 (Android 9.0)

## 1.1 `ActivityManagerService.getAppStartModeLocked()`

一个应用是否可以后台执行（Background Execution），与 `ActivityManagerService` 的 `getAppStartModeLocked()` 方法返回结果相关。返回结果在 `ActivityManager` 中定义。

```java
/** @hide Mode for {@link IActivityManager#isAppStartModeDisabled}: normal free-to-run operation. */
public static final int APP_START_MODE_NORMAL = 0;

/** @hide Mode for {@link IActivityManager#isAppStartModeDisabled}: delay running until later. */
public static final int APP_START_MODE_DELAYED = 1;

/** @hide Mode for {@link IActivityManager#isAppStartModeDisabled}: delay running until later, with
    * rigid errors (throwing exception). */
public static final int APP_START_MODE_DELAYED_RIGID = 2;

/** @hide Mode for {@link IActivityManager#isAppStartModeDisabled}: disable/cancel pending
    * launches; this is the mode for ephemeral apps. */
public static final int APP_START_MODE_DISABLED = 3;
```

`APP_START_MODE_NORMAL` 是可以执行；`APP_START_MODE_DELAYED` 针对的是 `packageTargetSdk < 28` 的应用，对于这些应用会悄悄地不执行，不会抛出错误。`APP_START_MODE_DELAYED_RIGID` 针对的是 `packageTargetSdk >= 28` 的应用，对于这些应用会抛出异常，并执行失败。`APP_START_MODE_DISABLED` 针对的是 InstantApp，本文不讨论。

```java
int getAppStartModeLocked(int uid, String packageName, int packageTargetSdk,
        int callingPid, boolean alwaysRestrict, boolean disabledOnly) {
    UidRecord uidRec = mActiveUids.get(uid);
    if (uidRec == null || alwaysRestrict || uidRec.idle) {
        if (ephemeral) {
            // We are hard-core about ephemeral apps not running in the background.
            return ActivityManager.APP_START_MODE_DISABLED;
        } else {
            if (disabledOnly) {
                // The caller is only interested in whether app starts are completely
                // disabled for the given package (that is, it is an instant app).  So
                // we don't need to go further, which is all just seeing if we should
                // apply a "delayed" mode for a regular app.
                return ActivityManager.APP_START_MODE_NORMAL;
            }
            final int startMode = (alwaysRestrict)
                    ? appRestrictedInBackgroundLocked(uid, packageName, packageTargetSdk)
                    : appServicesRestrictedInBackgroundLocked(uid, packageName,
                            packageTargetSdk);
            if (startMode == ActivityManager.APP_START_MODE_DELAYED) {
                // This is an old app that has been forced into a "compatible as possible"
                // mode of background check.  To increase compatibility, we will allow other
                // foreground apps to cause its services to start.
                if (callingPid >= 0) {
                    ProcessRecord proc;
                    synchronized (mPidsSelfLocked) {
                        proc = mPidsSelfLocked.get(callingPid);
                    }
                    if (proc != null &&
                            !ActivityManager.isProcStateBackground(proc.curProcState)) {
                        // Whoever is instigating this is in the foreground, so we will allow it
                        // to go through.
                        return ActivityManager.APP_START_MODE_NORMAL;
            }}}
            return startMode;
    }}
    return ActivityManager.APP_START_MODE_NORMAL;
}
```

当应用在以下三种情况时检查后台限制，即被认为是后台应用：

- 未启动，即 `uidRec == null`。
- `alwaysRestrict` 为 true。这个只有在检查隐式广播的时候传入 true，启动后台 Service 时传入的是 false。
- 应用在 idle 模式。

如果是后台应用的话，根据 `alwaysRestrict` 的值调用 `appRestrictedInBackgroundLocked()` 方法或 `appServicesRestrictedInBackgroundLocked()` 方法。

上面调用的方法返回的 `startMode` 是 `APP_START_MODE_DELAYED` 说明该应用的 `packageTargetSdk < 28`。为了兼容性，如果调用者是前台应用，则返回 `APP_START_MODE_NORMAL`。

`appRestrictedInBackgroundLocked()` 方法检查应用的后台执行限制。它首先判断 `packageTargetSdk`，如果小于 28，再判断 `AppOpsService` 是否允许。

```java
// Unified app-op and target sdk check
int appRestrictedInBackgroundLocked(int uid, String packageName, int packageTargetSdk) {
    // Apps that target O+ are always subject to background check
    if (packageTargetSdk >= Build.VERSION_CODES.O) {
        if (DEBUG_BACKGROUND_CHECK) {
            Slog.i(TAG, "App " + uid + "/" + packageName + " targets O+, restricted");
        }
        return ActivityManager.APP_START_MODE_DELAYED_RIGID;
    }
    // ...and legacy apps get an AppOp check
    int appop = mAppOpsService.noteOperation(AppOpsManager.OP_RUN_IN_BACKGROUND,
            uid, packageName);
    if (DEBUG_BACKGROUND_CHECK) {
        Slog.i(TAG, "Legacy app " + uid + "/" + packageName + " bg appop " + appop);
    }
    switch (appop) {
        case AppOpsManager.MODE_ALLOWED:
            return ActivityManager.APP_START_MODE_NORMAL;
        case AppOpsManager.MODE_IGNORED:
            return ActivityManager.APP_START_MODE_DELAYED;
        default:
            return ActivityManager.APP_START_MODE_DELAYED_RIGID;
    }
}
```

`appServicesRestrictedInBackgroundLocked()` 方法检查 Service 的后台执行限制。三种情况下，后台 Service 不受限制：

- 该应用是 Persistent，即 `ApplicationInfo.flags` 中包含 `ApplicationInfo.FLAG_SYSTEM` 和 `ApplicationInfo.FLAG_PERSISTENT`。
- 该 uid 在 background 白名单中。
- 该 uid 在 idle 白名单中。

不是上面的三种情况，则调用 `appRestrictedInBackgroundLocked()` 方法。

```java
// Service launch is available to apps with run-in-background exemptions but
// some other background operations are not.  If we're doing a check
// of service-launch policy, allow those callers to proceed unrestricted.
int appServicesRestrictedInBackgroundLocked(int uid, String packageName, int packageTargetSdk) {
    // Persistent app?
    if (mPackageManagerInt.isPackagePersistent(packageName)) {
        return ActivityManager.APP_START_MODE_NORMAL;
    }
    // Non-persistent but background whitelisted?
    if (uidOnBackgroundWhitelist(uid)) {
        return ActivityManager.APP_START_MODE_NORMAL;
    }
    // Is this app on the battery whitelist?
    if (isOnDeviceIdleWhitelistLocked(uid)) {
        return ActivityManager.APP_START_MODE_NORMAL;
    }
    // None of the service-policy criteria apply, so we apply the common criteria
    return appRestrictedInBackgroundLocked(uid, packageName, packageTargetSdk);
}
```

第一个 `if` 判断的 `mPackageManagerInt.isPackagePersistent(packageName)` 方法最终会调用 PackageManagerService.PackageManagerInternalImpl 的 `isPackagePersistent()` 方法。它会检查该应用是否同时拥有 `FLAG_SYSTEM` 和 `FLAG_PERSISTENT`。

```java
@Override
public boolean isPackagePersistent(String packageName) {
    synchronized (mPackages) {
        PackageParser.Package pkg = mPackages.get(packageName);
        return pkg != null
                ? ((pkg.applicationInfo.flags&(ApplicationInfo.FLAG_SYSTEM
                                | ApplicationInfo.FLAG_PERSISTENT)) ==
                        (ApplicationInfo.FLAG_SYSTEM | ApplicationInfo.FLAG_PERSISTENT))
                : false;
    }
}
```

## 1.2 `ActiveServices.startServiceLocked()`

`startService()` 方法最终会调用 `ActivityManagerService` 的 `startService()` 方法，而它又调用 `ActiveServices` 的 `startServiceLocked()` 方法。它会先查找对应的 ServiceRecord 对象，当 ServiceRecord 对象的 startRequested 字段为 false（说明未启动），且 fgRequired 为 false（说明是后台 Service），则调用 `getAppStartModeLocked()` 方法检查是否允许启动。

当返回的值不等于 `APP_START_MODE_NORMAL` 时启动失败。并且，当返回的是 `APP_START_MODE_DELAYED` 时，则直接返回 null，即上面说的悄悄地启动失败。否则返回 `mPackage` 字段值为 `?` 的 `ComponentName` 对象。

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
        throws TransactionTooLargeException {
    ServiceRecord r = res.record;
    // If this isn't a direct-to-foreground start, check our ability to kick off an
    // arbitrary service
    if (!r.startRequested && !fgRequired) {
        // Before going further -- if this app is not allowed to start services in the
        // background, then at this point we aren't going to let it period.
        final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                r.appInfo.targetSdkVersion, callingPid, false, false);
        if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
            if (allowed == ActivityManager.APP_START_MODE_DELAYED) {
                // In this case we are silently disabling the app, to disrupt as
                // little as possible existing apps.
                return null;
            }
            // This app knows it is in the new model where this operation is not
            // allowed, so tell it what has happened.
            UidRecord uidRec = mAm.mActiveUids.get(r.appInfo.uid);
            return new ComponentName("?", "app is in background uid " + uidRec);
}}}
```

在 `ComtextImpl` 中会处理这个特殊的 `ComponentName` 对象，处理方式是直接抛出 `IllegalStateException` 异常。

```java
private ComponentName startServiceCommon(Intent service, boolean requireForeground,
        UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess(this);
        ComponentName cn = ActivityManager.getService().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                        getContentResolver()), requireForeground,
                        getOpPackageName(), user.getIdentifier());
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) {
                throw new SecurityException(
                        "Unable to start service " + service
                        + ": " + cn.getClassName());
            } else if (cn.getPackageName().equals("?")) {
                throw new IllegalStateException(
                        "Not allowed to start service " + service + ": " + cn.getClassName());
            }
        }
        return cn;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```


**References:**
- [Background Execution Limits](https://developer.android.com/about/versions/oreo/background)