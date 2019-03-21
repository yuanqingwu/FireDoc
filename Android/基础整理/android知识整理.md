# 框架原理
## 1.Retrofit的实现与原理 
https://blog.csdn.net/jiankeufo/article/details/73186929

动态代理
```
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```
缓存
```
Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();
```
工厂模式


## 2.Glide缓存源码，加载原理 
#### 生命周期的监听

Glide中一个重要特性是Request可以随Activity或Fragment的onStart而resume，onStop而pause，onDestroy而clear，从而节约流量和内存，并且防止内存泄露


2.创建一个透明的 RequestManagerFragment 加入到FragmentManager 之中
通过添加的这个 Fragment 感知 Activity 、Fragment 的生命周期。

#### 缓存
https://www.cnblogs.com/ldq2016/p/8980272.html

在使用中的图片使用弱引用来进行缓存，不在使用中的图片使用LruCache来进行缓存

分为内存缓存与硬盘缓存。内存缓存的主要作用是防止应用重复将图片数据读取到内存当中，而硬盘缓存的主要作用是防止应用重复从网络或其他地方重复下载和读取数据

##### 内存缓存

缓存的key,决定缓存Key的参数有10个之多
```
 EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());
```
默认开启内存缓存，调用skipMemoryCache()方法并传入true，就表示禁用掉Glide的内存缓存功能

Glide内存缓存的实现使用的LruCache算法结合一种弱引用的机制

LruCache算法（Least Recently Used），也叫近期最少使用算法。它的主要算法原理就是把最近使用的对象用强引用存储在LinkedHashMap中，并且把最近最少使用的对象在缓存值达到预设定值之前从内存中移除

EngineResource是用一个acquired变量用来记录图片被引用的次数，调用acquire()方法会让变量加1，调用release()方法会让变量减1。
当acquired变量大于0的时候，说明图片正在使用中，也就应该放到activeResources弱引用缓存当中。而经过release()之后，如果acquired变量等于0了，说明图片已经不再被使用了，那么此时会调用listener的onResourceReleased()方法来释放资源，会将缓存图片从activeResources中移除，然后再将它put到LruResourceCache当中。这样也就实现了正在使用中的图片使用弱引用来进行缓存，不在使用中的图片使用LruCache来进行缓存的功能

##### 硬盘缓存

调用diskCacheStrategy()方法并传入DiskCacheStrategy.NONE，就可以禁用掉Glide的硬盘缓存功能了

Glide默认情况下在硬盘缓存的就是转换过后的图片，我们通过调用diskCacheStrategy()方法则可以改变这一默认行为，它可以接收四种参数：

- DiskCacheStrategy.NONE： 表示不缓存任何内容。
- DiskCacheStrategy.SOURCE： 表示只缓存原始图片。
- DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。
- DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。

硬盘缓存的实现也是使用的LruCache算法

如果我们是缓存的原始图片，只使用了id和signature这两个参数来构成缓存Key。而signature参数绝大多数情况下都是用不到的，因此基本上可以说就是由id（也就是图片url）来决定的Original缓存Key
```
public Key getOriginalKey() {
    if (originalKey == null) {
        originalKey = new OriginalKey(id, signature);
    }
    return originalKey;
}
```
 硬盘缓存写入：先判断是否允许缓存原始图片，如果允许的话又会调用cacheAndDecodeSourceData()方法。而在这个方法中同样调用了getDiskCache()方法来获取DiskLruCache实例，接着调用它的put()方法就可以写入硬盘缓存了，注意原始图片的缓存Key是用的resultKey.getOriginalKey()
 
 
 ###### 问题
 
 1. 图片url地址在基础之上带有一个token参数，如
```
http://url.com/image.jpg?token=d9caa6e02c990b0a
```
token作为一个验证身份的参数并不是一成不变的，很有可能时时刻刻都在变化。而如果token变了，那么图片的url也就跟着变了，图片url变了，缓存Key也就跟着变了。结果就造成了，明明是同一张图片，就因为token不断在改变，导致Glide的缓存功能完全失效了

解决：
GlideUrl类的构造函数接收两种类型的参数，一种是url字符串，一种是URL对象。然后getCacheKey()方法中的判断逻辑非常简单，如果传入的是url字符串，那么就直接返回这个字符串本身，如果传入的是URL对象，那么就返回这个对象toString()后的结果

创建一个MyGlideUrl继承自GlideUrl，重写了getCacheKey()方法，在里面加入了一段逻辑用于将图片url地址中token参数的这一部分移除掉。这样getCacheKey()方法得到的就是一个没有token参数的url地址，从而不管token怎么变化，最终Glide的缓存Key都是固定不变的了。如下调用即可：

```
Glide.with(this)
     .load(new MyGlideUrl(url))
     .into(imageView);
```


## ButterKnife实现原理 
Butterknife用的APT(Annotation Processing Tool)编译时解析技术（现在已经改成了谷歌的更强大的annotationProcessor，APT已经停止更新了）。大概原理就是你声明的注解的生命周期为CLASS,然后继承AbstractProcessor类。继承这个类后，在编译的时候，编译器会扫描所有带有你要处理的注解的类，然后再调用AbstractProcessor的process方法，对注解进行处理，那么我们就可以在处理的时候，动态生成绑定事件或者控件的java代码，然后在运行的时候，直接调用方法完成绑定。


- AbstractProcessor / Annotation Processor Tool  (apt)	是javac的一个工具，用来在编译时扫描和编译和处理注解（Annotation）。你可以自己定义注解和注解处理器去搞一些事情。一个注解处理器它以Java代码或者（编译过的字节码）作为输入，生成文件（通常是java文件）。这些生成的java文件不能修改，并且会同其手动编写的java代码一样会被javac编译。
- RetentionPolicy.SOURCE	编译之后抛弃，存活的时间在源码和编译时
- RetentionPolicy.CLASS	保留在编译后class文件中,但JVM将会忽略(对于 底层系统开发-开发编译过程 使用)    apt中source和class都一样
- RetentionPolicy.RUNTIME	能够使用反射读取和使用（运行时）

