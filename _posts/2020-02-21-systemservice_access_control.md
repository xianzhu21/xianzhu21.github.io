---
layout: post
title: SystemService 需要做的访问控制
date: 2020-02-21
modified: 2020-02-21
tags: [Android, SystemService, Permission]
categories: [Developer]
excerpt:  SystemService 是在 system_server 进程中执行，因此权限也比较高。虽然可以在 API 中做一些访问控制，但是因为有反射、动态代理等技术的存在，在 SystemService 的实际方法实现中做访问控制是必要的...
description: SystemService 是在 system_server 进程中执行，因此权限也比较高。虽然可以在 API 中做一些访问控制，但是因为有反射、动态代理等技术的存在，在 SystemService 的实际方法实现中做访问控制是必要的...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

SystemService 是在 system_server 进程中执行，因此权限也比较高。虽然可以在 API 中做一些访问控制，但是因为有反射、动态代理等技术的存在，在 SystemService 的实际方法实现中做访问控制是必要的。

ActivityManagerService 中定义了几个 enforce 开头的方法，这些都是访问控制相关的。

`enforceCallingPermission()` 是检查调用者是否拥有指定的 permission，最终调用的是 PackageManagerService 类的 `checkUidPermission()`。

```java
// com.android.server.am.ActivityManagerService
/**
    * This can be called with or without the global lock held.
    */
void enforceCallingPermission(String permission, String func) {
    if (checkCallingPermission(permission)
            == PackageManager.PERMISSION_GRANTED) {
        return;
    }
    throw new SecurityException(msg);
}

/**
    * Binder IPC calls go through the public entry point.
    * This can be called with or without the global lock held.
    */
int checkCallingPermission(String permission) {
    return checkPermission(permission, Binder.getCallingPid(),
            UserHandle.getAppId(Binder.getCallingUid()));
}

public int checkPermission(String permission, int pid, int uid) {
    if (permission == null) {
        return PackageManager.PERMISSION_DENIED;
    }
    return checkComponentPermission(permission, pid, uid, -1, true);
}

int checkComponentPermission(String permission, int pid, int uid,
        int owningUid, boolean exported) {
    if (pid == MY_PID) {
        return PackageManager.PERMISSION_GRANTED;
    }
    return ActivityManager.checkComponentPermission(permission, uid,
            owningUid, exported);
}

// android.app.ActivityManager
public static int checkComponentPermission(String permission, int uid,
        int owningUid, boolean exported) {
    // Root, system server get to do everything.
    final int appId = UserHandle.getAppId(uid);
    if (appId == Process.ROOT_UID || appId == Process.SYSTEM_UID) {
        return PackageManager.PERMISSION_GRANTED;
    }
    // If there is a uid that owns whatever is being accessed, it has
    // blanket access to it regardless of the permissions it requires.
    if (owningUid >= 0 && UserHandle.isSameApp(uid, owningUid)) {
        return PackageManager.PERMISSION_GRANTED;
    }
    if (permission == null) {
        return PackageManager.PERMISSION_GRANTED;
    }
    try {
        return AppGlobals.getPackageManager()
                .checkUidPermission(permission, uid);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
}}
```

AppOpsService 中定义了方法 `enforceManageAppOpsModes()`。它会调用 Context 类的 `enforcePermission()`，最后也是会调用到 ActivityManagerService 的 `checkPermission()`。

Context 类中已经定义了几个供使用的访问控制相关的方法：

```java
int checkPermission(String permission, int pid, int uid);
int checkPermission(String permission, int pid, int uid, IBinder callerToken);
int checkCallingPermission(String permission);
int checkCallingOrSelfPermission(String permission);
int checkSelfPermission(String permission);

void enforcePermission(String permission, int pid, int uid, String message);
void enforceCallingPermission(String permission, String message);
void enforceCallingOrSelfPermission(String permission, String message);
```

这么一看，SystemService 中的访问控制实际还是调用的 PackageManagerService 类的 `checkUidPermission()`，即 Android permission 机制。

在 API 的方法中可以使用 `@RequiresPermission` 注解，这样可以在构建 App 的时候检查是否声明了权限。

```java
// frameworks/base/core/java/android/os/PowerManager.java
/**
    * Notifies the power manager that user activity happened.
    * <p>
    * Resets the auto-off timer and brightens the screen if the device
    * is not asleep.  This is what happens normally when a key or the touch
    * screen is pressed or when some other user activity occurs.
    * This method does not wake up the device if it has been put to sleep.
    * </p><p>
    * Requires the {@link android.Manifest.permission#DEVICE_POWER} or
    * {@link android.Manifest.permission#USER_ACTIVITY} permission.
    * </p>
    *
    * @param when The time of the user activity, in the {@link SystemClock#uptimeMillis()}
    * time base.  This timestamp is used to correctly order the user activity request with
    * other power management functions.  It should be set
    * to the timestamp of the input event that caused the user activity.
    * @param event The user activity event.
    * @param flags Optional user activity flags.
    *
    * @see #wakeUp
    * @see #goToSleep
    *
    * @hide Requires signature or system permission.
    */
@SystemApi
@RequiresPermission(anyOf = {
        android.Manifest.permission.DEVICE_POWER,
        android.Manifest.permission.USER_ACTIVITY
})
public void userActivity(long when, int event, int flags) {
    try {
        mService.userActivity(when, event, flags);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
