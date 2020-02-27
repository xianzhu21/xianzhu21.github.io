---
layout: post
title: 修改 AlertDialog 的默认主题
date: 2019-07-30
modified: 2019-07-31
tags: [Android,Resource,Style]
categories: [Developer]
excerpt:  目前 AOSP 中有三个主题，分别是 themes.xml、themes_holo.xml、themes_materials.xml。其中 themes.xml 是 Android 4.0 以前使用的默认主题，themes_holo.xml 是 Android 4.0 ~ Android 4.4 使用的默认主题，themes_materials.xml 是 Android 5.0 之后使用的默认主题...
description: 目前 AOSP 中有三个主题，分别是 themes.xml、themes_holo.xml、themes_materials.xml。其中 themes.xml 是 Android 4.0 以前使用的默认主题，themes_holo.xml 是 Android 4.0 ~ Android 4.4 使用的默认主题，themes_materials.xml 是 Android 5.0 之后使用的默认主题...
---
<!-- more -->
> 欢迎转载，转载请注明出处 [xianzhu21.spaceo](xianzhu21.space)。

* TOC
{:toc}

# 1. 系统默认 Theme

目前 AOSP 中有三个主题，分别是 themes.xml、themes_holo.xml、themes_materials.xml。其中 themes.xml 是 Android 4.0 以前使用的默认主题，themes_holo.xml 是 Android 4.0 ~ Android 4.4 使用的默认主题，themes_materials.xml 是 Android 5.0 之后使用的默认主题。

还有一个文件叫 theme_device_defaults.xml，该文件的开头注释中写了如下文字，意思是，该文件中包含设备默认使用的主题，如果想修改系统默认主题，应该修改该文件，而不是 themes.xml 文件。

```xml
<!-- themes_device_defaults.xml -->
<!--
===============================================================
                        PLEASE READ
===============================================================
This file contains the themes that are the Device Defaults.
If you want to edit themes to skin your device, do it here.
We recommend that you do not edit themes.xml and instead edit
this file.

Editing this file instead of themes.xml will greatly simplify
merges for future platform versions and CTS compliance will be
easier.
===============================================================
                        PLEASE READ
===============================================================
    -->
```

# 2. AlertDialog 的默认主题

`AlertDialog` 的默认主题是如何设置的，先从 AlertDialog.java 代码开始分析。创建 `AlertDialog` 一般用内部类 `Builder` 来创建。实际的 `themeResId` 是 `resolveDialogTheme()` 方法返回的值，如果没有指定 `themeResId` 则会使用 `ResourceId.ID_NULL`。

在 `resolveDialogTheme()` 方法中如果没有指定 themeResId，当然 `ResourceId.ID_NULL` 也属于没用指定，则会使用当前主题的 `alertDialogTheme` 定义的 `AlertDialog` 主题。

```java
public class AlertDialog extends Dialog implements DialogInterface {
    public static class Builder {
        /**
            * Creates a builder for an alert dialog that uses the default alert
            * dialog theme.
            * <p>
            * The default alert dialog theme is defined by
            * {@link android.R.attr#alertDialogTheme} within the parent
            * {@code context}'s theme.
            *
            * @param context the parent context
            */
        public Builder(Context context) {
            this(context, resolveDialogTheme(context, ResourceId.ID_NULL));
    }}

    static @StyleRes int resolveDialogTheme(Context context, @StyleRes int themeResId) {
        if (themeResId == THEME_TRADITIONAL) {
            return R.style.Theme_Dialog_Alert;
        } else if (themeResId == THEME_HOLO_DARK) {
            return R.style.Theme_Holo_Dialog_Alert;
        } else if (themeResId == THEME_HOLO_LIGHT) {
            return R.style.Theme_Holo_Light_Dialog_Alert;
        } else if (themeResId == THEME_DEVICE_DEFAULT_DARK) {
            return R.style.Theme_DeviceDefault_Dialog_Alert;
        } else if (themeResId == THEME_DEVICE_DEFAULT_LIGHT) {
            return R.style.Theme_DeviceDefault_Light_Dialog_Alert;
        } else if (ResourceId.isValid(themeResId)) {
            // start of real resource IDs.
            return themeResId;
        } else {
            final TypedValue outValue = new TypedValue();
            context.getTheme().resolveAttribute(R.attr.alertDialogTheme, outValue, true);
            return outValue.resourceId;
}}}
```

