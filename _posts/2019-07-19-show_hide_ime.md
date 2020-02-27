---
layout: post
title: 输入法的弹出与收起及 FLAG_ALT_FOCUSABLE_IM 的影响
date: 2019-07-19
modified: 2019-07-31
tags: [Android,InputMethodManager,WindowManager]
categories: [Developer]
excerpt: 输入法弹出入口有 TextView、SearchView、NumberPicker 等。这几个控件都是调用 InputMethodManager 的 showSoftInput() 方法弹出的...
description: 输入法弹出入口有 TextView、SearchView、NumberPicker 等。这几个控件都是调用 InputMethodManager 的 showSoftInput() 方法弹出的...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

* TOC
{:toc}

# 1. 输入法的弹出

## 1.1 InputMethodManager.showSoftInput()

输入法弹出入口有 `TextView`、`SearchView`、`NumberPicker` 等。这几个控件都是调用 `InputMethodManager` 的 `showSoftInput()` 方法弹出的。 

`TextView` 的 `onTouchView()` 方法中，如果满足情况会调用 `showSoftInput()` 方法。

```java
if (touchIsFinished && (isTextEditable() || textIsSelectable)) {
    // Show the IME, except when selecting in read-only text.
    final InputMethodManager imm = InputMethodManager.peekInstance();
    viewClicked(imm);
    if (isTextEditable() && mEditor.mShowSoftInputOnFocus && imm != null) {
        imm.showSoftInput(this, 0);
    }
}
```

## 1.2 InputMethodManager.mServiceView

`showSoftInput()` 方法中 `mServedView` 和传入的 `View` 对象要一致，否则返回 false。`mServedView` 指的是输入法正在服务的 `View` 对象，其实就是接收输入法输入操作的 `View` 对象

最终会调用 `InputMethodManagerService` 的 `showSoftInput()` 方法弹出输入法窗口。

```java
public boolean showSoftInput(View view, int flags, ResultReceiver resultReceiver) {
    checkFocus();
    synchronized (mH) {
        if (mServedView != view && (mServedView == null
                || !mServedView.checkInputConnectionProxy(view))) {
            return false;
        }
        try {
            return mService.showSoftInput(mClient, flags, resultReceiver);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
}}}
```

在 `checkFocusNoStartInput()` 方法中把 `mNextServedView` 赋值给了 `mServedView`。`mNextServedView` 是在 `focusInLocked()` 方法中赋值的。

```java
private boolean checkFocusNoStartInput(boolean forceNewFocus) {
    // This is called a lot, so short-circuit before locking.
    if (mServedView == mNextServedView && !forceNewFocus) {
        return false;
    }
    synchronized (mH) {
        if (mServedView == mNextServedView && !forceNewFocus) {
            return false;
        }
        mServedView = mNextServedView;
    }
    return true;
}

void focusInLocked(View view) {
    mNextServedView = view;
    scheduleCheckFocusLocked(view);
}
```

`focusInLocked()` 方法中最后调用的 `scheduleCheckFocusLocked()` 方法会调用 `ViewRootIml` 对象的 `dispatchCheckFocus()` 方法，而这个方法会向 `Handler` 发送消息，最终会调用 `InputMethodManager` 的 `checkFocus()` 方法。在 `checkFocus()` 方法中会调用上面的提到的 `checkFocusNoStartInput()` 方法，就会把获取焦点的 `View` 对象设置为 `mServedView` 了。

{% highlight java %}
// android.view.inputmethod.InputMethodManager
static void scheduleCheckFocusLocked(View view) {
    ViewRootImpl viewRootImpl = view.getViewRootImpl();
    if (viewRootImpl != null) {
        viewRootImpl.dispatchCheckFocus();
}}

// android.view.ViewRootImpl
public void dispatchCheckFocus() {
    if (!mHandler.hasMessages(MSG_CHECK_FOCUS)) {
        // This will result in a call to checkFocus() below.
        mHandler.sendEmptyMessage(MSG_CHECK_FOCUS);
}}

final class ViewRootHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        case MSG_CHECK_FOCUS: {
            InputMethodManager imm = InputMethodManager.peekInstance();
            if (imm != null) {
                imm.checkFocus();
            }
        } break;
}}
{% endhighlight %}

如果 `checkFocusNoStartInput()` 方法返回 true，则调用 `startInputInner()` 方法，该方法会建立 `InputConnection` 把输入法和 `View` 绑定起来。

```java
// android.view.inputmethod.InputMethodManager
public void checkFocus() {
    if (checkFocusNoStartInput(false)) {
        startInputInner(InputMethodClient.START_INPUT_REASON_CHECK_FOCUS, null, 0, 0, 0);
}}
```

## 1.3 InputMethodManager.focusInLocked()

