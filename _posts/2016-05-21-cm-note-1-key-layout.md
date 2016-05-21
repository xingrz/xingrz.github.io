---
layout: post
category: 开发
title: CM 13.0 适配折腾笔记 1：按键布局修改
date:   2016-05-21 15:00:00 +0800
author: XiNGRZ
tags: [Android, CyanogenMod]
---

话说自[上次](/2016/2016-05-14/hello-cyanogen.html)入坑之后，这段时间为了「修正」Z5S 的触摸按键布局，又往深挖了一点点。这里写篇博客记录一下。

## 修改 Key Layout

按照[官方](https://source.android.com/devices/input/key-layout-files.html)文档的说法，厂商定义实体键时，**不应**直接修改 `Generic.kl`，而应该建一个以输入设备的名称为文件名的 `.kl` 文件。

如何知道这台手机的输入设备的名称呢？需要综合两处信息。

#### 其一，kernel 中的触屏驱动

触屏驱动源码位于 kernel 的 [`/drivers/input/touchscreen`](https://github.com/nx503a-dev/android_kernel_zte_nx503a/tree/cm-13.0/drivers/input/touchscreen) 目录下。由于同一个厂家可能几个不同型号共用了一套内核源码，所以我们还要找出它具体用的是哪个配置。

在 Z5S 的设备配置的 [`BoardConfig.mk`](https://github.com/nx503a-dev/android_device_zte_nx503a/blob/cm-13.0/BoardConfig.mk#L57) 中我们可以找到这么一行：

```mk
TARGET_KERNEL_CONFIG := msm8974-NX503A_defconfig
```

那么回到内核源码仓库，直奔 `/arch/arm/configs/` 目录，可以找到 [`msm8974-NX503A_defconfig`](https://github.com/nx503a-dev/android_kernel_zte_nx503a/blob/cm-13.0/arch/arm/configs/msm8974-NX503A_defconfig#L311) 这个文件。结合里面 `TOUCHSCREEN` 相关的字段和之前触屏源码目录里的 Makefile，可以得知 Z5S 有两种型号分别用到了 `synaptics_dsx` 和 `cyttsp4` 两种触摸屏。

#### 其二，真机上取得确切的设备名称

为什么我不直接写这个，而是要先翻内核。因为一开始我就只在我的 32G 版本上找到了 `synaptics_dsx`，结果放出去后有人反映 16G 版无效。一番研究之后才发现 Z5S 16G 和 32G 的触屏是不一样的。

由于我的 32G 前几天被我刷坏了，暂时无法开机。这里用前几天和 @BambooIV 一起解决这个 BUG 时的聊天记录举例一下。

Android 官方提供了 [`getevent`](https://source.android.com/devices/input/getevent.html) 这个工具让我们可以在 shell 里捕捉到手机的输入设备和事件。首先要让设备 `adb root` 授予 ADB root 权限，然后通过 `getevent -p` 取得手机上的输入设备列表：

```plain
$ adb shell getevent -p

<…省略一堆…>

add device 7: /dev/input/event0
  name:     "cyttsp4_mt"
  events:
    KEY (0001): 0044  0074
    ABS (0003): 002f  : value 0, min 0, max 15, fuzz 0, flat 0, resolution 0
                0035  : value 0, min 0, max 1079, fuzz 0, flat 0, resolution 0
                0036  : value 0, min 0, max 1919, fuzz 0, flat 0, resolution 0
                0039  : value 0, min 0, max 65535, fuzz 0, flat 0, resolution 0
                003a  : value 0, min 0, max 255, fuzz 0, flat 0, resolution 0
  input props:
    INPUT_PROP_DIRECT

add device 8: /dev/input/event1
  name:     "cyttsp4_btn"
  events:
    KEY (0001): 0066  008b  009e
  input props:
    <none>
```

有趣的是，32G 版只有 `synaptics_dsx` 一个设备同时兼作触屏和触屏底下的触摸键，而 16G 版则是分别由 `cyttsp4_mt` 和 `cyttsp4_btn` 两个设备承担。

为了验证 `cyttsp4_btn` 是不是就是我们要的底部触摸键对应的设备，可以用这条命令（后面的路径对应上文中 `cyttsp4_btn` 的路径）：

```plain
$ adb shell getevent -lt /dev/input/event1
```

并且在手机上触摸一下那几个按键，你会看到类似的输出：

```
[    1125.485711] EV_KEY       KEY_MENU             DOWN
[    1125.485726] EV_SYN       SYN_REPORT           00000000
[    1125.515822] EV_KEY       KEY_MENU             UP
[    1125.515834] EV_SYN       SYN_REPORT           00000000
[    1125.805195] EV_KEY       KEY_HOME             DOWN
[    1125.805215] EV_SYN       SYN_REPORT           00000000
[    1125.880776] EV_KEY       KEY_HOME             UP
[    1125.880785] EV_SYN       SYN_REPORT           00000000
[    1126.005205] EV_KEY       KEY_HOME             DOWN
[    1126.005226] EV_SYN       SYN_REPORT           00000000
[    1126.060987] EV_KEY       KEY_HOME             UP
[    1126.061001] EV_SYN       SYN_REPORT           00000000
[    1126.526975] EV_KEY       KEY_BACK             DOWN
[    1126.526979] EV_SYN       SYN_REPORT           00000000
[    1126.571815] EV_KEY       KEY_BACK             UP
[    1126.571822] EV_SYN       SYN_REPORT           00000000
```

就是它了。

在设备配置的 `/usr/keylayout` 里（其实哪里都可以）新建 `synaptics_dsx.kl` 和 `cyttsp4_btn.kl` 两个文件：

```shell
key 102     HOME          VIRTUAL
key 139     BACK          VIRTUAL
key 158     APP_SWITCH    VIRTUAL
```

并且在设备的构建配置里[加上](https://github.com/nx503a-dev/android_device_zte_nx503a/commit/08e5a73ddc19850dd4f93cb26e01252d72189202)对应的 `PRODUCT_COPY_FILES`，完成。

完事了吗？还没。

## 修改 CM 按钮设置里对应的选项

由于 CM 提供了对按键功能的自定义选项（系统设置 -- 按钮），我们还要修改一些参数让它正确地知道我们修改后的按键布局。

修改 [`/overlay/frameworks/base/core/res/res/values/config.xml`](https://github.com/nx503a-dev/android_device_zte_nx503a/blob/7960a1d312b2c71ed4523cc47a3f9008343382e9/overlay/frameworks/base/core/res/res/values/config.xml) 文件：

第一处，手机正面有哪些按键，`HOME` `BACK` `APP_SWITCH` 三者的二进制或运算结果为 `115`：

```xml
<!-- Hardware 'face' keys present on the device, stored as a bit field.
    This integer should equal the sum of the corresponding value for each
    of the following keys present:
        1 - Home
        2 - Back
        4 - Menu
        8 - Assistant (search)
        16 - App switch
        32 - Camera
        64 - Volume rocker
    For example, a device with Home, Back and Menu keys would set this
    config to 7. -->
<integer name="config_deviceHardwareKeys">115</integer>
```

第二处，由于我们已经将 `MENU` 改成 `APP_SWITCH` 了，所以把原本长按是 `APP_SWITCH` 的 `HOME` 改回和虚拟键一致的调起语音助手：

```xml
<!-- Control the behavior when the user long presses the home button.
        0 - Nothing
        1 - Menu key
        2 - Recent apps view in SystemUI
        3 - Launch assist intent
        4 - Voice Search
        5 - In-app Search
     This needs to match the constants in
     policy/src/com/android/internal/policy/impl/PhoneWindowManager.java
-->
<integer name="config_longPressOnHomeBehavior">3</integer>
```

好了，完成！
