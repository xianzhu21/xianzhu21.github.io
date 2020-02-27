---
layout: post
title: InputChannel and InputDispatcher in Android
date: 2020-02-27
modified: 2020-02-27
tags: [Android, InputManager, WindowManager]
categories: [Developer]
excerpt:  InputChannel 是分发 input 事件的通道，driver 产生 input 事件，发送到 system_server 后通过 socket 发送到 app 进程。每次创建 window 时会创建一对 InputChannel 对象...
description: InputChannel 是分发 input 事件的通道，driver 产生 input 事件，发送到 system_server 后通过 socket 发送到 app 进程。每次创建 window 时会创建一对 InputChannel 对象...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

*Based on Android 10.*

* TOC
{:toc}

# 0. InputChannel

InputChannel 是分发 input 事件的通道，driver 产生 input 事件，发送到 system_server 后通过 socket 发送到 app 进程。

每次创建 window 时会创建一对 InputChannel 对象，一个存到 app 进程的 ViewRootImpl 的 mInputChannel 变量，一个存到 system_server 的 WindowState 的 mInputChannel 字段。

# 1. Open and register input channel

## 1.1 Open a pair of input channels

### 1.1.1 WindowState#openInputChannel()

ViewRootImpl 类的 `setView()` 方法中会创建 InputChannel 对象作为 outInputChannel。然后在 WindowManagerService 类的 `addWindow()` 方法中调用 WindowState 类的 `openInputChannel()` 方法打开 server 端和 client 端的 InputChannel。

它实际调用 `InputChannel` 的静态方法 `openInputChannelPair()` 创建 InputChanel 对，它又调用 native 方法 `nativeOpenInputChannelPair()`。Server 端 InputChannel 存在 WindowState 的 mInputChannel 变量，Client 端 InputChanenl 调用 `transforTo()` 方法传给 ViewRootImpl 的 mInputChannel。

```java
// frameworks/base/services/core/java/com/android/server/wm/WindowState.java
void openInputChannel(InputChannel outInputChannel) {
    if (mInputChannel != null) {
        throw new IllegalStateException("Window already has an input channel.");
    }
    String name = getName();
    InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
    mInputChannel = inputChannels[0];
    mClientChannel = inputChannels[1];
    mInputWindowHandle.token = mClient.asBinder();
    if (outInputChannel != null) {
        mClientChannel.transferTo(outInputChannel);
        mClientChannel.dispose();
        mClientChannel = null;
    } else {
        // If the window died visible, we setup a dummy input channel, so that taps
        // can still detected by input monitor channel, and we can relaunch the app.
        // Create dummy event receiver that simply reports all events as handled.
        mDeadWindowEventReceiver = new DeadWindowEventReceiver(mClientChannel);
    }
    mWmService.mInputManager.registerInputChannel(mInputChannel, mClient.asBinder());
}
```

### 1.1.2 InputChannel#openInputChannelPair()

`openInputChannelPair()` 最终调用的是 `nativeOpenInputChannelPair()` 方法。代码注释说明，创建两个 InputChannel 后，一个提供给 InputDispatcher，即 server 端，用于分发 input 事件，另一个提供给应用的 input 队列用于消费 input 事件。

```java
// frameworks/base/core/java/android/view/InputChannel.java
/**
    * Creates a new input channel pair.  One channel should be provided to the input
    * dispatcher and the other to the application's input queue.
    * @param name The descriptive (non-unique) name of the channel pair.
    * @return A pair of input channels.  The first channel is designated as the
    * server channel and should be used to publish input events.  The second channel
    * is designated as the client channel and should be used to consume input events.
    */
public static InputChannel[] openInputChannelPair(String name) {
    if (name == null) {
        throw new IllegalArgumentException("name must not be null");
    }
    return nativeOpenInputChannelPair(name);
}
```

`android_view_InputChannel_nativeOpenInputChannelPair()` 函数中调用 `InputChannel::openInputChannelPair()` 创建两个 InputChannel 对象，然后再封装成 Java 层的 InputChannel 返回。

```java
// frameworks/base/core/jni/android_view_InputChannel.cpp
static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair(JNIEnv* env,
        jclass clazz, jstring nameObj) {
    ScopedUtfChars nameChars(env, nameObj);
    std::string name = nameChars.c_str();

    sp<InputChannel> serverChannel;
    sp<InputChannel> clientChannel;
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
    ...
    jobjectArray channelPair = env->NewObjectArray(2, gInputChannelClassInfo.clazz, NULL);
    if (env->ExceptionCheck()) {
        return NULL;
    }

    jobject serverChannelObj = android_view_InputChannel_createInputChannel(env,
            std::make_unique<NativeInputChannel>(serverChannel));
    if (env->ExceptionCheck()) {
        return NULL;
    }

    jobject clientChannelObj = android_view_InputChannel_createInputChannel(env,
            std::make_unique<NativeInputChannel>(clientChannel));
    if (env->ExceptionCheck()) {
        return NULL;
    }

    env->SetObjectArrayElement(channelPair, 0, serverChannelObj);
    env->SetObjectArrayElement(channelPair, 1, clientChannelObj);
    return channelPair;
}
```

### 1.1.3 InputChannel::openInputChannelPair()