## EventBus实现原理 
https://www.jianshu.com/p/bda4ed3017ba

## leakcanary原理
[LeakCanary，30分钟从入门到精通](https://www.jianshu.com/p/1e7e9b576391)

[LeakCanary原理解析](https://blog.csdn.net/xiaohanluo/article/details/78196755)

具体是这样做的，封装成带key的弱引用对象，然后GC看弱引用对象有没有回收，没有回收的话就怀疑是泄漏了，需要二次确认。然后生成HPROF文件，分析这个快照文件有没有存在带这个key值的泄漏对象，如果没有，那么没有泄漏，否则找出最短路径，打印给我们，我们就能够找到这个泄漏对象了。

1. RefWatcher.watch() 创建一个 KeyedWeakReference 到要被监控的对象。

2. 然后在后台线程检查引用是否被清除，如果没有，调用GC。

3. 如果引用还是未被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 .hprof 文件中。

4. 在另外一个进程中的 HeapAnalyzerService 有一个 HeapAnalyzer 使用HAHA 解析这个文件。

5. 得益于唯一的 reference key, HeapAnalyzer 找到 KeyedWeakReference，定位内存泄露。

6. HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄露。如果是的话，建立导致泄露的引用链。

7. 引用链传递到 APP 进程中的 DisplayLeakService， 并以通知的形式展示出来。

#### 1. Activity检测机制是什么？

答： 通过application.registerActivityLifecycleCallbacks来绑定Activity生命周期的监听，从而监控所有Activity; 在Activity执行onDestroy时，开始检测当前页面是否存在内存泄漏，并分析结果。因此，如果想要在不同的地方都需要检测是否存在内存泄漏，需要手动添加。

#### 2. 内存泄漏检测机制是什么？

答： KeyedWeakReference与ReferenceQueue联合使用，在弱引用关联的对象被回收后，会将引用添加到ReferenceQueue；清空后，可以根据是否继续含有该引用来判定是否被回收；判定回收， 手动GC, 再次判定回收，采用双重判定来确保当前引用是否被回收的状态正确性；如果两次都未回收，则确定为泄漏对象。

#### 3. 内存泄漏轨迹的生成过程 ？

答： 该版本采用eclipse.Mat来分析泄漏详细，从GCRoot开始逐步生成引用轨迹。

## AOP与OOP与APT
AOP为Aspect Oriented Programming的缩写，
意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。

- OOP（面向对象编程）针对业务处理过程的实体及其属性和行为进行抽象封装，以获得更加清晰高效的逻辑单元划分。


- 而AOP则是针对业务处理过程中的切面进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果。这两种设计思想在目标上有着本质的差异。

OOD/OOP面向名词领域，AOP面向动词领域。

[深入理解Android之AOP](https://blog.csdn.net/innost/article/details/49387395)

-  APT：Java5 中叫APT(Annotation Processing Tool)，在Java6开始，规范化为 Pluggable Annotation Processing

[APT实现BindView](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2018/0423/9629.html)

[安卓AOP三剑客:APT,AspectJ,Javassist](https://www.jianshu.com/p/dca3e2c8608a?from=timeline)

APT,AspectJ,Javassist对应的编译时期：
![APT,AspectJ,Javassist对应的编译时期](https://upload-images.jianshu.io/upload_images/751860-0641778f0bc265ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/540)

AOP仅仅只是个概念，实现它的方式（工具和库）有以下几种：

- AspectJ: 一个 JavaTM 语言的面向切面编程的无缝扩展（适用Android）。
- Javassist for Android: 用于字节码操作的知名 java 类库 Javassist 的 Android 平台移植版。
- DexMaker: Dalvik 虚拟机上，在编译期或者运行时生成代码的 Java API。
- ASMDEX: 一个类似 ASM 的字节码操作库，运行在Android平台，操作Dex字节码。

[深入理解Android之AOP](https://blog.csdn.net/innost/article/details/49387395)

AOP注解与使用:
- @Aspect：声明切面，标记类
- @Pointcut(切点表达式)：定义切点，标记方法
- @Before(切点表达式)：前置通知，切点之前执行
- @Around(切点表达式)：环绕通知，切点前后执行
- @After(切点表达式)：后置通知，切点之后执行
- @AfterReturning(切点表达式)：返回通知，切点方法返回结果之后执行
- @AfterThrowing(切点表达式)：异常通知，切点抛出异常时执行

切点表达式：
execution(<@注解类型模式>? <修饰符模式>? <返回类型模式> <方法名模式>(<参数模式>) <异常模式>?)

execution表示JPoint是执行方法的地方，AspectJ会对被执行方法做处理。而call表示JPoint是调用方法的地方，AspectJ会对调用处做处理。

## 热修复 
- 阿里系的从native层入手，见[AndFix](https://github.com/alibaba/AndFix)
- QQ空间的方案，插桩，见[安卓App热补丁动态修复技术介绍](http://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=0#wechat_redirect)
- 微信的方案，见[微信Android热补丁实践演进之路](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286306&idx=1&sn=d6b2865e033a99de60b2d4314c6e0a25&scene=0&uin=MTY4NDEwODc2Mg%3D%3D&key=cf237d7ae24775e8857eb5c8144bf27076228deefafb6a0afd4416d54ed479daa6a67f0363c96df893d5cf4e4d3db423&devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.11.6+build%2815G31%29&version=12000006&lang=zh_CN&nettype=WIFI&fontScale=100&pass_ticket=95LpBT0hMvr3CsOzDCiFvaRxltvzQUdRKYWgyVX2qwUNRC1PW%2Bmf8ebbmyh6bwrH)，dexDiff和dexPatch，方法很牛逼，需要全量插入，但是这个全量插入的dex中需要删除一些过早加载的类，不然同样会报class is pre verified异常，还有一个缺点就是合成占内存和内置存储空间。微信读书的方式和微信类似，见[Android Patch 方案与持续交付](http://wereadteam.github.io/2016/07/26/AndroidPatch/?from=timeline&isappinstalled=0)，不过微信读书是miniloader方式，启动时容易ANR，在我锤子手机上变现出来特别明显，长时间的卡图标现象。
- 美团的方案，也就是instant run的方案，见[Android热更新方案Robust](http://tech.meituan.com/android_robust.html)


## 热修复如何修复资源文件
[Android 热补丁技术——资源的热修复](https://blog.csdn.net/sbsujjbcy/article/details/52541803)

- 提前感知系统兼容性，不兼容则不进行后续操作
- 服务器端生成patch的资源，客户端应用patch的资源
- 替换系统AssetManger，加入patch的资源


# Android系统原理
## Application生命周期，activity 具体场景下的生命周期
- Application

https://developer.android.google.cn/reference/android/app/Application
onConfigurationChanged

onCreate

> Called when the application is starting, before any activity, service, or receiver objects (excluding content providers) have been created.
>
>Implementations should be as quick as possible (for example using lazy initialization of state) since the time spent in this function directly impacts the performance of starting the first activity, service, or receiver in a process.
>
>If you override this method, be sure to call super.onCreate().
>
>Be aware that direct boot may also affect callback order on Android Build.VERSION_CODES.N and later devices. Until the user unlocks the device, only direct boot aware components are allowed to run. You should consider that all direct boot unaware components, including such ContentProvider, are disabled until user unlock happens, especially when component callback order matters.
>
>If you override this method you must call through to the superclass implementation.
onLowMemory

onTerminate
> This method is for use in emulated process environments. It will never be called on a production Android device, where processes are removed by simply killing them; no user code (including this callback) is executed when doing so.
>
> If you override this method you must call through to the superclass implementation.

onTrimMemory

> Called when the operating system has determined that it is a good time for a process to trim unneeded memory from its process. This will happen for example when it goes in the background and there is not enough memory to keep as many background processes running as desired. You should never compare to exact values of the level, since new intermediate values may be added -- you will typically want to compare if the value is greater or equal to a level you are interested in.
>
> To retrieve the processes current trim level at any point, you can use ActivityManager.getMyMemoryState(RunningAppProcessInfo).
>
> If you override this method you must call through to the superclass implementation.



## activity启动模式（launchMode）
- standard
>   The default mode, which will usually create a new instance of the activity when it is started, though this behavior may change with the introduction of other options such as {@link android.content.Intent#FLAG_ACTIVITY_NEW_TASK Intent.FLAG_ACTIVITY_NEW_TASK}.
- singleTop
> If, when starting the activity, there is already an instance of the same activity class in the foreground that is interacting with the user, then re-use that instance. This existing instance will receive a call to {@link android.app.Activity#onNewIntent Activity.onNewIntent()} with the new Intent that is being started.
- singleInstance
> Only allow one instance of this activity to ever be running. This activity gets a unique task with only itself running in it; if it is ever launched again with the same Intent, then that task will be brought forward and its {@link android.app.Activity#onNewIntent Activity.onNewIntent()} method called. If this activity tries to start a new activity, that new activity will be launched in a separate task. See the Tasks and Back Stack document for more details about tasks.
- singleTask
> If, when starting the activity, there is already a task running that starts with this activity, then instead of starting a new instance the current task is brought to the front. The existing instance will receive a call to {@link android.app.Activity#onNewIntent Activity.onNewIntent()} with the new Intent that is being started, and with the {@link android.content.Intent#FLAG_ACTIVITY_BROUGHT_TO_FRONT Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT} flag set. This is a superset of the singleTop mode, where if there is already an instance of the activity being started at the top of the stack, it will receive the Intent as described there (without the FLAG_ACTIVITY_BROUGHT_TO_FRONT flag set). See the Tasks and Back Stack document for more details about tasks.


singleTask默认有clearTop效果

singleInstance是一种加强的singleTask模式，除了拥有singleTask模式所有的特性外，还加强了一点，就是此种模式的Activity只能单独地位于一个任务栈中。

TaskAffinity参数标识了一个activity所需要的任务栈的名字，默认情况下，所有activity所需的任务栈的名字为应用包名。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，在其他情况下没有意义。


Task是Android系统中的概念，它不同于进程Process的概念。简单地说，一个Task是一系列Activity的集合，这个集合是以堆栈的形式来组织的，遵循后进先出的原则。是用户为了完成某个目标而需要执行的一系列操作的过程的一种抽象。这是一个非常重要的概念，它从用户体验的角度出发，界定了应用程序的边界，极大地方便了开发者利用现成的组件（Activity）来搭建自己的应用程序，就像搭积木一样，而且，它还为应用程序屏蔽了底层的进程，即一个任务中的Activity可以都是运行在同一个进程中，也中可以运行在不同的进程中

## Activity的生命周期，finish调用后其他生命周期还会走么
Android中activity可以调用finish方法，结束自己，但是调用finish方法，activity到底会走那些生命周期方法：
- 在onCreate中：onCreate->onDestroy
- 在onStart中：onCreate->onStart->onStop->onDestroy
- 在onResume中：onCreate->onStart->onResume->onPause->onStop->onDestroy


## 应用详细启动过程，涉及的进程，fork新进程(Linux)
[Android应用程序启动过程源代码分析](https://blog.csdn.net/luoshengyang/article/details/6689748)

Android应用启动流程分析：https://blog.csdn.net/bfboys/article/details/52564531

## 详细描述应用从点击桌面图标到首页Activity展示的流程(应用启动流程，Activity，Window创建过程)

1.应用启动流程

- 冷启动

![image](http://hi.csdn.net/attachment/201108/14/0_1313305334OkCc.gif)
 在这个图中，ActivityManagerService和ActivityStack位于同一个进程中，而ApplicationThread和ActivityThread位于另一个进程中。其中，ActivityManagerService是负责管理Activity的生命周期的，ActivityManagerService还借助ActivityStack是来把所有的Activity按照后进先出的顺序放在一个堆栈中；对于每一个应用程序来说，都有一个ActivityThread来表示应用程序的主进程，而每一个ActivityThread都包含有一个ApplicationThread实例，它是一个Binder对象，负责和其它进程进行通信。

Step 1. 无论是通过Launcher来启动Activity，还是通过Activity内部调用startActivity接口来启动新的Activity，都通过Binder进程间通信进入到ActivityManagerService进程中，并且调用ActivityManagerService.startActivity接口； 

Step 2. ActivityManagerService调用ActivityStack.startActivityMayWait来做准备要启动的Activity的相关信息；

Step 3. ActivityStack通知ApplicationThread要进行Activity启动调度了，这里的ApplicationThread代表的是调用ActivityManagerService.startActivity接口的进程，对于通过点击应用程序图标的情景来说，这个进程就是Launcher了，而对于通过在Activity内部调用startActivity的情景来说，这个进程就是这个Activity所在的进程了；

Step 4. ApplicationThread不执行真正的启动操作，它通过调用ActivityManagerService.activityPaused接口进入到ActivityManagerService进程中，看看是否需要创建新的进程来启动Activity；

Step 5. 对于通过点击应用程序图标来启动Activity的情景来说，ActivityManagerService在这一步中，会调用startProcessLocked来创建一个新的进程，而对于通过在Activity内部调用startActivity来启动新的Activity来说，这一步是不需要执行的，因为新的Activity就在原来的Activity所在的进程中进行启动；

Step 6. ActivityManagerServic调用ApplicationThread.scheduleLaunchActivity接口，通知相应的进程执行启动Activity的操作；

Step 7. ApplicationThread把这个启动Activity的操作转发给ActivityThread，ActivityThread通过ClassLoader导入相应的Activity类，然后把它启动起来。



整个应用程序的启动过程要执行很多步骤，但是整体来看，主要分为以下五个阶段：

一. Step1 - Step 11：Launcher通过Binder进程间通信机制通知ActivityManagerService，它要启动一个Activity；

二. Step 12 - Step 16：ActivityManagerService通过Binder进程间通信机制通知Launcher进入Paused状态；

三. Step 17 - Step 24：Launcher通过Binder进程间通信机制通知ActivityManagerService，它已经准备就绪进入Paused状态，于是ActivityManagerService就创建一个新的进程，用来启动一个ActivityThread实例，即将要启动的Activity就是在这个ActivityThread实例中运行；

四. Step 25 - Step 27：ActivityThread通过Binder进程间通信机制将一个ApplicationThread类型的Binder对象传递给ActivityManagerService，以便以后ActivityManagerService能够通过这个Binder对象和它进行通信；

五. Step 28 - Step 35：ActivityManagerService通过Binder进程间通信机制通知ActivityThread，现在一切准备就绪，它可以真正执行Activity的启动操作了。

- 热启动

应用程序内部启动新的Activity的过程少了中间创建新的进程这一步，这是因为新的Activity是在已有的进程和任务中执行的，无须创建新的进程和任务。


2.activity创建

![image](https://img-blog.csdn.net/20150702153425952)
https://img-blog.csdn.net/20150702153425952

![image](https://img-blog.csdn.net/20151017183245096)
https://img-blog.csdn.net/20151017183245096


一个Activity中有一个Window对象用于描述当前应用的窗口，而Window是抽象类，实际上指向PhoneWindow类。PhoneWindow类中有一个内部类DecorView用于描述窗口的顶层视图，该视图才是真正装载layout.xml布局内容的,该类实际上是ViewGroup类型。当往DecorView上添加标题栏和窗口内容容器之后，最后将layout.xml布局添加到窗口的内容容器中，即完成了窗口Window添加视图View了

一个应用不管有多少个Activity，都只有一个WindowManager窗口管理器，也就只有一个WindowManagerGlobal对象通过维护三个数组 mViews，mRoots，mParams来管理所有窗口的添加，更新，删除。

一个窗口对应着一个ViewRootImpl类，该类主要的作用就是将窗口的顶层视图DecorView内容绘制到手机屏幕上。


1 Activity在其attach方法中创建了Window，实际上创建的是PhoneWindow，并通过各种回调等建立起与Activity的联系，我们在Activity中使用getWindow（）所得到的即是这个创建的Window。

2 在PhoneWindow中装载了一个顶级的View，即DecorView，它实际是一个FrameLayout，并通过各种主题样式选择加载不同的视图来填充DecorView，这部分被称作mContentRoot。

3 setContentView设置我们自己需要展示的视图，这部分视图被填充到了DecorView的一块子ViewGroup中，这块子ViewGroup被称作contentParent。

4 DecorView最终是通过WindowManager的addView方法添加到Window的，并且最终在Activity的onResume方法执行后显示出来。

5 在使用requestWindowFeature来设置样式时，实际上是调用了PhoneWindow的requestFeature方法，会将样式存储在Window的mLocalFeatures变量中，当installDecor时，会应用这些样式。也就是说，当需要通过requestWindowFeature来请求样式时，应该在setContentView方法之前调用，因为setContentView方法的调用会导致DecorView的创建并应用样式，如果在之后调用则会导致不会生效，因为此时DecorView已经创建完成了。

## Android两种虚拟机区别与联系
[JAVA虚拟机与Android虚拟机的区别](https://www.jianshu.com/p/8edac8e09b3e)

ART与DVM的区别

DVM中的应用每次运行时，字节码都需要通过即时编译器（JIT，just in time）转换为机器码，这会使得应用的运行效率降低。而在ART中，系统在安装应用时会进行一次预编译（AOT，ahead of time）,将字节码预先编译成机器码并存储在本地，这样应用每次运行时就不需要执行编译了，运行效率也大大提升。
ART占用空间比Dalvik大（字节码变为机器码之后，可能会增加10%-20%），这就是“时间换空间大法”。
预编译也可以明显改善电池续航，因为应用程序每次运行时不用重复编译了，从而减少了 CPU 的使用频率，降低了能耗。

## Activity的onNewIntent 
This is called for activities that set launchMode to "singleTop" in their package, or if a client used the Intent.FLAG_ACTIVITY_SINGLE_TOP flag when calling startActivity. In either case, when the activity is re-launched while at the top of the activity stack instead of a new instance of the activity being started, onNewIntent() will be called on the existing instance with the Intent that was used to re-launch it. 

An activity will always be paused before receiving a new intent, so you can count on onResume being called after this method. 

Note that getIntent still returns the original Intent. You can use setIntent to update it to this new Intent. 



## ActivityThread工作原理 

## service
[关于Android Service真正的完全详解，你需要知道的一切](https://blog.csdn.net/javazejian/article/details/52709857)



## 广播机制

[Android四大组件：BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)

//FIXME

从实现原理看上，Android中的广播使用了观察者模式，基于消息的发布/订阅事件模型。因此，从实现的角度来看，Android中的广播将广播的发送者和接受者极大程度上解耦，使得系统能够方便集成，更易扩展。具体实现流程要点粗略概括如下：

1.广播接收者BroadcastReceiver通过Binder机制向AMS(Activity Manager Service)进行注册；

2.广播发送者通过binder机制向AMS发送广播；

3.AMS查找符合相应条件（IntentFilter/Permission等）的BroadcastReceiver，将广播发送到BroadcastReceiver（一般情况下是Activity）相应的消息循环队列中；

4.消息循环执行拿到此广播，回调BroadcastReceiver中的onReceive()方法。

 对于不同的广播类型，以及不同的BroadcastReceiver注册方式，具体实现上会有不同。但总体流程大致如上。


 Android广播的分类：

1、 普通广播：这种广播可以依次传递给各个处理器去处理

2.System Broadcast: 系统广播

3、 有序广播：这种广播在处理器端的处理顺序是按照处理器的不同优先级来区分的，高优先级的处理器会优先截获这个消息，并且可以将这个消息删除

4、 粘性消息：粘性消息在发送后就一直存在于系统的消息容器里面，等待对应的处理器去处理，如果暂时没有处理器处理这个消息则一直在消息容器里面处于等待状态，粘性广播的Receiver如果被销毁，那么下次重建时会自动接收到消息数据。

(在 android 5.0/api 21中deprecated,不再推荐使用，相应的还有粘性有序广播，同样已经deprecated)

注意：普通广播和粘性消息不同被截获，而有序广播是可以被截获的。

在Android系统粘性广播一般用来确保重要的状态改变后的信息被持久保存，并且能随时广播给新的广播接收器，比如电源的改变，因为耗电需要一个过程，前一个过程必须提前得到，否则可能遇到下次刚好接收到的广播后系统自动关机了，随之而来的是kill行为，所以对某些未处理完的任务来说，后果很严重。

发送粘性广播需要权限（这里的权限是保存信息的权限和由系统发送未处理的广播的权限）
<uses-permission android:name="android.permission.BROADCAST_STICKY" />

5.Local Broadcast：App应用内广播

## Android消息机制，post与postDelay

设想一个场景，教室（线程）里的学生（Handler）有很多，只有一个老师（Looper），学生去不同教室（即不是创建该Handler对象的线程）学习写作业（创建Message），然后将作业带到自己教室里按顺序放到老师的桌子上（sendMessage插入消息队列），老师不断按顺序批改作业，每批改完一份作业就叫对应的学生过来拿作业本去修改错误（handleMessage）。

MessageQueue的中文翻译是消息队列，顾名思义，它的内部存储了一组消息，以队列的形式对外提供插入和删除的工作。虽然叫消息队列，但是它的内部存储结构并不是真正的队列，而是采用单链表的数据结构来存储消息列表。


采用的是无限循环，所以可能会有个疑问：该循环会不会特别消耗CPU资源？其实并不会，如果messageQueue有消息，自然是继续取消息；如果已经没有消息了，此时该线程便会阻塞在该next()方法的nativePollOnce() 方法中，主线程便会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生时，才通过往pipe管道写端写入数据来唤醒主线程工作。这里涉及到的是Linux的pipe/epoll机制，epoll机制是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。

在activity中直接用匿名内部类创建handler这样的写法，实际上是创建了一个继承Handler的Activity匿名内部类，而非静态的内部类会持有外部类的引用（即Activity的引用），而Message又持有了Handler对象的引用，所以如果当一个Message在队列中存放很久的时间或者设置Delay了很久，则Activity的视图和资源都无法及时释放内存。
建议写法：
```
     private static class MyHandler extends Handler {
        private final WeakReference<SampleActivity> mActivity;
            
        public MyHandler(SampleActivity activity) {
          mActivity = new WeakReference<SampleActivity>(activity);
        }
            
        @Override
        public void handleMessage(Message msg) {
          SampleActivity activity = mActivity.get();
          if (activity != null) {
            // ...
          }
        }
      }
```


## Binder机制，共享内存实现原理
https://blog.csdn.net/freekiteyu/article/details/70082302

[听说你Binder机制学的不错，来面试下这几个问题](https://www.jianshu.com/p/adaa1a39a274)

https://blog.csdn.net/universus/article/details/6211589


Android匿名共享内存（Ashmem）原理：https://www.jianshu.com/p/d9bc9c668ba6

Linux已经拥有的进程间通信IPC手段包括(Internet Process Connection)： 管道（Pipe）、信号（Signal）和跟踪（Trace）、插口（Socket）、报文队列（Message）、共享内存（Share Memory）和信号量（Semaphore）

一次拷贝的原理：
当数据从用户空间拷贝到内核空间的时候，是直从当前进程的用户空间接拷贝到目标进程的内核空间，这个过程是在请求端线程中处理的，操作对象是目标进程的内核空间。由于Binder内核空间的数据能直接映射到用户空间，这里就不在需要拷贝到用户空间。这就是一次拷贝的原理。内核空间的数据映射到用户空间其实就是添加一个偏移地址，并且将数据的首地址、数据的大小都复制到一个用户空间的Parcel结构体

## Parcelable与Serializable


对比 | Parcelable | Serializable
---|--- | ---
实现方式 | 实现Parcelable接口 |	实现Serializable接口
属于 | android 专用 | Java自带
内存消耗 | 优秀 | 一般
读写数据 | 内存中直接进行读写 | 通过使用IO流的形式将数据读写入在硬盘上
持久化 | 不可以 | 可以
速度 | 优秀 | 一般



Parcelable与Serializable的性能比较

  1）. 在内存的使用中,前者在性能方面要强于后者

  2）. 后者在序列化操作的时候会产生大量的临时变量,(原因是使用了反射机制)从而导致GC的频繁调用,因此在性能上会稍微逊色

  3）. Parcelable是以Ibinder作为信息载体的.在内存上的开销比较小,因此在内存之间进行数据传递的时候,Android推荐使用Parcelable,既然是内存方面比价有优势,那么自然就要优先选择.

  4）. 在读写数据的时候,Parcelable是在内存中直接进行读写,而Serializable是通过使用IO流的形式将数据读写入在硬盘上.

  但是：虽然Parcelable的性能要强于Serializable,但是仍然有特殊的情况需要使用Serializable,而不去使用Parcelable,因为Parcelable无法将数据进行持久化,因此在将数据保存在磁盘的时候,仍然需要使用后者,因为前者无法很好的将数据进行持久化.(原因是在不同的Android版本当中,Parcelable可能会不同,因此数据的持久化方面仍然是使用Serializable)

## 垃圾回收机制

## 主线程Looper.loop为什么不会造成死循环 
程序无响应：主线程执行任务时间较长，导致其他需要立刻在主线程处理的事件无法得到处理。
线程阻塞：线程处于等待状态
线程结束：线程的run方法返回

阻塞与程序无响应没有必然关系，虽然主线程在没有消息可处理的时候是阻塞的，但是只要保证有消息的时候能够立刻处理，程序是不会无响应的。
阻塞与线程退出也没有必然联系，线程完全可以在不阻塞的情况下死循环，同样达到不退出的效果。阻塞考虑到节约系统资源而做的处理，和线程退出没有关系。

## AsyncTask


## Android权限管理


## Android电量管理优化
[Optimize for battery life](https://developer.android.google.cn/topic/performance/power/)



# Android UI

## View的绘制原理

## 绘制
性能分析：https://blog.csdn.net/itachi85/article/details/61914979

布局优化：https://blog.csdn.net/itachi85/article/details/66475877

## SurfaceView, TextureView, SurfaceTexture等的区别
- SurfaceView是一个有自己独立Surface的View, 它的渲染可以放在单独线程而不是主线程中, 其缺点是不能做变形和动画。
- SurfaceTexture可以用作非直接输出的内容流，这样就提供二次处理的机会。与SurfaceView直接输出相比，这样会有若干帧的延迟。同时，由于它本身管理BufferQueue，因此内存消耗也会稍微大一些。
- TextureView是一个可以把内容流作为外部纹理输出在上面的View, 它本身需要是一个硬件加速层。
事实上TextureView本身也包含了SurfaceTexture, 它与SurfaceView+SurfaceTexture组合相比可以完成类似的功能（即把内容流上的图像转成纹理，然后输出）, 区别在于TextureView是在View hierachy中做绘制，因此一般它是在主线程上做的（在Android 5.0引入渲染线程后，它是在渲染线程中做的）。而SurfaceView+SurfaceTexture在单独的Surface上做绘制，可以是用户提供的线程，而不是系统的主线程或是渲染线程。另外，与TextureView相比，它还有个好处是可以用Hardware overlay进行显示。

## Fragment的懒加载实现，参数传递与保存
在Fragment中有一个setUserVisibleHint这个方法，而且这个方法是优于onCreate()方法的，所以也可以作为Fragment的一个生命周期来看待，它会通过isVisibleToUser告诉我们当前Fragment我们是否可见，我们可以在可见的时候再进行网络加载。

```
public void setUserVisibleHint(boolean isVisibleToUser
```

setUserVisibleHint(boolean isVisibleToUser) 这个方法的回调不是和fragment 的生命周期方法同步的, 也就是说该方法的调用有可能会在Fragemnt的onCreateView()方法被调用之前调用，这样一来就会出现NullPointerException空指针异常。所以就需要满足控件初始化完成，用户可见，之后才能加载数据。

setUserVisibleHint（）方法只有在ViewPager+**FragmentPagerAdapter**+Fragment结合使用的时候才会被调用，单用Fragment的时候可以考虑使用onHiddenChanged（boolean hidden）方法

setUserVisibleHint()在Fragment创建时会先被调用一次，传入isVisibleToUser = false
如果当前Fragment可见，那么setUserVisibleHint()会再次被调用一次，传入isVisibleToUser = true
如果Fragment从可见->不可见，那么setUserVisibleHint()也会被调用，传入isVisibleToUser = false
总结：setUserVisibleHint()除了Fragment的可见状态发生变化时会被回调外，在new Fragment()时也会被回调


setUserVisibleHint(boolean isVisibleToUser)方法是比onCreate更早调用的，但是我们一般在加载数据时，都会在数据加载完成时进行UI更新，所以这就有了一个问题，假如拉取数据是秒回，但是我们还没有进行UI绑定，或者是Adapter初始化等，那么我们就无法更新UI了，所以Fragment给我们提供了另一个方法getUserVisibleHint()，它就是用来判断当前Fragment是否可见，所以我们就可以在一系列变量初始化完成后再判断是否可见，若可见再进行数据拉取：

```
@Override
public void onStart() {
    super.onStart();
    Log.d("TAG", mTagName + " onStart()");

    ...

    if(getUserVisibleHint()) {
        pullData();
    }

}

```

## ViewPager的缓存实现
limit默认值为1，意味着会默认预加载当前页面之前和之后的一个页面

## 自定义View里，onDraw详细优化
尽可能的减少onDraw被调用的次数，大多数时候导致onDraw都是因为调用了invalidate().因此请尽量减少调用invaildate()的次数。如果可能的话，尽量调用含有4个参数的invalidate()方法而不是没有参数的invalidate()。没有参数的invalidate会强制重绘整个view。


### Canvas.save()跟Canvas.restore()的调用时机
- save：用来保存Canvas的状态。save之后，可以调用Canvas的平移、放缩、旋转、错切、裁剪等操作。
- restore：用来恢复Canvas之前保存的状态。防止save后对Canvas执行的操作对后续的绘制有影响。

对canvas中特定元素的旋转平移等操作实际上是对整个画布进行了操作，所以如果不对canvas进行save以及restore，那么每一次绘图都会在上一次的基础上进行操作，最后导致错位。比如说你相对于起始点每次30度递增旋转，30，60，90.如果不使用save 以及 restore 就会变成30, 90, 150，每一次在前一次基础上进行了旋转。


## Android动画 

## requestLayout，invalidate，postInvalidate区别与联系
![image](http://upload-images.jianshu.io/upload_images/1734948-b4493f7b0234dd69.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## RecyclerView与ListView(缓存原理，区别联系，优缺点) ，局部刷新，ItemTouchHelper，
[深入理解Android中的缓存机制(二)RecyclerView跟ListView缓存机制对比](https://www.jianshu.com/p/01f161cb498c)

[Android ListView 与 RecyclerView 对比浅析--缓存机制](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&scene=21#wechat_redirect)

[RecyclerView一些你可能需要知道的优化技术](https://www.jianshu.com/p/1d2213f303fc)


在RecyclerView的Adapter中被缓存的单位已经不是Item  View了，而是一个ViewHolder,而原来的ListView则是缓存的View。在RecyclerView.Recycler类中有mAttachedScrap,mChangedScrap,mCachedViews几个ViewHoler列表对象，他们就是用于缓存ViewHolder的，在通过LayoutState的next函数获取Item View时，实际上调用的是RecyclerView.Recycler的getViewForPosition函数，该函数会首先从这几个ViewHolder缓存中获取对应位置的ViewHolder，如果没有缓存则调用RecyclerView.Adapter中的createViewHolder函数来创建一个新的ViewHolder。

```
    /**
     * A Recycler is responsible for managing scrapped or detached item views for reuse.
     *
     * <p>A "scrapped" view is a view that is still attached to its parent RecyclerView but
     * that has been marked for removal or reuse.</p>
     *
     * <p>Typical use of a Recycler by a {@link LayoutManager} will be to obtain views for
     * an adapter's data set representing the data at a given position or item ID.
     * If the view to be reused is considered "dirty" the adapter will be asked to rebind it.
     * If not, the view can be quickly reused by the LayoutManager with no further work.
     * Clean views that have not {@link android.view.View#isLayoutRequested() requested layout}
     * may be repositioned by a LayoutManager without remeasurement.</p>
     */
    public final class Recycler {
        final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
        ArrayList<ViewHolder> mChangedScrap = null;
        //缓存的全部View，包含可见跟不可见的ViewHolder
        final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();

        private final List<ViewHolder>
                mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);
        //缓存的最大数量，默认为2
        private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
        int mViewCacheMax = DEFAULT_CACHE_SIZE;

        RecycledViewPool mRecyclerPool;

        private ViewCacheExtension mViewCacheExtension;
        //默认缓存数量
        static final int DEFAULT_CACHE_SIZE = 2;
        
        ......
        }
```


RecyclerView 滑动场景下的回收复用涉及到的结构体两个：mCachedViews 和 RecyclerViewPool。

mCachedViews 优先级高于 RecyclerViewPool，回收时，最新的 ViewHolder 都是往 mCachedViews 里放，mCachedViews默认大小是2，如果它满了，那就移出一个扔到 ViewPool 里好空出位置来缓存最新的 ViewHolder。
复用时，也是先到 mCachedViews 里找 ViewHolder，找到的话就不用再绑定数据。但需要各种匹配条件，概括一下就是只有原来位置的卡位可以复用存在 mCachedViews 里的 ViewHolder，如果 mCachedViews 里没有，那么才去 ViewPool 里找。
在 ViewPool 里的 ViewHolder 都是跟全新的 ViewHolder 一样，只要 type 一样，有找到，就可以拿出来复用，重新绑定下数据即可。


实际上RecyclerView做局部刷新是非常容易的，其实就是使用好带payload参数的这个notifyItemRangeChanged方法，以及override带payload的这个onBindViewHolder方法，在onBindViewHolder中去刷新你想更新的控件即可

## 自定义LayoutManager 

## 嵌套滑动实现原理 



# Android 项目结构

## 插件化

## MVC，MVP，MVVM模式理解与使用

#### MVP
MVP模式会解除View与Model的耦合，同时又带来良好的可扩展性，可测试性，保证了系统的简洁性，灵活性。

只要保证我们是通过Presenter将View和Model解耦合，降低类型复杂度，各个模块可以独立测试，独立变化。

###### 不依赖于具体而是依赖于抽象
Presenter可以运用于任何实现了View逻辑接口的UI，


###### MVP内存泄漏
由于Presenter经常需要执行一些耗时操作，而且持有了Activity的强引用，如果Activity先于耗时操作返回而被销毁了，那么Activity就会发生内存泄漏。

解决：

通过弱引用和Activity，Fragment的生命周期解决。

#### MVVM



# 优化

## 如何给一个app瘦身
[App瘦身最佳实践](https://www.jianshu.com/p/8f14679809b3)

- 第一种：配置build.gradle文件，开启minifyEnabled，作用是启用混淆压缩模式，会过滤掉整个项目中未使用到的jar和class文件，对代码进行混淆，从而减少dex文件大小。
- 配置build.gradle文件，开启shrinkResources，作用是将res目录下未使用到的图片文件进行特殊处理，其具体做法是将未使用到的图片全部变成1x1像素的小图，从而减少res目录的大小。
- 配置build.gradle文件，指定resConfigs，作用是指定打包时编译的语言包类型，未指定的其他语言包，将不会打包到apk文件中，从而减少apk体积的大小。 
一般我们只支持中文和英文。
- 采用三方工具（如tinypng）来进一步压缩项目中的所有png图片，从而进一步减小apk体积。
- 选择合适的图片格式
谷歌给出的建议，简单来说就是：VD->WebP->Png->JPG
如果是纯色的icon，那么用svg
 如果是两种以上颜色的icon，用webp
 如果webp无法达到效果，选择png
 如果图片没有alpha通道，可以考虑jpg
 VD即VectorDrawable，android上的svg实现类。
WebP格式，谷歌（google）开发的一种旨在加快图片加载速度的图片格式。图片压缩体积大约只有JPEG的2/3，并能节省大量的服务器带宽资源和数据空间。


# 应用性能监控

## 无埋点统计
- 字节码插桩（https://www.jianshu.com/p/b5ffe845fe2d）
  
关键技术：
1. View的唯一标识（ID）
2. 页面的划分
3. 无需埋点轻松收集定制的业务数据

asm介绍:https://www.ibm.com/developerworks/cn/java/j-lo-asm30/

- AspectJ、AOP编程、Android数据埋点、Android性能监控 （https://blog.csdn.net/xinanheishao/article/details/74082605）
- 利用无障碍服务来获取所有的屏幕交互事件，然后实现无埋点统计（https://www.jianshu.com/p/d1bf6934fb95）


## transient关键字
为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。当一个对象被序列化的时候，transient型变量的值不包括在序列化的表示中，然而非transient型的变量是被包括进去的。

1）一旦变量被transient修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。

2）transient关键字只能修饰变量，而不能修饰方法和类。注意，本地变量是不能被transient关键字修饰的。变量如果是用户自定义类变量，则该类需要实现Serializable接口。

3）被transient关键字修饰的变量不再能被序列化，一个静态变量不管是否被transient修饰，均不能被序列化。

## GC机制

[Hotspot JVM GC](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)

我们当前使用的sun(oracle)的java版本(应该是1.3以上)都是内置的HotSpot VM实现
![image](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/images/gcslides/Slide5.png)

1. Young Generation

新生代。
所有new的对象。
该区域的内存管理使用minor garbage collection(小GC).
更进一步分成Eden space, Survivor 0 和 Survivor 1 三个部分
2. Old Generation

老年区.
新生代中执行小粒度的GC幸存下来的"老"对象.
该区域的内存管理使用major garbage collection(大GC).
3. Permanent Generation

持久代.
包含应用的类/方法信息, 以及JRE库的类和方法信息.

小GC执行非常频繁, 而且速度特别快.
大GC一般会比小GC慢十倍以上.
大小GC都会发出"Stop the World"事件, 也就是说中断程序运行, 直至GC完成. 这也是我们在App优化之消除卡顿中为什么说频繁GC会造成用户感知卡顿.

###### GC Root
直译GC根, 我们姑且不译了吧.
所谓GC Root我们可以理解为是一个Heap内存之外的对象, 通常包括但不仅限于如下几种:

- System Class 系统Class Loader加载的类. 例如java运行环境中rt.jar中类, 比如java.util.* package中的类.
- Thread 运行中的线程
- JNI 中的本地/全局变量, 用户自定义的JNI代码或是JVM内部的.
- Busy Monitor 任何调用了wait()或notify()方法, 或是同步化的(synchronized)的东西. 可以理解为同步监控器.
- Java本地实例, 还在运行的Thread的stack中的方法创建的对象.



## 类的加载机制
http://www.importnew.com/25295.html/4003106-dd3ff46b4f9f3574


## Java反射

## RXJava