`alertDialogTheme` 在哪定义的，就得看 themes_device_defaults.xml 了，其中有 `Theme.DeviceDefault` 和 `Theme.DeviceDefault.Light`。`Theme.DeviceDefault` 是暗色主题，一般的 Android 默认都不是暗色主题，所以我们看 `Theme.DeviceDefault.Light`。其中定义了 `alertDialogTheme`，它的值是 `@style/Theme.DeviceDefault.Light.Dialog.Alert`，且它的 parent 是 `Theme.Material.Light`。

```xml
<!-- themes_device_defaults.xml -->
<style name="Theme.DeviceDefault.Light" parent="Theme.Material.Light" >
    ...
    <!-- AlertDialog attributes -->
    <item name="alertDialogTheme">@style/Theme.DeviceDefault.Light.Dialog.Alert</item>
    <item name="alertDialogStyle">@style/AlertDialog.DeviceDefault.Light</item>
    ...
</style>

<style name="Theme.DeviceDefault.Light.Dialog.Alert" parent="Theme.Material.Light.Dialog.Alert">
    ...
    <!-- Dialog attributes -->
    <item name="dialogCornerRadius">@dimen/config_dialogCornerRadius</item>
    <item name="alertDialogTheme">@style/Theme.DeviceDefault.Light.Dialog.Alert</item>
    ...
</style>
```

style 标签有两种继承方法，一个是 name 用 `.` 分隔，一个是 parent 属性。例如 `Theme.DeviceDefault.Light` 是继承自 `Theme.DeviceDefault`，但是它还有 parent 属性 `Theme.Material.Light`。如果两个都存在则以 parent 为准，即 `Theme.DeviceDefault.Light` 实际继承 `Theme.Material.Light`。

继续跟一下 `Theme.Material.Light.Dialog.Alert`，它在 theme_materials.xml 中。它继承 `Theme.Material.Light.Dialog.BaseAlert`，而在这个 style 中只定义了 `windowMinWidthMajor` 和 `windowMinWidthMinor`，而它又自动继承了 `Theme.Material.Light`，在这个主题中定义了 `alertDialogStyle` 的值为 `AlertDialog.Material.Light`。

```xml
<!-- themes_device_defaults.xml -->
<!-- Material theme (light version). -->
<style name="Theme.Material.Light" parent="Theme.Light">
    ...
    <!-- AlertDialog attributes -->
    <item name="alertDialogTheme">@style/ThemeOverlay.Material.Dialog.Alert</item>
    <item name="alertDialogStyle">@style/AlertDialog.Material.Light</item>
    <item name="alertDialogCenterButtons">false</item>
    <item name="alertDialogIcon">@drawable/ic_dialog_alert_material</item>
    ...
</style>

<style name="Theme.Material.Light.Dialog.BaseAlert">
    <item name="windowMinWidthMajor">@dimen/dialog_min_width_major</item>
    <item name="windowMinWidthMinor">@dimen/dialog_min_width_minor</item>
</style>

<!-- Material light theme for alert dialog windows, which is used by the
        {@link android.app.AlertDialog} class.  This is basically a dialog
        but sets the background to empty so it can do two-tone backgrounds.
        For applications targeting Honeycomb or newer, this is the default
        AlertDialog theme. -->
<style name="Theme.Material.Light.Dialog.Alert" parent="Theme.Material.Light.Dialog.BaseAlert"/>
```

`AlertDialog.Material.Light` 定义在 styles_material.xml。其实 `AlertDialog.Material.Light` 没有定义任何 item，但它又继承自 `AlertDialog.Material`。在 `AlertDialog.Material` 中定义了 layout 属性，因此，`alert_dialog_material` 就是 `AlertDialog` 使用的 layout。

```xml
<!-- styles_device_defaults.xml -->
<!-- Dialog styles -->
<style name="AlertDialog.Material" parent="AlertDialog">
    ...
    <item name="layout">@layout/alert_dialog_material</item>
    <item name="listLayout">@layout/select_dialog_material</item>
    <item name="progressLayout">@layout/progress_dialog_material</item>
    <item name="horizontalProgressLayout">@layout/alert_dialog_progress_material</item>
    <item name="listItemLayout">@layout/select_dialog_item_material</item>
    <item name="multiChoiceItemLayout">@layout/select_dialog_multichoice_material</item>
    <item name="singleChoiceItemLayout">@layout/select_dialog_singlechoice_material</item>
    <item name="controllerType">@integer/config_alertDialogController</item>
    <item name="selectionScrollOffset">@dimen/config_alertDialogSelectionScrollOffset</item>
</style>

<style name="AlertDialog.Material.Light" />
```