上面提到调用 `focusInLocked()` 方法后，最终会把参数 `View` 对象赋值给 `mServedView`。该方法有两个地方调用，一个是 `focusIn()` 方法，一个是 `onPostWindowFocus()` 方法。

### 1.3.1 InputMethodManager.focusIn()

`focusIn()` 方法是在 `View` 中在四种情况下会调用。

1. `Window` 已经获得了焦点的情况下 `View` 获得焦点。
2. 暂时的 detach 结束后。
3. 当 `View` 获得焦点的情况下 `Window` 重新获得焦点。
4. 当 `View` attach 到 `Window` 时获得焦点。

```java
// android.view.View
/**
 * Called by the view system when the focus state of this view changes.
 */
protected void onFocusChanged(boolean gainFocus, @FocusDirection int direction,
        @Nullable Rect previouslyFocusedRect) {
    InputMethodManager imm = InputMethodManager.peekInstance();
    if (!gainFocus) {
        ...
    } else if (imm != null && mAttachInfo != null && mAttachInfo.mHasWindowFocus) {
        imm.focusIn(this);
}}

/**
 * Dispatch {@link #onFinishTemporaryDetach()} to this View and its direct children if this is
 * a container View.
 */
public void dispatchFinishTemporaryDetach() {
    mPrivateFlags3 &= ~PFLAG3_TEMPORARY_DETACH;
    onFinishTemporaryDetach();
    if (hasWindowFocus() && hasFocus()) {
        InputMethodManager.getInstance().focusIn(this);
}}

public void onWindowFocusChanged(boolean hasWindowFocus) {
    InputMethodManager imm = InputMethodManager.peekInstance();
    if (!hasWindowFocus) {
        ...
    } else if (imm != null && (mPrivateFlags & PFLAG_FOCUSED) != 0) {
        imm.focusIn(this);
}}

protected void onAttachedToWindow() {
    if (isFocused()) {
        InputMethodManager imm = InputMethodManager.peekInstance();
        if (imm != null) {
            imm.focusIn(this);
}}}
```

### 1.3.2 InputManagerMethod.onPostWindowFocus()

`onPostWindowFocus()` 方法是在 `Window` 获得焦点的时候被调用，是在 `ViewImplRoot` 的 `performTraversals()` 方法和 `handleWindowFocusChanged()` 方法中当获得焦点的时候调用。

# 2. 输入法的收起

输入法的收起与弹出类似，最终是调用 InputMethodManagerService 的 `hideSoftInput()` 方法收起的。而调用该方法的有两处，一个是 `closeCurrentInput()` 方法，一个是 `hideSoftInputFromWindow()` 方法。

这两个方法都是 Client 主动调用的，还有一种情况是，当一个新的 `Window` 获得焦点的时候输入法会自动收起来的。这个机制的目的是新弹出的 `Window` 也有可能需要使用输入法，所以需要先收起来，然后使用的时候再弹出来。

## 2.1 InputMethodManager.closeCurrentInput()

`closeCurentInput()` 方法的调用有两处，一个是在 `startInputInner()` 方法中如果 `mServedView` 的 `getHandler()` 返回 null，则调用 `closeCurrentInput()` 方法，属于异常情况。

```java
boolean startInputInner(@InputMethodClient.StartInputReason final int startInputReason,
        IBinder windowGainingFocus, int controlFlags, int softInputMode,
        int windowFlags) {
    final View view;
    synchronized (mH) {
        view = mServedView;
    }

    // Now we need to get an input connection from the served view.
    // This is complicated in a couple ways: we can't be holding our lock
    // when calling out to the view, and we need to make sure we call into
    // the view on the same thread that is driving its view hierarchy.
    Handler vh = view.getHandler();
    if (vh == null) {
        // If the view doesn't have a handler, something has changed out
        // from under us, so just close the current input.
        // If we don't close the current input, the current input method can remain on the
        // screen without a connection.
        if (DEBUG) Log.v(TAG, "ABORT input: no handler for view! Close current input.");
        closeCurrentInput();
        return false;
}}
```

另一个是 `checkFocusNoStartInput()` 方法中当 `mNextServedView == null` 时调用。

```java
private boolean checkFocusNoStartInput(boolean forceNewFocus) {
    synchronized (mH) {
        if (mNextServedView == null) {
            finishInputLocked();
            // In this case, we used to have a focused view on the window,
            // but no longer do.  We should make sure the input method is
            // no longer shown, since it serves no purpose.
            closeCurrentInput();
            return false;
}}
```

