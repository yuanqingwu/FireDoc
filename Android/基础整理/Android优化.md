## 布局优化
根据Google官方出品的Android性能优化典范，60帧每秒是目前最合适的图像显示速度，事实上绝大多数的Android设备也是按照每秒60帧来刷新的。为了让屏幕的刷新帧率达到60fps，我们需要确保在时间16ms（1000/60Hz）内完成单次刷新的操作（包括measure、layout以及draw），这也是Android系统每隔16ms就会发出一次VSYNC信号触发对UI进行渲染的原因。

##### 减少嵌套层次及控件个数：使用<include>标签重用布局、<merge>标签减少层级、<ViewStub>标签懒加载。使用ConstraintLayout，自定义布局减少布局层级。
- Android的布局文件的加载是LayoutInflater利用pull解析方式来解析，然后根据节点名通过反射的方式创建出View对象实例；
- 同时嵌套子View的位置受父View的影响，类如RelativeLayout、LinearLayout等经常需要measure两次才能完成，而嵌套、相互嵌套、深层嵌套等的发生会使measure次数呈指数级增长，所费时间呈线性增长；

LinearLayout使用weight会测量两遍。

###### merge标签
merge标签常用于减少布局嵌套层次，但是只能用于根布局。

###### ViewStub标签
推迟创建对象、延迟初始化，不仅可以提高性能，也可以节省内存（初始化对象不被创建）。Android定义了ViewStub类，ViewStub是轻量级且不可见的视图，它没有大小，没有绘制功能，也不参与measure和layout，资源消耗非常低。
App里常见的视图如蒙层、小红点，以及网络错误、没有数据等公共视图，使用频率并不高，如果每一次都参与绘制其实是浪费资源的，都可以借助ViewStub标签进行延迟初始化，仅当使用时才去初始化。

###### include标签
include标签和布局性能关系不大，主要用于布局重用，一般和merge标签配合使用

###### 自定义控件
自定义控件时，注意在onDraw不能进行复杂运算

###### 工具
有Hierarchy Viewer这个方便可视化的工具，可以得到：树形结构总览、布局view、每一个View（包含子View）绘制所花费的时间及View总个数。

打开设备的GPU配置渲染工具在屏幕上显示为条形图，可以协助我们定位UI渲染问题。

## 启动优化
https://developer.android.google.cn/topic/performance/vitals/launch-time
1. 利用提前展示出来的Window，快速展示出来一个界面，给用户快速反馈的体验；
2. 避免在启动时做密集沉重的初始化（Heavy app initialization）；
3. 定位问题：避免I/O操作、反序列化、网络操作、布局嵌套等。

###### 启动加速之主题切换
使用Activity的windowBackground主题属性来为启动的Activity提供一个简单的drawable。
Layout XML file:
```
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" android:opacity="opaque">
  <!-- The background color, preferably the same as your normal theme -->
  <item android:drawable="@android:color/white"/>
  <!-- Your product logo - 144dp color version of your app icon -->
  <item>
    <bitmap
      android:src="@drawable/product_logo_144dp"
      android:gravity="center"/>
  </item>
</layer-list>
```
Manifest file:

```
<activity ...
android:theme="@style/AppTheme.Launcher" />
```


将主题切换回来最简单的方法就是在调用super.onCreate() 和 setContentView()之前调用setTheme(R.style.AppTheme) 
```
public class MyMainActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    // Make sure this is before calling super.onCreate
    setTheme(R.style.Theme_MyApp);
    super.onCreate(savedInstanceState);
    // ...
  }
}
```
###### 启动加速之Avoid Heavy App Initialization
考虑异步初始化三方组件，不阻塞主线程

注意：闪屏页的2秒停留可以利用，把耗时操作延迟到这个时间间隔里。

###### 启动加速之Diagnosing The Problem
在开发阶段我们一般使用BlockCanary或者ANRWatchDog找耗时操作，简单明了，但是无法得到每一个方法的执行时间以及更详细的对比信息。我们可以通过Method Tracing或者DDMS来获得更全面详细的信息。
启动应用，点击 Start Method Tracing，应用启动后再次点击，会自动打开刚才操作所记录下的.trace文件，建议使用DDMS来查看，功能更加方便全面。


