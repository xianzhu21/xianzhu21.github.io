---
layout: post
title: "编译Android模块及调试"
date: 2017-04-08
tags: [Android]
categories: [I'm a coder]
# excerpt: 介绍了Android中RILJ的req/resp和ind服务的机制，服务中使用的RILRequest数据结构与RILRegistrant的机制
description: 如何编译Android系统的单个模块，又如何替换新的模块及调试
---

> 欢迎转载，转载请注明出处 [xianzhuliu.github.io](xianzhuliu.github.io)。

## 编译单个模块

编译单个模块之前先要把整个Android源码编译一遍。

编译前需要加载编译环境，在Android源码目录下执行如下命令。

{% highlight shell %}
$ source build/envsetup.sh
{% endhighlight %}

之后要选择编译目标设备和编译版本。

{% highlight shell %}
$ lunch <product_name>-<build_variant>
{% endhighlight %}

目标设备可以在[这里](https://source.android.com/source/running)查看，如果开发机是Google手机的话，例如Google Pixel的代码是"aosp_sailfish"。编译版本有3个，分别是"user"、"user-debug"、"eng"。user 是发布板，user-debug 是开发版，就是说 adb 可以获取 root 权限，eng 包含更多 debug 工具 (没有用过)。因此，我们在这里执行如下命令。

{% highlight shell %}
$ lunch aosp_sailfish-userdebug
{% endhighlight %}

最后执行编译命令，编译模块的命令有```make```、```mm```、```mmm```等。

{% highlight shell %}
$ make <module-name>
{% endhighlight %}

```make```命令会编译所有依赖的其他模块，因此如果没有对其他模块的修改尽量不要使用该命令，因为会特别慢。

以编译 telephony.jar 模块为例说明```mm```和```mmm```命令。

{% highlight shell %}
$ cd frameworks/opt/telephony
$ mm
{% endhighlight %}

```mm```命令是编译当前目录，当然当前目录是可编译的才行。如果当前目录中有 Android.mk 文件就可编译。

{% highlight shell %}
$ mmm frameworks/opt/telephony
{% endhighlight %}

```mmm```命令是编译某个目录，结果与```mm```命令相同。

因为```mm```命令和```mmm```命令是只编译指定目录，所以编译会很快，基本在10分钟以内。```mm```命令和```mmm```命令如果有参数```-a```，则是重新编译所有依赖的模块，因此结果与```make```命令相同。

如果编译成功，在编译的输出 log 中可以看到模块输出的目录。

{% highlight shell %}
...
Install: out/target/product/xxx/system/framework/telephony.jar
...
{% endhighlight %}

## 替换编译的模块

如果要替换模块，adb 需要获取 root 权限。如果是刷了自带 root 权限的 CM 等系统的话，在开发者模式中给 adb root 权限即可，否则需要刷入 userdebug 版本的 image。手机用 USB 连接电脑后在 terminal 中输入```$ adb shell```，如果第一个字符是```#```，则说明 adb 拥有 root 权限 (如果是 userdebug 版本，可能需要```sudo```权限)。adb 拥有 root 权限后执行以下命令。

{% highlight shell %}
$ adb remount  # 设置手机文件系统为可写
$ adb push out/target/product/xxx/system/framework/telephony.jar system/framework  # 用新编译的模块替换
{% endhighlight %}

最后重启，就是漫长了应用更新了，时间根据手机中应用数的不同而不同。需要替换的模块在手机系统中所在的路径一般跟生成路径中```out/target/product/xxx/```之后的路径相同。也可以进入手机文件系统查看模块所在的路径。以 telephony.jar 为例。

{% highlight a %}
$ adb shell
# ls system/framework/
{% endhighlight %}

## 调试系统

Android framework 的中编写 log 代码的时候，根据模块的不同可以使用 android.util.Log (Android 提供给 App 使用的 Log)，android.util.Slog 或 android.telephony.Rlog 输出 log 信息。打印 log 的命令如下。

{% highlight a %}
$ adb logcat [-b <buffer>]
{% endhighlight %}

如果没有```-b```参数，输出的是 Log 信息。如果```-b```参数是```radio```，输出的是 Rlog 信息。如果```-b```参数是```system```，输出的是 Slog 信息。因为 telephony.jar 是 telephony 模块，因此使用 Rlog 会比较好，因为这样就可以与系统的 telephony 模块的 log 信息一起查看，也过滤掉了其他的 log 信息。如果想查看特定类的 log 信息，可以使用 Linux shell 的 grep 命令，因为 log 方法中传进去的 tag 一般是类名。

{% highlight shell %}
$ adb logcat -b radio |grep "RILJ"
{% endhighlight %}

更多的 logcat 的使用方法可以参考官网的 [logcat 命令行工具](https://developer.android.google.cn/studio/command-line/logcat.html)部分。

<br>
**References:**

- [Preparing to Build](https://source.android.com/source/building)
- [build/envsetup.sh](https://android.googlesource.com/platform/build/+/master/envsetup.sh)
- [Android 调试桥](https://developer.android.google.cn/studio/command-line/adb.html)
- [logcat 命令行工具](https://developer.android.google.cn/studio/command-line/logcat.html)


end.