那什么时候 `mNextServedView` 会赋值为 null。这也有两种情况，一个是当 `removeView()` 的时候调用 `InputMethodManager` 的 `windowDismissed()` 方法，它又调用 `finishInputLocked()` 方法，在这个方法中会给 `mNextServedView` 赋值为 null。这个时候焦点会发生改变，进而调用 `checkFocusNoStartInput()` 方法，如果这时没有获得焦点的 `View`，则 `mNextServedView` 始终为 null，所以最终会调用 `closeCurrentInput()`。

还有一种情况，当 `View` 从 `Window` detach 的时候，即调用 `onViewDetachedFromWindow()` 方法时会把 `mNextServedView` 会赋值为 null，然后调用 `scheduleCheckFocusLocked()` 方法触发收起输入法。

## 2.2 InputMethodManager.hideSoftInputFromWindow

`hideSoftInputFromWindow()` 方法是 public 的，需要传入参数 `IBinder` 对象 `windowToken`。例如，TextView 的 `setEnable()` 方法中，如果 `enable` 为 false，则会调用该方法。

```java
// android.widget.TextView
public void setEnabled(boolean enabled) {
    if (enabled == isEnabled()) {
        return;
    }

    if (!enabled) {
        // Hide the soft input if the currently active TextView is disabled
        InputMethodManager imm = InputMethodManager.peekInstance();
        if (imm != null && imm.isActive(this)) {
            imm.hideSoftInputFromWindow(getWindowToken(), 0);
}}}

// android.view.inputmethod.InputMethodManager
public boolean hideSoftInputFromWindow(IBinder windowToken, int flags) {
    return hideSoftInputFromWindow(windowToken, flags, null);
}
public boolean hideSoftInputFromWindow(IBinder windowToken, int flags,
        ResultReceiver resultReceiver) {
    checkFocus();
    synchronized (mH) {
        if (mServedView == null || mServedView.getWindowToken() != windowToken) {
            return false;
        }
        try {
            return mService.hideSoftInput(mClient, flags, resultReceiver);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
}}}
```

## 2.3 新的 Window 获得焦点时的输入法收起

当一个新的 `Window` 获得焦点的时候 `InputMethodManager` 的 `onPostWindowFocus()` 方法被调用，在 1.3.2 中也有说明。在该方法中也会调用 `startInputInner()` 方法。

```java
// android.view.inputmethod.InputMethodManager
/**
    * Called by ViewAncestor when its window gets input focus.
    * @hide
    */
public void onPostWindowFocus(View rootView, View focusedView,
        @SoftInputModeFlags int softInputMode, boolean first, int windowFlags) {
    boolean forceNewFocus = false;
    synchronized (mH) {
        focusInLocked(focusedView != null ? focusedView : rootView);
    }
    if (checkFocusNoStartInput(forceNewFocus)) {
        // We need to restart input on the current focus view.  This
        // should be done in conjunction with telling the system service
        // about the window gaining focus, to help make the transition
        // smooth.
        if (startInputInner(InputMethodClient.START_INPUT_REASON_WINDOW_FOCUS_GAIN,
                rootView.getWindowToken(), controlFlags, softInputMode, windowFlags)) {
            return;
}}}
```

`startInputInner()` 方法比较长，在上面也说明了，这个方法主要是绑定输入法和 `View`。在这个方法中会调用 `InputMethodManagerService` 的 `startInputOrWindowGainedFocus()` 方法绑定输入法。

```java
// android.view.inputmethod.InputMethodManager
boolean startInputInner(@InputMethodClient.StartInputReason final int startInputReason,
        IBinder windowGainingFocus, int controlFlags, int softInputMode,
        int windowFlags) {
    ...
    synchronized (mH) {
        try {
            if (DEBUG) Log.v(TAG, "START INPUT: view=" + dumpViewInfo(view) + " ic="
                    + ic + " tba=" + tba + " controlFlags=#"
                    + Integer.toHexString(controlFlags));
            final InputBindResult res = mService.startInputOrWindowGainedFocus(
                    startInputReason, mClient, windowGainingFocus, controlFlags, softInputMode,
                    windowFlags, tba, servedContext, missingMethodFlags,
                    view.getContext().getApplicationInfo().targetSdkVersion);
}}}
```

`startInputOrWindowGainedFocus()` 方法名中可看到，它有两种情况。看代码可知，当 `windowToken` 不为 null 的时候会调用 `windowGainedFocus()` 方法，否则调用 `startInput()` 方法。

```java
// com.android.server.InputMethodManagerService
public InputBindResult startInputOrWindowGainedFocus(
        /* @InputMethodClient.StartInputReason */ final int startInputReason,
        IInputMethodClient client, IBinder windowToken, int controlFlags, int softInputMode,
        int windowFlags, @Nullable EditorInfo attribute, IInputContext inputContext,
        /* @InputConnectionInspector.missingMethods */ final int missingMethods,
        int unverifiedTargetSdkVersion) {
    final InputBindResult result;
    if (windowToken != null) {
        result = windowGainedFocus(startInputReason, client, windowToken, controlFlags,
                softInputMode, windowFlags, attribute, inputContext, missingMethods,
                unverifiedTargetSdkVersion);
    } else {
        result = startInput(startInputReason, client, inputContext, missingMethods, attribute,
                controlFlags);
    }
    return result;
}
```

