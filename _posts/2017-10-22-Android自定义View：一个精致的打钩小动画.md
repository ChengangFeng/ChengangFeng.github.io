> Github地址：TickView，一个精致的打钩小动画
[https://github.com/ChengangFeng/TickView](https://github.com/ChengangFeng/TickView)

## 前言

最近在看轻芒杂志的时候，看到一个动画很带感很精致；

恰好这段时间也在看[【HenCoder】](http://hencoder.com)的自定义view教程（里面写得非常非常详细，也有相应的习题等等），所以就趁热打铁，熟悉一下学习的知识。

国际惯例，先上轻芒杂志标记已读的动画


![qingmang.gif](http://upload-images.jianshu.io/upload_images/956714-38124933370504ab.gif?imageMogr2/auto-orient/strip)

看了后是不是感觉很精致，很带感？

---

那下面来看一下我自己模仿的效果


![my.gif](http://upload-images.jianshu.io/upload_images/956714-43eee0d2501a1433.gif?imageMogr2/auto-orient/strip)

静态图

![静态图](http://upload-images.jianshu.io/upload_images/956714-05b3b5e0afeee5c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


是不是模仿得有几分相似，哈哈~，下面来看一下我实现的思路吧

# 2. 分析

这个动画实现起来并不复杂，掌握几个基本的自定义view的方法即可。

实现的思路分为`选中状态`和`未选中状态`


#### 2.1 未选中的状态


![未选择.png](http://upload-images.jianshu.io/upload_images/956714-d2447c0e31c805df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


未选中的状态很简单，需要绘制的有两个图形
* 圆环
* 勾

#### 2.2 选中的状态
绘制选中的动画稍微复杂一点，主要包括
1. **绘制圆环进度条**
这个简单，直接使用`drawArc()`即可实现

2. **绘制向圆心收缩的动画**
这个一开始的时候想用`drawArc()`加上设置画笔的宽度`strokeWidth`来实现，不过改变的宽度是往外扩张的，所以这个想法果断放弃。
之后，我的想法是这样的，看下图
![向圆心收缩的动画分析](http://upload-images.jianshu.io/upload_images/956714-b853ae5d7285da99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我就打算**先绘制一个黄色的背景，然后在这个图层上面绘制一个白色的圆，半径不断的缩小，直至为0，这就反过来得到了一个向中心收缩的动画**，这可以叫逆转思维吧，最近看的一本书里面说到有时候反过来思考也许会有不一样的效果。


3. **显示勾出来**
关于这个√，我在网上搜了一波，也没有明确的指明怎么画法才是标准的，所以这里可以随意发挥，自己觉得好看就行。这里直接可以使用`drawLine()`可以一步搞定。
4. **最后是圆环放大再回弹的效果**
放大回弹可以使用`drawArc()`，配合改变画笔的宽度来实现即可

# 3.具体实现

#### 3.1 确定进度圆环和钩的位置

经过上面分析，无论是选中状态还是未选中状态，进度圆环和钩的位置是不变的，所以我们先来确定圆环的位置和钩的位置

``` java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    ...
    //设置圆圈的外切矩形,radius是圆的半径，centerX，centerY是控件中心的坐标
    mRectF.set(centerX - radius, centerY - radius, centerX + radius, centerY + radius);

    //设置打钩的几个点坐标（具体坐标点的位置不用怎么理会，自己定一个就好，没有统一的标准）
    //画一个√，需要确定3个坐标点的位置
    //所以这里我先用一个float数组来记录3个坐标点的位置，
    //最后在onDraw()的时候使用canvas.drawLines(mPoints, mPaintTick)来画出来
    //其中这里mPoint[0]~mPoint[3]是确定第一条线"\"的两个坐标点位置
    //mPoint[4]~mPoint[7]是确定第二条线"/"的两个坐标点位置

    mPoints[0] = centerX - tickRadius + tickRadiusOffset;
    mPoints[1] = (float) centerY;
    mPoints[2] = centerX - tickRadius / 2 + tickRadiusOffset;
    mPoints[3] = centerY + tickRadius / 2;
    mPoints[4] = centerX - tickRadius / 2 + tickRadiusOffset;
    mPoints[5] = centerY + tickRadius / 2;
    mPoints[6] = centerX + tickRadius * 2 / 4 + tickRadiusOffset;
    mPoints[7] = centerY - tickRadius * 2 / 4;
}
```

#### 3.2 定义变量，标记状态
既然分选中状态和未选中状态，那个绘制过程中，就必须判断当前究竟是绘制未选中的呢还是选中了的呢。

因此在这里，我定义了一个变量`isChecked`
``` java
//是否被点亮
private boolean isChecked = false;

//暴露外部接口，改变绘制状态
public void setChecked(boolean checked) {
    if (this.isChecked != checked) {
        isChecked = checked;
        reset();
    }
}
```

#### 3.3 绘制未选中状态

绘制过程中那些画笔就不详细说了，一开始初始化画笔最后绘制的时候调用即可

```  java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    if (!isChecked) {
        //绘制圆环，mRectF就是之前确定的外切矩形
        //因为是静态的，所以设置扫过的角度为360度
        canvas.drawArc(mRectF, 90, 360, false, mPaintRing);

        //根据之前定好的钩的坐标位置，进行绘制
        canvas.drawLines(mPoints, mPaintTick);
        return;
    }
}
```

#### 3.4 绘制选中状态
选中状态是个动画，因此我们这里需要调用`postInvalidate()`**不断进行重绘**，直到动画执行完毕；另外，我这里用计数器的方式来控制绘制的进度。

#### 3.4.1 绘制圆环进度条

绘制进度圆环这里，我们定义一个计数器`ringCounter`,峰值为360（也就是360度），每执行一次`onDraw()`方法，我们对`ringCounter`进行自加，进而模拟进度。

最后记得调用`postInvalidate()`进行重绘


```  java
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
这一步后效果图如下

![绘制圆环进度条.gif](http://upload-images.jianshu.io/upload_images/956714-409731a1ccf23cd8.gif?imageMogr2/auto-orient/strip)


#### 3.4.2 绘制向圆心收缩的动画
圆心收缩的动画在圆环进度达到100%的时候才进行，同理，也采用计数器`circleCounter`的方法来控制绘制的时间和速度
``` java
//计数器
private int circleCounter = 0;

@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    ...
    //在圆环进度达到100%的时候才开始绘制
    if (ringCounter == 360) {
        //先绘制背景的圆
        mPaintCircle.setColor(checkBaseColor);
        canvas.drawCircle(centerX, centerY, radius, mPaintCircle);
        //然后在背景圆的图层上，再绘制白色的圆(半径不断缩小)
        //半径不断缩小，背景就不断露出来，达到向中心收缩的效果
        mPaintCircle.setColor(checkTickColor);
        //收缩的单位先试着设置为6，后面可以进行自己自定义
        circleCounter += 6;
        canvas.drawCircle(centerX, centerY, radius - circleCounter, mPaintCircle);
    }
    //必须重绘
    postInvalidate();
}
```
这一步后效果图如下


![绘制向圆心收缩的动画.gif](http://upload-images.jianshu.io/upload_images/956714-91606ea8b700cf45.gif?imageMogr2/auto-orient/strip)

#### 3.4.3 绘制钩
当白色的圆半径收缩到0后,就该绘制打钩了。

绘制打钩，这里问题不大，因为在`onMeasure()`中已经将钩的三个坐标点已经计算出来了，直接使用`drawLine()`即可画出来。

``` java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    ...
    canvas.drawCircle(centerX, centerY, radius - circleCounter, mPaintCircle);
    //当白色的圆半径收缩到0后，
    //也就是计数器circleCounter大于背景圆的半径的时候，就该将钩√显示出来了
    //这里加40是为了加一个延迟时间，不那么仓促的将钩显示出来
    if (circleCounter >= radius + 40) {
        //显示打钩（外加一个透明的渐变）
        alphaCount += 20;
        if (alphaCount >= 255) alphaCount = 255;
        mPaintTick.setAlpha(alphaCount);
        //最后就将之前在onMeasure中计算好的坐标传进去，绘制钩出来
        canvas.drawLines(mPoints, mPaintTick);
    }
    postInvalidate();
}
```
这一步后效果图如下

![绘制钩后效果图.gif](http://upload-images.jianshu.io/upload_images/956714-c3a7b86593b1e4ec.gif?imageMogr2/auto-orient/strip)

#### 3.4.4 绘制放大再回弹的效果

放大再回弹的效果，开始的时机应该也是收缩动画结束后开始，也就是说跟打钩的动画同时进行

因为这里要放大并且回弹，所以这里的计数器我设置成一个不为0的数值，先设置成45（随意，这不是标准），然后没重绘一次，自减4个单位。

最后画笔的宽度是关键的地方，画笔的宽度根据`scaleCounter`的正负来决定是加还是减

``` java
//计数器
private int scaleCounter = 45;

@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    ...
    if (circleCounter >= radius + 40) {
        //显示打钩
       ...
        //显示放大并回弹的效果
        scaleCounter -= 4;
        if (scaleCounter <= -45) {
            scaleCounter = -45;
        }
        //放大回弹，主要看画笔的宽度
        float strokeWith = mPaintRing.getStrokeWidth() +
            (scaleCounter > 0 ? dp2px(mContext, 1) : -dp2px(mContext, 1));
        mPaintRing.setStrokeWidth(strokeWith);
        canvas.drawArc(mRectF, 90, 360, false, mPaintRing);
    }
    //动画执行完毕，就补在需要重绘了
    if (scaleCounter != -45) {
        postInvalidate();
    }
}

```
完成最后一步的最终效果图


![最终的效果图](http://upload-images.jianshu.io/upload_images/956714-c741924df3b9099a.gif?imageMogr2/auto-orient/strip)

#### 3.5 暴露外部接口
为了灵活的可以控制绘制的状态，我们可以暴露一个接口给外部设置是否选中

```
/**
 *  是否选中
 */
public void setChecked(boolean checked) {
    if (this.isChecked != checked) {
        isChecked = checked;
        reset();
    }
}

/**
 *  重置，并重绘
 */
private void reset() {
    //画笔重置
    ...
    //计数器重置
    ringCounter = 0;
    circleCounter = 0;
    scaleCounter = 45;
    alphaCount = 0;
    ...
    invalidate();
}
```

#### 3.6 添加点击事件
控件到这里已经基本做好了，但还不是特别的完善。

想想`checkbox`，它不需要暴露外部接口也能通过点击控件来实现选中还是取消选中，所以接下来要实现的就是为控件添加点击事件

先定义一个接口`OnCheckedChangeListener`,实现监听此控件的监听事件
```
private OnCheckedChangeListener mOnCheckedChangeListener;

public interface OnCheckedChangeListener {
    void onCheckedChanged(TickView tickView, boolean isCheck);
}

public void setOnCheckedChangeListener(OnCheckedChangeListener listener) {
    this.mOnCheckedChangeListener = listener;
}
```

接下来，初始化控件的点击事件

```
/**
 * 在构造函数中初始化
 */
public TickView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    ...
    setUpEvent();
}

/**
 * 初始化点击事件
 */
private void setUpEvent() {
    this.setOnClickListener(new OnClickListener() {
        @Override
        public void onClick(View view) {
            isChecked = !isChecked;
            reset();
            if (mOnCheckedChangeListener != null) {
                //此处回调
                mOnCheckedChangeListener.onCheckedChanged((TickView) view, isChecked);
            }
        }
    });
}
```
看看效果图


![添加点击事件.gif](http://upload-images.jianshu.io/upload_images/956714-dc33b8f565be34aa.gif?imageMogr2/auto-orient/strip)


#### 3.7 自定义配置项

```
<declare-styleable name="TickView">
    <!--没有选中的基调颜色-->
    <attr name="uncheck_base_color" format="color" />
    <!--选中后的基调颜色-->
    <attr name="check_base_color" format="color" />
    <!--选中后钩的颜色-->
    <attr name="check_tick_color" format="color" />
    <!--圆的半径-->
    <attr name="radius" format="dimension" />
    <!--动画执行的速度-->
    <attr name="rate">
        <enum name="slow" value="0"/>
        <enum name="normal" value="1"/>
        <enum name="fast" value="2"/>
    </attr>
</declare-styleable>
```
这里简单说一下动画执行速度的配置，这里我设置了3档速度，我用枚举定义了三个速度的配置项

```
enum TickRateEnum {

    //低速
    SLOW(6, 4, 2),
    //正常速度
    NORMAL(12, 6, 4),
    //高速
    FAST(20, 14, 8);

    public static final int RATE_MODE_SLOW = 0;
    public static final int RATE_MODE_NORMAL = 1;
    public static final int RATE_MODE_FAST = 2;

    //圆环进度增加的单位
    private int ringCounterUnit;
    //圆圈收缩的单位
    private int circleCounterUnit;
    //圆圈最后放大收缩的单位
    private int scaleCounterUnit;

    public static TickRateEnum getRateEnum(int rateMode) {
        TickRateEnum tickRateEnum;
        switch (rateMode) {
            case RATE_MODE_SLOW:
                tickRateEnum = TickRateEnum.SLOW;
                break;
            case RATE_MODE_NORMAL:
                tickRateEnum = TickRateEnum.NORMAL;
                break;
            case RATE_MODE_FAST:
                tickRateEnum = TickRateEnum.FAST;
                break;
            default:
                tickRateEnum = TickRateEnum.NORMAL;
                break;
        }
        return tickRateEnum;
    }

    ...
}

```
获取xml的配置，获取对应的枚举，从而得到配好的动画速度的一些参数
```
/**
 * 构造函数
 */
public TickView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    ...
    initAttrs(attrs);
}

/**
 * 获取自定义配置
 */
private void initAttrs(AttributeSet attrs) {
    TypedArray typedArray = mContext.obtainStyledAttributes(attrs, R.styleable.TickView);
    ...
    //获取配置的动画速度
    int rateMode = typedArray.getInt(R.styleable.TickView_rate, TickRateEnum.RATE_MODE_NORMAL);
    mTickRateEnum = TickRateEnum.getRateEnum(rateMode);
    typedArray.recycle();
}
```

最终成果图

![最终成果图](http://upload-images.jianshu.io/upload_images/956714-43eee0d2501a1433.gif?imageMogr2/auto-orient/strip)
---

That ' s all~
感谢大家阅读，最后再放一下项目的github地址

> Github地址：TickView，一个精致的打钩小动画
[https://github.com/ChengangFeng/TickView](https://github.com/ChengangFeng/TickView)
