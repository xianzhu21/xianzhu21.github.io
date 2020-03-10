---
layout: post
title: Dropping event bug in Dialog
date: 2020-03-10
modified: 2020-03-10
tags: [Android, Bug, Dialog, InputManager, WindowManager]
categories: [Developer]
excerpt:  某个 App 启动后显示用户协议相关 Dialog。这时不相应 touch event。复现后，回到 Home，重新进入也不恢复。
description: 某个 App 启动后显示用户协议相关 Dialog。这时不相应 touch event。复现后，回到 Home，重新进入也不恢复。
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

* TOC
{:toc}

# Bug description

某个 App 启动后显示用户协议相关 Dialog。这时不相应 touch event。复现后，回到 Home，重新进入也不恢复。

概率性发生的。重启后概率性恢复。

相关 log：

```shell
2713  2713 W ViewRootImpl[MainActivity]: Dropping event due to no window focus: MotionEvent { action=ACTION_DOWN, actionButton=0, id[0]=0, x[0]=934.4269, y[0]=460.23578, toolType[0]=TOOL_TYPE_FINGER, buttonState=0, metaState=0, flags=0x0, edgeFlags=0x0, pointerCount=1, historySize=0, eventTime=89426937, downTime=89426937, deviceId=4, source=0x1002 }
```

# Recurrence step

这是跑 monkey 的时候复现的。终端是个双屏终端。手动复现步骤如下：

- 点击副屏，focus 在副屏。
- 点击主屏的 X App。
- 用非常快的速度点击副屏，再次点击主屏的另一个应用，例如 fmradio。
- 回到 Home，再次进入 X App。
- 这时的 Dialog 已经无法相应 touch event 了。

# Root cause

直接原因是 ViewRootImpl 对象的 mStopped 字段为 true。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
protected boolean shouldDropInputEvent(QueuedInputEvent q) {
    if (mView == null || !mAdded) {
        Slog.w(mTag, "Dropping event due to root view being removed: " + q.mEvent);
        return true;
    } else if ((!mAttachInfo.mHasWindowFocus
                && !q.mEvent.isFromSource(InputDevice.SOURCE_CLASS_POINTER)
            || mStopped
            || (mIsAmbientMode && !q.mEvent.isFromSource(InputDevice.SOURCE_CLASS_BUTTON))
            || (mPausedForTransition && !isBack(q.mEvent))) {
        // Drop non-terminal input events.
        Slog.w(mTag, "Dropping event due to no window focus: " + q.mEvent);
        return true;
    }
    return false;
}
```

mStopped 表示相关 Window 的状态，如果是 true 则表明 inactive，所以会 drop event。而 mStopped 就是在 Activity 的 `performRestart()` 和 `performStop()` 方法中改变的。

```java
// frameworks/base/core/java/android/app/Activity.java
final void performRestart() {
    if (mToken != null && mParent == null) {
        // No need to check mStopped, the roots will check if they were actually stopped.
        WindowManagerGlobal.getInstance().setStoppedState(mToken, false /* stopped */);
}}
final void performStop(boolean preserveWindow) {
    // If we're preserving the window, don't setStoppedState to true, since we
    // need the window started immediately again. Stopping the window will
    // destroys hardware resources and causes flicker.
    if (!preserveWindow && mToken != null && mParent == null) {
        WindowManagerGlobal.getInstance().setStoppedState(mToken, true);
}}

// frameworks/base/core/java/android/view/WindowManagerGlobal.java
public void setStoppedState(IBinder token, boolean stopped) {
    synchronized (mLock) {
        int count = mViews.size();
        for (int i = 0; i < count; i++) {
            if (token == null || mParams.get(i).token == token) {
                ViewRootImpl root = mRoots.get(i);
                root.setWindowStopped(stopped);
}}}}

// frameworks/base/core/java/android/view/ViewRootImpl.java
// Set to true if the owner of this window is in the stopped state,
// so the window should no longer be active.
boolean mStopped = false;
void setWindowStopped(boolean stopped) {
    if (mStopped != stopped) {
        mStopped = stopped;
        ...
}}
```

注意 `setStoppedState()` 方法的第一个参数是 token，这是 WindowManager.LayoutParams 类的 token 字段。用 token 来表示改变 mStopped 的是哪一个 Window。这里会遍历所有 ViewRootImpl，如果 token 相同则调用 `setWindowStopped()` 方法。

接下来看一下，这个 token 是怎么来的。对于一个 dialog window，它的 token 是在 `addView()` 时赋值的，其实就是启动该 dialog 的 activity window 的 mAppToken。最终是在 `adjustLayoutParamsForSubWindow()` 方法中赋值的。

```java
// frameworks/base/core/java/android/view/WindowManagerImpl.java
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
// frameworks/base/core/java/android/view/WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
    }
    ViewRootImpl root;
    root = new ViewRootImpl(view.getContext(), display);
    view.setLayoutParams(wparams);
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);

    root.setView(view, wparams, panelParentView);
}

