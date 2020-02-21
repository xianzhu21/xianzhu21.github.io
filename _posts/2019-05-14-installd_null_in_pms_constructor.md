---
layout: post
title: PMS 初始化时抛出 installd 空指针问题分析
date: 2019-05-14
modified: 2019-06-26
tags: [Android,PackageManager,installd]
categories: [Developer]
excerpt:  从 log 中看出异常原因是调用 IInstalld 方法的时候抛出了空指针异常，IInstalld 其实就是 installd 在 SystemServer 中的代理，Java 层对应的 Service 是 Installer...
description: 从 log 中看出异常原因是调用 IInstalld 方法的时候抛出了空指针异常，IInstalld 其实就是 installd 在 SystemServer 中的代理，Java 层对应的 Service 是 Installer...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

# 0. 错误 log

Android 8.1 系统。

```shell
1. E PackageManager: Failed to create app data for com.xxxx.iot.service, but trying to recover: com.android.server.pm.Installer$InstallerException: java.lang.NullPointerException: Attempt to invoke interface method 'long android.os.IInstalld.createAppData(java.lang.String, java.lang.String, int, int, int, java.lang.String, int)' on a null object reference
2. W PackageManager: com.android.server.pm.Installer$InstallerException: java.lang.NullPointerException: Attempt to invoke interface method 'void android.os.IInstalld.destroyAppData(java.lang.String, java.lang.String, int, int, long)' on a null object reference
3. D PackageManager: Recovery failed!
4. W PackageManager: Failed to migrate com.xxxx.iot.service: java.lang.NullPointerException: Attempt to invoke interface method 'void android.os.IInstalld.migrateAppData(java.lang.String, java.lang.String, int, int)' on a null object reference
```

# 1. SystemServer 连接 installd

从 log 中看出异常原因是调用 IInstalld 方法的时候抛出了空指针异常，IInstalld 其实就是 installd 在 SystemServer 中的代理，Java 层对应的 Service 是 Installer，在 systemServer 的 `startBootstrapServices()` 方法中连接 installd。

```java
// frameworks/base/services/java/com/android/server/SystemServer.java
/**
 * Starts the small tangle of critical services that are needed to get
 * the system off the ground.  These services have complex mutual dependencies
 * which is why we initialize them all in one place here.  Unless your service
 * is also entwined in these dependencies, it should be initialized in one of
 * the other functions.
 */
private void startBootstrapServices() {
    ...
    Installer installer = mSystemServiceManager.startService(Installer.class);
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    ...
}
```

Android 8.0 之前 SystemServer 与 installd 是通过 socket 连接的，在连接的时候会阻塞等待 installd 连接成功。

```java
// frameworks/base/core/java/com/android/internal/os/InstallerConnection.java
public void waitForConnection() {
    for (;;) {
        try {
            execute("ping");
            return;
        } catch (InstallerException ignored) {}
        Slog.w(TAG, "installd not ready");
        SystemClock.sleep(1000);
}}
```

但是在 Android 8.0 把 installd 的连接方式改成了 binder，且不会阻塞等待 installd 连接成功，而是每隔一秒重试，直到连接成功。

```java
// frameworks/base/services/core/java/com/android/server/pm/Installer.java
private void connect() {
    IBinder binder = ServiceManager.getService("installd");
    if (binder != null) {
        try {
            binder.linkToDeath(new DeathRecipient() {
                @Override
                public void binderDied() {
                    Slog.w(TAG, "installd died; reconnecting");
                    connect();
                }
            }, 0);
        } catch (RemoteException e) {
            binder = null;
        }
    }
    if (binder != null) {
        mInstalld = IInstalld.Stub.asInterface(binder);
    } else {
        Slog.w(TAG, "installd not found; trying again");
        // 一秒后重试
        BackgroundThread.getHandler().postDelayed(() -> {
            connect();
        }, DateUtils.SECOND_IN_MILLIS);
}}
```

# 2. 初始化 PackageManagerService

PackageManagerService（下面称为 PMS）的初始化也是在 `startBootstrapServices()` 方法中，且是在创建 Installer 对象之后。调用 PMS 的静态方法 `main()` 进行初始化，参数传入了 Installer 对象。

```java
// frameworks/base/services/java/com/android/server/SystemServer.java
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
```

PMS 的初始化中会扫描 APK，并为应用创建私有目录。扫描 APK 时如果没有需要清除的 APK 则不会调用 installd，所以我们关注给应用创建私有目录的过程，它是调用 `reconcileAppsDataLI()` 方法。

```java
// frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
// Prepare storage for system user really early during boot,
// since core system apps like SettingsProvider and SystemUI
// can't wait for user to start
final int storageFlags;
if (StorageManager.isFileEncryptedNativeOrEmulated()) {
    storageFlags = StorageManager.FLAG_STORAGE_DE;
} else {
    storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
}
List<String> deferPackages = reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL,
        UserHandle.USER_SYSTEM, storageFlags, true /* migrateAppData */,
        true /* onlyCoreApps */);
```