- 卡顿不能都靠异步来解决，错误的使用工程线程不仅不能改善卡顿，反而可能加剧卡顿。是否需要开启工作线程需要根据具体的性能瓶颈根源具体分析，对症下药，不可一概而论；
- 而如何开启线程同样也有学问：Thread、ThreadPoolExecutor、AsyncTask、HandlerThread、IntentService等都各有利弊；例如通常情况下ThreadPoolExecutor比Thread更加高效、优势明显，但是特定场景下单个时间点的表现Thread会比ThreadPoolExecutor好：同样的创建对象，ThreadPoolExecutor的开销明显比Thread大；
- 正确的开启线程也不能包治百病，例如执行网络请求会创建线程池，而在Application中正确的创建线程池势必也会降低启动速度；因此延迟操作也必不可少。


## 内存优化
https://developer.android.google.cn/topic/performance/memory-overview.html#gc

https://developer.android.google.cn/topic/performance/memory

[Android内存优化之OOM](http://hukai.me/android-performance-oom/)

通过命令行adb shell dumpsys meminfo packagename查看内存详细占用情况

ActivityManager.getMemoryClass()可以用来查询当前应用的Heap Size阈值，这个方法会返回一个整数，表明你的应用的Heap Size阈值是多少Mb(megabates)。

- 在Dalvik下，大部分Davik采取的都是标记-清理回收算法，而且具体使用什么算法是在编译期决定的，无法在运行的时候动态更换。标记-清理回收算法无法对Heap中空闲内存区域做碎片整理。系统仅仅会在新的内存分配之前判断Heap的尾端剩余空间是否足够，如果空间不够会触发gc操作，从而腾出更多空闲的内存空间；这样内存空洞就产生了。
- ART在GC上不像Dalvik仅有一种回收算法，ART在不同的情况下会选择不同的回收算法。应用程序在前台运行时，响应性是最重要的，因此也要求执行的GC是高效的。相反，应用程序在后台运行时，响应性不是最重要的，这时候就适合用来解决堆的内存碎片问题。因此，Mark-Sweep GC适合作为Foreground GC，而Mark-Compact GC适合作为Background GC。由于有Compact的能力存在，内存碎片在ART上可以很好的被避免，这个也是ART一个很好的能力。

#### 内存泄漏

[Android内存泄漏分析心得](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=2649796884&idx=1&sn=92b4e344060362128e4a86d6132c3736&scene=0#wechat_redirect)

###### 单例造成的内存泄露
解决方案：
> 将该属性的引用方式改为弱引用;

> 如果传入Context，使用ApplicationContext;

###### 匿名内部类（匿名内部类会引用外部类，导致无法释放，比如各种回调）
在Java中，非静态内部类 和 匿名类 都会潜在的引用它们所属的外部类，但是，静态内部类却不会。如果这个非静态内部类实例做了一些耗时的操作，就会造成外围对象不会被回收，从而导致内存泄漏。

解决方案：
> 将内部类变成静态内部类;

> 如果有强引用Activity中的属性，则将该属性的引用方式改为弱引用;

> 在业务允许的情况下，当Activity执行onDestory时，结束这些耗时任务;

###### Activity Context 的不正确使用
解决方案：
> 使用ApplicationContext代替ActivityContext，因为ApplicationContext会随着应用程序的存在而存在，而不依赖于activity的生命周期；

> 对Context的引用不要超过它本身的生命周期，慎重的对Context使用“static”关键字。Context里如果有线程，一定要在onDestroy()里及时停掉。

######  Handler引起的内存泄漏
解决方案：
> 可以把Handler类放在单独的类文件中，或者使用静态内部类便可以避免泄露;

> 如果想在Handler内部去调用所在的Activity,那么可以在handler内部使用弱引用的方式去指向所在Activity.使用Static + WeakReference的方式来达到断开Handler与Activity之间存在引用关系的目的。

###### 注册监听器的泄漏
系统服务可以通过Context.getSystemService 获取，它们负责执行某些后台任务，或者为硬件访问提供接口。如果Context 对象想要在服务内部的事件发生时被通知，那就需要把自己注册到服务的监听器中。然而，这会让服务持有Activity 的引用，如果在Activity onDestory时没有释放掉引用就会内存泄漏。

解决方案：
> 使用ApplicationContext代替ActivityContext;

> 在Activity执行onDestory时，调用反注册;


```
InputMethodManager imm = (InputMethodManager) context.getApplicationContext().getSystemService(Context.INPUT_METHOD_SERVICE);
```

###### AsyncTask/Runnable以匿名内部类的方式存在，会隐式持有对所在Activity的引用。
将AsyncTask和Runnable设为静态内部类或独立出来；在线程内部采用弱引用保存Context引用

###### 未及时注销资源导致内存泄漏，如BraodcastReceiver、File、Cursor、Stream、Bitmap等。View没有recycle。
在Activity销毁的时候要及时关闭或者注销。

BraodcastReceiver：调用unregisterReceiver()注销；
Cursor，Stream、File：调用close()关闭；
动画：在Activity.onDestroy()中调用Animator.cancel()停止动画

###### 集合中对象没清理造成的内存泄漏
> 在Activity退出之前，将集合里的东西clear，然后置为null，再退出程序。

###### WebView造成的泄露
当我们不要使用WebView对象时，应该调用它的destory()函数来销毁它，并释放其占用的内存，否则其占用的内存长期也不能被回收，从而造成内存泄露。

解决方案：
> 为webView开启另外一个进程，通过AIDL与主线程进行通信，WebView所在的进程可以根据业务的需要选择合适的时机进行销毁，从而达到内存的完整释放。


#### OOM
 Android系统的每个进程都有一个最大内存限制，如果申请的内存资源超过这个限制，系统就会抛出OOM错误。
 
 - Android 2.x系统，当dalvik allocated + external allocated + 新分配的大小 >= dalvik heap 最大值时候就会发生OOM。其中bitmap是放于external中 。
- Android 4.x系统，废除了external的计数器，类似bitmap的分配改到dalvik的java heap中申请，只要allocated + 新分配的内存 >= dalvik heap 最大值的时候就会发生OOM（art运行环境的统计规则还是和dalvik保持一致）

根据[《Manage Your App's Memory》](https://developer.android.google.cn/topic/performance/memory.html)我们可以对内存的状态进行监听，在Activity中覆写此方法，根据不同的case进行不同的处理：

```
public class MainActivity extends AppCompatActivity
    implements ComponentCallbacks2 {

    // Other activity code ...

    /**
     * Release memory when the UI becomes hidden or when system resources become low.
     * @param level the memory-related event that was raised.
     */
    public void onTrimMemory(int level) {

        // Determine which lifecycle or system event was raised.
        switch (level) {

            case ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN:

                /*
                   Release any UI objects that currently hold memory.

                   The user interface has moved to the background.
                */

                break;

            case ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE:
            case ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW:
            case ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL:

                /*
                   Release any memory that your app doesn't need to run.

                   The device is running low on memory while the app is running.
                   The event raised indicates the severity of the memory-related event.
                   If the event is TRIM_MEMORY_RUNNING_CRITICAL, then the system will
                   begin killing background processes.
                */

                break;

            case ComponentCallbacks2.TRIM_MEMORY_BACKGROUND:
            case ComponentCallbacks2.TRIM_MEMORY_MODERATE:
            case ComponentCallbacks2.TRIM_MEMORY_COMPLETE:

                /*
                   Release as much memory as the process can.

                   The app is on the LRU list and the system is running low on memory.
                   The event raised indicates where the app sits within the LRU list.
                   If the event is TRIM_MEMORY_COMPLETE, the process will be one of
                   the first to be terminated.
                */

                break;

            default:
                /*
                  Release any non-critical data structures.

                  The app received an unrecognized memory level value
                  from the system. Treat this as a generic low-memory message.
                */
                break;
        }
    }
}
```
The onTrimMemory() callback was added in Android 4.0 (API level 14). For earlier versions, you can use the onLowMemory(), which is roughly equivalent to the TRIM_MEMORY_COMPLETE event.


对高风险OOM代码块如展示高清大图等进行try catch，在catch块加载非高清的图片并做相应内存回收的处理。注意OOM是OutOfMemoryError，不能使用Exception进行捕获。

###### 避免内存抖动
Memory Churn内存抖动：大量的对象被创建又在短时间内马上被释放。

例如：在For循环中分配了多个临时对象，或在onDraw()方法中创建了Paint、Bitmap对象，应用产生了大量的对象；这会很快耗尽young generation的可用内存，导致GC发生。
使用Analyze your RAM usage中的工具找出代码里内存抖动的地方。考虑把操作移出内部循环，或者将其移动到基于工厂的分配结构中。


### Bitmap优化
[细说Bitmap](https://www.jianshu.com/p/e49ec7d053b3)

###### Bitmap的内存回收
Android 2.3.3（API10）之前，Bitmap的像素数据存放在Native内存，而Bitmap对象本身则存放在Dalvik Heap中。而在Android3.0之后，Bitmap的像素数据也被放在了Dalvik Heap中。

###### Bitmap占用内存的计算

- getByteCount()方法是在API12加入的，代表存储Bitmap的像素需要的最少内存；
- getAllocationByteCount()在API19加入，代表在内存中为Bitmap分配的内存大小；
- 在复用Bitmap的情况下，getAllocationByteCount()可能会比getByteCount()大；

计算公式：

- 对资源文件：
 width * height * nTargetDensity/inDensity * nTargetDensity/inDensity * 一个像素所占的内存。
- 其他文件：
width * height * 一个像素所占的内存。

######  一个像素占用多大内存？
 Bitmap.Config用来描述图片的像素是怎么被存储的
- ARGB_8888: 每个像素4字节. 共32位，默认设置。
- Alpha_8: 只保存透明度，共8位，1字节。
- ARGB_4444: 共16位，2字节。
- RGB_565:共16位，2字节，只存储RGB值。

##### Bitmap如何复用
1. 使用LruCache和DiskLruCache做内存和磁盘缓存；
2. 使用Bitmap复用，同时针对版本进行兼容。

```
BitmapFactory.Options options = new BitmapFactory.Options();
// 图片复用，这个属性必须设置；
options.inMutable = true;
// 手动设置缩放比例，使其取整数，方便计算、观察数据；
options.inDensity = 320;
options.inTargetDensity = 320;
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.resbitmap, options);
// 对象内存地址；
Log.i(TAG, "bitmap = " + bitmap);
Log.i(TAG, "bitmap：ByteCount = " + bitmap.getByteCount() + ":::bitmap：AllocationByteCount = " + bitmap.getAllocationByteCount());

// 使用inBitmap属性，这个属性必须设置；
options.inBitmap = bitmap;
options.inDensity = 320;
// 设置缩放宽高为原始宽高一半；
options.inTargetDensity = 160;
options.inMutable = true;
Bitmap bitmapReuse = BitmapFactory.decodeResource(getResources(), R.drawable.resbitmap_reuse, options);
// 复用对象的内存地址；
Log.i(TAG, "bitmapReuse = " + bitmapReuse);
Log.i(TAG, "bitmap：ByteCount = " + bitmap.getByteCount() + ":::bitmap：AllocationByteCount = " + bitmap.getAllocationByteCount());
Log.i(TAG, "bitmapReuse：ByteCount = " + bitmapReuse.getByteCount() + ":::bitmapReuse：AllocationByteCount = " + bitmapReuse.getAllocationByteCount());

输出：
I/lz: bitmap = android.graphics.Bitmap@35ac9dd4
I/lz: width:1024:::height:594
I/lz: bitmap：ByteCount = 2433024:::bitmap：AllocationByteCount = 2433024
I/lz: bitmapReuse = android.graphics.Bitmap@35ac9dd4 // 两个对象的内存地址一致
I/lz: width:512:::height:297
I/lz: bitmap：ByteCount = 608256:::bitmap：AllocationByteCount = 2433024
I/lz: bitmapReuse：ByteCount = 608256:::bitmapReuse：AllocationByteCount = 2433024 // ByteCount比AllocationByteCount小

```


##### Bitmap如何压缩

###### Bitmap.compress()
质量压缩：
它是在保持像素的前提下改变图片的位深及透明度等，来达到压缩图片的目的，不会减少图片的像素。进过它压缩的图片文件大小会变小，但是解码成bitmap后占得内存是不变的。

###### BitmapFactory.Options.inSampleSize
内存压缩：

- 解码图片时，设置BitmapFactory.Options类的inJustDecodeBounds属性为true，可以在Bitmap不被加载到内存的前提下，获取Bitmap的原始宽高。而设置BitmapFactory.Options的inSampleSize属性可以真实的压缩Bitmap占用的内存，加载更小内存的Bitmap。
- 设置inSampleSize之后，Bitmap的宽、高都会缩小inSampleSize倍。例如：一张宽高为2048x1536的图片，设置inSampleSize为4之后，实际加载到内存中的图片宽高是512x384。占有的内存就是0.75M而不是12M，足足节省了15倍。

> inSampleSize值的大小不是随便设、或者越大越好，需要根据实际情况来设置
> inSampleSize比1小的话会被当做1，任何inSampleSize的值会被取接近2的幂值。



###### BitmapFactory 在解码图片时，可以带一个Options


##### 使用Glide来做Bitmap的加载。


## 绘制优化
##### 降低View.onDraw（）的复杂度
###### onDraw（）中不要创建新的局部对象
######  避免onDraw（）执行大量 & 耗时操作

##### 避免过度绘制（Overdraw）
###### 移除默认的 Window 背景
###### 移除 控件中不必要的背景
###### 减少布局文件的层级（嵌套）
###### 自定义控件View优化：使用 clipRect() 、 quickReject()

使用 canvas.clipRect()来给 Canvas 设置一个裁剪区域，只有在该区域内才会被绘制，区域之外的都不绘制

## 响应速度优化

##### ANR触发场景

- InputDispatching Timeout ：输入事件分发超时5s未响应完毕；
- BroadcastQueue Timeout ：前台广播在10s内、后台广播在60秒内未执行完成；
- Service Timeout ：前台服务在20s内、后台服务在200秒内未执行完成；
- ContentProvider Timeout ：内容提供者,在publish过超时10s；


> service本身也运行于主线程，执行耗时任务同样会发生ANR。
流程总结：
1. Service创建之前会延迟发送一个消息，而这个消息就是ANR的起源；
2. Service创建完毕，在规定的时间之内执行完毕onCreate()方法就移除这个消息，就不会产生ANR了；
3. 在规定的时间之内没有完成onCreate()的调用，消息被执行，ANR发生。

## 网络优化

###### Network Monitor，Charles、Fiddler等抓包工具

Android Studio自带的Network Monitor简单直观，可以看出时间段之内的网络请求数量及访问速率；

使用Charles、Fiddler等抓包工具同样可以实现Network Monitor的功能，而且更加强大。

对于真正的弱网，可以使用抓包工具进行模拟，也有聪明的小伙伴使用wifi精灵进行限速

Facebook的开源项目augmented-traffic-control可以模拟不同的网络环境，针对带宽、时延抖动、丢包率、错包率、包重排序率等方面，堪称弱网调试神器；

##### 网络优化主要从三个方面进行：1. 速度；2. 成功率；3. 流量。
###### Gzip压缩
HTTP协议上的Gzip编码是一种用来改进WEB应用程序性能的技术，用来减少传输数据量大小，从而减少流量消耗与传输时间。

######  IP直连与HttpDns
DNS解析的失败率占联网失败中很大一种，而且首次域名解析一般需要几百毫秒。针对此，我们可以不用域名，才用IP直连省去 DNS 解析过程，节省这部分时间。

HttpDNS基于Http协议的域名解析，替代了基于DNS协议向运营商Local DNS发起解析请求的传统方式，可以避免Local DNS造成的域名劫持和跨网访问问题，解决域名解析异常带来的困扰。

###### 图片上传与下载
- 使用WebP格式；同样的照片，采用WebP格式可大幅节省流量，相对于JPG格式的图片，流量能节省将近 25% 到 35 %；相对于 PNG 格式的图片，流量可以节省将近80%。最重要的是使用WebP之后图片质量也没有改变。
- 使用缩略图；App中需要加载的图片按需加载，列表中的图片根据需要的尺寸加载合适的缩略图即可，只有用户查看大图的时候才去加载原图。
- 图片上传：
避免整文件传输，采用分片传输；
根据网络类型以及传输过程中的变化动态的修改分片大小；
每个分片失败重传的机会。

###### 协议层的优化
使用最新的协议，Http协议有多个版本：0.9、1.0、1.1、2等。新版本的协议经过再次的优化

###### 请求打包
合并网络请求，减少请求次数。对于一些接口类如统计，无需实时上报，将统计信息保存在本地，然后根据策略统一上传。这样头信息仅需上传一次，减少了流量也节省了资源。

###### 网络缓存
对服务端返回数据进行缓存，设定有效时间，有效时间之内不走网络请求，减少流量消耗

我们也可以自定义缓存的实现，一些网络库例如：Volley、Okhttp等都有好的实践供参考

###### 其他
断点续传，文件、图片等的下载，采用断点续传，不浪费用户之前消耗过的流量

重试策略，一次网络请求的失败，需要多次的重试来断定最终的失败，可以参考Volley的重试机制实现。

Protocol Buffer是Google的一种数据交换的格式，它独立于语言，独立于平台。相较于目前常用的Json，数据量更小，意味着传输速度也更快。

尽量避免客户端的轮询，而使用服务器推送的方式；

数据更新采用增量，而不是全量，仅将变化的数据返回，客户端进行合并，减少流量消耗；

## 耗电优化
[Battery Historian](https://github.com/google/battery-historian)

通过Battery Historian可以方便的看到各耗电模块随着时间的耗电情况：包含操作类型、执行时间、对应App等；还可以进行筛选特定的App，给出一个总结性的说明，包括：Network Information、 Syncs、WakeLock、Services、Process info、Scheduled Job、Sensor Use等，查看每一个模块的总结，可以看出来每一项的耗时以及执行次数。当发现异常的时候可以针对性的进行排查。


###### 及时注销定位监听
###### 谨慎使用WakeLock
- App在前台不要申请WakeLock，此时无需申请，申请的话会计算到应用电量消耗；
- App在后台由于业务需要必须要申请WakeLock时使用带有超时参数的方法，防止由于忘记或者异常情况下没有释放；
- App申请使用WakeLock，任务结束之后及时释放，让系统再次进入休眠状态。

```
PowerManager pm = (PowerManager)mContext.getSystemService(Context.POWER_SERVICE);
 PowerManager.WakeLock wl = pm.newWakeLock(PowerManager.SCREEN_DIM_WAKE_LOCK| PowerManager.ON_AFTER_RELEASE,TAG);
 wl.acquire(TIMEOUT);// 使用带有超时参数的acquire方法
 // ... do work...
 wl.release();
```
如果只是需要屏幕常亮的话，可以使用FLAG_KEEP_SCREEN_ON，无需考虑释放WakeLock的问题。

######  传感器使用
使用传感器，选择合适的采样率，越高的采样率类型则越费电；

###### JobScheduler
使用JobScheduler，一些任务通过JobScheduler来触发，例如可推迟的网络请求、下载、GPS等，可以在特定场景：连接Wifi、连接电源等场景触发。既完成了任务，也无需考虑由于一些任务导致的电量消耗。


## 包大小优化

https://developer.android.google.cn/studio/build/shrink-code

###### 代码混淆。使用proGuard 代码混淆器工具，它包括压缩、优化、混淆等功能。

###### 避免重复功能的库

###### 资源优化。比如使用 Android Lint 删除冗余资源，资源文件最少化等。

移除无用资源文件要比移除无用代码容易，在Android Studio的任何文件中右击，选择清除无用资源即可删除没有用到的资源文件。

可以考虑使用TinyPng、pngquant、ImageOptim等工具对图片进行压缩，这些工具可以减少PNG文件大小，同时保持图像质量。

图片优化。比如利用 AAPT 工具对 PNG 格式的图片做压缩处理，降低图片色彩位数等。

备注：需要注意的是在Android构建流程中AAPT会使用内置的压缩算法来优化res/drawable/目录下的PNG图片，但也可能会导致本来已经优化过的图片体积变大，可以通过在build.gradle中设置cruncherEnabled来禁止AAPT采用默认方式优化我们已经优化过的图片。

```
aaptOptions {
    cruncherEnabled = false
}
```

只保留指定的语言文件
```
resConfigs('zh-rCN','en')
```
只保留指定的SO：

```
ndk{
    abiFilters('armeabi','armeabi-v7a')
    }
```
使用Tint着色器

###### 使用矢量图
使用矢量图片能够有效的减少App中图片所占用的大小，矢量图形在Android中表示为VectorDrawable对象。

优点

- 图片扩展性：不损伤图片质量，一套图适配所有；
- 图片非常小：比使用位图小十几倍，有利于减小apk体积；

缺点

- 性能优损失，系统渲染VectorDrawable需要花费更多时间，因为矢量图的初始化加载会比相应的光栅图片消耗更多的CPU周期，但是两者之间的内存消耗和性能接近；
- 矢量图主要用在色调单一的icon。

###### 使用 WebP图片格式
优点：

- WebP在同画质下体积更小，WebP支持透明度，压缩比比JPEG更高但显示效果却不输于JPEG；
- 可以通过工具、云服务等进行PNG到WebP的转换；

缺点：
- Android从4.0才开始WebP的原生支持，意味着要兼容4.0以下机型需要添加适配库；当然现在市面上适配4.0以下的应用已经很少了。
- Android 4.2.1+才支持显示含透明度的WebP，因此最低版本小于4.2.1的App也不是想用就能用的。可以将不显示透明度的图片转换为WebP。

###### 资源混淆
在Apk打包过程中，aapt会将每一个资源生成一个对应的int数值，而我们通过这个int值来查找使用资源。在Apk构成中，我们可以看到里面有一个resources.arsc文件，里面保存着资源id和资源key的映射关系。

当调用图片时，先找到drawable分类，再根据当前的系统config找到匹配的config表，根据id找到对应的res数据。drawable在arsc中是当做string类型保存的，res数据中有这个资源在res string pool池中的索引。根据这个索引可以在字符串池中找到一个字符串。这个字符串其实就是一个路径，比如：res/drawable-xhdpi/icon.png；混淆就是将这个路径改为R/s/f.png；同时修改resources.arsc文件的映射关系。这样就能清楚的看出来资源混淆能减小Apk的原因：

- resources.arsc变小；
- 文件信息变小，采用了超短路径，res/drawable-xhdpi/icon.png被修改为R/s/f.png。

推荐微信的资源混淆方案：[AndResGuard](https://github.com/shwenzhang/AndResGuard)。
######  资源在线化
将部分使用频率不高的资源例如图片，放在网上，在恰当的时机提前下载，
###### 插件化。比如功能模块放在服务器上，按需下载，可以减少安装包大小。

###### 7Zip压缩
我们知道Apk文件实际上就是一个Zip文件。Android SDK的打包工具apkbuilder采用的是Deflate算法将Android App的代码、资源等文件进行压缩，压缩成Zip格式，然后签名发布。

使用7Zip对Apk进行极限压缩。需要注意这样极限压缩之后的签名被破坏，需要重新签名。

## 列表优化

## 性能优化工具
###### 查看 Layout 层次的 Hierarchy View

###### Android 系统上带的 GPU Profile 工具

###### LeakCanary工具

###### Android Lint 工具

###### Memory Analyzer Tool(MAT)
MAT 是一个快速，功能丰富的 Java Heap 分析工具，通过分析 Java 进程的内存快照 HPROF 分析，从众多的对象中分析，快速计算出在内存中对象占用的大小，查看哪些对象不能被垃圾收集器回收，并可以通过视图直观地查看可能造成这种结果的对象。




