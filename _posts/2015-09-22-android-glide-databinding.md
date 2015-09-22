---
layout: post
category: 开发
tags: [Android, Glide, DataBinding]
title: 更新一下弹药库：Glide + Android Data Binding
---

趁着最近比较空闲，赶紧恶补一些东西，更新一下自己的「弹药库」。

[Glide](https://github.com/bumptech/glide) 是继 Picasso 后又一个屌炸天的图片加载库。相较于 Picasso，它对滚动列表做了更好的优化。

另一方面，Google 最近搞了个 [Android Data Binding](https://developer.android.com/tools/data-binding/guide.html)，把数据绑定的概念引入到了 Android 开发中。

这里假设你已经看过上面这两个东西了。好，他们在一起能怎么组合呢？

## 一行代码绑定圆形图片

```java
public class ImageViewBindingUtil {

    @BindingAdapter("bind:roundImage")
    public static void loadRoundImage(ImageView view, String url) {
        Glide.with(view.getContext())
                .load(url)
                .bitmapTransform(new CropCircleTransformation(view.getContext()))
                .into(view);
    }

}
```

```xml
<ImageView
    android:id="@+id/avatar"
    android:layout_width="40dp"
    android:layout_height="40dp"
    app:roundImage="@{user.avatarUrl}" />
```

你没看错！就一个 `app:roundImage`！

对了，需要配合一个 Glide 的插件库 [glide-transformations](https://github.com/wasabeef/glide-transformations)。

简洁到爆了哈哈哈哈哈哈！！！

Binding Adapter 还有更多类似的玩法，比如绑定日期：

```java
public class TextViewBindingUtil {

    @BindingAdapter("bind:relativeDateTime")
    public static void setRelativeDateTime(TextView view, Date date) {
        view.setText(DateUtils.getRelativeTimeSpanString(view.getContext(), date.getTime()));
    }

}
```

自己发掘吧~