继续看 `Window` 获得焦点情况，即 `windowGainedFocus()` 方法。其中 `startInputMode` 参数是 `Window` 要求的输入法状态，如果没有指定则是 `SOFT_INPUT_STATE_UNSPECIFIED`。然后如果 `isTextEditor` 为 false 或 `doAutoShow` 为 false，则继续判断该 `Window` 是否可能使用输入法，它调用 `mayUseInputMethod()` 方法判断。

```java
// com.android.server.InputMethodManagerService
private InputBindResult windowGainedFocus(
        /* @InputMethodClient.StartInputReason */ final int startInputReason,
        IInputMethodClient client, IBinder windowToken, int controlFlags,
        /* @android.view.WindowManager.LayoutParams.SoftInputModeFlags */ int softInputMode,
        int windowFlags, EditorInfo attribute, IInputContext inputContext,
        /* @InputConnectionInspector.missingMethods */  final int missingMethods,
        int unverifiedTargetSdkVersion) {
    try {
        synchronized (mMethodMap) {
            switch (softInputMode&WindowManager.LayoutParams.SOFT_INPUT_MASK_STATE) {
                case WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED:
                    if (!isTextEditor || !doAutoShow) {
                        if (WindowManager.LayoutParams.mayUseInputMethod(windowFlags)) {
                            // There is no focus view, and this window will
                            // be behind any soft input window, so hide the
                            // soft input window if it is shown.
                            if (DEBUG) Slog.v(TAG, "Unspecified window will hide input");
                            hideCurrentInputLocked(InputMethodManager.HIDE_NOT_ALWAYS, null);
}}}}}}}
```

`mayUseInputMethod()` 方法检查传入的 `windowFlags` 是否设置了 `FLAG_NOT_FOCUSABLE` 或 `FLAG_ALT_FOCUSABLE_IM`。如果没有设置这两个 Flag 或同时设置了这两个 Flag 都会返回 true，即，该 `Window` 可能会使用输入法。默认状态下这两个 Flag 都不会设置，因此 `windowGainedFocus()` 方法中最终会调用 `hideCurrentInputLocked()` 方法收起输入法。

```java
// android.view.WindowManager
public static class LayoutParams extends ViewGroup.LayoutParams implements Parcelable {    
    /**
        * Given a particular set of window manager flags, determine whether
        * such a window may be a target for an input method when it has
        * focus.  In particular, this checks the
        * {@link #FLAG_NOT_FOCUSABLE} and {@link #FLAG_ALT_FOCUSABLE_IM}
        * flags and returns true if the combination of the two corresponds
        * to a window that needs to be behind the input method so that the
        * user can type into it.
        *
        * @param flags The current window manager flags.
        *
        * @return Returns true if such a window should be behind/interact
        * with an input method, false if not.
        */
    public static boolean mayUseInputMethod(int flags) {
        switch (flags&(FLAG_NOT_FOCUSABLE|FLAG_ALT_FOCUSABLE_IM)) {
            case 0:
            case FLAG_NOT_FOCUSABLE|FLAG_ALT_FOCUSABLE_IM:
                return true;
        }
        return false;
}}
```

# 3. FLAG_ALT_FOCUSABLE_IM 的影响

`FLAG_ALT_FOCUSABLE_IM` 是文章标题中出现的，为什么关注该 Flag？因为当 `addWindow()` 的时候默认是会收起输入法的，但是当显示一个 `AlertDialog` 的时候并不会。分析后得知，当 `AlertDialog` 调用 `show()` 时，如果不是自定义 `View` 或自定义 `View` 不能输入，则会设置 `FLAG_ALT_FOCUSABLE_IM`。

```java
// com.android.internal.app.AlertController
private void setupCustomContent(ViewGroup customPanel) {
    final View customView;
    if (mView != null) {
        customView = mView;
    } else if (mViewLayoutResId != 0) {
        final LayoutInflater inflater = LayoutInflater.from(mContext);
        customView = inflater.inflate(mViewLayoutResId, customPanel, false);
    } else {
        customView = null;
    }
    final boolean hasCustomView = customView != null;
    if (!hasCustomView || !canTextInput(customView)) {
        mWindow.setFlags(WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM,
                WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM);
}}
```

因此，`AlertDialog` 弹出的时候 `mayUseInputMethod()` 方法会返回 false，所以也不会调用 `hideCurrentInputLocked()` 方法收起输入法。