// frameworks/base/core/java/android/view/Window.java
void adjustLayoutParamsForSubWindow(WindowManager.LayoutParams wp) {
    if (wp.token == null) {
        wp.token = mContainer == null ? mAppToken : mContainer.mAppToken;
}}
```

已经知道了 dialog window 的 token 就是 show dialog 时的 mAppToken。而在 `addView()` 方法中会创建 ViewRootImpl 对象，然后会 add 到 mRoots 中，wparams 会 add 到 mParams。这说明所有的 window 都存在 mRoots 中，且它的配置存在 mParams。

重新看下 `setStoppedState()` 方法，这里遍历的是 mParams 变量，然后找到与参数 token 匹配的  ViewRootImpl 对象后调用 `setWindowStopped()` 方法。因为 dialog token 就是 mAppToken，所以 dialog 的 mStopped 值应该和 activity 的 mStopped 值相同。

这是正常逻辑，但在我们的 case 中没有改变 mStopped 值，说明 dialog token 变了，因此在 `performRestart()` 时也没能改变 dialog 的 mStopped 值。

上面分析了，加入到 mParams 时的 LayoutParams 变量中 token 正确的，那我们看看这个变量是从哪来的。Dialog 类的 `show()` 方法中调用 `addView()` 时传进去的第二个参数 `l` 就是这个 LayoutParams 变量。

```java
// Android 8.1 or earlier version.
// frameworks/base/core/java/android/app/Dialog.java
public void show() {
    WindowManager.LayoutParams l = mWindow.getAttributes();
    if ((l.softInputMode
            & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
        WindowManager.LayoutParams nl = new WindowManager.LayoutParams();
        nl.copyFrom(l);
        nl.softInputMode |=
                WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
        l = nl;
    }
    mWindowManager.addView(mDecor, l);
}

// frameworks/base/core/java/android/view/Window.java
public final WindowManager.LayoutParams getAttributes() {
    return mWindowAttributes;
}
```

但调用 `addView()` 方法前有个 if 语句，会做一些 softInputMode 字段相关的处理。这里它新建一个 LayoutParams 变量 `nl`，处理完后赋给了 `l` 变量。就是说，本来 `l` 是 `mWindow.getAttributes()` 返回的 mWindowAttributes，但是在调用 `addView()` 前被赋值成了 `nl`。

这导致了，给 token 赋值的 LayoutParams 对象是 `nl`，而不是 mWindowAttributes。mWindowAttributes 的 token 至始至终没有被赋值。这是问题的根本原因。

那什么时候会修改 mParams 中 dialog 对应的 LayoutParams 变量？在 activity resume 的时候会调用 `updateViewLayout()` 方法。这个方法中的第二个参数 params 就是 Window 类的 mWindowAttributes 变量。从方法实现中可以看到，就是先 remove 然后再 add，就是直接替换了。这样在 mParams 中 dialog 对应的变量是 mWindowAttributes，而它的 token 一直是 null。

```java
// frameworks/base/core/java/android/view/WindowManagerGlobal.java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
    view.setLayoutParams(wparams);
    synchronized (mLock) {
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        mParams.add(index, wparams);
        root.setLayoutParams(wparams, false);
}}
```

这就是出现问题的点。在调用 `updateViewLayout()` 时用 mWindowAttributes（token==null）替换了在 `addView()` 方法中添加的 `nl`（token==mAppToken）。所以之后调用 `setStoppedState()` 时找不到 mAppToken 相同的 token，也就不会进入 if 语句，这就导致了不会调用 ViewRootImpl 的 `setWindowStopped()` 方法。最终结果是 ViewRootImpl 中的 mStopped 变量一直不会改变。

那为什么是概率出现的。双屏的 Activity 生命周期成为了触发点。当跟着复现步骤操作的时候，system_server 会在 activity resume 之前调用 `stopActivityLocked()`，进而调用 `performStop()` 的时机在 `updateViewLayout()` 之前。这导致了 mStopped 为 true，然后紧接着 `updateViewLayout()` 方法中 mWindowAttributes 替换了 `nl`。之后的 `setStoppedState()` 不会再改变 mStopped 值。

而正常的时候，`updateViewLayout()` 是在 `stopActivityLocked()` 之前调用了，所以 mStopped 一直是 `false`，从来没有改变过。虽然，之后的 `setStoppedState()` 也不能改变 mStopped 值，但它一直是 `false`，所以也就没有问题了。

# Solution

解决方法很简单，就是在 Dialog 类的 `show()` 方法中不要新建 LayoutParams 对象，直接用 mWindowAttribute，这样 mAppToken 会直接赋值给 mWindowAttribute 的 token 字段。

其实 Google 已经在 Android 9.0 中修复了该问题，而且在 commit message 中写了一大段文字说明该 Bug。

```diff
commit 15d403dd6a3b780141530f1f44185256fd3f4aed
Author: Felipe Leme <felipeal@google.com>
Date:   Thu Mar 29 10:02:32 2018 -0700

    Don't use a copy of window params when showing a dialog.
    
    When a Dialog's show() method is called, it makes a copy (l) of its window param
    and change the copy's softInputMode before calling wm.addView(). This call ends
    up calling WindowManagerGlobal.addView(view, l, display, parentWindow),
    which in turn sets the application token from the parentWindow into l and stores
    l on its mParams map.
    
    Later, when the dialog layout is changed (for example, if it's resized), the
    original params ends up passed to WindowManagerGlobal.updateViewLayout(),
    which in turn updates it's internal mParams with it, hence losing the
    application token (as the token was set in the copy).
    
    Then, when Autofill (and possibly Assist) is triggered to that activity, the
    Dialog's view hierarchy is ignored because WindowManagerGlobal.getRootViews()
    ignores views whose params do not have an application token.
    
    This CL fixes this issue by passing the original dialog's param to the wm
    method and resetting the softInputMode that was changed, rather than making a
    copy.
    
    Test: atest DialogLauncherActivityTest
    Test: manual verification with Twitch
    
    Fixes: 68816440
    
    Change-Id: I55f510ab7a44030bc368221b7db1a221bc2e09c8

diff --git a/core/java/android/app/Dialog.java b/core/java/android/app/Dialog.java
index 4a168fe..e4a0583 100644
--- a/core/java/android/app/Dialog.java
+++ b/core/java/android/app/Dialog.java
@@ -321,16 +321,20 @@ public class Dialog implements DialogInterface, Window.Callback,
            }
    
            WindowManager.LayoutParams l = mWindow.getAttributes();
+        boolean restoreSoftInputMode = false;
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
-            WindowManager.LayoutParams nl = new WindowManager.LayoutParams();
-            nl.copyFrom(l);
-            nl.softInputMode |=
+            l.softInputMode |=
                        WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
-            l = nl;
+            restoreSoftInputMode = true;
            }
    
            mWindowManager.addView(mDecor, l);
+        if (restoreSoftInputMode) {
+            l.softInputMode &=
+                    ~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
+        }
+
            mShowing = true;
    
            sendShowMessage();
```