`InputChannel.openInputChannelPair()` 函数定义在 `InputTransport.cpp` 中。函数中调用 Linux 的 `socketpair()` 函数创建一对已连接的 socket，可以把这一对 socket 当成 pipe 返回的文件描述符使用，并且这两个文件描述符都是可读可写的，参考「[Linux 上实现双向进程间通信管道](https://www.ibm.com/developerworks/cn/linux/l-pipebid/index.html)」。然后封装成 InputChannel 对象。

```
// frameworks/native/libs/input/InputTransport.cpp
status_t InputChannel::openInputChannelPair(const String8& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        ...
        return result;
    }

    int bufferSize = SOCKET_BUFFER_SIZE;
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    std::string serverChannelName = name;
    serverChannelName.append(" (server)");
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);

    std::string clientChannelName = name;
    clientChannelName.append(" (client)");
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}

InputChannel::InputChannel(const std::string& name, int fd) :
        mName(name) {
    setFd(fd);
}
```

## 1.2 Register input channel of server

Server 端的 InputChannel 注册就在 WindowState 类的 `openInputChannel()` 方法的最后一行，调用 InputManagerService 的 `registerInputChannel()` 方法注册。

### 1.2.1 InputManagerService#registerInputChannel()

创建好一对 InputChannel 后，调用 `registerInputChannel()` 方法注册 server 端的 InputChannel。

```java
// frameworks/base/services/core/java/com/android/server/input/InputManagerService.java
/**
    * Registers an input channel so that it can be used as an input event target.
    * @param inputChannel The input channel to register.
    * @param inputWindowHandle The handle of the input window associated with the
    * input channel, or null if none.
    */
public void registerInputChannel(InputChannel inputChannel, IBinder token) {
    if (inputChannel == null) {
        throw new IllegalArgumentException("inputChannel must not be null.");
    }
    if (token == null) {
        token = new Binder();
    }
    inputChannel.setToken(token);
    nativeRegisterInputChannel(mPtr, inputChannel, Display.INVALID_DISPLAY);
}
```

`nativeRegisterInputChannel()` 函数先把 Java 层的 InputChannel 转换成 native 层变量， 然后调用 `NativeInputManager::registerInputChannel()`，这个函数直接获取 InputDispathcer 后 调用 `reigisterInputChannel()` 函数。

```java
// frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp
static void nativeRegisterInputChannel(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobject inputChannelObj, jint displayId) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    if (inputChannel == nullptr) {
        throwInputChannelNotInitialized(env);
        return;
    }

    status_t status = im->registerInputChannel(env, inputChannel, displayId);
    if (status) {
        std::string message;
        message += StringPrintf("Failed to register input channel.  status=%d", status);
        jniThrowRuntimeException(env, message.c_str());
        return;
    }

    android_view_InputChannel_setDisposeCallback(env, inputChannelObj,
            handleInputChannelDisposed, im);
}

status_t NativeInputManager::registerInputChannel(JNIEnv* /* env */,
        const sp<InputChannel>& inputChannel, int32_t displayId) {
    ATRACE_CALL();
    return mInputManager->getDispatcher()->registerInputChannel(
            inputChannel, displayId);
}
```

### 1.2.2 InputDispatcher::registerInputChannel()

首先调用 `getConnectionIndexLocked()` 函数检查是否已经注册过。若没有注册过，创建一个 Connection 对象封装 inputChannel 和 inputWindowHandle。调用 `inputChannel->getFd()` 获得文件描述符。最后加入到 mLooper 对象中，如果 client 发来消息，回调 `handleReceiveCallback()` 函数。

```java
// frameworks/native/services/inputflinger/InputDispatcher.cpp
status_t InputDispatcher::registerInputChannel(const sp<InputChannel>& inputChannel,
                                                int32_t displayId) {
    { // acquire lock
        std::scoped_lock _l(mLock);
        if (getConnectionIndexLocked(inputChannel) >= 0) {
            ALOGW("Attempted to register already registered input channel '%s'",
                    inputChannel->getName().c_str());
            return BAD_VALUE;
        }

        sp<Connection> connection = new Connection(inputChannel, false /*monitor*/);
        int fd = inputChannel->getFd();
        mConnectionsByFd.add(fd, connection);
        mInputChannelsByToken[inputChannel->getToken()] = inputChannel;
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
    } // release lock

    // Wake the looper because some connections have changed.
    mLooper->wake();
    return OK;
}
```

## 1.3 Register input channel of client

Client 端的 InputChannel 注册是在 ViewRootImpl 类的 `setView()` 方法的 `mWindowSession.addToDisplay()` 之后，即注册完 server 端 InputChannel 之后。这里创建了 WindowInputEventReceiver 对象，它是从 InputEventReceiver 继承而来的。InputEventReceiver 的构造方法中调用了 `nativeInit()` 方法。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    if ((mWindowAttributes.inputFeatures
            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
        mInputChannel = new InputChannel();
    }
    ...
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
            mTempInsets);
    ...
    if (mInputChannel != null) {
        mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                Looper.myLooper());
    }
    ...
}

// frameworks/base/core/java/android/view/InputEventReceiver.java
/**
    * Creates an input event receiver bound to the specified input channel.
    *
    * @param inputChannel The input channel.
    * @param looper The looper to use when invoking callbacks.
    */
