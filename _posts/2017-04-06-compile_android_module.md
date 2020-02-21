---
layout: post
title: "编译 Android 模块及调试"
date: 2017-04-08
modified: 2019-04-03
tags: [Android, Build]
categories: [Developer]
excerpt: 首先加载编译环境，然后编译 Android 系统的单个模块，最后替换模块。
description: 首先加载编译环境，然后编译 Android 系统的单个模块，最后替换模块。
---

> 欢迎转载，转载请注明出处 [xianzhu21.space](xianzhu21.space)。

## 编译单个模块

编译前需要加载编译环境，在 Android 源码目录下执行如下命令。
<!-- more -->

{% highlight shell %}
$ source build/envsetup.sh
{% endhighlight %}

之后要选择编译目标设备和编译版本。

{% highlight shell %}
$ lunch <product_name>-<build_variant>
{% endhighlight %}

目标设备可以在[这里](https://source.android.com/setup/build/running#selecting-device-build)查看，如果开发机是 Google 手机的话，例如 Google Pixel 的代码是 "aosp_sailfish"。编译版本有 3 个，分别是 "user"、"user-debug"、"eng"。user 是发布板，user-debug 是开发版，就是说 adb 可以获取 root 权限，eng 包含更多 debug 工具。我们在这里执行如下命令。

{% highlight shell %}
$ lunch aosp_sailfish-userdebug
{% endhighlight %}

最后执行编译命令，编译模块的命令有 ```make```、```mm```、```mmm```、```mma```、```mmma``` 等。

{% highlight shell %}
$ make <module-name>
{% endhighlight %}

```make``` 命令按照模块编译，而且会编译所有依赖的其他模块，因此如果没有对其他模块的修改尽量不要使用该命令，因为会特别慢。

以编译 telephony.jar 模块为例说明 ```mm```、```mmm```、```mma```、```mmma``` 命令。

{% highlight shell %}
$ cd frameworks/opt/telephony
$ mm
{% endhighlight %}

```mm``` 命令是编译当前目录，当然当前目录是可编译的才行。如果当前目录中有 Android.mk 文件就可编译。

{% highlight shell %}
$ mmm frameworks/opt/telephony
{% endhighlight %}

```mmm``` 命令是编译某个目录，结果与 ```mm``` 命令相同。

因为 ```mm``` 命令和 ```mmm``` 命令是只编译指定目录，不会编译依赖的模块，所以编译会很快。但如果没有编译过依赖，则会编译失败。这时可以使用 ```mma``` 命令和 ```mmma``` 命令，这两个命令会一同编译所有依赖。

如果编译成功，在编译的输出 log 中可以看到模块输出的目录。

{% highlight shell %}
...
Install: out/target/product/xxx/system/framework/telephony.jar
...
{% endhighlight %}

## 替换编译的模块

如果要替换模块，adb 需要获取 root 权限。如果是刷了自带 root 权限的 CM 等系统的话，在开发者模式中给 adb root 权限即可，否则需要刷入 userdebug 版本的 image。手机用 USB 连接电脑后在 terminal 中输入 ```$ adb root```。adb 拥有 root 权限后执行以下命令。

{% highlight shell %}
$ adb root # 如果该命令执行失败，说明 adb 无法获得 root 权限，无法替换模块
$ adb disable-verity # Android 7 之后需要输入该命令关闭 dm_verity
$ adb remount  # 重新挂在为可写
$ adb push out/target/product/xxx/system/framework/telephony.jar system/framework  # 用新编译的模块替换
{% endhighlight %}

最后重启。需要替换的模块在手机系统中所在的路径一般跟生成路径中 ```out/target/product/xxx/``` 之后的路径相同。也可以进入手机文件系统查看模块所在的路径。以 telephony.jar 为例。

{% highlight shell %}
$ adb shell
# ls system/framework/
{% endhighlight %}

## ODEX 优化

ODEX 优化指的是编译时提前把 class 文件转换成 odex 文件的操作。以 telephony.jar 为例，如果关闭 ODEX 优化 Java 代码会编译到 telephony.jar 文件中，而开启 ODEX 优化 Java 代码会编译到 /system/framework/oat/arm64/telephony.odex 文件中。

因此，如果开启了 ODEX 优化后替换 telephony.jar 文件不会生效。这时有两种方法让代码生效：一种是关闭 ROM 和模块的 ODEX 优化；另一种是替换 odex 文件。

### 关闭 ODEX 优化

### 替换 odex 文件

odex 文件中包含签名，所以不能直接替换 odex 文件。需要先用 ROM 里的 odex 文件中提取签名数据并写入到新 odex 文件才能正常启动。

## 调试

Android framework 的中添加 log 的时候，根据模块的不同可以使用 android.util.Log (Android 提供给 App 使用的 Log)，android.util.Slog 或 android.telephony.Rlog 输出 log 信息。打印 log 的命令如下。

{% highlight a %}
$ adb logcat [-b <buffer>]
{% endhighlight %}

如果没有 ```-b``` 参数，输出的是 Log 信息。如果 ```-b``` 参数是 ```radio```，输出的是 Rlog 信息。如果 ```-b``` 参数是 ```system``` ，输出的是 Slog 信息。因为 telephony.jar 是 telephony 模块，因此使用 Rlog 会比较好，因为这样就可以与系统的 telephony 模块的 log 信息一起查看，也过滤掉了其他的 log 信息。如果想查看特定类的 log 信息，可以使用 Linux shell 的 grep 命令，因为 log 方法中传进去的 tag 一般是类名。

{% highlight shell %}
$ adb logcat -b radio |grep RILJ
{% endhighlight %}

更多的 logcat 的使用方法可以参考官网的 [logcat 命令行工具](https://developer.android.google.cn/studio/command-line/logcat.html)部分。

<br>
**References:**

- [Preparing to Build](https://source.android.com/source/building)
- [build/envsetup.sh](https://android.googlesource.com/platform/build/+/master/envsetup.sh)
- [Android 调试桥](https://developer.android.google.cn/studio/command-line/adb.html)
- [logcat 命令行工具](https://developer.android.google.cn/studio/command-line/logcat.html)


end.