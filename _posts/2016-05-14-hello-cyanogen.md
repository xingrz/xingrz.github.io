---
layout: post
category: 开发
title: CyanogenMod 13.0 折腾手记
date:   2016-05-14 23:00:00 +0800
author: XiNGRZ
tags: [Android, CyanogenMod]
---

![](https://www.sinaimg.cn/large/4b263fe4gw1f3se8cpo7zj22eo37ke84.jpg)

最近入了 CM 这个坑 —— 给手上这台原本是 Android 4.2 的 Nubia Z5S (`nx503a`) 弄个 CM13。完全小白地折腾了两个星期，算是搞明白了一点，随手写点笔记。

> 话说[上次](https://xingrz.me/2014/2014-10-04/compile-aosp-with-vagrant-on-osx.html)折腾 ROM 的时候 CM 才到 11…那次最后是不了了之了

以 `nx503a` 为例，要给一台设备编译出一个 CM，至少需要这三套源码：

- [`android_device_zte_nx503a`](https://github.com/xingrz/android_device_zte_nx503a) - 设备配置
- [`android_kernel_zte_nx503a`](https://github.com/xingrz/android_kernel_zte_nx503a) - 内核
- [`android_vendor_zte_nx503a`](https://github.com/xingrz/android_vendor_zte_nx503a) - 厂家闭源的二进制

我的这三套源码都 fork 自 [@BambooIV](https://github.com/BambooIV) 前辈的成果，感谢！

## device

主要参考 CM 官方 wiki 里的 [How To Port CyanogenMod Android To Your Own Device](http://wiki.cyanogenmod.org/w/Doc:_porting_intro)。

- `BoardConfig.mk` 里，`BOARD_*_PARTITION_SIZE` 这几个变量指定了分区的大小，必须和分区的真实大小相等或略小。否则即便编译出来的 `img` 没达到分区的上限，`fastboot flash` 的时候也会报错
- 大多数适配贡献者会在 `patches` 或其它类似的目录里包含了需要对基础源码做的修改，需要自己应用

## kernel

- 如果 `boot.img` 或者 `recovery.img` 超过了分区的真实大小，一个办法是[启用内核压缩](https://github.com/xingrz/android_kernel_zte_nx503a/commit/2a08a9aa9d49c86cf013324c87e9d8b454b2a9ae)

## 导航键

Z5S 是一台带有触摸式导航键的机器，但我不太喜欢国内很多厂商把右键作为返回、左键作为菜单的做法，因为菜单键在 Android 上已经淘汰了。以往我的做法是在系统设置里将菜单改为多任务切换，然而这么做这两个按键的位置还是和 Android 原生的虚拟按键相反的。

其实有个更彻底的办法是[修改](https://github.com/xingrz/android_device_zte_nx503a/blob/a38a8271a2860080aff4fbf28b8d4b4b38b3c076/patches/keyboards.patch) `frameworks/base/data/keyboards/Generic.kl`。

```diff
-key 139   MENU
+key 139   BACK              VIRTUAL

-key 158   BACK
+key 158   APP_SWITCH        VIRTUAL
```

这里加上 `VIRTUAL` flag 的作用[官方有说明](https://source.android.com/devices/input/key-layout-files.html#virtual-soft-keys)。简单地说就是让这几个触摸键带上震动反馈。

## 编译

一开始我做了一条歪路：`device` `kernel` `vendor` 这三者不应该自己 `git clone` 到源码树里的。正确的做法应该是在 `.repo/local_manifests` 里建立一个 `roomservice.xml`，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="xingrz/android_device_zte_nx503a" path="device/zte/nx503a" remote="github" revision="cm-13.0" />
  <project name="xingrz/android_kernel_zte_nx503a" path="kernel/zte/nx503a" remote="github" revision="cm-13.0" />
  <project name="xingrz/android_vendor_zte_nx503a" path="vendor/zte/nx503a" remote="github" revision="cm-13.0" />
</manifest>
```

这个文件的作用是告诉 `repo sync`，除了拉取官方的源码库，还要拉取这些仓库，并把它们放到正确的位置。

虽然官方文档说 XML 的文件名随便你取，但我发现如果不用 `roomservice.xml` 的话，`breakfast nx503a` 的时候就不会自动解析 `device` 里的 `cm.dependencies`，也就导致编译或使用的时候缺少某些的库。

添加完这个文件，在 CM 源码库根目录下运行 `breakfast nx503a`，它就会自动把这个设备依赖的其它源码库加入到 `roomservices.xml` 里并拉取下来了。

> 一开始我编译时提示缺少 `device/qcom/common`，于是我自作聪明地自己 `git clone` 下来了。后来发现相机只能预览和录像，不能拍照。经 @updating [提点](http://weibo.com/1260797924/Dv9Ltw6qr)，发现是缺少 `libboringssl-compat.so`，进而是发现 `cm.dependencies` 没有起作用。

## TWRP

CM 附带的是它们自己的 CyanogenMod Recovery，你也可以将它换成 [TWRP](https://github.com/omnirom/android_bootable_recovery)。参考 Dees Troy 的 [How to compile TWRP touch recovery](http://forum.xda-developers.com/showthread.php?t=1943625)。

在 `roomservice.xml` 里加上这两个仓库，然后 `repo sync` 一下：

```xml
<project name="CyanogenMod/android_external_busybox" path="external/busybox" remote="github" revision="cm-13.0" />
<project name="omnirom/android_bootable_recovery" path="bootable/recovery-twrp" remote="github" revision="android-6.0" />
```

由于 TWRP 还在用老的 `fstab` 分区表配置文件，所以要单独给它写一份。这里我放在了设备目录的 `recovery/twrp.fstab` 里。如果你的设备官方有 4.X 的 ROM，可以参考官方的 `recovery` 里的配置。

最后一步在设备的 `BoardConfig.mk` 最后加上：

```Makefile
# TWRP
RECOVERY_VARIANT := twrp
TW_THEME := portrait_hdpi
PRODUCT_COPY_FILES += device/zte/nx503a/recovery/twrp.fstab:recovery/root/etc/twrp.fstab
```

然后 `make recoveryimage` 出来的 `recovery.img` 就是 TWRP 的了。

## 结尾

这篇博客只是我自己的一些笔记，稍微有点粗糙。

最后感谢 [@马丁龙猪](http://weibo.com/u/1803822891)、[@BambooIV](http://weibo.com/u/2986119755)、[@Aeeeb_Ping](http://weibo.com/u/2626788773)、[@updating](http://weibo.com/updateing) 几位前辈的帮助！