public InputEventReceiver(InputChannel inputChannel, Looper looper) {
    mInputChannel = inputChannel;
    mMessageQueue = looper.getQueue();
    mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
            inputChannel, mMessageQueue);
}
```

`nativeInit()` 函数中先把 InputChannel 和 MessageQueue 对象转换成 native 层的对象，然后创建 NativeInputEventReceiver 对象并调用 `initialize()` 函数。

```java
// frameworks/base/core/jni/android_view_InputEventReceiver.cpp
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    if (inputChannel == NULL) {
        jniThrowRuntimeException(env, "InputChannel is not initialized.");
        return 0;
    }

    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    if (messageQueue == NULL) {
        jniThrowRuntimeException(env, "MessageQueue is not initialized.");
        return 0;
    }

    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    status_t status = receiver->initialize();
    if (status) {
        String8 message;
        message.appendFormat("Failed to initialize input event receiver.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
        return 0;
    }

    receiver->incStrong(gInputEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
```

`initialize()` 函数中调用了 `setFdEvents()` 函数，在这里与 server 端注册类似，先获取 InputChannel 对象的文件描述符，然后调用 `addFd()` 函数加入到 Looper 对象中。这样 Client 端的 InputChannel 注册完成。

注意，mMessageQueue 对象从 Java 层传过来的，当时调用 `Looper.myLooper()` 的 `getQueue()` 方法返回的。因此，client 端中处理 input 事件的是调用 `Looper.myLooper()` 的线程，即调用 `ViewRootImpl.setView()` 的线程，而 `setView()` 是在 `WindowManager.addView()` 时调用。

```java
// frameworks/base/core/jni/android_view_InputEventReceiver.cpp
status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}

void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
}}}
```

# 2. Set the focused window to InputDispatcher

调用 Session 类的 `addToDisplay()` 方法后，在 WindowManagerService 会新建 IWindow 对应的 WindowState 对象。之后会调用 ViewRootImpl 的 `performTraversals()` 方法。该方法中又会调用 `relayoutWindow()` 方法，这个方法最终会调用 WindowManagerService 的 `relayoutWindow()` 方法。它会调用 `mInputMonitor.updateInputWindowsLw()` 更新 InputDispatcher 中的 focused window。

WindowManagerService 中当 focus 改变的时候，会调用 InuptMonitor 类的 `updateInputWindowsLw()` 方法。 

## 2.1 InputMonitor#updateInputWindowsLw()

该方法首先调用 `scheduleUpdateInputWindows()` 方法，最后执行 Runnable 对象 mUpdateInputWindows 的 `run()` 方法。mUpdateInputWindows 的 `run()` 方法中调用 UpdateInputForAllWindowsConsumer 类的 `updateInputWindows()` 方法遍历所有 window。

```java
// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
/* Updates the cached window information provided to the input dispatcher. */
void updateInputWindowsLw(boolean force) {
    if (!force && !mUpdateInputWindowsNeeded) {
        return;
    }
    scheduleUpdateInputWindows();
}

private void scheduleUpdateInputWindows() {
    if (mDisplayRemoved) {
        return;
    }

    if (!mUpdateInputWindowsPending) {
        mUpdateInputWindowsPending = true;
        mHandler.post(mUpdateInputWindows);
}}

private final Runnable mUpdateInputWindows = new Runnable() {
    @Override
    public void run() {
        synchronized (mService.mGlobalLock) {
        ...
        // Add all windows on the default display.
            mUpdateInputForAllWindowsConsumer.updateInputWindows(inDrag);
}}};

private final class UpdateInputForAllWindowsConsumer implements Consumer<WindowState> {
    private void updateInputWindows(boolean inDrag) {
    Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "updateInputWindows");
    ...
    mDisplayContent.forAllWindows(this,
            true /* traverseTopToBottom */);
    ...
    if (mApplyImmediately) {
        mInputTransaction.apply();
    } else {
        mDisplayContent.getPendingTransaction().merge(mInputTransaction);
        mDisplayContent.scheduleAnimation();
    }

    Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
}}
```

遍历所有的 WindowState 对象，然后调用 `populateInputWindowHandle()` 填充 inputWindowHandle，如果该 window 如果获得了焦点，在赋值给 mFocusedInputWindowHandle 变量，其实这个变量没有在其他地方使用，应该算是一个遗留的无用代码。

```java
// frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java
private final class UpdateInputForAllWindowsConsumer implements Consumer<WindowState> {
    @Override
    public void accept(WindowState w) {
        final InputChannel inputChannel = w.mInputChannel;
        final InputWindowHandle inputWindowHandle = w.mInputWindowHandle;
        if (inputChannel == null || inputWindowHandle == null || w.mRemoved
                || w.cantReceiveTouchInput()) {
            if (w.mWinAnimator.hasSurface()) {
                mInputTransaction.setInputWindowInfo(
                        w.mWinAnimator.mSurfaceController.mSurfaceControl, mInvalidInputWindow);
            }
            // Skip this window because it cannot possibly receive input.
            return;
        }

        final int flags = w.mAttrs.flags;
        final int privateFlags = w.mAttrs.privateFlags;
        final int type = w.mAttrs.type;
        final boolean hasFocus = w.isFocused();
        final boolean isVisible = w.isVisibleLw();
        ...

        if (w.inPinnedWindowingMode()) {
            ...
        }
        ...
        // If there's a drag in progress and 'child' is a potential drop target,
        // make sure it's been told about the drag
        if (inDrag && isVisible && w.getDisplayContent().isDefaultDisplay) {
            mService.mDragDropController.sendDragStartedIfNeededLocked(w);
        }

        populateInputWindowHandle(
                inputWindowHandle, w, flags, type, isVisible, hasFocus, hasWallpaper);

        if (w.mWinAnimator.hasSurface()) {
            mInputTransaction.setInputWindowInfo(
                    w.mWinAnimator.mSurfaceController.mSurfaceControl, inputWindowHandle);
}}}
```

填充 inputWindowHandle 后调用 SurfaceControl.Transaction 类的 `setInputWindowInfo()` 方法。它会调用 `nativeSetInputWindowInfo()` 方法。

在 JNI 层的 `nativeSetInputWindows()` 函数先获取 NativeInputWindowHandle 后更新属性，最后调用 `setInputWindowInfo()` 函数更新 layer_state_t 的 inputInfo 属性。

```java
// frameworks/base/core/jni/android_view_SurfaceControl.cpp
static void nativeSetInputWindowInfo(JNIEnv* env, jclass clazz, jlong transactionObj,
        jlong nativeObject, jobject inputWindow) {
    auto transaction = reinterpret_cast<SurfaceComposerClient::Transaction*>(transactionObj);

    sp<NativeInputWindowHandle> handle = android_view_InputWindowHandle_getHandle(
            env, inputWindow);
    handle->updateInfo();

    SurfaceControl* const ctrl = reinterpret_cast<SurfaceControl *>(nativeObject);
    transaction->setInputWindowInfo(ctrl, *handle->getInfo());
}

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