# 3. 如何修改 AlertDialog 的默认 Theme

`AlertDialog` 使用的 layout 定义在 `AlertDialog.Material` 中，那直接修改 `alert_dialog_material` 就可以吗？这样修改也可以，但不建议，从 styles_material.xml 的注释中也可以看到，为了通过 CTS 不能修改该文件。它建议修改 styles_device_defaults.xml。

```xml
<!--
===============================================================
                        PLEASE READ
===============================================================

The Material themes must not be modified in order to pass CTS.
Many related themes and styles depend on other values defined in this file.
If you would like to provide custom themes and styles for your device,
please see styles_device_defaults.xml.

===============================================================
                        PLEASE READ
===============================================================
-->
```

在 themes_device_defaults.xml 中的默认主题 Theme.DeviceDefault.Light 定义了 `alertDialogStyle` 属性为 `@style/AlertDialog.DeviceDefault.Light`，它就是定义在 styles_device_defaults.xml，它的值也是 `AlertDialog.Material.Light`，那是不是直接替换这个 `AlertDialog.DeviceDefault.Light` 的值就可以？

```xml
<!-- themes_device_defaults.xml -->
<style name="Theme.DeviceDefault.Light" parent="Theme.Material.Light" >
    <!-- AlertDialog attributes -->
    <item name="alertDialogTheme">@style/Theme.DeviceDefault.Light.Dialog.Alert</item>
    <item name="alertDialogStyle">@style/AlertDialog.DeviceDefault.Light</item>
</style>

<!----------------------------------------------->

<!-- styles_device_defaults.xml -->
<!-- AlertDialog Styles -->
<style name="AlertDialog.DeviceDefault" parent="AlertDialog.Material"/>
<style name="AlertDialog.DeviceDefault.Light" parent="AlertDialog.Material.Light"/>
```

答案是否定的。

上面说过 `AlertDialog` 使用的默认主题是当前主题的 `alertDialogTheme` 属性指定的，即 `Theme.DeviceDefault.Light.Dialog.Alert`。虽然从命名方式看到它继承自 `Theme.DeviceDefault.Light`，但是它有 parent 属性，所以它继承 `Theme.DeviceDefault.Light` 是无效的，它实际继承的是 `Theme.Material.Light.Dialog.Alert`。而且也没用重写 `alertDialogStyle` 属性，因此，替换 styles_device_defaults.xml 中的 `AlertDialog.DeviceDefault.Light` 的值是没有效果的，因为根本没有用到 `Theme.DeviceDefault.Light`。

```xml
<!-- themes_device_defaults.xml -->
<style name="Theme.DeviceDefault.Light.Dialog.Alert" parent="Theme.Material.Light.Dialog.Alert">
    <!-- Dialog attributes -->
    <item name="dialogCornerRadius">@dimen/config_dialogCornerRadius</item>
    <item name="alertDialogTheme">@style/Theme.DeviceDefault.Light.Dialog.Alert</item>
</style>
```

那应该怎么改？很简单，在 `Theme.DeviceDefault.Light.Dialog.Alert` 中定义 `alertDialogStyle` 属性，覆盖掉 `Theme.Material.Light` 中定义的 `alertDialogStyle` 就可以了。实例代码如下所示。

```xml
<!-- themes_device_defaults.xml -->
<style name="Theme.DeviceDefault.Light.Dialog.Alert" parent="Theme.Material.Light.Dialog.Alert">
    <!-- Dialog attributes -->
    <item name="dialogCornerRadius">@dimen/config_dialogCornerRadius</item>
    <item name="alertDialogTheme">@style/Theme.DeviceDefault.Light.Dialog.Alert</item>

    <!-- Use the attributes designed by Xianzhu21 -->
    <item name="alertDialogStyle">@style/AlertDialog.X21</item>
</style>

<!----------------------------------------------->

<!-- styles_x21.xml -->
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- Dialog styles -->
    <style name="AlertDialog.X21" parent="AlertDialog.Material">
        <item name="layout">@layout/alert_dialog_x21</item>
    </style>
</resources>
```