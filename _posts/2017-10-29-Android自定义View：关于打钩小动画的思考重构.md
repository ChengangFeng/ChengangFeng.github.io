> Github地址：TickView，一个精致的打钩小动画
[https://github.com/ChengangFeng/TickView](https://github.com/ChengangFeng/TickView)

先上效果图，不然读不下去了，right？

**动图**

![动图.gif](http://upload-images.jianshu.io/upload_images/956714-54cdce326517b896.gif?imageMogr2/auto-orient/strip)

**静态图**
![静态图](http://upload-images.jianshu.io/upload_images/956714-82e91058c278cae2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---

## 1. 回顾
 [【Android自定义View：一个精致的打钩小动画】](http://www.jianshu.com/p/1b2cdba03d23)
上一篇文章，我们已经实现了基本上实现了控件的效果了，但是...但是...过了三四天后，仔细看回自己写的代码，虽然思路还在，但是部分代码还是不能一下子的看得明白...

我的天，这得立马重构啊~ 恰好，有个简友 [ChangQin](http://www.jianshu.com/u/601bff1a5d52) 模仿写了一下这个控件，我看了后觉得我也可以这样实现一下。

## 2. 深思
关于控件绘制的思路，可以去看看 [上一篇文章](http://www.jianshu.com/p/1b2cdba03d23)，这里就不再分析了。
这里先来分析一下上一篇文章里面，控件里面的一些顽处，哪些地方需要改进。

就拿 **绘制圆环进度** 这一步来看
``` java
//计数器
private int ringCounter = 0;

@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    if (!isChecked) {
        ...
        return;
    }
    //画圆弧进度，每次绘制都自加12个单位，也就是圆弧又扫过了12度
    //这里的12个单位先写死，后面我们可以做一个配置来实现自定义
    ringCounter += 12;
    if (ringCounter >= 360) {
        ringCounter = 360;
    }
    canvas.drawArc(mRectF, 90, ringCounter, false, mPaintRing);
    ...
    //强制重绘
    postInvalidate();
}
```

这里，我们定义了一个计数器`ringCounter`, 当绘制的时候，是根据12个单位进行自增到达360，从而模拟进度的变化。

仔细想想
1. 通过改变自增的单位来控制动画速度的变化，这很难调整得使自己满意，此时我们可以想到，使动画速度执行快慢的根本就是控制时间啊，如果可以**用时间来控制动画速度**那得方便多了
2. 动画分为4步执行，如果每一步动画都用手写计数器来实现，那得定义4个成员变量或者更多，**太多成员变量只会让代码更加混乱**
3. 如果动画要加上**插值器**，那手写的计数器根本无法满足
4. 看到上面的分析，**我无法接收了**

---

## 3. 改改改
那么怎么去改善上面所说的问题呢，答案就是用自定义的属性动画来解决了，所以这篇文章主要的讲的地方就是用属性动画来替换手写的计数器，尽可能的保证代码逻辑的清晰，特别是`onDraw()`方法中的代码。

使用属性动画的一个好处就是，给定数值的范围，它会帮你生成一堆你想要的数值，配合插值器还要意想不到的效果呢，下一面就一步一步针对动画执行的部分进行重构

###  3.1 绘制圆环进度条
首先，使用自定义的`ObjectAnimator`来模拟进度
``` java
//ringProgress是自定义的属性名称，生成数值的范围是0 - 360,就是一个圆的角度
ObjectAnimator mRingAnimator = ObjectAnimator.ofInt(this, "ringProgress", 0, 360);
//定义动画执行的时间，很好的替代之前使用自增的单位来控制动画执行的速度
mRingAnimator.setDuration(mRingAnimatorDuration);
//暂时不需要插值器
mRingAnimator.setInterpolator(null);
```
自定义属性动画，还需要配置相应的`setter`和`getter`，因为在动画执行的时候，会找相应的`setter`去改变相应的值。

``` java
private int getRingProgress() {
    return ringProgress;
}

private void setRingProgress(int ringProgress) {
    //动画执行的时候，会调用setter
    //这里我们可以将动画生成的数值记录下来,用变量存起来，在ondraw的时候用
    this.ringProgress = ringProgress;
    //记得重绘
    postInvalidate();
}
```
最后，在`onDraw()`中画图
``` java
//画圆弧进度
canvas.drawArc(mRectF, 90, ringProgress, false, mPaintRing);
```

###  3.2 绘制向圆心收缩的动画
同理，也是造一个属性动画
``` java
//这里自定义的属性是圆收缩的半径
ObjectAnimator mCircleAnimator = ObjectAnimator.ofInt(this, "circleRadius", radius - 5, 0);
//加一个减速的插值器
mCircleAnimator.setInterpolator(new DecelerateInterpolator());
mCircleAnimator.setDuration(mCircleAnimatorDuration);
```
setter/getter也是类似就不说了

最后`onDraw()`中绘制
``` java
//画背景
mPaintCircle.setColor(checkBaseColor);
canvas.drawCircle(centerX, centerY, ringProgress == 360 ? radius : 0, mPaintCircle);
//当进度圆环绘制好了，就画收缩的圆
if (ringProgress == 360) {
    mPaintCircle.setColor(checkTickColor);
    canvas.drawCircle(centerX, centerY, circleRadius, mPaintCircle);
}
```
###  3.3 绘制钩和放大再回弹的效果
这是两个独立的效果，这里同时执行，我就合在一起说了

首先也是定义属性动画
```java
//勾出来的透明渐变
ObjectAnimator mAlphaAnimator = ObjectAnimator.ofInt(this, "tickAlpha", 0, 255);
mAlphaAnimator.setDuration(200);
//最后的放大再回弹的动画，改变画笔的宽度来实现
//而画笔的宽度，则是的变化范围是
//首先从初始化宽度开始，再到初始化宽度的n倍，最后又回到初始化的宽度
ObjectAnimator mScaleAnimator = ObjectAnimator.ofFloat(this, "ringStrokeWidth", mPaintRing.getStrokeWidth(), mPaintRing.getStrokeWidth() * SCALE_TIMES, mPaintRing.getStrokeWidth() / SCALE_TIMES);
mScaleAnimator.setInterpolator(null);
mScaleAnimator.setDuration(mScaleAnimatorDuration);

//打钩和放大回弹的动画一起执行
AnimatorSet mAlphaScaleAnimatorSet = new AnimatorSet();
mAlphaScaleAnimatorSet.playTogether(mAlphaAnimator, mScaleAnimator);
```

getter/setter

``` java
private int getTickAlpha() {
    return 0;
}

private void setTickAlpha(int tickAlpha) {
    //设置透明度，可以不用变量来保存了
    //直接将透明度的值设置到画笔里面即可
    mPaintTick.setAlpha(tickAlpha);
    postInvalidate();
}

private float getRingStrokeWidth() {
    return mPaintRing.getStrokeWidth();
}

private void setRingStrokeWidth(float strokeWidth) {
    //设置画笔宽度，可以不用变量来保存了
    //直接将画笔宽度设置到画笔里面即可
    mPaintRing.setStrokeWidth(strokeWidth);
    postInvalidate();
}
```

最后，同理在`onDraw()`中绘制即可
``` java
if (circleRadius == 0) {
    canvas.drawLines(mPoints, mPaintTick);
    canvas.drawArc(mRectF, 0, 360, false, mPaintRing);
}
```

###  3.4 依次执行动画
执行多个动画，可以用到`AnimatorSet`,其中`playTogether()`是一起执行，`playSequentially()`是一个挨着一个，step by step执行。

``` java
mFinalAnimatorSet = new AnimatorSet();
mFinalAnimatorSet.playSequentially(mRingAnimator, mCircleAnimator, mAlphaScaleAnimatorSet);
```

最后在`onDraw()`中执行动画
``` java
//这里定义了一个标识符，用于告诉程序，动画每次只能执行一次
if (!isAnimationRunning) {
    isAnimationRunning = true;
    //执行动画
    mFinalAnimatorSet.start();
}
```

###  3.5 每个方法最好能有单一的职责
如果将定义属性动画的方法放在`onDraw()`中，我个人感觉很乱，并且再仔细看看，这几个属性动画是不需要动态变化的，为什么不抽出来在一开始的时候就初始化呢？

so，我们将定义属性动画的代码抽出来,并且放到构造函数中初始化
``` java
public TickView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    ...
    initAnimatorCounter();
}
```

``` java
/**
 * 用ObjectAnimator初始化一些计数器
 */
private void initAnimatorCounter() {
    //圆环进度
    ObjectAnimator mRingAnimator = ObjectAnimator.ofInt(this, "ringProgress", 0, 360);
    ...
    //收缩动画
    ObjectAnimator mCircleAnimator = ObjectAnimator.ofInt(this, "circleRadius", radius - 5, 0);
    ...
    //勾出来的透明渐变
    ObjectAnimator mAlphaAnimator = ObjectAnimator.ofInt(this, "tickAlpha", 0, 255);
    ...
    //最后的放大再回弹的动画，改变画笔的宽度来实现
    ObjectAnimator mScaleAnimator = ObjectAnimator.ofFloat(this, "ringStrokeWidth", mPaintRing.getStrokeWidth(), mPaintRing.getStrokeWidth() * SCALE_TIMES, mPaintRing.getStrokeWidth() / SCALE_TIMES);
    ...

    //打钩和放大回弹的动画一起执行
    AnimatorSet mAlphaScaleAnimatorSet = new AnimatorSet();
    mAlphaScaleAnimatorSet.playTogether(mAlphaAnimator, mScaleAnimator);

    mFinalAnimatorSet = new AnimatorSet();
    mFinalAnimatorSet.playSequentially(mRingAnimator, mCircleAnimator, mAlphaScaleAnimatorSet);
}
```

最后，`onDraw()`方法中，只负责简单的绘制，什么都不管
``` java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    if (!isChecked) {
        canvas.drawArc(mRectF, 90, 360, false, mPaintRing);
        canvas.drawLines(mPoints, mPaintTick);
        return;
    }
    //画圆弧进度
    canvas.drawArc(mRectF, 90, ringProgress, false, mPaintRing);
    //画黄色的背景
    mPaintCircle.setColor(checkBaseColor);
    canvas.drawCircle(centerX, centerY, ringProgress == 360 ? radius : 0, mPaintCircle);
    //画收缩的白色圆
    if (ringProgress == 360) {
        mPaintCircle.setColor(checkTickColor);
        canvas.drawCircle(centerX, centerY, circleRadius, mPaintCircle);
    }
    //画勾,以及放大收缩的动画
    if (circleRadius == 0) {
        canvas.drawLines(mPoints, mPaintTick);
        canvas.drawArc(mRectF, 0, 360, false, mPaintRing);
    }
    //ObjectAnimator动画替换计数器
    if (!isAnimationRunning) {
        isAnimationRunning = true;
        mFinalAnimatorSet.start();
    }
}
```
最终效果是一样的，代码逻辑一目了然
![最终效果.gif](http://upload-images.jianshu.io/upload_images/956714-54cdce326517b896.gif?imageMogr2/auto-orient/strip)

所以，个人觉得，在开发中，定时review一下自己的代码，无论对自己，还是对以后维护，是很有帮助的。

That ' s all~
感谢大家阅读，最后再放一下项目的github地址

> Github地址：TickView，一个精致的打钩小动画
[https://github.com/ChengangFeng/TickView](https://github.com/ChengangFeng/TickView)