`FLAG_STORAGE_CE` 指的是 `/data/user/0/`，即 `/data/data`，该目录只能在解锁屏幕后才能访问。而 `FLAG_STORAGE_DE` 指的是 `/data/user_de/0`，该目录在有屏幕锁的时候也能访问，参考 Android 官网关于 [DirectBoot](https://developer.android.com/training/articles/direct-boot.html) 的介绍。 

`reconcileAppsDataLI()` 方法中判断 `ceDir` 和 `deDir` 中的应用是否是已知的（参考[系统启动时 PMS 的扫描应用分析](https://www.notion.so/89cd25c3-6c49-4606-bf5e-d109e9ace62d)）且已安装的，如果不是的话调用 installd 的 `destroyAppData()` 函数删除对应私有目录。

之后对于已安装的应用调用 `prepareAppDataAndMigrateLIF()` 方法。

```java
// frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
if (ps.getInstalled(userId)) {
    prepareAppDataAndMigrateLIF(ps.pkg, userId, flags, migrateAppData);
    preparedCount++;
}
```

`prepareAppDataAndMigrateLIF()` 最终会调用 Installd 的 `createAppData()` 函数创建应用的私有目录。第一行 log 就是调用 `createAppData()` 时抛出的异常。就是说当调用 `createAppData()` 时 installd 还没连接成功，所以抛出了空指针异常。这样就不能创建私有目录（包括 CE 和 DE）。这不就是启动失败吗？没有创建应用私有目录，如果应用需要访问就会抛出 IO 异常。但是在两种情况下是没有问题的：一种是系统不是第一次启动，之前启动的时候已经创建好了私有目录；另一种是应用在 boot complete 之后才访问 CE 目录，且永不访问 DE 目录。

# 3. Boot complete

第一种情况比较好理解，因为之前已经创建好了已经安装的私有目录，所以没有什么问题。第二种情况换句话说在 boot complete 的时候会创建 CE 私有目录。

在 ActivityManagerService 的 `finishBooting()` 方法会调用 UserController 的 `sendBootCompletedLocked()` 方法。它会调用 `finishUserBoot()` 方法改变 UserState 的状态。

```java
// frameworks/base/services/core/java/com/android/server/am/UserController.java
private void finishUserBoot(UserState uss, IIIntentReceiver resultTo) {
    if (uss.setState(STATE_BOOTING, STATE_RUNNING_LOCKED)) {
        mInjector.getUserManagerInternal().setUserState(userid, uss.state);
    }
    maybeUnlockUser(userId);
}
```

把状态改为 `RUNNING_LOCKED` 后尝试解锁用户，当设备没有 credential-encrypted storage 时会解锁成功。

```java
/**
    * Attempt to unlock user without a credential token. This typically
    * succeeds when the device doesn't have credential-encrypted storage, or
    * when the the credential-encrypted storage isn't tied to a user-provided
    * PIN or pattern.
    */
boolean maybeUnlockUser(final int userId) {
    // Try unlocking storage using empty token
    return unlockUserCleared(userId, null, null, null);
}
```

如果解锁成功则调用 `finishUserUnlocking()` 方法。它会把状态改为 `RUNNING_UNLOCKING`，并调用 UserManagerService 的 `onBeforeUnlockUser()` 方法。

```java
// frameworks/base/services/core/java/com/android/server/am/UserController.java
private void finishUserUnlocking(final UserState uss) {
    if (uss.setState(STATE_RUNNING_LOCKED, STATE_RUNNING_UNLOCKED)) {
        mInjector.getUserManagerInternal().setUserState(userid, uss.state);
        proceedWithUnlock = true;
    }
    if (proceedWithUnlock) {
        mInjector.getUserManager().onBeforeUnlockUser(userId);
    }
}
```

在 `onBeforeUnlockUser()` 方法中会调用 PMS 的 `reconcileAppsData()` 方法，传入参数是 `FLAG_STORAGE_CE`。这时会创建应用的 CE 私有目录。所以说，如果系统应用（三方应用不会在 boot complete 之前启动）在 boot complete 之前不访问 CE 目录，且永不访问 DE 目录则不会有问题。

其实 boot complete 之后 DE 目录也有创建的情况，那就是当切换到非 `USER_SYSTEM` 的时候。这时会调用 `startUser()` 方法，它会调用 UserManagerService 的 `onBeforeStartUser()` 方法。在 `onBeforeStartUser()` 中会调用 `reconcileAppsData()` 方法（`FLAG_STORAGE_DE`）。

# 4. 分析问题原因

那为什么 installd 会空？从 log 分析是因为 SystemServer 启动的比较快，即 PMS 的初始化太快了。为什么不是 installd 启动太慢而是 SystemServer 启动太快？

我计算了 installd 启动时间，从 vold 启动开始计时（`Vold 3.0 (the awakening) firing up`），到 installd 启动（`installd firing up`），花了 5.379 秒。我又测试了正常情况下的 installd 启动时间，均在 5.4 秒左右，所以可以判断 installd 启动正常。

但是 vold 启动到 SystemServer StartInstallers 花了 4.250 秒，而正常启动 3 次的平均花费是 6.157 秒。

在第一次连接 installd 失败一秒后尝试时 installd 还没启动，所以又要等一秒，而这时 PMS 已经开始初始化了，所以当 PMS 调用 installd 的函数的时候抛出了空指针异常。

那为什么 SystemServer 启动这么快，分析了 Zygote 和 SystemServer 相关 log 后无法找出原因，因为与正常启动的 log 相同。

那 Android 8.0 之前是等待 installd 连接，为什么 Android 8.0 之后开始异步连接了呢？我猜测是为了加快开机启动速度，而出现 SystemServer 需要使用 installd 的时候 installd 还未连接的状况，我觉得应该当成启动不正常来对待。

# 5. 总结

抛出空指针异常的直接原因是 installd 还未连接，未连接的原因是 SystemServer 连接 installd 的时候还未启动。为什么还未启动，原因是 SystemServer 启动快，而不是 installd 启动慢。

导致的结果分两种情况，第一次启动和其他：第一次启动时应用的私有目录还未创建，installd 未连接导致私有目录无法创建，但是在 boot complete 的时候（假设这时 installd 连接成功）会再次创建 CE 的私有目录，但 DE 私有目录一直不会创建；但如果 installd 长时间未连接会出现更严重的问题，因为 PMS 的安装卸载等等操作实际是 installd 来实现的。