---
layout: post
title: NavigationBar 和 StatusBar 窗口大小与布局分析
date: 2020-02-21
modified: 2020-02-21
tags: [Android,SystemUI,WindowManager]
categories: [Developer]
excerpt:  NavigationBar 和 StatusBar 都属于 SystemBar，也叫做 decor，就是说给 App 装饰的意思。SystemBar 是在 beginLayoutLw() 方法中布局...
description: NavigationBar 和 StatusBar 都属于 SystemBar，也叫做 decor，就是说给 App 装饰的意思。SystemBar 是在 beginLayoutLw() 方法中布局...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

*基于 Android 9 源码*

* TOC
{:toc}

# 0. 概述

NavigationBar 和 StatusBar 都属于 SystemBar，也叫做 decor，就是说给 App 装饰的意思。一般的 window 的布局是在 PhoneWindowManager 的 `layoutWindowLw()` 方法中，而 SystemBar 是在 `beginLayoutLw()` 方法中布局。

当前最上层的 Activity 可以修改 SystemBar 的 visibility，可以调用 `View#setSystemUiVisibility()` 方法，系统也有一些针对 SystemBar visibility 的策略。最终的 visibility 保存在 PhoneWindowManager 中的 `mLastSystemUiFlags` 变量中。

# 1. NavigationBar 的布局

PhoneWindowManager 实现了 WindowManagerPolicy 接口。`beginLayoutLw()` 中会调用 `layoutNavigationBar()` 方法布局 NavigationBar。

