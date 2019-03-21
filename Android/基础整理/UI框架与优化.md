## activity生命周期
1.onCreate和OnDestory是配对的，分别标志着activity的创建和销毁，并且只能会有一次调用。从activity是否可见来说，onStart和onStop是配对的，随着用户的操作或者设备屏幕的点亮和熄灭，这两个方法可能被调用多次；从activity是否在前台来说，onResume和onPause是配对的，随着用户操作或者屏幕的点亮和熄灭，这两个方法可能被调用多次。

onStart和onStop是从activity是否可见这个角度来回调的；onResume和onPause是从activity是否位于前台这个角度来回调的。

从当前activityA打开一个新的activityB,A的onPause会先调用，然后B才会启动。
A onPause->B onCreate->B onStart->B onResume->A onStop


## activity启动的时候获取view宽高
在onCreate,onStart,onResume中均无法得到某个View的宽高信息，因为View的mesure过程和Activity的生命周期方法不是同步执行的，因此无法确保view是否测量完毕。

1. Activity/View # onWindowFocusChanged
onWindowFocusChanged的含义是：View已经初始化完毕了，宽高已经准备好了。
需要注意的是onWindowFocusChanged在Activity的窗口得到焦点和失去焦点的时候都会被调用一次。

2. view.post(runnable)
通过post将runnable投递到消息队列尾部，等待Looper调用此runnable
3. ViewTreeObserver
通过ViewTreeObserver .addOnGlobalLayoutListener来获得宽高，当获得正确的宽高后，请移除这个观察者，否则回调会多次执行：

```
//获得ViewTreeObserver 
ViewTreeObserver observer=view.getViewTreeObserver();
//注册观察者，监听变化
observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
     @Override
     public void onGlobalLayout() {
            //判断ViewTreeObserver 是否alive，如果存活的话移除这个观察者
           if(observer.isAlive()){
             observer.removeGlobalOnLayoutListener(this);
             //获得宽高
             int viewWidth=view.getMeasuredWidth();
             int viewHeight=view.getMeasuredHeight();
           }
        }
   });
```
4. view.mesure(int widthMesureSpec,int heightMwsureSpec)

通过手动进行mesure来得到View的宽高，需要分情况处理：
- match_parent:无法测量出View的大小，需要知道父容器的剩余空间
- 具体数值（比如都是100px）：

```
int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
view.measure(widthMeasureSpec,heightMeasureSpec);
```

- wrap_content:

```
int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOTST);
int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);
view.measure(widthMeasureSpec,heightMeasureSpec);
```



# layout
## ConstraintLayout 

