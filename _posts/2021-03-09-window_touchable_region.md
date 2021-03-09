---
layout: post
title: Window Touchable Region
date: 2021-03-09
modified: 2021-03-09
tags: [Android, InputManager, WindowManager]
categories: [Developer]
excerpt:  当用户触屏后，InputReader 从驱动读取一个输入事件加入到队列，InputDispatcher 从队列中读取一个输入事件准备分发。如果该输入事件是一个触摸事件...
description: 当用户触屏后，InputReader 从驱动读取一个输入事件加入到队列，InputDispatcher 从队列中读取一个输入事件准备分发。如果该输入事件是一个触摸事件...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

* TOC
{:toc}

# 0. Android 输入事件分发概述

当用户触屏后，InputReader 从驱动读取一个输入事件加入到队列，InputDispatcher 从队列中读取一个输入事件准备分发。

如果该输入事件是一个触摸事件，则先查找最上层可接受该 Touch 的窗口。然后通过 Socket 发送给对应的 InputConsumer。

InputConsumer 会发送给相应的 InputEventReceiver 实现类。该实现类中分发给每一个 InputStage 责任链，InputStage 实现类会最终处理该输入事件。

参考「[InputChannel and InputDispatcher in Android](https://xianzhu21.space/developer/inputchannel_inputdispatcher/)」。

Gityuan 的「[Input 系统—事件处理全过程](http://gityuan.com/2016/12/31/input-ipc/)」有个整体框架图如下：

<figure>
    <a href="/images/input_summary.jpg"><img src="/images/input_summary.jpg" alt="Data flow in input system"></a>
    <figcaption>Data flow in input system</figcaption>
</figure>

# 1. 查找接收 Touch 的窗口

一个触摸事件该不该发给一个窗口，取决于该窗口的可触摸区域是否包含该触摸的坐标，且没有被别的窗口的可触摸区域盖住。

该逻辑实现在 `findTouchedWindowAtLocked()` 函数中，它会遍历所有的 InputWindowHandle，然后调用 `windowInfo->touchableRegionContainsPoint(x, y)` 检查可触摸区域是否包含该触摸点坐标。如果满足则返回。

```cpp
// frameworks/native/services/inputflinger/InputDispatcher.cpp
sp<InputWindowHandle> InputDispatcher::findTouchedWindowAtLocked(int32_t displayId,
        int32_t x, int32_t y, bool addOutsideTargets, bool addPortalWindows) {
    // Traverse windows from front to back to find touched window.
    const std::vector<sp<InputWindowHandle>> windowHandles = getWindowHandlesLocked(displayId);
    for (const sp<InputWindowHandle>& windowHandle : windowHandles) {
        const InputWindowInfo* windowInfo = windowHandle->getInfo();
        if (windowInfo->displayId == displayId) {
            int32_t flags = windowInfo->layoutParamsFlags;
            if (windowInfo->visible) {
                if (!(flags & InputWindowInfo::FLAG_NOT_TOUCHABLE)) {
                    bool isTouchModal = (flags & (InputWindowInfo::FLAG_NOT_FOCUSABLE
                            | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
                    if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
                        ...
                        // Found window.
                        return windowHandle;
                }}
                ...
    }}}
    return nullptr;
}
```

这里为什么会叫可触摸区域（Touchable Region），而不是窗口大小，是因为可触摸区域可以和显示窗口不一样，甚至可以由多个矩形组成（一个显示窗口只能是一个矩形）。

InputWindowHandle 持有 InputWindowInfo 对象，InputWindowInfo 有成员 `touchableRegion`。它是 Region 类对象，注意是在 framework/native/include/ui/Region.h 中声明的 Region 类。

```cpp
// frameworks/native/include/input/InputWindow.h
struct InputWindowInfo {
    ...
    Region touchableRegion;
    void addTouchableRegion(const Rect& regin);
    bool touchableRegionContainsPoint(int32_t x, int32_t y) const;
    ...
}

class InputWindowHandle : public RefBase {
public:
    inline const InputWindowInfo* getInfo() const {
        return &mInfo;
    }
    ...
protected:
    InputWindowInfo mInfo;
};

// frameworks/native/include/ui/Region.h
class Region : public LightFlattenable<Region>
{
public:
    inline bool contains(int x, int y) const;
    ...
    // STL-like iterators
    typedef Rect const* const_iterator;
    const_iterator begin() const;
    const_iterator end() const;
    ...
    // mStorage is a (manually) sorted array of Rects describing the region
    // with an extra Rect as the last element which is set to the
    // bounds of the region. However, if the region is
    // a simple Rect then mStorage contains only that rect.
    Vector<Rect> mStorage;
};
```

Region 类的成员 `mStorage` 是一个 `Vector<Rect>` 对象，从这里也可以知道可触摸区域由多个矩形组成的。

# 2. 可触摸区域的计算

InputWindowInfo 结构体中的 `touchableRegion` 其实是从 Java 层过来的，是 Java InputWindowHandle 类的 `touchableRegion` 成员。

```java
// frameworks/base/core/java/android/view/InputWindowHandle.java
public final class InputWindowHandle {
    ...
    public final Region touchableRegion = new Region();
    ...
}
```

这个 `touchableRegion` 是在 UpdateInputForAllWindowsConsumer 的 `updateInputWindows()` 方法中遍历 DisplayContent 的所有 WindowState 时计算的。

```java
// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
final class InputMonitor {
    ...
    private final class UpdateInputForAllWindowsConsumer implements Consumer<WindowState> {
        ...
        private void updateInputWindows(boolean inDrag) {
            ...
            mDisplayContent.forAllWindows(this, true /* traverseTopToBottom */);
            ...
}}}
```

UpdateInputForAllWindowsConsumer 实现了 Consumer，所以会对每个 WindowState 调用它的 `accept()` 方法。该方法中会调用 `populateInputWindowHandle()` 方法计算 `touchableRegion`，最后调用 `setInputWindowInfo()` 方法使之生效。

```java
// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
private final class UpdateInputForAllWindowsConsumer implements Consumer<WindowState> {
    ...
    @Override
    public void accept(WindowState w) {
        final InputChannel inputChannel = w.mInputChannel;
        final InputWindowHandle inputWindowHandle = w.mInputWindowHandle;
        ...
        populateInputWindowHandle(
                inputWindowHandle, w, flags, type, isVisible, hasFocus, hasWallpaper);
        if (w.mWindowAnimator.hasSurface()) {
            mInputTransaction.setInputWindowInfo(
                    w.mWindowAnimator.mSurfaceController.mSurfaceControl, inputWindowHandle);
}}}

```

## 2.1 InputMonitor#populateInputWindowHandle()

`populateInputWindowHandle()` 方法中 `touchableRegion` 成员是调用 WindowState 类的 `getSurfaceTouchableRegion()` 方法获取的。

```java
// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
void populateInputWindowHandle(final InputWindowHandle inputWindowHandle,
            final WindowState child, int flags, final int type, final boolean isVisible,
            final boolean hasFocus, final boolean hasWallpaper) {
    inputWindowHandle.name = child.toString();
    flags = child.getSurfaceTouchableRegion(inputWindowHandle, flags);
    ...
    inputWindowHandle.layer = child.mLayer;
    inputWindowHandle.displayId = child.getDisplayId();
    final Rect frame = child.getFrameLw();
    inputWindowHandle.frameLeft = frame.left;
    inputWindowHandle.frameTop = frame.top;
    inputWindowHandle.frameRight = frame.right;
    inputWindowHandle.frameBottom = frame.bottom;
    ...
}
```

## 2.2 WindowState#getSurfaceTouchableRegion()

Touchable Region 的计算一共有 4 种情况：兼容不可修改大小的 Activity 窗口、普通 Activity 窗口、ActivityView 窗口、其他窗口。这里我们只看普通 Activity 窗口和其他窗口。

**普通 Activity 窗口**

如果有 Letterbox 则使用 Letterbox 的内边界。Letterbox 是用来兼容显示宽高比不匹配的窗口而加的边框。否则使用 Task 或 Stack 的边界作为可触摸区域。从这里可以看到，Activity 窗口的可触摸区域其实就是和显示区域一致。

**其他窗口**

调用 `getTouchableRegion()` 方法。详细看 2.2.1 WindowState#getTouchableRegion()。

拿到 touchable region 后会调用 `translate()` 方法偏移到 Surface 坐标轴，即与 `mFrame` 的起始位置一致。这可能是为了方便做裁剪、缩放等操作。

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
int getSurfaceTouchableRegion(InputWindowHandle inputWindowHandle, int flags) {
    final boolean modal = (flags & (FLAG_NOT_TOUCH_MODAL | FALG_NOT_FOCUSABLE)) == 0;
    final Region region = inputWindowHandle.touchableRegion;
    setTouchableRegionCropIfNeeded(inputWindowHandle);

    final Rect appOverrideBounds = mAppToken != null
            ? mAppToken.getResolvedOverrideBounds() : null;
    // 1.
    if (appOverrideBounds != null && !appOverrideBounds.isEmpty()) {
        // skip
    }

    // 2.
    if (modal && mAppToken != null) {
        // Limit the outer touch to the activity stack region.
        flags != FLAG_NOT_TOUCH_MODAL;
        // If the inner bounds of letterbox is available, then it will be used as the touchable
        // region so it won't cover the touchable letterbox and the touch events can slip to
        // activity from letterbox.
        mAppToken.getLetterboxInnerBounds(mTmpRect);
        if (TmpRect.isEmpty()) {
            final Task task = getTask();
            if (task != null)
                task.getDimBounds(mTmpRect);
            else {
                getStack().getDimBounds(mTmpRect);
        }}
        ...
        region.set(mTmpRect);
        cropRegionToStackBoundsIfNeeded(region);
        subtractTouchExcludeRegionIfNeeded(region);
    } else if (modal && mTapExcludeRegionHolder != null) {
    // 3.
        ...
    } else {
    // 4.
        // Not modal or full screen modal
        getTouchableRegion(region);
    }
    // Translate to surface based coordinates.
    region.translate(-mWindowFrames.mFrame.left, -mWindowFrames.mFrame.top);

    return flags;
```

### 2.2.1 WindowState#getTouchableRegion()

`getTouchableRegion()` 根据 `mTouchableInsets` 方法分 4 种情况。`mTouchableInsets` 表明应该使用哪个区域作为 Touchable Region。

`mTouchableInsets` 默认值是 `TOUCHABLE_INSETS_FRAME`，即默认的触摸区域是 `mWindowFrames.mFrame` 矩形。`mWindowFrames.mFrame` 其实与 Surface 的窗口一致。

如果是 `TOUCHABLE_INSETS_REGION`，则使用 `mGivenTouchableRegion` 作为可触摸区域，而 `mGivenTouchableRegion` 是在客户端进程中设置的。

注意，`mGivenTouchableRegion` 需要以 `mFrame` 左上角作为坐标原点，即要给出窗口的相对位置坐标，因为会调用 `translate()` 方法偏移到与 `mFrame` 一样的起始位置。

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
/**
 * Flag indicating whether the touchable region should be adjusted by
 * the visible insets; if false the area outside the visible insets is
 * NOT touchable, so we must use those to adjust the frame during hit
 * tests.
 */
int mTouchableInsets = ViewTreeObserver.InternalInsetsInfo.TOUCHABLE_INSETS_FRAME;

/** Get the touchable region in global coordinates. */
void getTouchableRegion(Region outRegion) {
    final Rect frame = mWindowFrames.mFrame;
    switch (mTouchableInsets) {
        default:
        case TOUCHABLE_INSETS_FRAME:
            outRegion.set(frame);
            break;
        case TOUCHABLE_INSETS_CONTENT:
            applyInsets(outRegion, frame, mGivenContentInsets);
            break;
        case TOUCHABLE_INSETS_VISIBLE:
            applyInsets(outRegion, frame, mGivenVisibleInsets);
            break;
        case TOUCHABLE_INSETS_REGION: {
            outRegion.set(mGivenTouchableRegion);
            outRegion.translate(frame.left, frame.top);
            break;
        }
    }
    cropRegionToStackBoundsIfNeeded(outRegion);
    subtractTouchExcludeRegionIfNeeded(outRegion);
}
```

# 3. 可触摸区域的传递

在上一节，设置 touchable region 后，调用 `Transaction#setInputWindowInfo()` 方法把 InputWindowHandle 对象传递给 SurfaceFlinger，更新 `Layer.mDrawingState.inputInfo`。

之后在 `SurfaceFlinger#updateInputWindowInfo()` 函数中生成一个 `std::vector<InputWindowInfo>` 对象后传递给 InputFlinger。

### 3.1 Transaction#setInputWindowInfo()

`setInputWindowInfo()` 方法就是调用了 JNI 函数 `nativeSetInputWindowInfo()`。

```java
// framework/base/core/java/android/view/SurfaceControl.java
/**
 * An atomic set of changes to a set of SurfaceControl.
 */
public static class Transaction implements Closeable {
    public Transaction setInputWindowInfo(SurfaceControl sc, InputWindowHandle handle) {
        sc.checkNotReleased();
        nativeSetInputWindowInfo(mNativeObject, sc.mNativeObject, handle);
        return this;
}}
```

`nativeSetInputWindowInfo()` 函数中会调用 `updateInfo()` 函数，它从 Java 对象中读取数据，填充 native InputWindowHandle 的 `mInfo` 成员。InputWindowHandle 的相关内容可查看第 1 节。

```cpp
// framework/base/core/jni/android_view_SurfaceControl.cpp
static void nativeSetInputWindowInfo(JNIEnv* env, jclass clazz, jlong transactionObj,
        jlong nativeObject, jobject inputWindow) {
    auto transaction = reinterpret_cast<SurfaceComposerClient::Transaction*>(transactionObj);

    sp<NativeInputWindowHandle> handle = android_view_InputWindowHandle_getHandle(
            env, inputWindow);
    handle->updateInfo();

    SurfaceControl* const ctrl = reinterpret_cast<SurfaceControl *>(nativeObject);
    transaction->setInputWindowInfo(ctrl, *handle->getInfo());
}
```

Java 层 InputWindowHandle 类的成员 `touchableRegion` 是一个 Region 对象。其实 Java 层 Region 类是个 Proxy ，它实际实现是 native 层 SkRegion。所以在 `updateInfo()` 函数中会先从 Java 对象获取到 SkRegion 对象，然后遍历所有矩形，再调用 `addTouchableRegion()` 函数，存到 native InputWindowInfo 对象。

```cpp
// frameworks/base/core/jni/android_hardware_input_InputWindowHandle.cpp
bool NativeInputWindowHandle::updateInfo() {
    ...
    mInfo.touchableRegion.clear();
    ...
    jobject regionObj = env->GetObjectField(obj,
            gInputWindowHandleClassInfo.touchableRegion);
    if (regionObj) {
        SkRegion* region = android_graphics_Region_getSkRegion(env, regionObj);
        for (SkRegion::Iterator it(*region); !it.done(); it.next()) {
            const SkIRect& rect = it.rect();
            mInfo.addTouchableRegion(Rect(rect.fLeft, rect.fTop, rect.fRight, rect.fBottom));
        }
        env->DeleteLocalRef(regionObj);
}}
```

填充 native InputWindowInfo 后，调用 `Transaction::setInputWindowInfo()` 函数更新相应 layer 的 `inputInfo` 成员。最后加上 `layer_state_t::eInputInfoChanged` flag。

```cpp
// frameworks/native/libs/gui/SurfaceComposerClient.cpp
SurfaceComposerClient::Transaction& SurfaceComposerClient::Transaction::setInputWindowInfo(
        const sp<SurfaceControl>& sc,
        const InputWindowInfo& info) {
    layer_state_t* s = getLayerState(sc);
    if (!s) {
        mStatus = BAD_INDEX;
        return *this;
    }
    s->inputInfo = info;
    s->what |= layer_state_t::eInputInfoChanged;
    return *this;
}
```

## 3.2 SurfaceFlinger#updateInputWindowInfo()

每当 SurfaceFlinger 收到 `MessageQueue::INVALIDATE` 消息时，会调用 `updateInputFlinger()` 函数。该函数中，如果 `mInputInfoChanged` 为 true，则调用 `updateInputWindowInfo()` 函数。因为在 3.1 节最后给 `what` 成员加上了 `layer_state_t::eInputInfoChanged` flag，所以这里 `mInputInfoChanged` 为 true，会调用 `updateInputWindowInfo()` 函数。

```cpp
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::onMessageReceived(int32_t what) NO_THREAD_SAFETY_ANALYSIS {
    switch (what) {
        case MessageQueue::INVALIDATE: {
            ...
            updateInputFlinger();
            ...
	      break;
        }
        case MessageQueue::REFRESH: {
            handleMessageRefresh();
            break;
}}}

void SurfaceFlinger::updateInputFlinger() {
    ATRACE_CALL();
    if (!mInputFlinger) {
        return;
    }

    if (mVisibleRegionsDirty || mInputInfoChanged) {
        mInputInfoChanged = false;
        updateInputWindowInfo();
    }
    ...
}

void SurfaceFlinger::updateInputWindowInfo() {
    std::vector<InputWindowInfo> inputHandles;

    mDrawingState.traverseInReverseZOrder([&](Layer* layer) {
        if (layer->hasInput()) {
            // When calculating the screen bounds we ignore the transparent region since it may
            // result in an unwanted offset.
            inputHandles.push_back(layer->fillInputInfo());
        }
    });

    mInputFlinger->setInputWindows(inputHandles,
                                   mInputWindowCommands.syncInputWindows ? mSetInputWindowsListener
                                                                         : nullptr);
}
```

`updateInputWindowInfo()` 函数中会遍历每个 Layer，并调用 `fillInputInfo()` 函数从 Layer 中获取 InputWindowInfo 对象。

之前在 `getSurfaceTouchableRegion()` 方法中，把 `touchableRegion` 平移到了 Surface 坐标系。

所以在 `fillInputInfo()` 函数中又调用 `translate()` 函数平移到了屏幕坐标系。

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
...
    // Translate to surface based coordinates.
    region.translate(-mWindowFrames.mFrame.left, -mWindowFrames.mFrame.top);
...
```

```cpp
// frameworks/native/services/surfaceflinger/Layer.cpp
InputWindowInfo Layer::fillInputInfo() {
    InputWindowInfo info = mDrawingState.inputInfo;
    ...
    // Position the touchable region relative to frame screen location and restrict it to frame
    // bounds.
    info.touchableRegion = info.touchableRegion.translate(info.frameLeft, info.frameTop);
    info.visible = canReceiveInput();
    ...
}
```

在 `updateInputWindowInfo()` 函数中获取 InputWindowInfo 之后调用 IInputFlinger 的 `setInputWindows()` 函数把 `inputHandles` 传递给 IInputFlinger。

InputManager 继承了 BnInputFlinger 并实现了 IInputFlinger 定义的函数。它在 `setInputWindows()` 函数中，先根据 displayId 把 InputWindowHandle 分类到不同的集合中。最后每个屏幕调用一次 InputDispatcher 的 `setInputWindows()` 函数，第一个参数是 `inputWindowHandles`，第二个参数是 `displayId`。

```cpp
// frameworks/native/services/inputflinger/InputFlinger.h
class InputManager : public InputManagerInterface, public BnInputFlinger {
...
public:
    ...
    virtual void setInputWindows(const std::vector<InputWindowInfo>& handles,
            const sp<ISetInputWindowsListener>& setInputWindowsListener);
    ...
private:
    ...
    sp<InputDispatcherInterface> mDispatcher;
    ...
};

// frameworks/native/services/inputflinger/InputFlinger.cpp
void InputManager::setInputWindows(const std::vector<InputWindowInfo>& infos,
        const sp<ISetInputWindowsListener>& setInputWindowsListener) {
    std::unordered_map<int32_t, std::vector<sp<InputWindowHandle>>> handlesPerDisplay;

    std::vector<sp<InputWindowHandle>> handles;
    for (const auto& info : infos) {
        handlesPerDisplay.emplace(info.displayId, std::vector<sp<InputWindowHandle>>());
        handlesPerDisplay[info.displayId].push_back(new BinderWindowHandle(info));
    }
    for (auto const& i : handlesPerDisplay) {
        mDispatcher->setInputWindows(i.second, i.first, setInputWindowsListener);
    }
}
```

`InputDispatcher::setInputWindows()` 函数的主要作用是更新 mWindowHandlesByDisplay 中的 InputWindowHandle。

1. 如果传进来的 `inputWindowHandles` 是空的，删除该 Display 的所有 InputWindowHandle。
2. 否则：
    1. 遍历传进来的 `inputWindowHandles`。
        1. 如果当前 `handle` 没有 InputChannel，则略过。
        2. 如果当前 `handle` 的 `displayId` 与传进来的 `displayId` 不一致，则略过。
        3. 在 `oldHandlesByTokens` 中查找该 `handle` 是否已经存在。
            1. 如果存在，更新后加入到 `newHandles`。
            2. 否则直接加入到 `newHandles`。
    2. 在 `newHandles` 中查找最顶层获得焦点的窗口 `newFocusedWindowHandle`。
    3. 用 `newHandles` 更新该 `displayId` 中的 `InputWindowHandle` 集合。
3. 更新该 `displayId` 的焦点窗口。
4. 更新 TouchState。
5. 释放已经消失的窗口的 InputChannel。

```cpp
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::setInputWindows(const std::vector<sp<InputWindowHandle>>& inputWindowHandles,
        int32_t displayId, const sp<ISetInputWindowsListener>& setInputWindowsListener) {
    ...
    // Copy old handles for release if they are no longer present.
    const std::vector<sp<InputWindowHandle>> oldWindowHandles =
            getWindowHandlesLocked(displayId);

    sp<InputWindowHandle> newFocusedWindowHandle = nullptr;

    // 1.
    if (inputWindowHandles.empty()) {
        // Remove all handles on a display if there are no windows left.
        mWindowHandlesByDisplay.erase(displayId);
    } else {
    // 2.
        const std::vector<sp<InputWindowHandle>>& oldHandles =
                mWindowHandlesByDisplay[displayId];
        std::unordered_map<sp<IBinder>, sp<InputWindowHandle>, IBinderHash> oldHandlesByTokens;
        for (const sp<InputWindowHandle>& handle : oldHandles) {
            oldHandlesByTokens[handle->getToken()] = handle;
        }

        std::vector<sp<InputWindowHandle>> newHandles;
        // 2.1
        for (const sp<InputWindowHandle>& handle : inputWindowHandles) {
            const InputWindowInfo* info = handle->getInfo();
            // 2.1.1
            if ((getInputChannelLocked(handle->getToken()) == nullptr &&
                 info->portalToDisplayId == ADISPLAY_ID_NONE)) {
                ...
                continue;
            }
            // 2.1.2
            if (info->displayId != displayId) {
                ALOGE("Window %s updated by wrong display %d, should belong to display %d",
                      handle->getName().c_str(), displayId, info->displayId);
                continue;
            }
            // 2.1.3
            if (oldHandlesByTokens.find(handle->getToken()) != oldHandlesByTokens.end()) {
                const sp<InputWindowHandle> oldHandle =
                        oldHandlesByTokens.at(handle->getToken());
                oldHandle->updateFrom(handle);
                newHandles.push_back(oldHandle);
            } else {
                newHandles.push_back(handle);
        }}
        // 2.2
        for (const sp<InputWindowHandle>& windowHandle : newHandles) {
            // Set newFocusedWindowHandle to the top most focused window instead of the last one
            if (!newFocusedWindowHandle && windowHandle->getInfo()->hasFocus
                    && windowHandle->getInfo()->visible) {
                newFocusedWindowHandle = windowHandle;
            }
            ...
        }
        // 2.3
        // Insert or replace
        mWindowHandlesByDisplay[displayId] = newHandles;
    }

    // 3.
    sp<InputWindowHandle> oldFocusedWindowHandle =
            getValueByKey(mFocusedWindowHandlesByDisplay, displayId);
    if (oldFocusedWindowHandle != newFocusedWindowHandle) {
        if (oldFocusedWindowHandle != nullptr) {
            ...
            mFocusedWindowHandlesByDisplay.erase(displayId);
        }
        if (newFocusedWindowHandle != nullptr) {
            mFocusedWindowHandlesByDisplay[displayId] = newFocusedWindowHandle;
    }}

    // 4.
    ssize_t stateIndex = mTouchStatesByDisplay.indexOfKey(displayId);
    if (stateIndex >= 0) {
        TouchState& state = mTouchStatesByDisplay.editValueAt(stateIndex);
        for (size_t i = 0; i < state.windows.size(); ) {
            TouchedWindow& touchedWindow = state.windows[i];
            if (!hasWindowHandleLocked(touchedWindow.windowHandle)) {
                ...
                state.windows.erase(state.windows.begin() + i);
            } else {
              ++i;
    }}}

    // 5.
    // Release information for windows that are no longer present.
    // This ensures that unused input channels are released promptly.
    // Otherwise, they might stick around until the window handle is destroyed
    // which might not happen until the next GC.
    for (const sp<InputWindowHandle>& oldWindowHandle : oldWindowHandles) {
        if (!hasWindowHandleLocked(oldWindowHandle)) {
            oldWindowHandle->releaseChannel();
    }}
    ...
}
```

这样就把本次的 InputWindowHandle 更新完成了。重新回到第 1 节，当 InputFlinger 接受一个 Touch 后，会调用 `InputDispatcher::findTouchedWindowAtLocked()` 函数查找接收该 Touch 的窗口。

```cpp
// frameworks/native/services/inputflinger/InputDispatcher.cpp
sp<InputWindowHandle> InputDispatcher::findTouchedWindowAtLocked(int32_t displayId,
        int32_t x, int32_t y, bool addOutsideTargets, bool addPortalWindows) {
    // Traverse windows from front to back to find touched window.
    const std::vector<sp<InputWindowHandle>> windowHandles = getWindowHandlesLocked(displayId);
    for (const sp<InputWindowHandle>& windowHandle : windowHandles) {
        const InputWindowInfo* windowInfo = windowHandle->getInfo();
        if (windowInfo->displayId == displayId) {
            int32_t flags = windowInfo->layoutParamsFlags;

            if (windowInfo->visible) {
                if (!(flags & InputWindowInfo::FLAG_NOT_TOUCHABLE)) {
                    bool isTouchModal = (flags & (InputWindowInfo::FLAG_NOT_FOCUSABLE
                            | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
                    if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
                        // Found window.
                        return windowHandle;
    }}}}}
    return nullptr;
}
```