在 UpdateInputForAllWindowsConsumer  的 `updateInputWindows()` 方法的最后部分，如果 mApplyImmediately 是 true，则调用 `mInputTransaction.apply()`，否则把 mInputTransaction 合并到 mPendingTransaction，下次一起 apply。

## 2.2 InputDispatcher::setInputWindows()

`populateInputWindowHandle()` 方法是 Android 10 中新增的，而在 Android 9 中会调用 `addInputWindowHandle()` 方法把所有 inputWindowHandle 存到 mInputWindowHandles 变量中。然后调用 InpurManagerService 类的 `nativeSetInputWindows()` 方法，JNI 层的对应函数会调用 `InputDispatcher::setInputWindows()` 函数。

而在 Android 10 中先更新 native 层的 InputWindowHandle，然后通知 SurfaceFlinger apply 一次 transaction 来设置 Layer::eInputInfoChanged。当下一次 VSYNC 信号到来时，在 `onMessageReceived()` 回调函数中，SurfaceFlinger 调用 `handleMessageTransaction()` 函数后最终在 `handleTransactionLocked()` 函数中给 mInputInfoChanged 设置为 true，然后调用 `updateInputFlinger()` 函数。

```java
// frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
void SurfaceFlinger::handleTransactionLocked(uint32_t transactionFlags)
{
    if ((transactionFlags & eTraversalNeeded) || mTraversalNeededMainThread) {
        mCurrentState.traverseInZOrder([&](Layer* layer) {
            uint32_t trFlags = layer->getTransactionFlags(eTransactionNeeded);
            if (!trFlags) return;

            const uint32_t flags = layer->doTransaction(0);
            if (flags & Layer::eVisibleRegion)
                mVisibleRegionsDirty = true;

            if (flags & Layer::eInputInfoChanged) {
                mInputInfoChanged = true;
            }
        });
        mTraversalNeededMainThread = false;
    }
    ...
}

void SurfaceFlinger::updateInputFlinger() {
    ATRACE_CALL();
    if (!mInputFlinger) {
        return;
    }
    if (mVisibleRegionsDirty || mInputInfoChanged) {
        mInputInfoChanged = false;
        updateInputWindowInfo();
    } else if (mInputWindowCommands.syncInputWindows) {
        // If the caller requested to sync input windows, but there are no
        // changes to input windows, notify immediately.
        setInputWindowsFinished();
    }
    executeInputWindowCommands();
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

IInputFlinger 的 server 端是 InputManager，它又会调用 InputDispatcher 的 `setInputWindows()` 函数，传入的第二个参数是 displayId。

```java
// frameworks/native/services/inputflinger/InputManager.cpp
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
}}
```

对比 Android 9 和 Android 10 中更新 input window 的路径：

- Android 9: InputMonitor -> InputManagerService -> jni InputManagerService -> InputDispatcher
- Android 10: InputMonitor -> SurfaceControl -> SurfaceFlinger -> IInputFlinger -> InputManager -> InputDispatcher

不管是 Android 9 还是 Android 10 都会调用 InputDispatcher 的 `setInputWindows()` 函数。

`setInputWindows()` 函数简单说是，遍历所有 InputWindowHandle，找到获得焦点的 InputWindowHandle，具体有一下几个部分：

1. 遍历 inputWindowHandles，其中把无效的或 not touchable 或 not focusable 或无 input channel 的去除后存到 newHandles 变量，然后更新 mWindowHandlesByDisplay。
2. 遍历 newHandles，找到第一个 hasFocus 的 InputWindowHandle，存到 newFocusedWindowhandle 变量。
3. 如果 `oldFocusedWindowHandle != newFocusedWindowHandle` 为 true，先对 oldFocusedWindowHandle 绑定的 InputChannel 发送取消事件 `CANCEL_NON_POINTER_EVENTS`，然后更新 mFocusedWindowHandlesByDisplay，最后回调 `onFocusChangedLocked()`。
4. 向已经不存在的 touchedWindow 的 InputChannel 发送取消事件 `CANCEL_POINTER_EVENTS`。
5. 释放已经不存在的 oldWindowHandle 的 input channel。
6. 唤醒 Lopper。

```
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
/**
    * Called from InputManagerService, update window handle list by displayId that can receive input.
    * A window handle contains information about InputChannel, Touch Region, Types, Focused,...
    * If set an empty list, remove all handles from the specific display.
    * For focused handle, check if need to change and send a cancel event to previous one.
    * For removed handle, check if need to send a cancel event if already in touch.
    */