[ConstraintLayout](https://developer.android.google.cn/reference/android/support/constraint/ConstraintLayout)

###### Bias
layout_constraintHorizontal_bias  
layout_constraintVertical_bias  

The default when encountering such opposite constraints is to center the widget; but you can tweak the positioning to favor one side over another using the bias attributes:


```
 <Button android:id="@+id/button" ...
    app:layout_constraintHorizontal_bias="0.3"
    app:layout_constraintLeft_toLeftOf="parent"
    app:layout_constraintRight_toRightOf="parent/>
```

![image](https://developer.android.google.cn/reference/android/support/constraint/resources/images/centering-positioning-bias.png)

例如上图控制左边偏向30%而不是默认的50% 

Using bias, you can craft User Interfaces that will better adapt to screen sizes changes.

######  Circular positioning (Added in 1.1)

- layout_constraintCircle : references another widget id
- layout_constraintCircleRadius : the distance to the other widget center
- layout_constraintCircleAngle : which angle the widget should be at (in degrees, from 0 to 360)

###### Chain Style
layout_constraintHorizontal_chainStyle
layout_constraintVertical_chainStyle

- CHAIN_SPREAD -- the elements will be spread out (default style)
- Weighted chain -- in CHAIN_SPREAD mode, if some widgets are set to MATCH_CONSTRAINT, they will split the available space
- CHAIN_SPREAD_INSIDE -- similar, but the endpoints of the chain will not be spread out
- CHAIN_PACKED -- the elements of the chain will be packed together. The horizontal or vertical bias attribute of the child will then affect the positioning of the packed elements

![image](https://developer.android.google.cn/reference/android/support/constraint/resources/images/chains-styles.png)

###### Weighted chains
layout_constraintHorizontal_weight
layout_constraintVertical_weight

control how the space will be distributed among the elements using MATCH_CONSTRAINT. For exemple, on a chain containing two elements using MATCH_CONSTRAINT, with the first element using a weight of 2 and the second a weight of 1, the space occupied by the first element will be twice that of the second element.

###### Guideline
android.support.constraint.Guideline
用于辅助布局，即类似为辅助线，横向的、纵向的。该布局是不会显示到界面上的。

Currently, the Guideline object allows you to create Horizontal and Vertical guidelines which are positioned relative to the ConstraintLayout container. Widgets can then be positioned by constraining them to such guidelines. In 1.1, Barrier and Group were added too.

android:orientation取值为”vertical”和”horizontal”.决定其方向

    layout_constraintGuide_begin
    layout_constraintGuide_end
    layout_constraintGuide_percent
决定其确定位置


```
<android.support.constraint.ConstraintLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <android.support.constraint.Guideline
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/guideline"
            app:layout_constraintGuide_begin="100dp"
            android:orientation="vertical"/>

    <Button
            android:text="Button"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:id="@+id/button"
            app:layout_constraintLeft_toLeftOf="@+id/guideline"
            android:layout_marginTop="16dp"
            app:layout_constraintTop_toTopOf="parent" />

</android.support.constraint.ConstraintLayout>
```


###### Barrier
A Barrier references multiple widgets as input, and creates a virtual guideline based on the most extreme widget on the specified side. For example, a left barrier will align to the left of all the referenced views.


```
<android.support.constraint.Barrier
              android:id="@+id/barrier"
              android:layout_width="wrap_content"
              android:layout_height="wrap_content"
              app:barrierDirection="start"
              app:constraint_referenced_ids="button1,button2" />
```
With the barrier direction set to start, we will have the following result:
![image](https://developer.android.google.cn/reference/android/support/constraint/resources/images/barrier-start.png)

Reversely, with the direction set to end, we will have:
![image](https://developer.android.google.cn/reference/android/support/constraint/resources/images/barrier-end.png)

If the widgets dimensions change, the barrier will automatically move according to its direction to get the most extreme widget:
![image](https://developer.android.google.cn/reference/android/support/constraint/resources/images/barrier-adapt.png)

Other widgets can then be constrained to the barrier itself, instead of the individual widget. This allows a layout to automatically adapt on widget dimension changes (e.g. different languages will end up with different length for similar worlds).

###### Optimizer (in 1.1)
In 1.1 we exposed the constraints optimizer. You can decide which optimizations are applied by adding the tag app:layout_optimizationLevel to the ConstraintLayout element.

- none : no optimizations are applied
- standard : Default. Optimize direct and barrier constraints only
- direct : optimize direct constraints
- barrier : optimize barrier constraints
- chain : optimize chain constraints (experimental)
- dimensions : optimize dimensions measures (experimental), reducing the number of measures of match constraints elements
This attribute is a mask, so you can decide to turn on or off specific optimizations by listing the ones you want. For example: app:layout_optimizationLevel="direct|barrier|chain"



# 动画 与 变换

[Animations and Transitions](https://developer.android.google.cn/training/animation/)

## Animations

##### Property Animation
Creates an animation by modifying an object's property values over a set period of time with an Animator.

[Property Animation](https://developer.android.google.cn/guide/topics/graphics/prop-animation)

##### View Animation
There are two types of animations that you can do with the view animation framework:

- Tween animation: Creates an animation by performing a series of transformations on a single image with an Animation
- Frame animation: or creates an animation by showing a sequence of images in order with an AnimationDrawable.

## Interpolator
An interpolator define how specific values in an animation are calculated as a function of time. For example, you can specify animations to happen linearly across the whole animation, meaning the animation moves evenly the entire time, or you can specify animations to use non-linear time, for example, using acceleration or deceleration at the beginning or end of the animation.


```
AccelerateDecelerateInterpolator

@Override
public float getInterpolation(float input) {
    return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
}
```

```
LinearInterpolator

@Override
public float getInterpolation(float input) {
    return input;
}
```

- AccelerateDecelerateInterpolator 在动画开始与结束的地方速率改变比较慢，在中间的时候加速
- AccelerateInterpolator 在动画开始的地方速率改变比较慢，然后开始加速 
- AnticipateInterpolator 开始的时候向后然后向前甩
- AnticipateOvershootInterpolator 开始的时候向后然后向前甩一定值后返回最后的值
- BounceInterpolator 动画结束的时候弹起
- CycleInterpolator 动画循环播放特定的次数，速率改变沿着正弦曲线
- DecelerateInterpolator 在动画开始的地方快然后慢
- LinearInterpolator 以常量速率改变
- OvershootInterpolator 向前甩一定值后再回到原来位置


## AnimatedVectorDrawable
[Animate drawable graphics](https://developer.android.google.cn/guide/topics/graphics/drawable-animation)

## FlingAnimation 
[Move views using a fling animation](https://developer.android.google.cn/guide/topics/graphics/fling-animation)

## SpringAnimation
[Animate movement using spring physics](https://developer.android.google.cn/guide/topics/graphics/spring-animation)


## Transition 
[Animate layout changes using a transition](https://developer.android.google.cn/training/transitions/)

[Create a custom transition animation](https://developer.android.google.cn/training/transitions/custom-transitions)

[Start an activity using an animation](https://developer.android.google.cn/training/transitions/start-activity)


# UI优化
### 界面卡顿的原因以及解决方法
- 内存抖动
- 方法耗时
- 避免过度绘制（overdraw），
 手机开发者选项里面找到工具：Debug GPU overdraw，其中，不同颜色代表了绘制了几次

### LinearLayout、FrameLayout、RelativeLayout性能对比
1. RelativeLayout会让子View调用2次onMeasure，LinearLayout 在有weight时，也会调用子View2次onMeasure
2. RelativeLayout的子View如果高度和RelativeLayout不同，则会引发效率问题，当子View很复杂时，这个问题会更加严重。如果可以，尽量使用padding代替margin。
3. 在不影响层级深度的情况下,使用LinearLayout和FrameLayout而不是RelativeLayout。
最后再思考一下文章开头那个矛盾的问题，为什么Google给开发者默认新建了个RelativeLayout，而自己却在DecorView中用了个LinearLayout。因为DecorView的层级深度是已知而且固定的，上面一个标题栏，下面一个内容栏。采用RelativeLayout并不会降低层级深度，所以此时在根节点上用LinearLayout是效率最高的。而之所以给开发者默认新建了个RelativeLayout是希望开发者能采用尽量少的View层级来表达布局以实现性能最优，因为复杂的View嵌套对性能的影响会更大一些。

4. 能用两层LinearLayout，尽量用一个RelativeLayout，在时间上此时RelativeLayout耗时更小。另外LinearLayout慎用layout_weight,也将会增加一倍耗时操作。由于使用LinearLayout的layout_weight,大多数时间是不一样的，这会降低测量的速度。这只是一个如何合理使用Layout的案例，必要的时候，你要小心考虑是否用layout weight。总之减少层级结构，才是王道，让onMeasure做延迟加载，用viewStub，include等一些技巧。


### 使用矢量图


### 屏幕适配
#### 为什么要屏幕适配

##### 基础概念
[支持多种屏幕](https://developer.android.google.cn/guide/practices/screens_support#DeclaringTabletLayouts)

[提供资源](https://developer.android.google.cn/guide/topics/resources/providing-resources)

- px

也称为图像元素，是作为图像构成的基本单元，单个像素的大小并不固定，跟随屏幕大小和像素数量的关系变化（屏幕越大，像素越低，单个像素越大，反之亦然）。所以在使用像素作为设计单位时，在不同的设备上可能会有缩放或拉伸的情况。

- dpi:

是指屏幕上每英寸（1英寸 = 2.54 厘米）距离中有多少个像素点。如果屏幕为 320*240，屏幕长 2 英寸宽 1.5 英寸，Dpi = 320 / 2 = 240 / 1.5 = 160。




系统 dpi 跟物理 dpi 并不一定相同。在系统中使用的全部都是系统 dpi，没有使用物理 dpi，也获取不到物理 dpi。物理 dpi 主要用于厂家对于手机的参数描述（也可以看做 ppi ）

为简便起见，Android 将所有屏幕密度分组为六种通用密度： 低、中、高、超高、超超高和超超超高。

六种通用的密度：
- ldpi（低）~120dpi
- mdpi（中）~160dpi
- hdpi（高）~240dpi
- xhdpi（超高）~320dpi
- xxhdpi（超超高）~480dpi
- xxxhdpi（超超超高）~640dpi

要为不同的密度创建替代位图可绘制对象，应遵循六种通用密度之间的 3:4:6:8:12:16 缩放比率

通用的尺寸和密度按照基线配置（即正常尺寸和 mdpi（中）密度）排列。 此基线基于第一代 Android 设备 (T-Mobile G1) 的屏幕配置，该设备采用 HVGA 屏幕（在 Android 1.6 之前，这是 Android 支持的唯一屏幕配置）。

![image](https://developer.android.google.cn/images/screens_support/screens-ranges.png)

上图说明 Android 如何将实际尺寸和密度粗略地 对应到通用的尺寸和密度（数据并不精确）
- Density（密度）

这个是指屏幕上每平方英寸（2.54 ^ 2 平方厘米）中含有的像素点数量。

- dp:密度无关像素 (density-independent pixel):

在定义 UI 布局时应使用的虚拟像素单位，用于以密度无关方式表示布局维度 或位置。
密度无关像素等于 160 dpi 屏幕上的一个物理像素，这是 系统为“中”密度屏幕假设的基线密度。在运行时，系统 根据使用中屏幕的实际密度按需要以透明方式处理 dp 单位的任何缩放 。dp 单位转换为屏幕像素很简单： px = dp * (dpi / 160)。 例如，在 240 dpi 屏幕上，1 dp 等于 1.5 物理像素。在定义应用的 UI 时应始终使用 dp 单位 ，以确保在不同密度的屏幕上正常显示 UI。


###### 屏幕尺寸的新配置限定符 （在 Android 3.2 中引入）

屏幕配置 | 限定符值  | 说明
----|---|---
smallestWidth | sw<N>dp </br>示例：sw600dp sw720dp | 屏幕的基本尺寸，由可用屏幕区域的最小尺寸指定。<br> 设备的 smallestWidth 是屏幕可用高度和宽度的最小尺寸（您也可以将其视为屏幕的“最小可能宽度”）。<br>无论屏幕的当前方向如何，您均可使用此限定符确保应用 UI 的可用宽度至少为 <N>dp。
可用屏幕宽度 | w<N>dp<br>示例：w720dp w1024dp | 指定资源应该使用的最小可用宽度（dp 单位） — 由 <N> 值定义。<br>当屏幕的方向在横屏与竖屏之间切换时，系统对应的 宽度值将会变化，以 反映 UI 可用的当前实际宽度。<br>这对于确定是否使用多窗格布局往往很有用。
可用屏幕高度 | h<N>dp<br>示例：h720dp h1024dp 等等 | 指定资源应该使用的最小屏幕高度（dp 单位） — 由 <N> 值定义。<br>当屏幕的方向在横屏与竖屏之间切换时，系统 对应的高度值将会变化，以 反映 UI 可用的当前实际高度。<br>大多数应用不需要此限定符，考虑到 UI 经常垂直滚动， 因此高度更弹性，而宽度更刚性。


- 使用ConstraintLayout 

The Guidelines are used to define each percentage break point, and then a Button view is stretched to fill the gap:

```
<androidx.constraintlayout.widget.ConstraintLayout
         xmlns:android="http://schemas.android.com/apk/res/android"
         xmlns:app="http://schemas.android.com/apk/res-auto"
         android:layout_width="match_parent"
         android:layout_height="match_parent">

     <androidx.constraintlayout.widget.Guideline
         android:layout_width="wrap_content"
         android:layout_height="wrap_content"
         android:id="@+id/left_guideline"
         app:layout_constraintGuide_percent=".15"
         android:orientation="vertical"/>

     <androidx.constraintlayout.widget.Guideline
         android:layout_width="wrap_content"
         android:layout_height="wrap_content"
         android:id="@+id/right_guideline"
         app:layout_constraintGuide_percent=".85"
         android:orientation="vertical"/>

     <androidx.constraintlayout.widget.Guideline
         android:layout_width="wrap_content"
         android:layout_height="wrap_content"
         android:id="@+id/top_guideline"
         app:layout_constraintGuide_percent=".15"
         android:orientation="horizontal"/>

     <androidx.constraintlayout.widget.Guideline
         android:layout_width="wrap_content"
         android:layout_height="wrap_content"
         android:id="@+id/bottom_guideline"
         app:layout_constraintGuide_percent=".85"
         android:orientation="horizontal"/>

     <Button
         android:text="Button"
         android:layout_width="0dp"
         android:layout_height="0dp"
         android:id="@+id/button"
         app:layout_constraintLeft_toLeftOf="@+id/left_guideline"
         app:layout_constraintRight_toRightOf="@+id/right_guideline"
         app:layout_constraintTop_toTopOf="@+id/top_guideline"
         app:layout_constraintBottom_toBottomOf="@+id/bottom_guideline" />

 </androidx.constraintlayout.widget.ConstraintLayout>
```

##### px/dp（百分比）适配
[Android 屏幕适配方案](https://blog.csdn.net/lmj623565791/article/details/45460089)

[Android dp方式的屏幕适配-原理](https://blog.csdn.net/fesdgasdgasdg/article/details/82054971)

##### 修改 density适配(今日头条)
[一种极低成本的Android屏幕适配方式](https://mp.weixin.qq.com/s/d9QCoBP6kV9VSWvVldVVwA)

###### DisplayMetrics

- DisplayMetrics#density 就是上述的density

- DisplayMetrics#densityDpi 就是上述的dpi

- DisplayMetrics#scaledDensity 字体的缩放因子，正常情况下和density相等，但是调节系统字体大小后会改变这个值

布局文件中dp的转换，最终都是调用 TypedValue#applyDimension(int unit, float value, DisplayMetrics metrics) 来进行转换

```
    public static float applyDimension(int unit, float value,
                                       DisplayMetrics metrics)
    {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }
```

图片的decode，BitmapFactory#decodeResourceStream方法也是通过 DisplayMetrics 中的值来计算的：
```
    public static Bitmap decodeResourceStream(Resources res, TypedValue value,
            InputStream is, Rect pad, Options opts) {
        validate(opts);
        if (opts == null) {
            opts = new Options();
        }

        if (opts.inDensity == 0 && value != null) {
            final int density = value.density;
            if (density == TypedValue.DENSITY_DEFAULT) {
                opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
            } else if (density != TypedValue.DENSITY_NONE) {
                opts.inDensity = density;
            }
        }
        
        if (opts.inTargetDensity == 0 && res != null) {
            opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
        }
        
        return decodeStream(is, pad, opts);
    }
```


# UI框架

### databinding

### Design Support Library
Design Support Library是Google在2015年的IO大会上发布的全新Material Design支持库,在这个support库里面主要包含了 8 个新的 Material Design组件,最低支持 Android 2.1。

- TextInputLayout 	EditText辅助控件
- FloatingActionButton 	MD风格的圆形按钮
- Snackbar 	提示框
- TabLayout 	选项卡
- NavigationView 	DrawerLayout的侧滑界面
- CoordinatorLayout 	超级FrameLayout
- AppBarLayout 	MD风格的滑动Layout
- CollapsingToolbarLayout 	可折叠的MD风格ToolbarLayout


### Tangram

