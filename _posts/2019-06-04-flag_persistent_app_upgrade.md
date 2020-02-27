---
layout: post
title: FLAG_PERSISTENT 应用是否可以更新
date: 2019-06-04
modified: 2019-06-20
tags: [Android,PackageManager]
categories: [Developer]
excerpt:  Persistent flag 指的是 ApplicationInfo 类中的常量 FLAG_PERSISTENT (1<<3)。如果一个应用是 persistent 的，则永远在运行状态，即时被 kill 也会被重新启动。SystemUI 就是 persistent 的应用，可以想象如果 SystemUI 停止运行了...
description: Persistent flag 指的是 `ApplicationInfo` 类中的常量 `FLAG_PERSISTENT (1<<3)`。如果一个应用是 persistent 的，则永远在运行状态，即时被 kill 也会被重新启动。SystemUI 就是 persistent 的应用，可以想象如果 SystemUI 停止运行了...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

* TOC
{:toc}

# 0. Persistent flag 介绍

Persistent flag 指的是 `ApplicationInfo` 类中的常量 `FLAG_PERSISTENT (1<<3)`。如果一个应用是 persistent 的，则一直在运行状态，即使被 kill 也会被重新启动。SystemUI 就是 persistent 的应用，可以想象如果 SystemUI 停止运行了，StatusBar 和 NavigationBar 也就没有了，这肯定是不正常的。Persistent 属性如何设置，很简单，在 AndroidManifest.xml 的 application 标签中声明就可以。

```xml
<application
    android:name=".SystemUIApplication"
    android:persistent="true"
    ...
    >
    ...
</application>
```

所有应用安装后会在 `/data/system/packages.xml` 中更新相关信息，其中 publicFlag 就是该应用的所拥有的 flags。可以把该值转成二进制后查看千位是否是 1 来判断该应用是否拥有 persistent 属性。

# 1. 应用的 Persistent flag 权限

不是说只要在 AndroidManifest.xml 中声明 persistent 便可以拥有 persistent flag。Persistent flag 只有系统应用才会生效，而且是安装在 system 或 vendor 分区的系统应用。

`ApplicationInfo.FLAG_PERSISTENT` 是在 `PackageParser` 类的 `parseBaseApplication()` 方法中设置的。因为 Android 9.0 对 persistent flag 做了一些修改，所以下面分为 Android 8.1 和 Android 9.0 来分析源码。

## 1.1 Android 8.1

对于 Android 8.1，能否设置 `FLAG_PERSISTENT` 有两个条件，第一个是 `parseBaseApplication()` 方法的参数 `flags` 中包含 `PARSE_IS_SYSTEM`，第二个是 AndroidManifest 中声明了 persistent。

```java
package android.content.pm;
public class PackageParser {
    /**
    * Parse the {@code application} XML tree at the current parse location in a
    * <em>base APK</em> manifest.
    * <p>
    * When adding new features, carefully consider if they should also be
    * supported by split APKs.
    */
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
        throws XmlPullParserException, IOException {
        ...
        if ((flags&PARSE_IS_SYSTEM) != 0) {
            if (sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestApplication_persistent,
                    false)) {
                // Check if persistence is based on a feature being present
                final String requiredFeature = sa.getNonResourceString(
                    com.android.internal.R.styleable.
                    AndroidManifestApplication_persistentWhenFeatureAvailable);
                if (requiredFeature == null || mCallback.hasFeature(requiredFeature)) {
                    ai.flags |= ApplicationInfo.FLAG_PERSISTENT;
        }}}
        ...
}}
```

参数 `flags` 是，PackageManagerService 中调用 `scanDirTracedLI()` 方法或调用 `installPackageLI()` 方法时传入的 `parseFlags` 参数。

在调用 `installPackageLI()` 时没用设置 `PARSE_IS_SYSTEM`，而调用 `scanDirTracedLI()` 时会根据扫描的目录来设置的。

```java
// Find base frameworks (resource packages without code).
scanDirTracedLI(frameworkDir, mDefParseFlags
        | PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED,
        scanFlags | SCAN_NO_DEX, 0);

// Collected privileged system packages.
final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
scanDirTracedLI(privilegedAppDir, mDefParseFlags
        | PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

// Collect ordinary system packages.
final File systemAppDir = new File(Environment.getRootDirectory(), "app");
scanDirTracedLI(systemAppDir, mDefParseFlags
        | PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
...
// mAppInstallDir = new File(dataDir, "app");
scanDirTracedLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
```

上面的代码只是列出了部分目录的扫描。从源码中可知，当扫描以下目录时会设置 `PARSE_IS_SYSTEM`。 

```
/vender/overlay
/system/framework
/system/priv-app
/system/app
/vendor/app
/oem/app
```

因此，不管是安装应用还是扫描 `/data/app` 中的 APK，都不会设置 `PARSE_IS_SYSTEM`，即普通应用是没有 persistent flag 权限的，即使在 AndroidManifest 中声明了也不会生效。

## 1.2 Android 9.0

Android 9.0 的 `parseBaseApplication()` 方法中设置 `FLAG_PERSISTENT` 的那个 `if` 代码块没有检查 `PARSE_IS_SYSTEM` flag（其实 Android 9.0 去掉了 `PARSE_IS_SYSTEM` flag），而只要在 AndroidManifest 中声明了 persistent 属性就会设置 `FLAG_PERSISTENT`。