void InputDispatcher::setInputWindows(const std::vector<sp<InputWindowHandle>>& inputWindowHandles,
                                        int32_t displayId,
                                        const sp<ISetInputWindowsListener>& setInputWindowsListener) {
    { // acquire lock
        std::scoped_lock _l(mLock);

        // Copy old handles for release if they are no longer present.
        const std::vector<sp<InputWindowHandle>> oldWindowHandles =
                getWindowHandlesLocked(displayId);
        sp<InputWindowHandle> newFocusedWindowHandle = nullptr;
        if (inputWindowHandles.empty()) {
            // Remove all handles on a display if there are no windows left.
            mWindowHandlesByDisplay.erase(displayId);
        } else {
            // Since we compare the pointer of input window handles across window updates, we need
            // to make sure the handle object for the same window stays unchanged across updates.
            const std::vector<sp<InputWindowHandle>>& oldHandles =
                    mWindowHandlesByDisplay[displayId];
            std::unordered_map<sp<IBinder>, sp<InputWindowHandle>, IBinderHash> oldHandlesByTokens;
            for (const sp<InputWindowHandle>& handle : oldHandles) {
                oldHandlesByTokens[handle->getToken()] = handle;
            }
            // --1--
            std::vector<sp<InputWindowHandle>> newHandles;
            for (const sp<InputWindowHandle>& handle : inputWindowHandles) {
                ... // skip some invalid handles
                if (oldHandlesByTokens.find(handle->getToken()) != oldHandlesByTokens.end()) {
                    const sp<InputWindowHandle> oldHandle =
                            oldHandlesByTokens.at(handle->getToken());
                    oldHandle->updateFrom(handle);
                    newHandles.push_back(oldHandle);
                } else {
                    newHandles.push_back(handle);
            }}
            // --2--
            for (const sp<InputWindowHandle>& windowHandle : newHandles) {
                // Set newFocusedWindowHandle to the top most focused window instead of the last one
                if (!newFocusedWindowHandle && windowHandle->getInfo()->hasFocus &&
                    windowHandle->getInfo()->visible) {
                    newFocusedWindowHandle = windowHandle;
            }}
        // Insert or replace
        mWindowHandlesByDisplay[displayId] = newHandles;
        }
        // --3--
        sp<InputWindowHandle> oldFocusedWindowHandle =
                getValueByKey(mFocusedWindowHandlesByDisplay, displayId);
        if (oldFocusedWindowHandle != newFocusedWindowHandle) {
            if (oldFocusedWindowHandle != nullptr) {
                sp<InputChannel> focusedInputChannel =
                        getInputChannelLocked(oldFocusedWindowHandle->getToken());
                if (focusedInputChannel != nullptr) {
                    CancelationOptions options(CancelationOptions::CANCEL_NON_POINTER_EVENTS,
                                                "focus left window");
                    synthesizeCancelationEventsForInputChannelLocked(focusedInputChannel, options);
                }
                mFocusedWindowHandlesByDisplay.erase(displayId);
            }
            if (newFocusedWindowHandle != nullptr) {
                mFocusedWindowHandlesByDisplay[displayId] = newFocusedWindowHandle;
            }
            if (mFocusedDisplayId == displayId) {
                onFocusChangedLocked(oldFocusedWindowHandle, newFocusedWindowHandle);
        }}

        // --4-- Cancel pointer-events to the touched windows that removed
        ssize_t stateIndex = mTouchStatesByDisplay.indexOfKey(displayId);
        if (stateIndex >= 0) {
            TouchState& state = mTouchStatesByDisplay.editValueAt(stateIndex);
            for (size_t i = 0; i < state.windows.size();) {
                TouchedWindow& touchedWindow = state.windows[i];
                if (!hasWindowHandleLocked(touchedWindow.windowHandle)) {
                    sp<InputChannel> touchedInputChannel =
                            getInputChannelLocked(touchedWindow.windowHandle->getToken());
                    if (touchedInputChannel != nullptr) {
                        CancelationOptions options(CancelationOptions::CANCEL_POINTER_EVENTS,
                                                    "touched window was removed");
                        synthesizeCancelationEventsForInputChannelLocked(touchedInputChannel,
                                                                            options);
                    }
                    state.windows.erase(state.windows.begin() + i);
                } else {
                    ++i;
        }}}
        // --5--
        // Release information for windows that are no longer present.
        // This ensures that unused input channels are released promptly.
        // Otherwise, they might stick around until the window handle is destroyed
        // which might not happen until the next GC.
        for (const sp<InputWindowHandle>& oldWindowHandle : oldWindowHandles) {
            if (!hasWindowHandleLocked(oldWindowHandle)) {
                oldWindowHandle->releaseChannel();
    }}} // release lock
    // --6--
    // Wake up poll loop since it may need to make new input dispatching choices.
    mLooper->wake();
}
```

# 3. Dispatch input key event

## 3.1 Send an event to the target InputChannels

### 3.1.1 InputDispatcherThread

InputDispatcherThread 继承 Thread，是用于无限排队并分发 input 事件。它在 InputManager 的 `initialize()` 函数中创建，在 `start()` 函数中执行。

分发 input 事件调用 InputDispatcher 的 `dispatcherOnce()` 函数。

```java
// frameworks/native/services/inputflinger/InputManager.cpp
void InputManager::initialize() {
    mReaderThread = new InputReaderThread(mReader);
    mDispatcherThread = new InputDispatcherThread(mDispatcher);
}

status_t InputManager::start() {
    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
    ...
    return OK;
}

// frameworks/native/services/inputflinger/dispatcher/include/InputDispatcherThread.h
class InputDispatcherThread : public Thread

// frameworks/native/services/inputflinger/dispatcher/InputDispatcherThread.cpp
bool InputDispatcherThread::threadLoop() {
    mDispatcher->dispatchOnce();
    return true;
}
```

### 3.1.2 InputDispatcher::dispatchOnceInnerLocked()

`dispatchOnce()` 函数会调用 `dispatchOnceInnerLocked()` 函数。从 mInboundQueue 中获取一个要分发的 EventEntry。一共有 4 种 event type，我们以 TYPE_KEY 为例来说明。TYPE_KEY 的处理调用 `dispatchKeyLocked()` 函数。

inputTargets 中保存着需要分发的目标，InputTarget 中有需要分发的 inputChannel 变量。

调用 `findFocusedWindowTargetsLocked()` 函数填充 inputTargets，然后调用 dispatchEventLocke() 函数分发 key event。

```java
// frameworks/native/services/inputflinger/InputDispatcher.cpp
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    ...
    // If dispatching is frozen, do not process timeouts or try to deliver any new events.
    if (mDispatchFrozen) {
        return;
    }
    ...
    // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (!mPendingEvent) {
        ...
        // Inbound queue has at least one entry.
        mPendingEvent = mInboundQueue.dequeueAtHead();
        ...
        // Get ready to dispatch the event. Reset input target wait timeout.
        resetANRTimeoutsLocked();
    }
    ...
    switch (mPendingEvent->type) {
        case EventEntry::TYPE_CONFIGURATION_CHANGED: { ... }
        case EventEntry::TYPE_DEVICE_RESET: { ... }
        case EventEntry::TYPE_KEY: {
            KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
            done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        }
        case EventEntry::TYPE_MOTION: { ... }
}}

bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ...
    // Identify targets.
    std::vector<InputTarget> inputTargets;
    int32_t injectionResult =
            findFocusedWindowTargetsLocked(currentTime, entry, inputTargets, nextWakeupTime);
    ...
    // Add monitor channels from event's or focused display.
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(entry));
    // Dispatch the key.
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

`findFocusedWindowTargetsLocked()` 函数中，先根据 displayId 获取 focusedWindowHandle，然后检查权限等操作，最后调用 `addWindowTargetLocked()` 函数把 focusedWindowHandle 加到 inputTargets 中。

```java
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
                                                        const EventEntry* entry,
                                                        std::vector<InputTarget>& inputTargets,
                                                        nsecs_t* nextWakeupTime) {
    int32_t injectionResult;
    std::string reason;

    int32_t displayId = getTargetDisplayId(entry);
    sp<InputWindowHandle> focusedWindowHandle =
            getValueByKey(mFocusedWindowHandlesByDisplay, displayId);
    sp<InputApplicationHandle> focusedApplicationHandle =
            getValueByKey(mFocusedApplicationHandlesByDisplay, displayId);
    // If there is no currently focused window and no focused application
    // then drop the event.
    ...
    // Check permissions.
    ...
    // Check whether the window is ready for more input.
    ...
    // Success!  Output targets.
    injectionResult = INPUT_EVENT_INJECTION_SUCCEEDED;
    addWindowTargetLocked(focusedWindowHandle,
                            InputTarget::FLAG_FOREGROUND | InputTarget::FLAG_DISPATCH_AS_IS,
                            BitSet32(0), inputTargets);
    ...
    return injectionResult;
}
```

### 3.1.3 InputDispatcher::dispatchEventLocked()

`dispatchEventLocked()` 中遍历所有 inputTargets，根据 inputChannel 获取 connection，然后调用 `prepareDispatchCycleLocked()` 函数。这个函数中，如果针对 FLAG_SPLIT 的 motion event 生成 splitMotionEntry，但最后都会调用 `enqueueDispatchEntriesLocked()` 函数。

```java
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime, EventEntry* eventEntry,
                                            const std::vector<InputTarget>& inputTargets) {
    ...
    for (const InputTarget& inputTarget : inputTargets) {
        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
}}}

void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
                                                    const sp<Connection>& connection,
                                                    EventEntry* eventEntry,
                                                    const InputTarget* inputTarget) {
    bool wasEmpty = connection->outboundQueue.isEmpty();

    // Enqueue dispatch entries for the requested modes.
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                                InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                                InputTarget::FLAG_DISPATCH_AS_OUTSIDE);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                                InputTarget::FLAG_DISPATCH_AS_HOVER_ENTER);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                                InputTarget::FLAG_DISPATCH_AS_IS);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                                InputTarget::FLAG_DISPATCH_AS_SLIPPERY_EXIT);
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
                                InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER);
    // If the outbound queue was previously empty, start the dispatch cycle going.
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
}}
```

`enqueueDispatchEntriesLocked()` 函数中，先调用 `enqueueDispatchEntryLocked()` 函数，在这先创建 DispatchEntry 后 enqueue 到 outboundQueue 中。outboundQueue 是需要分发的 event 队列。

```java
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
void InputDispatcher::enqueueDispatchEntryLocked(const sp<Connection>& connection,
                                                    EventEntry* eventEntry,
                                                    const InputTarget* inputTarget,
                                                    int32_t dispatchMode) {
    int32_t inputTargetFlags = inputTarget->flags;
    if (!(inputTargetFlags & dispatchMode)) {
        return;
    }
    ...
    // This is a new event.
    // Enqueue a new dispatch entry onto the outbound queue for this connection.
    DispatchEntry* dispatchEntry =
            new DispatchEntry(eventEntry, // increments ref
                                inputTargetFlags, inputTarget->xOffset, inputTarget->yOffset,
                                inputTarget->globalScaleFactor, inputTarget->windowXScale,
                                inputTarget->windowYScale);
    ...
    // Remember that we are waiting for this dispatch to complete.
    if (dispatchEntry->hasForegroundTarget()) {
        incrementPendingForegroundDispatches(eventEntry);
    }

    // Enqueue the dispatch entry.
    connection->outboundQueue.enqueueAtTail(dispatchEntry);
}
```

把 DispatchEntry 加入到队列后调用 `startDispatchCycleLocked()` 函数。

`startDispatchCycleLocked()` 函数中遍历 outboundQueue，根据 event type 调用不同的分发函数。对于 TYPE_KEY 调用 `publishKeyEvent()` 函数。

每发送一个 EventEntry 后，从 outboundQueue 取出，并加入到 waitQueue 中。waitQueue 是用于等待从 app 进程发回的 finished signal。

```java
// frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
                                                const sp<Connection>& connection) {
    while (connection->status == Connection::STATUS_NORMAL &&
            !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;

        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
            case EventEntry::TYPE_KEY: {
                KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);

                // Publish the key event.
                status = connection->inputPublisher
                                    .publishKeyEvent(dispatchEntry->seq, keyEntry->deviceId,
                                                    keyEntry->source, keyEntry->displayId,
                                                    dispatchEntry->resolvedAction,
                                                    dispatchEntry->resolvedFlags, keyEntry->keyCode,
                                                    keyEntry->scanCode, keyEntry->metaState,
                                                    keyEntry->repeatCount, keyEntry->downTime,
                                                    keyEntry->eventTime);
                break;
            case EventEntry::TYPE_MOTION: { ... }
        }
        ...
        // Re-enqueue the event on the wait queue.
        connection->outboundQueue.dequeue(dispatchEntry);
        traceOutboundQueueLength(connection);
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        traceWaitQueueLength(connection);
}}
```

