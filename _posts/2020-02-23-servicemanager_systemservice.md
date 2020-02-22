---
layout: post
title: ServiceManager 与 SystemService
date: 2020-02-23
modified: 2020-02-23
tags: [Android,ServiceManager,SystemService]
categories: [Developer]
excerpt: SystemServer 是 system_server 进程的 Java 层入口，main() 方法中创建 SystemServer 对象，并执行 run()。run() 方法中创建 SystemServiceManager 对象...
description: SystemServer 是 system_server 进程的 Java 层入口，main() 方法中创建 SystemServer 对象，并执行 run()。run() 方法中创建 SystemServiceManager 对象...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

# 0. SystemServer.main()

SystemServer 是 system_server 进程的 Java 层入口，`main()` 方法中创建 SystemServer 对象，并执行 `run()`。`run()` 方法中创建 SystemServiceManager 对象，然后调用 `startBootstrapServices()`、`startCoreServices()`、`startOtherServices()`。

`startBootstrapServices()` 中创建 ActivityManagerService、PowerManagerService、PackageManagerService 等。

# 1. SystemService

SystemService 是在 system_server 进程中执行的服务的抽象类，它定义了一些回调接口，例如 `onStart()`、`onBootPhase()`、`onStartUser()` 等等。还有 `publicBinderService()` 实现了向 servicemanager 注册服务，`publicLocalService()` 实现了注册 system_server 本地服务。

系统服务不是必须继承 SystemService，关键是调用 ServiceManager 类的 `addService()` 向 servicemanager 注册服务及调用 ServiceManagerRegistry 类的 `registerService()` 向 client 端注册服务的代理对象。

# 2. PowerManagerService 的启动

PowerManagerService 是比较标准的系统服务实现方式，它继承了 SystemService。

PowerManagerService 是调用 SystemServiceManager 类的 `startService()` 方法创建并注册的。

```java
mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
```

看一下 `startService()` 方法的实现。首先反射调用构造器创建对象，然后调用 `startService(service)` 注册 Service。

```java
// com.android.server.SystemServiceManager
/**
    * Creates and starts a system service. The class must be a subclass of
    * {@link com.android.server.SystemService}.
    */
public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);
        final T service;
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } ...
        startService(service);
        return service;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
}}
```

`startService(service)` 就是把该 Service 添加到 `mServices` 中，然后调用 `onStart()`。

```java
// com.android.server.SystemServiceManager
public void startService(@NonNull final SystemService service) {
    mServices.add(service);
    long time = SystemClock.elapsedRealtime();
    try {
        service.onStart();
    } ...
}
```

# 3. PowerManagerService 的注册

那 PowerManager 如何调用到 PowerManagerService 的？ServiceManager 类是 servicemanager 进程的 client端，servicemanager 是管理所有 service 的程序，代码在 `framework/native/cmds/servicemanager/service_manager.c`。把一个服务注册到 servicemanager 中需要调用 ServiceManager 类的 `addService()`，看一下 PowerManagerService 类的 `onStart()` 的实现。

`onStart()` 中调用了 `publishBinderService()` 和 `publishLocalService()`，第一个就是往 servicemanager 注册服务的，第二个是注册本地 service，供 system_server 进程内调用。

```java
// frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java
public void onStart() {
    publishBinderService(Context.POWER_SERVICE, new BinderService());
    publishLocalService(PowerManagerInternal.class, new LocalService());

    Watchdog.getInstance().addMonitor(this);
    Watchdog.getInstance().addThread(mHandler);
}
```

`publishBinderService()` 和 `publishLocalService()` 都是 SystemService 的方法。第一个方法最终会调用 `ServiceManager#addService()`，第二个方法会调用 `LocalServices#addService()`。

`publishBinderService()` 的第二个参数中创建的 BinderService 类就是 IPowerManager 在 service 端的实现，就是说它实现了 IPowerManager.Stub。

```java
// frameworks/base/services/core/java/com/android/server/power/PowerManagerService.java
final class BinderService extends IPowerManager.Stub
```

IPowerManager 在 client 端的实现是在 PowerManager 的构造方法中传进来的，而这个是 SystemServiceRegistry 中调用 `reigsterService()` 方法时创建的匿名类的 `createService()` 方法中创建的。