`layoutNavigationBar()` 方法会先调用 `navigationBarPosition()` 方法返回 NavigationBar 的位置，根据不同的位置布局方式不同。以布局在下边为例，会计算 NavigationBar 的 top，然后更新 `mTmpNavigationFrame` 和 DisplayFrames 的相关属性。其中比较重要的是更新 `DisplayFrames.mDock` 变量，然后用这个变量设置 `mCurrent`、`mVoiceContent`、`mContent` 等变量。DisplayFrames 的各个属性在「[DisplayFrames 分析：Android 中都有哪些 Frame？](https://xianzhu21.space/developer/displayframes_analysis/)」有说明。

```java
// frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
public class PhoneWindowManager implements WindowManagerPolicy {
    private boolean layoutNavigationBar(DisplayFrames displayFrames, int uiMode, Rect dcf,
            boolean navVisible, boolean navTranslucent, boolean navAllowedHidden,
            boolean statusBarExpandedNotKeyguard) {
        if (mNavigationBar == null) {
            return false;
        }
        boolean transientNavBarShowing = mNavigationBarController.isTransientShowing();
        // Force the navigation bar to its appropriate place and size. We need to do this directly,
        // instead of relying on it to bubble up from the nav bar, because this needs to change
        // atomically with screen rotations.
        final int rotation = displayFrames.mRotation;
        final int displayHeight = displayFrames.mDisplayHeight;
        final int displayWidth = displayFrames.mDisplayWidth;
        final Rect dockFrame = displayFrames.mDock;
        mNavigationBarPosition = navigationBarPosition(displayWidth, displayHeight, rotation);

        if (mNavigationBarPosition == NAV_BAR_BOTTOM) {
            // It's a system nav bar or a portrait screen; nav bar goes on bottom.
            final int top = cutoutSafeUnrestricted.bottom
                    - getNavigationBarHeight(rotation, uiMode);
            mTmpNavigationFrame.set(0, top, displayWidth, displayFrames.mUnrestricted.bottom);
            displayFrames.mStable.bottom = displayFrames.mStableFullscreen.bottom = top;
            if (transientNavBarShowing) {
                mNavigationBarController.setBarShowingLw(true);
            } else if (navVisible) {
                mNavigationBarController.setBarShowingLw(true);
                dockFrame.bottom = displayFrames.mRestricted.bottom
                        = displayFrames.mRestrictedOverscan.bottom = top;
            } else {
                // We currently want to hide the navigation UI - unless we expanded the status bar.
                mNavigationBarController.setBarShowingLw(statusBarExpandedNotKeyguard);
            }
            if (navVisible && !navTranslucent && !navAllowedHidden
                    && !mNavigationBar.isAnimatingLw()
                    && !mNavigationBarController.wasRecentlyTranslucent()) {
                // If the opaque nav bar is currently requested to be visible and not in the process
                // of animating on or off, then we can tell the app that it is covered by it.
                displayFrames.mSystem.bottom = top;
            }
        } else if (mNavigationBarPosition == NAV_BAR_RIGHT) {
            // Landscape screen; nav bar goes to the right.
            ...
        } else if (mNavigationBarPosition == NAV_BAR_LEFT) {
            // Seascape screen; nav bar goes to the left.
            ...
        }

        // Make sure the content and current rectangles are updated to account for the restrictions
        // from the navigation bar.
        displayFrames.mCurrent.set(dockFrame);
        displayFrames.mVoiceContent.set(dockFrame);
        displayFrames.mContent.set(dockFrame);
        mStatusBarLayer = mNavigationBar.getSurfaceLayer();
        // And compute the final frame.
        mNavigationBar.computeFrameLw(mTmpNavigationFrame, mTmpNavigationFrame,
                mTmpNavigationFrame, displayFrames.mDisplayCutoutSafe, mTmpNavigationFrame, dcf,
                mTmpNavigationFrame, displayFrames.mDisplayCutoutSafe,
                displayFrames.mDisplayCutout, false /* parentFrameWasClippedByDisplayCutout */);
        mNavigationBarController.setContentFrame(mNavigationBar.getContentFrameLw());
        return mNavigationBarController.checkHiddenLw();
    }
}
```

NavigationBar 的 top 是 `cutoutSafeUnrestricted.bottom` 减去 NavigationBar 高度的结果。`cutoutSafeUnrestricted` 是安全的窗口（Android 9 针对刘海平新增的），当没有 Overscan 的时候与 `mUnrestricted` 相同，即 `cutoutSafeUnrestricted.bottom` 的值与 `DisplayFrames.mDisplayHeight` 值相同。计算 NavigationBar 的 top 后设置 `mTmpNavigationFrame`。`mTmpNavigationFrame` 就是 NavigationBar 的窗口区域。

之后更新 `mStable` 和 `mFullscreen` 的 bottom 值为 top。`mStable` 就是应用窗口的显示区域，这就是 NavigationBar 占用应用显示区域的原因。

如果 NavigationBar 可见的话更新 `dockFrame`、`mRestricted`、`mRestrictedOverscan` 的 bottom 值。然后用 dockFrame 去更新 `mCurrent`、`mVoiceContent`、`mContent`。

最后调用 `computeFrameLw()` 方法计算 NavigationBar 的 `contentFrame` 大小，然后绑定到 `mNavigationBarController`。

# 2. StatusBar 的布局

StatusBar 的布局在 PhoneWindowManager 的 `beginLayoutLw()` 方法中，调用 `layoutStatusBar()` 方法。

```java
// frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
public class PhoneWindowManager implements WindowManagerPolicy {
    @Override
    public void beginLayoutLw(DisplayFrames displayFrames, int uiMode) {
        boolean updateSysUiVisibility |= layoutNavigationBar(displayFrames, uiMode, dcf,
                    navVisible, navTranslucent, navAllowedHidden, statusBarExpandedNotKeyguard);
        updateSysUiVisibility |= layoutStatusBar(
                    displayFrames, pf, df, of, vf, dcf, sysui, isKeyguardShowing);
        if (updateSysUiVisibility) {
            updateSystemUiVisibilityLw();
        }
    }

    private boolean layoutStatusBar(DisplayFrames displayFrames, Rect pf, Rect df, Rect of, Rect vf,
            Rect dcf, int sysui, boolean isKeyguardShowing) {
        // decide where the status bar goes ahead of time
        if (mStatusBar == null) {
            return false;
        }
        // apply any navigation bar insets
        of.set(displayFrames.mUnrestricted);
        df.set(displayFrames.mUnrestricted);
        pf.set(displayFrames.mUnrestricted);
        vf.set(displayFrames.mStable);

        mStatusBarLayer = mStatusBar.getSurfaceLayer();

        // Let the status bar determine its size.
        mStatusBar.computeFrameLw(pf /* parentFrame */, df /* displayFrame */,
                vf /* overlayFrame */, vf /* contentFrame */, vf /* visibleFrame */,
                dcf /* decorFrame */, vf /* stableFrame */, vf /* outsetFrame */,
                displayFrames.mDisplayCutout, false /* parentFrameWasClippedByDisplayCutout */);

        // For layout, the status bar is always at the top with our fixed height.
        displayFrames.mStable.top = displayFrames.mUnrestricted.top
                + mStatusBarHeightForRotation[displayFrames.mRotation];
        // Make sure the status bar covers the entire cutout height
        displayFrames.mStable.top = Math.max(displayFrames.mStable.top,
                displayFrames.mDisplayCutoutSafe.top);

        // Tell the bar controller where the collapsed status bar content is
        mTmpRect.set(mStatusBar.getContentFrameLw());
        mTmpRect.intersect(displayFrames.mDisplayCutoutSafe);
        mTmpRect.top = mStatusBar.getContentFrameLw().top;  // Ignore top display cutout inset
        mTmpRect.bottom = displayFrames.mStable.top;  // Use collapsed status bar size
        mStatusBarController.setContentFrame(mTmpRect);

        boolean statusBarTransient = (sysui & View.STATUS_BAR_TRANSIENT) != 0;
        boolean statusBarTranslucent = (sysui
                & (View.STATUS_BAR_TRANSLUCENT | View.STATUS_BAR_TRANSPARENT)) != 0;
        if (!isKeyguardShowing) {
            statusBarTranslucent &= areTranslucentBarsAllowed();
        }

        // If the status bar is hidden, we don't want to cause windows behind it to scroll.
        if (mStatusBar.isVisibleLw() && !statusBarTransient) {
            // Status bar may go away, so the screen area it occupies is available to apps but just
            // covering them when the status bar is visible.
            final Rect dockFrame = displayFrames.mDock;
            dockFrame.top = displayFrames.mStable.top;
            displayFrames.mContent.set(dockFrame);
            displayFrames.mVoiceContent.set(dockFrame);
            displayFrames.mCurrent.set(dockFrame);

            if (DEBUG_LAYOUT) Slog.v(TAG, "Status bar: " + String.format(
                    "dock=%s content=%s cur=%s", dockFrame.toString(),
                    displayFrames.mContent.toString(), displayFrames.mCurrent.toString()));

            if (!mStatusBar.isAnimatingLw() && !statusBarTranslucent
                    && !mStatusBarController.wasRecentlyTranslucent()) {
                // If the opaque status bar is currently requested to be visible, and not in the
                // process of animating on or off, then we can tell the app that it is covered by it.
                displayFrames.mSystem.top = displayFrames.mStable.top;
            }
        }
        return mStatusBarController.checkHiddenLw();
}}
```

首先设定 `of /* overscanFrame */`、`df /* displayFrame */`、`pf /* parentFrame */`、`vf /* visibleFrame */`，其中 `of`、`df`、`pf` 设定为 `displayFrames.mUnrestricted`，即屏幕大小。`vf` 设定为 `displayFrames.mStable`，`mStable` 的大小在调用 `layoutNavigationBar()` 方法后变成了除去 NavigationBar 的窗口。然后调用 `mStatusBar.computeFrameLw()` 方法计算 StatusBar 的 `mContentFrame` 大小。`mContentFrame` 在 IME 不存在时与 `mDecorFrame` 相同，IME 存在时是 `mDecorFrame` 除去 IME 窗口的大小。

```java
    // apply any navigation bar insets
    of.set(displayFrames.mUnrestricted);
    df.set(displayFrames.mUnrestricted);
    pf.set(displayFrames.mUnrestricted);
    vf.set(displayFrames.mStable);
    
    mStatusBarLayer = mStatusBar.getSurfaceLayer();
    
    // Let the status bar determine its size.
    mStatusBar.computeFrameLw(pf /* parentFrame */, df /* displayFrame */,
            vf /* overlayFrame */, vf /* contentFrame */, vf /* visibleFrame */,
            dcf /* decorFrame */, vf /* stableFrame */, vf /* outsetFrame */,
            displayFrames.mDisplayCutout, false /* parentFrameWasClippedByDisplayCutout */);
```

下一步修改 `displayFrames.mStable`，即应用的窗口。因为 StatusBar 默认显示在顶部的，所以修改 `mStable.top` 值。

```java
    // For layout, the status bar is always at the top with our fixed height.
    displayFrames.mStable.top = displayFrames.mUnrestricted.top
            + mStatusBarHeightForRotation[displayFrames.mRotation];
    // Make sure the status bar covers the entire cutout height
    displayFrames.mStable.top = Math.max(displayFrames.mStable.top,
            displayFrames.mDisplayCutoutSafe.top);
```

计算 StatusBar 的可显示区域，绑定到 `mStatusBarController`。

```java
    // Tell the bar controller where the collapsed status bar content is
    mTmpRect.set(mStatusBar.getContentFrameLw());
    mTmpRect.intersect(displayFrames.mDisplayCutoutSafe);
    mTmpRect.top = mStatusBar.getContentFrameLw().top;  // Ignore top display cutout inset
    mTmpRect.bottom = displayFrames.mStable.top;  // Use collapsed status bar size
    mStatusBarController.setContentFrame(mTmpRect);
```

判断 StatusBar 是否是短暂显示的（Transient）或是半透明的（Translucent）。其中 `sysui` 就是 `mLastSystemUiFlags`。

```java
    boolean statusBarTransient = (sysui & View.STATUS_BAR_TRANSIENT) != 0;
    boolean statusBarTranslucent = (sysui
            & (View.STATUS_BAR_TRANSLUCENT | View.STATUS_BAR_TRANSPARENT)) != 0;
    if (!isKeyguardShowing) {
        statusBarTranslucent &= areTranslucentBarsAllowed();
    }
```

如果 StatusBar 是可见的，且不是暂时显示的，则修改 displayFrames 的 `mDock`、`mContent`、`mVoiceContent`、`mCurrent`，其实就是除去 StatusBar 的窗口大小。还有，如果 StatusBar 是不透明的，则修改 `mSystem.top` 值为 `mStable.top` 值，这时 Activity 使用 `@android:style/Theme.NoTitleBar.Fullscreen` 也不会隐藏 StatusBar。

```java
    // If the status bar is hidden, we don't want to cause windows behind it to scroll.
    if (mStatusBar.isVisibleLw() && !statusBarTransient) {
        // Status bar may go away, so the screen area it occupies is available to apps but just
        // covering them when the status bar is visible.
        final Rect dockFrame = displayFrames.mDock;
        dockFrame.top = displayFrames.mStable.top;
        displayFrames.mContent.set(dockFrame);
        displayFrames.mVoiceContent.set(dockFrame);
        displayFrames.mCurrent.set(dockFrame);
    
        if (DEBUG_LAYOUT) Slog.v(TAG, "Status bar: " + String.format(
                "dock=%s content=%s cur=%s", dockFrame.toString(),
                displayFrames.mContent.toString(), displayFrames.mCurrent.toString()));
    
        if (!mStatusBar.isAnimatingLw() && !statusBarTranslucent
                && !mStatusBarController.wasRecentlyTranslucent()) {
            // If the opaque status bar is currently requested to be visible, and not in the
            // process of animating on or off, then we can tell the app that it is covered by it.
            displayFrames.mSystem.top = displayFrames.mStable.top;
        }
    }
```