`publicKeyEvent()` 函数中根据传进来的参数生成 InputMessage，然后调用 InputChannel 的 `sendMessage()` 发送 InputMessage。`sendMessage()` 函数就是调用 socket 的 `send()` 函数把数据传过去。

```java
// frameworks/native/libs/input/InputTransport.cpp
status_t InputPublisher::publishKeyEvent(
        uint32_t seq,
        int32_t deviceId,
        int32_t source,
        int32_t action,
        int32_t flags,
        int32_t keyCode,
        int32_t scanCode,
        int32_t metaState,
        int32_t repeatCount,
        nsecs_t downTime,
        nsecs_t eventTime) {
    InputMessage msg;
    msg.header.type = InputMessage::TYPE_KEY;
    msg.body.key.seq = seq;
    msg.body.key.deviceId = deviceId;
    msg.body.key.source = source;
    msg.body.key.displayId = displayId;
    msg.body.key.action = action;
    msg.body.key.flags = flags;
    msg.body.key.keyCode = keyCode;
    msg.body.key.scanCode = scanCode;
    msg.body.key.metaState = metaState;
    msg.body.key.repeatCount = repeatCount;
    msg.body.key.downTime = downTime;
    msg.body.key.eventTime = eventTime;
    return mChannel->sendMessage(&msg);
}

status_t InputChannel::sendMessage(const InputMessage* msg) {
    const size_t msgLength = msg->size();
    InputMessage cleanMsg;
    msg->getSanitizedCopy(&cleanMsg);
    ssize_t nWrite;
    do {
        nWrite = ::send(mFd, &cleanMsg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
    } while (nWrite == -1 && errno == EINTR);
    ...
    return OK;
}
```

这样，InputDispatcher 完成了从 mInboundQueue 中获取一个 EventEntry 并向目标 InputChannel 发送 event。

<figure>
    <a href="/images/dispatch_input.jpg"><img src="/images/dispatch_input.jpg" alt="Dispatch input event"></a>
    <figcaption>Dispatch input event</figcaption>
</figure>

## 3.2 Consume events and callback