```java
// frameworks/base/core/java/android/os/PowerManager.java
public final class PowerManager {
    final IPowerManager mService;
    public PowerManager(Context context, IPowerManager service, Handler handler) {
        mContext = context;
        mService = service;
        mHandler = handler;
}}

// frameworks/base/core/java/android/app/SystemServiceRegistry.java
static {
    registerService(Context.POWER_SERVICE, PowerManager.class,
        new CachedServiceFetcher<PowerManager>() {
    @Override
    public PowerManager createService(ContextImpl ctx) throws ServiceNotFoundException {
        IBinder b = ServiceManager.getServiceOrThrow(Context.POWER_SERVICE);
        IPowerManager service = IPowerManager.Stub.asInterface(b);
        return new PowerManager(ctx.getOuterContext(),
                service, ctx.mMainThread.getHandler());
}});}
```

# 4. addService, getService, call API

## addService()

添加服务最终要调用 ServiceManager 类的 `addService()`，需要传入 name 和 IBinder 对象（例如 IPowerManager.Stub 的实现类）。`addService()` 中会调用 IServiceManager 的 `addService()`，最终把 service 注册到 servicemanager 进程中。

```java
/* ADD SERVICE */
SystemServer.startBootstrapServices() -> PowerManagerService() ->
SystemService.publishBinderService() -> ServiceManager.addService() ->
IServiceManager.addService() -> servicemanager
```

## getService()

获取服务最终要调用 ServiceManager 类的 `getService()`，而它会调用 IServiceManager 的 `getService()` 获取 IBinder 对象。拿到 IBinder 对象后，调用 Stub 类的 `asInterface()` 获取 client 端的代理类。

```java
/* GET SERVICE */
SystemService
ContextImpl.getSystemService() -> SystemServiceRegistry.getSystemService() ->
CachedServiceFetcher.createService() -> ServiceManager.getServiceOrThrow() ->
ServiceManager.getService() -> ServiceManager.rawGetService() ->
getIServiceManager().getService() -> servicemanager
```

## call API

获取服务的代理对象后，会通过 `IBinder` 到服务所在进程，执行实际实现代码。

```java
/* CALL API */
PowerManager.wakeUp() -> IPowerManager.Stub.Proxy.wakeUp() -> IBinder.transact() ->
IPowerManager.Stub.onTransact() -> BinderService.wakeUp() ->
PowerManagerService.wakeUpInternal()
```

「[红茶一杯话 Binder（ServiceManager篇）](https://my.oschina.net/youranhongcha/blog/149578)」中的一张图说明了 app、system_server 和 servicemanager 三个进程的关系。

<figure>
    <a href="/images/servicemanager.png"><img src="/images/servicemanager.png" alt="App, system_server and servicemanager"></a>
    <figcaption>App, system_server and servicemanager</figcaption>
</figure>

# 5. ServiceManager, ServiceManagerRegistry, ServiceManagerNative, SystemServiceManger

ServiceManager 是 servicemanager 在 client 端的代理，拥有 IServiceManager 代理对象。SystemServer 调用 IServiceManager 的 `addService()` 把系统服务注册到 servicemanager。

ServiceManagerRegistry 就是保存各个系统服务在 client 端的代理对象的类。在 static 代码块中调用 `registerService()` 注册了服务。Client 端调用 `getSystemService()` 获取服务代理对象，最终会调用 IServiceManager 的 `getService()` 获取 IBinder 对象。

ServiceManagerNative 的作用只有一个，就是调用 `asInterface()` 获取 IServiceManager 的代理对象。代理对象是 ServiceManagerNative 的内部类 ServiceManagerProxy。

```java
// frameworks/base/core/java/android/os/ServiceManager.java
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    sServiceManager = ServiceManagerNative
            .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
    return sServiceManager;
}
```

SystemServiceManager 像类型所示的，就是用来管理 SystemService 的，当调用 `startService()` 的使用会通过反射调用构造方法创建 SystemService 对象，然后回调 `onStart()`。像 PowerManagerService，在 `onStart()` 的实现中调用了 `publisBinderService()` 向 servicemanager 注册自己。

## Reference

1. 「[红茶一杯话 Binder（ServiceManager篇）](https://my.oschina.net/youranhongcha/blog/149578)