---
layout: post
category: 开发
tags: [Android, 自定义控件]
title: Android 自定义控件（一）：View
---

好吧，三个半月没写博客了。终于把毕业设计全部搞定了，就等着明天去取胶装好的论文交给老师就没我事咯~

最近在做一个项目要用到自己实现一些控件，写出来备忘一下。

这篇博客的例子，我要自己实现一个 [`View`](http://developer.android.com/reference/android/view/View.html)，在上面通过 [`Canvas`](http://developer.android.com/reference/android/graphics/Canvas.html) 相关的方法绘制时分秒三支指针。姑且就叫它 `WatchView` 好了。

## 入门

一些基本的知识：

- Android 的所有 UI 控件都继承自 `View`，所谓 `View` 就是一个带有面积但是什么内容都没有的 UI 元素。相当于 HTML 中的 `div` 标签。
- `ViewGroup` 也是一种 `View`，它里面可以包含其它 `View`（当然也包括 `ViewGroup` 了）。没听过？`LinearLayout` `RelativeLayout` `FrameLayout`…它们都是 `ViewGroup`。
- 当系统需要绘制这个 `View` 的时候，它会调用它的 `onDraw(canvas)` 方法，我们可以调用 `invalidate()` 方法来让这个 `View` 重新绘制。

## 开始吧

不说废话，建工程，继承 `View`。

```java
public class WatchView extends View {

    private static final double ROUND = 2d * Math.PI;

    private final Calendar calendar = new GregorianCalendar();

    private final Paint paintSecond = new Paint();

    public WatchView(Context context) {
        this(context, null);
    }

    public WatchView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public WatchView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        float density = getResources().getDisplayMetrics().density;

        paintSecond.setAntiAlias(true);
        paintSecond.setColor(Color.RED);
        paintSecond.setStyle(Paint.Style.STROKE);
        paintSecond.setStrokeWidth(2f * density);
        paintSecond.setStrokeCap(Paint.Cap.ROUND);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        // 圆心的 x, y 坐标
        int x = canvas.getWidth() / 2;
        int y = canvas.getHeight() / 2;

        // 半径
        int radius = Math.min(x, y);

        // 为了让秒针连续转动，所以秒针的角度肯定不是从「秒」这个整数里来
        // 刚好秒针走一圈是 1 分钟，那么，就用分乘以「一圈（2π）」就是秒针要走的弧度数了
        float minute = ((float) calendar.get(Calendar.SECOND) + (float) calendar.get(Calendar.MILLISECOND) / 1000f) / 60f;

        // 三角函数的坐标轴是以 3 点方向为 0 的，所以记得要减去四分之一个圆周哦
        double secondRadians = (minute - QUARTER) * ROUND;

        canvas.drawLine(x, y,
                x + (float) Math.cos(secondRadians) * radius,
                y + (float) Math.sin(secondRadians) * radius,
                paintSecond);
    }

}
```

这里用到了两个东西：[`Paint`](http://developer.android.com/reference/android/graphics/Paint.html) 和 [`Canvas`](http://developer.android.com/reference/android/graphics/Canvas.html)，查文档吧，这里不重复了。

编译，运行。咦，不动？没错，我们还没做动画呢。

## 让它动起来

动起来，本质上就是定时重绘这个视图。如何定时呢？用 `Handler` 就好了。

```java
public class WatchView extends View implements Runnable {

    private final Handler handler = new Handler(Looper.getMainLooper());

    /* ... */

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        handler.post(this);
    }

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        handler.removeCallbacksAndMessages(null);
    }

    @Override
    public void run() {
        calendar.setTimeInMillis(System.currentTimeMillis());
        invalidate();
        handler.postDelayed(this, 20);
    }

    /* ... */

}
```

好了，图书馆要关门了，走。完整源代码见：[xingrz/WatchView](https://github.com/xingrz/WatchView)。