在「1.3 Register input channel of client」中调用 `setFdEvents()` 函数向 Looper 注册了 ALOOPER_EVENT_INPUT。从「[Android 消息机制 2-Handler（Native 层）](http://gityuan.com/2015/12/27/handler-message-native/)」可知，在 Looper 的 `pollInner()` 函数中会回调 callback 的 `handleEvent()` 函数，在我们这里 callback 就是 NativeInputEventReceiver。

### 3.2.1 NativeInputEventReceiver::consumeEvents()

NativeInputEventReceiver 的 `handleEvent()` 函数中，如果 events 是 ALOOPER_EVENT_INPUT，则调用 `consumeEvents()` 函数。

`consumeEvents` 会调用 InputConsumer 的 `consume()` 函数获取 InputEvent 后转换成 Java 层的对象 inputEventObj，然后调用 Java 层的 InputEventReceiver 类的 `dispatchInputEvent()` 方法分发给 Java 层。

```java
// frameworks/base/core/jni/android_view_InputEvent_Receiver.cpp
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
    for (;;) {
        uint32_t seq;
        InputEvent* inputEvent;
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches, frameTime, &seq, &inputEvent);

        if (!skipCallbacks) {
            jobject inputEventObj;
            switch (inputEvent->getType()) {
            case AINPUT_EVENT_TYPE_KEY:
                inputEventObj = android_view_KeyEvent_fromNative(env,
                        static_cast<KeyEvent*>(inputEvent));
                break;

            case AINPUT_EVENT_TYPE_MOTION: {
                MotionEvent* motionEvent = static_cast<MotionEvent*>(inputEvent);
                if ((motionEvent->getAction() & AMOTION_EVENT_ACTION_MOVE) && outConsumedBatch) {
                    *outConsumedBatch = true;
                }
                inputEventObj = android_view_MotionEvent_obtainAsCopy(env, motionEvent);
                break;
            }}

            if (inputEventObj) {
                env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
                if (env->ExceptionCheck()) {
                    ALOGE("Exception dispatching input event.");
                    skipCallbacks = true;
                }
                env->DeleteLocalRef(inputEventObj);
        }}
        if (skipCallbacks) {
            mInputConsumer.sendFinishedSignal(seq, false);
}}
```

### 3.2.2 InputConsumer::consume()

上面 3.1 分析最后调用 InputChannel 的 `sendMessage()` 函数发送 event 数据，而 `receiveMessage()` 函数就是从 socket 中取数据。

如果取到的 event type 是 TYPE_KEY，则调用 `initializeKeyEvent()` 函数填充 keyEvent。

```java
// frameworks/native/libs/input/InputTransport.cpp
status_t InputConsumer::consume(InputEventFactoryInterface* factory,
        bool consumeBatches, nsecs_t frameTime, uint32_t* outSeq, InputEvent** outEvent,
        int32_t* displayId, int* motionEventType, int* touchMoveNumber, bool* flag) {
    // Fetch the next input message.
    // Loop until an event can be returned or no additional events are received.
    while (!*outEvent) {
        if (mMsgDeferred) {
            // mMsg contains a valid input message from the previous call to consume
            // that has not yet been processed.
            mMsgDeferred = false;
        } else {
            // Receive a fresh message.
            status_t result = mChannel->receiveMessage(&mMsg);
            if (result) {
                // Consume the next batched event unless batches are being held for later.
                if (consumeBatches || result != WOULD_BLOCK) {
                    result = consumeBatch(factory, frameTime, outSeq, outEvent);
                    if (*outEvent) {
                        break;
                }}
                return result;
        }}

        switch (mMsg.header.type) {
        case InputMessage::TYPE_KEY: {
            KeyEvent* keyEvent = factory->createKeyEvent();
            if (!keyEvent) return NO_MEMORY;

            initializeKeyEvent(keyEvent, &mMsg);
            *outSeq = mMsg.body.key.seq;
            *outEvent = keyEvent;
            break;
        }
        case InputMessage::TYPE_MOTION: { ... }
    }}
    return OK;
}
```

### 3.2.3 InputEventReceiver#dispatchInputEvent()

获取 keyEvent 之后调用 Java 层 InputEventReceiver 类的 `dispatchInputEvent()` 方法。

InputEventReceiver 是抽象类，在 `dispatchInputEvent()` 方法中回调 InputEventReceiver 的 `onInputEvent()` 方法。ViewRootImpl 的内部类 WindowInputEventReceiver 实现了 InputEventReceiver。

在「1.3 Register input channel of client」中说过，在 `setView()` 方法中会创建 WindowInputEventReceiver 对象时传入了 InputChannel 对象。因此，当该 window 是 target 的时候会回调 `onInputEvent()` 方法。

它在 `onInputEvent()` 方法中会调用 `enqueueInputEvent()` 方法。这个方法中用对象生成 QueuedInputEvent 对象后加入链表。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
final class WindowInputEventReceiver extends InputEventReceiver {
    @Override
    public void onInputEvent(InputEvent event, int displayId) {
        enqueueInputEvent(event, this, 0, true);
}}

/* Input event queue.
    * Pending input events are input events waiting to be delivered to the input stages
    * and handled by the application.
    */
QueuedInputEvent mPendingInputEventHead;
QueuedInputEvent mPendingInputEventTail;
int mPendingInputEventCount;

void enqueueInputEvent(InputEvent event,
        InputEventReceiver receiver, int flags, boolean processImmediately) {
    QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
    // Always enqueue the input event in order, regardless of its time stamp.
    // We do this because the application or the IME may inject key events
    // in response to touch events and we want to ensure that the injected keys
    // are processed in the order they were received and we cannot trust that
    // the time stamp of injected events are monotonic.
    QueuedInputEvent last = mPendingInputEventTail;
    if (last == null) {
        mPendingInputEventHead = q;
        mPendingInputEventTail = q;
    } else {
        last.mNext = q;
        mPendingInputEventTail = q;
    }
    mPendingInputEventCount += 1;
    Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);
    if (processImmediately) {
        doProcessInputEvents();
    } else {
        scheduleProcessInputEvents();
}}
```

`doProcessInputEvents()` 方法中会遍历 pending input event 链表，调用 `deliverInputEvent()` 方法分发 QueuedInputEvent。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
void doProcessInputEvents() {
    // Deliver all pending input events in the queue.
    while (mPendingInputEventHead != null) {
        QueuedInputEvent q = mPendingInputEventHead;
        mPendingInputEventHead = q.mNext;
        if (mPendingInputEventHead == null) {
            mPendingInputEventTail = null;
        }
        q.mNext = null;

        mPendingInputEventCount -= 1;
        Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                mPendingInputEventCount);

        long eventTime = q.mEvent.getEventTimeNano();
        long oldestEventTime = eventTime;
        if (q.mEvent instanceof MotionEvent) {
            MotionEvent me = (MotionEvent)q.mEvent;
            if (me.getHistorySize() > 0) {
                oldestEventTime = me.getHistoricalEventTimeNano(0);
            }
        }
        mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

        deliverInputEvent(q);
    }

    // We are done processing all input events that we can process right now
    // so we can clear the pending flag immediately.
    if (mProcessInputEventsScheduled) {
        mProcessInputEventsScheduled = false;
        mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
    }
}
```

`deliverInputEvent()` 方法中首先找到合适的 InputStage。如果找到合适的 InputStage了，先调用 `handleWindowFocusChanged()` 方法更新 focused 状态，然后调用 InputStage 的 `deliver()` 方法分发事件。

```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
private void deliverInputEvent(QueuedInputEvent q) {
    Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
            q.mEvent.getSequenceNumber());
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
    }

    InputStage stage;
    if (q.shouldSendToSynthesizer()) {
        stage = mSyntheticInputStage;
    } else {
        stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
    }

    if (q.mEvent instanceof KeyEvent) {
        mUnhandledKeyManager.preDispatch((KeyEvent) q.mEvent);
    }

    if (stage != null) {
        handleWindowFocusChanged();
        stage.deliver(q);
    } else {
        finishInputEvent(q);
    }
}
```

InputStage 是处理 input event 责任链的抽象类，调用 `deliver()` 后把 input event 发送给 InputStage，然后它处理后可以发送给下一个 InputStage 或者结束这次分发。所有需要处理的 InputStage 都处理完后，调用 `finishInputEvent()` 方法。

在 `finishInputEvent()` 方法中会向 InputDispatcher 发送处理完成的 signal，这时会从 waitQueue 中移除对应的 input event，然后启动下一个 dispatch。

「[Input 系统—事件处理全过程](http://gityuan.com/2016/12/31/input-ipc/)」有个整体框架图如下：

<figure>
    <a href="/images/input_summary.jpg"><img src="/images/input_summary.jpg" alt="Data flow in input system"></a>
    <figcaption>Data flow in input system</figcaption>
</figure>

### Reference
{:.no_toc}

- [Android 应用程序键盘（Keyboard）消息处理机制分析](https://blog.csdn.net/luoshengyang/article/details/6882903)
- [Input系统—InputDispatcher线程](http://gityuan.com/2016/12/17/input-dispatcher/)
- [Input 系统—事件处理全过程](http://gityuan.com/2016/12/31/input-ipc/)