```java
/**
    * Parse the {@code application} XML tree at the current parse location in a
    * <em>base APK</em> manifest.
    * <p>
    * When adding new features, carefully consider if they should also be
    * supported by split APKs.
    */
private boolean parseBaseApplication(Package owner, Resources res,
        XmlResourceParser parser, int flags, String[] outError) 
    throws XmlPullParserException, IOException {
    ...
    if (sa.getBoolean(
            com.android.internal.R.styleable.AndroidManifestApplication_persistent,
            false)) {
        // Check if persistence is based on a feature being present
        final String requiredFeature = sa.getNonResourceString(com.android.internal.R.styleable
                .AndroidManifestApplication_persistentWhenFeatureAvailable);
        if (requiredFeature == null || mCallback.hasFeature(requiredFeature)) {
            ai.flags |= ApplicationInfo.FLAG_PERSISTENT;
    }}
    ...
}
```

这是不是说明，普通应用也能赋予 persistent flag 呢？答案当然是否定的。在后续的安装或扫描流程中，在 `scanPackageNewLI()` 方法会调用 `applyPolicy()` 方法。在这个方法中，如果不是系统应用，则会清除 `FLAG_PERSISTENT`。

```java
/**
    * Applies policy to the parsed package based upon the given policy flags.
    * Ensures the package is in a good state.
    * <p>
    * Implementation detail: This method must NOT have any side effect. It would
    * ideally be static, but, it requires locks to read system state.
    */
private static void applyPolicy(PackageParser.Package pkg, final @ParseFlags int parseFlags,
        final @ScanFlags int scanFlags, PackageParser.Package platformPkg) {
    ...
    if ((scanFlags & SCAN_AS_SYSTEM) != 0) {
        pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SYSTEM;
        ...
    } else {
        // non system apps can't be flagged as core
        pkg.coreApp = false;
        // clear flags not applicable to regular apps
        pkg.applicationInfo.flags &=
                ~ApplicationInfo.FLAG_PERSISTENT;
        ...
}}
```

上面的代码中 `SCAN_AS_SYSTEM` 的设置与 Android 8.0 中 `PARSE_IS_SYSTEM` 一样，是扫描 system 分区和 vendor 分区的 APK 时设置的，

# 2. Persistent 应用是否可以更新？

更新指的是走安装流程，即 PackageManagerService 的 `installPackageLI()` 方法。在 Android 9.0 之前对于 persistent 应用的升级没有任何限制，但升级后的 APK 在 `/data/app` 中。从上文可知，扫描 `/data/app` 的 APK 时不会设置 `PARSE_IS_SYSTEM` flag，因此，persistent flag 也会失效。

原本 persistent 应用失去 persistent flag 肯定会受影响，以 SystemUI 为例，SystemUI 是 Service，并且在 SystemServer 中会调用 `startServiceAsUser()` 方法启动 SystemUI。在 Android 8.0 中加入了后台 Service 启动限制，参考 [Android 8.0 加入的「Background Service Limitations」](../background_service_imitations/)，对于没有 `ApplicationInfo.FLAG_SYSTEM` 和 `ApplicationInfo.FLAG_PERSISTENT` 的应用无法后台启动 Service。因此，SystemServer 是无法启动升级后的 SystemUI。

在 Android 9.0 之后的版本，对 persistent 应用的更新做了限制。如果要替换的应用是 persistent，则直接抛出异常，安装失败。

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    ...
    // Prevent persistent apps from being updated
    if ((oldPackage.applicationInfo.flags & ApplicationInfo.**FLAG_PERSISTENT**) != 0) {
    res.setError(PackageManager.INSTALL_FAILED_INVALID_APK,
            "Package " + oldPackage.packageName + " is a persistent app. "
                    + "Persistent apps are not updateable.");
        return;
    }
    ...
}
```

下面的 Git commit message 中说明了，persistent 应用无法用 adb 或其他 installer 更新，而且说明了之前如果更新了 persistent 应用，会丢失 persistent flag。

```
Block the upgrade of persistent apps

Currently when upgrading a persistent app the persistent flag will
be silently stripped from the newly installed app, resulting in
inconsitent app behaviour.

Persistent apps by definiton should not be possible to upgrade via
adb or any 3rd party installer. This commit adds a check which will
fail the upgrade of an app marked as persistent.
Bug: 66889554
Test: Manual
        - Create an app with the persistent flag set to true,
        push to /system/priv-app and reboot
        - Increment the app versionCode, build and install using
        adb install -r
        The attempted adb install should fail with:
        INSTALL_FAILED_INVALID_APK
Change-Id: Iaa00addcfbbb2adba319bea96b37e8bfb6e11372
```

在 Google 的 issue tracker 中，有人提问了 Android 9.0 为什么无法升级 persistent 应用的问题「[failed update to persistent application on Android P platform.](https://issuetracker.google.com/issues/110321068)」，Google 工程师的回复是 persistent 应用作为系统的一部分，不应该被 kill，这是一种 policy，不是技术问题。

总结，Android 9.0 之前可以更新 persistent 应用，但是更新后会丢失 persistent flag，Android 9.0 及之后版本做了限制，无法更新 persistent 应用。

> **References**:
> - https://android-review.googlesource.com/c/platform/frameworks/base/+/495670
> - https://issuetracker.google.com/issues/110321068