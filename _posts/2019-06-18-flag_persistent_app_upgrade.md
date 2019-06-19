---
layout: post
title: FLAG_PERSISTENT 应用是否可以更新
date: 2019-06-04
modified: 2019-06-18
tags: [Android,PackageManager]
categories: [I'm a coder]
excerpt:  Persistent flag 指的是 ApplicationInfo 类中的常量 FLAG_PERSISTENT (1<<3)。如果一个应用是 persistent 的，则永远在运行状态，即时被 kill 也会被重新启动。SystemUI 就是 persistent 的应用，可以想象如果 SystemUI 停止运行了...
description: Persistent flag 指的是 `ApplicationInfo` 类中的常量 `FLAG_PERSISTENT (1<<3)`。如果一个应用是 persistent 的，则永远在运行状态，即时被 kill 也会被重新启动。SystemUI 就是 persistent 的应用，可以想象如果 SystemUI 停止运行了...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.github.io](xianzhu21.github.io)。

# 0. Persistent flag 介绍

Persistent flag 指的是 `ApplicationInfo` 类中的常量 `FLAG_PERSISTENT (1<<3)`。如果一个应用是 persistent 的，则永远在运行状态，即使被 kill 也会被重新启动。SystemUI 就是 persistent 的应用，可以想象如果 SystemUI 停止运行了，StatusBar 和 NavigationBar 也就没有了，这肯定是不正常的。Persistent 属性如何设置，很简单，在 AndroidManifest.xml 的 application 标签中声明就可以。
```xml
<application
    android:name=".SystemUIApplication"
    android:persistent="true"
    ...>
    ...
</application>
```
所有应用安装后会在 `/data/system/packages.xml` 中更新相关信息，其中 publicFlag 就是该应用的所拥有的 flags，可以查看千位是否是 1 来判断该应用是否拥有 persistent 属性。

# 1. 应用的 Persistent flag 权限

不是所有应用只要在 AndroidManifest.xml 中声明 persistent 便可以拥有 persistent flag。Persistent flag 只有系统应用才会生效，而且是安装在 system 或 vendor 分区的系统应用。

下面分析应用的 persistent flag 是如何设置的。

`ApplicationInfo.FLAG_PERSISTENT` 是在 `PackageParser` 类的 `parseBaseApplication()` 方法中设置的。能否设置 `FLAG_PERSISTENT` 有两个条件，第一个是 `parseFlags` 中包含 `PARSE_IS_SYSTEM`，第二个是 AndroidManifest 中声明了 persistent。

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
        if ((flags&**PARSE_IS_SYSTEM**) != 0) {
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

`PARSE_IS_SYSTEM` 是 parseFlag 的一种。在 PackageManagerService 的构造方法中调用 `scanDirTracedLI()` 方法时根据扫描的目录不同会设置 `PARSE_IS_SYSTEM`。从代码中可以知道，当扫描以下目录时会设置 `PARSE_IS_SYSTEM`。 

```java
/vender/overlay
/system/framework
/system/priv-app
/system/app
/vendor/app
/oem/app
```

而不管是安装应用的流程还是扫描 `/data/app` 中的 APK，都不会设置 `PARSE_IS_SYSTEM`，所以普通应用是没有 persistent flag 权限的，即使在 AndroidManifest 中声明了也不会生效。

# 2. Persistent 应用是否可以更新？

更新指的是走安装流程，即 PackageManagerService 的 `installPackageLI()` 方法。在 Android 9.0 之前对于 persistent 应用的升级没有任何限制，但升级后的 APK 在 `/data/app` 中。从上文可知，扫描 `/data/app` 的 APK 时不会设置 `PARSE_IS_SYSTEM` flag，因此，persistent flag 也会失效。

原本 persistent 应用失去 persistent flag 肯定会受影响，以 SystemUI 为例，SystemUI 是 Service，并且在 SystemServer 中会调用 `startServiceAsUser()` 方法启动 SystemUI。在 Android 8.0 中加入了后台 Service 启动限制，参考 [Android 8.0 加入的「Background Service Limitations」](https://www.notion.so/efceffd6-6e85-4ea5-a200-3535e985478a)，对于没有 `ApplicationInfo.FLAG_SYSTEM` 和 `ApplicationInfo.FLAG_PERSISTENT` 的应用无法后台启动 Service。因此，SystemServer 是无法启动升级后的 SystemUI。

在 Android 9.0 之后的版本，对 persistent 应用的更新做了限制。如果要替换的应用是 persistent，则直接抛出异常，安装失败。

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    ...
    // Prevent persistent apps from being updated
    if ((oldPackage.applicationInfo.flags & ApplicationInfo.FLAG_PERSISTENT) != 0) {
    res.setError(PackageManager.INSTALL_FAILED_INVALID_APK,
            "Package " + oldPackage.packageName + " is a persistent app. "
                    + "Persistent apps are not updateable.");
        return;
    }
    ...
}
```

该 Git commit message 中说明了，persistent 应用无法用 adb 或其他 installer 更新，而且说明了如果更新了 persistent 应用，会丢失 persistent flag。

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

总结，Android 9.0 之前可以更新 persistent 应用，但是更新后会丢失 persistent flag，Android 9.0 之后系统做了限制，无法更新 persistent 应用。

> **References**:
> - https://android-review.googlesource.com/c/platform/frameworks/base/+/495670
> - https://issuetracker.google.com/issues/110321068