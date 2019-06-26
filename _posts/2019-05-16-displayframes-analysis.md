---
layout: post
title: DisplayFrames 分析：Android 中都有哪些 Frame？
date: 2019-05-16
modified: 2019-06-27
tags: [Android,WindowManager]
categories: [Developer]
excerpt:  DisplayFrames 中包含了屏幕区域、应用区域、内容区域、安全显示区域等等。DisplayFrames 中用十几个 Rect 对象来表示这些 frame...
description: DisplayFrames 中包含了屏幕区域、应用区域、内容区域、安全显示区域等等。DisplayFrames 中用十几个 Rect 对象来表示这些 frame...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.github.io](xianzhu21.github.io)。

`mDisplayId` 是跟物理屏幕相关的，`DEFAULT_DISPLAY` 的值是 0，`mDisplayWidth` 是物理屏幕宽度，`mDisplayHeight` 是物理屏幕高度。

```java
/**
 * The current size of the screen; really; extends into the overscan area of the screen and
 * doesn't account for any system elements like the status bar.
 */
public final Rect mOverscan = new Rect();
/**
 * The current visible size of the screen; really; (ir)regardless of whether the status bar can
 * be hidden but not extending into the overscan area.
 */
public final Rect mUnrestricted = new Rect();
/** Like mOverscan*, but allowed to move into the overscan region where appropriate. */
public final Rect mRestrictedOverscan = new Rect();
/**
 * The current size of the screen; these may be different than (0,0)-(dw,dh) if the status bar
 * can't be hidden; in that case it effectively carves out that area of the display from all
 * other windows.
 */
public final Rect mRestricted = new Rect();
```

- `mOverscan` 是当前的屏幕大小，包括过扫描区域。过扫描区域在输出到 TV 时会用到，对于移动设备来说 mOverscan 大小就是物理设备的大小 `(0,0)-(dw,dh)`。关于 Overscan 可参考 [Android 官网](https://developer.android.com/training/tv/start/layouts#overscan)和[这篇博客](https://www.cnblogs.com/all-for-fiona/p/4054527.html)。
- `mUnrestricted` 是当前可见的屏幕大小，其实就是 `(0,0)-(dw,dh)`。
- 一般情况下 `mRestrictedOverscan` 与 `mRestricted` 相同，是应用可显示的区域，包含 StatusBar 的区域，不包含 NavigationBar 区域。

```java
/**
 * During layout, the current screen borders accounting for any currently visible system UI
 * elements.
 */
public final Rect mSystem = new Rect();

/** For applications requesting stable content insets, these are them. */
public final Rect mStable = new Rect();

/**
 * For applications requesting stable content insets but have also set the fullscreen window
 * flag, these are the stable dimensions without the status bar.
 */
public final Rect mStableFullscreen = new Rect();
```

- `mSystem` 是布局过程中，当前画面的边界，包含 Translucent（半透明）区域。一般情况下 NavigationBar 区域不是 Translucent，而 StatusBar 是 Translucent。
- `mStable` 是应用窗口的显示区域，不包含 StatusBar 和 NavigationBar。
- `mStableFullscreen` 是当 Activity 设置 Fullscreen flag 时候的窗口显示区域，这时 StatusBar 会隐藏。

```java
/**
 * During layout, the current screen borders with all outer decoration (status bar, input method
 * dock) accounted for.
 */
public final Rect mCurrent = new Rect();

/**
 * During layout, the frame in which content should be displayed to the user, accounting for all
 * screen decoration except for any space they deem as available for other content. This is
 * usually the same as mCurrent*, but may be larger if the screen decor has supplied content
 * insets.
 */
public final Rect mContent = new Rect();

/**
 * During layout, the frame in which voice content should be displayed to the user, accounting
 * for all screen decoration except for any space they deem as available for other content.
 */
public final Rect mVoiceContent = new Rect();

/** During layout, the current screen borders along which input method windows are placed. */
public final Rect mDock = new Rect();
```

- `mCurrent` 是布局的时候除去外部装饰的窗口（例如 StatusBar 和输入法窗口）。
- `mContent` 是当前应该给用户显示的窗口，通常与 `mCurrent` 相同。当装饰窗口提供内容插入的时候，有可能比 `mCurrent` 更大。
- `mVoiceContent` 通常与 `mContent` 相同。
- `mDock` 是输入法布局时的边界。

```java
/** The display cutout used for layout (after rotation) */
@NonNull public WmDisplayCutout mDisplayCutout = WmDisplayCutout.NO_CUTOUT;

/** The cutout as supplied by display info */
@NonNull public WmDisplayCutout mDisplayInfoCutout = WmDisplayCutout.NO_CUTOUT;

/**
 * During layout, the frame that is display-cutout safe, i.e. that does not intersect with it.
 */
public final Rect mDisplayCutoutSafe = new Rect();
```

- `mDisplayCutout` 和 `mDisplayInfoCutout` 都是用于刘海屏布局的剪刀工具，Android 9.0 新加入的。可参考「[Support display cutouts](https://developer.android.com/guide/topics/display-cutout)」和「[Display Cutouts](https://source.android.com/devices/tech/display/display-cutouts)」。
- `mDisplayCutoutSafe` 是在刘海屏上可以安全显示的区域，即这个区域与刘海区域没有交集。
