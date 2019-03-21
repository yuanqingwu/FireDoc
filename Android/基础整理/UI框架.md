# UI架构
### Window 与WindowManager
Window表示一个窗口的概念。Window是个抽象类，他的具体实现是PhoneWindow。WindowManager是外界访问Window的入口，Window的具体实现在WindowManagerService中，WindowManager和WindowManagerService交互是个IPC过程。Android中所有的视图都是通过Window来呈现的，不管是Activity，Dialog还是Toast,他们的视图都是依附在Window上的。每一个Window都对应着一个View和一个ViewRootImpl,Window和View是通过ViewRootImpl来建立联系，因此Window并不是实际存在的，它是以View的形式存在。

Window 有三种类型，分别是应用 Window、子 Window 和系统 Window。应用类 Window 对应一个 Acitivity，子 Window 不能单独存在，需要依附在特定的父 Window 中，比如常见的一些 Dialog 就是一个子 Window。系统 Window是需要声明权限才能创建的 Window，比如 Toast 和系统状态栏都是系统 Window。

Window 是分层的，每个 Window 都有对应的 z-ordered，层级大的会覆盖在层级小的 Window 上面，这和 HTML 中的 z-index 概念是完全一致的。在三种 Window 中，应用 Window 层级范围是 1~99，子 Window 层级范围是 1000~1999，系统 Window 层级范围是 2000~2999。
这些层级范围对应着 WindowManager.LayoutParams 的 type 参数，如果想要 Window 位于所有 Window 的最顶层，那么采用较大的层级即可，很显然系统 Window 的层级是最大的，当我们采用系统层级时，需要声明权限。

Window是一个抽象类，目前只有PhoneWindow是其实现类。

有三个值得关注的：

- Window.Callback

    该接口使得依靠（可以确定的是所有界面都是通过Window展示的）Window来显示内容的其它对象——主要就是Activity——接收Window发送的交互等事件和消息回调。例如Activity.onAttachedToWindow()这样的回调。

- getDecorView

    DecorView就是和Window关联的用来显示内容的ViewTree的root。所有地方都使用Window来显示内容，不同的Window使用者会需要不同的DecorView，比如Activity就需要DecorView本身具备ActionBar这类的“界面装饰部分”。而Dialog明显没有。

- setContentView

    Window显示的自定义内容。Activity中的setContentView正是调用关联的Window对象的此方法。将界面内容附加到DecorView作为其子树。



##### Window的内部机制

- **ViewManager** 接口定义addView，updateViewLayout，removeView三个接口->
    
```
public interface ViewManager
{
    /**
     * Assign the passed LayoutParams to the passed View and add the view to the window.
     * <p>Throws {@link android.view.WindowManager.BadTokenException} for certain programming
     * errors, such as adding a second view to a window without removing the first view.
     * <p>Throws {@link android.view.WindowManager.InvalidDisplayException} if the window is on a
     * secondary {@link Display} and the specified display can't be found
     * (see {@link android.app.Presentation}).
     * @param view The view to be added to this window.
     * @param params The LayoutParams to assign to view.
     */
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

- **WindowManager** 接口继承ViewManager->

- **WindowManagerImpl** 实现WindowManager接口，获取WindowManagerGlobal单例，具体工作交给mGlobal实例 （桥接模式）->


```
private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();

 @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
    
   @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

    //在WindowManager中增加的removeViewImmediate接口
    @Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
```


- **WindowManagerGlobal** 创建ViewRootImpl并将View添加到列表中 ->

```
//所有window所对应的view
private final ArrayList<View> mViews = new ArrayList<View>();
//所有window所对应的ViewRootImpl
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
//所有window所对应的布局参数
private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
//正在被删除的View对象,或者说那些已经调用removeView方法但是删除操作还未完成的window对象
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```
++addView++
```
 public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
            //检查参数是否合法
            ......
            
            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
            
            // do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
            
    }
```

++removeView++
```
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            //查找待删除的View的索引
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            //通过ViewRootImpl进行删除，调用root.die(immediate)方法
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```

++updateViewLayout++
```
    public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        //更新View的LayoutParams
        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            //更新ViewRootImpl的LayoutParams
            root.setLayoutParams(wparams, false);
        }
    }
```


- **ViewRootImpl** ->

++addView++

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {

......
                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                requestLayout();

}

  @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            //View绘制的初始入口,scheduleTraversals->mTraversalRunnable->doTraversal()->performTraversals()
            scheduleTraversals();
        }
    }
```
接着会通过WindowSession最终来完成Window的添加过程


++removeView++
```
    /**
     * @param immediate True, do now if not in traversal. False, put on queue and do later.
     * @return True, request has been queued. False, request has been completed.
     */
    boolean die(boolean immediate) {
        // Make sure we do execute immediately if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
        if (immediate && !mIsInTraversal) {
        //同步删除，直接调用doDie方法
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
        //异步删除，Handler会调用doDie()方法
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
```
doDie()会调用dispatchDetachedFromWindow（）进行真正的删除操作


```
dispatchDetachedFromWindow主要做四件事：
1.垃圾回收相关的工作，比如清除数据和消息，移除回调
2.通过Session的remove方法删除Window: mWindowSession.remove(mWindow);经过IPC调用WindowManagerService的removeWindow方法
3.调用View的dispatchDetachedFromWindow方法，在内部会调用View的onDetachedFromWindow()以及onDetachedFromWindowInternal()。（onDetachedFromWindow在View从Window移除时会调用，我们可以在这个方法内部做一些资源回收的工作，比如终止动画，停止线程等）
4.调用WindowManagerGlobal 的doRemoveView方法刷新数据，包括mRoots,mParams以及mDyingViews,需要将当前Window所关联的这三类对象从列表中移除
```

++updateViewLayout++

```
void setLayoutParams(WindowManager.LayoutParams attrs, boolean newView) {
    ......
    //对View进行重新布局，包括测量，布局，重绘三个过程
    scheduleTraversals();
}
```

```
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            //mTraversalRunnable->doTraversal()->performTraversals()
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

除了View本身的重绘以外，ViewRootImpl还会通过WindowSession来更新Window的视图，这个过程最终是通过WIndowManagerService的relayoutWindow()来具体实现的。

- **IWindowSession** 是个Binder对象，实现类为Session->

- **Session** 通过WindowManagerService来实现window的添加，删除与更新->

- **WindowManagerService** 为每一个应用保留一个单独的Session

window的添加，删除与更新具体实现全部在WindowManagerService中


#### Window的创建
##### Activity的Window创建过程

- ActivityThread 

Activity的启动最终会由performLaunchActivity()来完成整个启动过程，在这个方法内会通过类加载器创建Activity的实例对象，并调用其attach方法为其关联运行过程中的一系列上下文环境变量：
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
 ......
//创建Activity实例
ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
         ......
        } catch (Exception e) {
           ......
        }

       try {
            //创建Application对象
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                //关联运行过程中的一系列上下文环境变量
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                        
                        ......
                //调用Activity的onCreate方法  
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
    }
......
return activity;
}
```
- Activity 在attach方法里系统会创建Activity所属的Window对象并为其设置回调接口

```
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
            
            //创建window
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        //实现了window的回调接口，当window接收到外接状态改变时就会回调Activity的方法，
        //比如onAttachedToWindow,onDetachedFromWindow,dispatchTouchEvent,等等
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();
            
            }
```



- PhoneWindow 创建DecorView

```
    /**
     * Constructor for main window of an activity.
     */
    public PhoneWindow(Context context, Window preservedWindow,
            ActivityConfigCallback activityConfigCallback) {
        this(context);
        // Only main activity windows use decor context, all the other windows depend on whatever
        // context that was given to them.
        mUseDecorContext = true;
        if (preservedWindow != null) {
        //创建DecorView
            mDecor = (DecorView) preservedWindow.getDecorView();
            mElevation = preservedWindow.getElevation();
            mLoadElevation = false;
            mForceDecorInstall = true;
            // If we're preserving window, carry over the app token from the preserved
            // window, as we'll be skipping the addView in handleResumeActivity(), and
            // the token will not be updated as for a new window.
            getAttributes().token = preservedWindow.getAttributes().token;
        }
        // Even though the device doesn't support picture-in-picture mode,
        // an user can force using it through developer options.
        boolean forceResizable = Settings.Global.getInt(context.getContentResolver(),
                DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
        mSupportsPictureInPicture = forceResizable || context.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_PICTURE_IN_PICTURE);
        mActivityConfigCallback = activityConfigCallback;
    }
```
- ActivityThread handleResumeActivity方法中会先调用Activity的onResume方法，接着调用r.activity.makeVisible();
- Activity 在makeVisible方法中DecorView才完成添加和显示两个过程，至此Activity的视图才能被用户看到

```
   void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```

##### Dialog的Window创建过程

- Dialog 创建window


```
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {

......

        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        //创建window
        final Window w = new PhoneWindow(mContext);
        mWindow = w;
        w.setCallback(this);
        w.setOnWindowDismissedCallback(this);
        w.setOnWindowSwipeDismissedCallback(() -> {
            if (mCancelable) {
                cancel();
            }
        });
        w.setWindowManager(mWindowManager, null, null);
        w.setGravity(Gravity.CENTER);

        mListenersHandler = new ListenersHandler(this);

}
```
- PhoneWindow 创建DecorView
- Dialog 在show方法中会通过WindowManager将DecorView添加到Window中

```
    /**
     * Start the dialog and display it on screen.  The window is placed in the
     * application layer and opaque.  Note that you should not override this
     * method to do initialization when the dialog is shown, instead implement
     * that in {@link #onStart}.
     */
    public void show() {
    ......
        mWindowManager.addView(mDecor, l);
        mShowing = true;
        
        sendShowMessage();
    }
```

- 关闭Dialog会通过WindowManager来移除DecorView

```
    void dismissDialog() {
    ......
     try {
            mWindowManager.removeViewImmediate(mDecor);
        } finally {
            if (mActionMode != null) {
                mActionMode.finish();
            }
            mDecor = null;
            mWindow.closeAllPanels();
            onStop();
            mShowing = false;

            sendDismissMessage();
    }
```

**普通Dialog的特殊之处**
普通Dialog必须采用Activity的Context，如果采用Application的Context就会报错 ：android.view.WindowManager$BadTokenException:Unable to add window--token null is not for an application

错误信息指出是没有应用token导致的，而应用token一般只有Activity有，所以这里需要用Activity作为Context来显示Dialog。另外系统Window比较特殊，它可以不需要token，因此也可以指定Dialog的Window为系统Window,也可以弹出Dialog.
WindowManager.LayoutParams中的type表示WIndow的类型，系统Window的层级范围是2000-2999，这里我们可以选用TYPE_SYSTEM_OVERLAY来指定对话框Window的类型为系统Window

```
dialog.getWindow().setType(LayoutParams.TYPE_SYSTEM_ERROR)

还需要在AndroidManifest文件中声明权限
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```

##### Toast的Window创建过程

### LayoutInflater的工作流程

LayoutInflater是一个抽象类：

```
@SystemService(Context.LAYOUT_INFLATER_SERVICE)
public abstract class LayoutInflater {
......
}
```
在加载ContextImpl时会通过如下代码将LayoutInflater的ServiceFetcher注入到容器中：

```
final class SystemServiceRegistry {

......

 static {
 ......
 registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
            ......
        }
    }

```
LayoutInflater的真正实现类是PhoneLayoutInflater：


```
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };

    /**
     * Instead of instantiating directly, you should retrieve an instance
     * through {@link Context#getSystemService}
     *
     * @param context The Context in which in which to find resources and other
     *                application-specific things.
     *
     * @see Context#getSystemService
     */
    public PhoneLayoutInflater(Context context) {
        super(context);
    }

    protected PhoneLayoutInflater(LayoutInflater original, Context newContext) {
        super(original, newContext);
    }

    /** Override onCreateView to instantiate names that correspond to the
        widgets known to the Widget factory. If we don't find a match,
        call through to our super class.
    */
    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }

        return super.onCreateView(name, attrs);
    }

    public LayoutInflater cloneInContext(Context newContext) {
        return new PhoneLayoutInflater(this, newContext);
    }
}
```
核心方法就是覆写了LayoutInflater的onCreateView方法，该方法就是在传递进来的View名字前面加上"android.widget.","android.webkit.","android.app."前缀以得到该内置View类的完整路径，最后根据类的完整路径来构造对应的View对象。

#### 以Activity的SetContentView为例

##### PhoneWindow
```
public class PhoneWindow extends Window implements MenuBuilder.Callback {
......
 public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        //mContentParent为空时先构建DecorView,并且将DecorView包裹到mContentParent中
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
        //解析layoutResID
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }


}
```
##### LayoutInflater.java

```
   public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }
        //获取XML资源解析器
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```

```
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            //Context对象
            mConstructorArgs[0] = inflaterContext;
            //存储父视图
            View result = root;

            try {
                // Look for the root node.
                int type;
                //找到root元素
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();

                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
                //解析merge标签
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    //不是merge标签那么就直接解析布局中的视图
                    //通过XML的tag来解析layout根视图，name就是要解析的视图的类名，如RelativeLayout
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        //生成布局参数
                        params = root.generateLayoutParams(attrs);
                        //如果attachToRoot为false，那么将给temp设置布局参数
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    //解析temp视图下的所有子view
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    //如果root不为空，且attachToRoot为true，那么将temp添加到父视图中
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    //如果root为空或者attachToRoot为false，那么返回的结果就是temp
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                        + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;

                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            return result;
        }
    }
```
上述inflate方法中，主要有下面几步：
1. 解析XML中的根标签（第一个元素）。
2. 如果根标签是merge，那么调用rInflate进行解析，rInflate会将merge标签下的所有子View直接添加到根标签中；
3. 如果根标签是普通元素那么调用createViewFromTag对该元素进行解析。
4. 调用rInflateChildren解析temp根元素下的所有子View,并且将这些子View都添加到temp下；
5. 返回解析到的根视图。
 


```
   View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
            
            ......
            
            try {
            View view;
            //用户可以通过设置LayoutInflater的factory来自行解析View，默认这些factory都为空，可以忽略
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            //没有Factory的情况下通过onCreateView或者createView创建View
            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                //内置View控件的解析
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                    //自定义控件的解析
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch
            ......省略多个catch块
        }
```
PhoneLayoutInflater覆写了onCreateView方法，该方法就是在View标签的前面设置一个 "android.widget."的前缀，然后再传递给createView解析。也就是说内置View和自定义View最终都是调用了createView进行解析，只是Google为了让开发者在XML中更方便定义View,只需要写View名称而不需要写完整路径。


```
 public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
        //从缓存中获取构造函数
        Constructor<? extends View> constructor = sConstructorMap.get(name);
        if (constructor != null && !verifyClassLoader(constructor)) {
            constructor = null;
            sConstructorMap.remove(name);
        }
        Class<? extends View> clazz = null;

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);
            //如果没有缓存构造函数
            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                //如果prefix不为空，那么构造完整的View路径，并且加载该类
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);

                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                //从Class对象中获取构造函数
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                //将构造函数存入缓存中
                sConstructorMap.put(name, constructor);
            } else {
                // If we have a filter, apply it to cached constructor
                if (mFilter != null) {
                    // Have we seen this name before?
                    Boolean allowedState = mFilterMap.get(name);
                    if (allowedState == null) {
                        // New class -- remember whether it is allowed
                        clazz = mContext.getClassLoader().loadClass(
                                prefix != null ? (prefix + name) : name).asSubclass(View.class);

                        boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                        mFilterMap.put(name, allowed);
                        if (!allowed) {
                            failNotAllowed(name, prefix, attrs);
                        }
                    } else if (allowedState.equals(Boolean.FALSE)) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
            }

            Object lastContext = mConstructorArgs[0];
            if (mConstructorArgs[0] == null) {
                // Fill in the context if not already within inflation.
                mConstructorArgs[0] = mContext;
            }
            Object[] args = mConstructorArgs;
            args[1] = attrs;
            //通过反射构造View
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            mConstructorArgs[0] = lastContext;
            return view;

        } catch(){}
        ......省略多个catch与finally代码块
        
    }
```
**createView**相对简单，如果有前缀，那么构造View的完整路径，并且将该类加载到虚拟机中，然后获取该类的构造函数并且缓存起来，再通过构造函数来创建该View的对象，最后将这个对象返回，这就是解析单个View的过程。

但是我们的窗口中是一颗视图树，LayoutInflate需要解析完这棵树，就得依靠**rInflate**方法：

```
 void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
        //获取树的深度，深度优先遍历
        final int depth = parser.getDepth();
        int type;
        boolean pendingRequestFocus = false;
        //挨个元素解析
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();

            if (TAG_REQUEST_FOCUS.equals(name)) {
                pendingRequestFocus = true;
                consumeChildElements(parser);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
            //解析inclue标签
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
            //解析merge标签，抛出异常，因为merge标签必须为根视图
                throw new InflateException("<merge /> must be the root element");
            } else {
            //根据元素名进行解析
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                //递归调用进行解析，也就是深度优先遍历
                rInflateChildren(parser, view, attrs, true);
                //将解析到的View添加到viewgroup中，也就是他的parent
                viewGroup.addView(view, params);
            }
        }

        if (pendingRequestFocus) {
            parent.restoreDefaultFocus();
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```

```
  final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
```
rInflate通过深度优先遍历来构造视图树，每解析到一个视图元素就会递归调用rInflate，直到这条路径下的最后一个元素，然后再回溯过来将每个View添加到他们的parent中。通过rInflate解析之后整棵视图树就构建完毕。当调用了Activity的onResume之后，我们通过setContentView设置的内容就会出现在我们的视野中。


### View 工作原理

View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure,layout,draw三个流程才能最终将一个View绘制出来。performTraversals会依次调用performMeasure,performLayout和performDraw三个方法，这三个方法分别完成顶级View的measure,layout,draw这三大流程，其中在performMeasure中会调用measure方法，在measure中又会调用onMeasure方法，在onMeasure中会对所有子元素进行measure过程，这时候measure流程就从父容器传递到子元素中了，这样就完成了依次measure过程。接着子元素会重复父容器的measure过程，如此反复就完成了整个View树的遍历。performLayout和performDraw的传递过程和performMeasure是类似的，唯一不同的是，performDraw的传递是在draw方法中通过dispatchDraw来实现的，不过这并没有本质区别。

measure过程决定了View的宽高，measure完成后可以通过getMeasuredWidth和getMeasuredHeight方法来获取到View的测量宽高。

layout决定了View的四个顶点的坐标和实际的宽高，完成后可以通过getTop,getBottom,getLeft和getRight来拿到View四个顶点的位置，并可以通过getWidth和getHeight方法来拿到View最终的宽高。

draw过程则决定了View的显示，只有draw方法完成后View的内容才能呈现在屏幕上。

###### DecorView
DecorView作为顶级View,一般会包含两部分，上面是标题栏，下面是内容栏（具体情况视android版本及主题有关）。Activity中通过setContentView()所设置的布局文件就是被添加到内容栏之中的。


##### ViewRoot
ViewRoot类在Android2.2之后就被ViewRootImpl替换了。
,它是连接WindowManager和DecorView的纽带，View的三大流程都是通过ViewRootImpl来完成的。
```
/**
 * The top of a view hierarchy, implementing the needed protocol between View
 * and the WindowManager.  This is for the most part an internal implementation
 * detail of {@link WindowManagerGlobal}.
 *
 * {@hide}
 */
 
 ViewRootImpl是一个视图层次结构的顶部，它实现了View与WindowManager之间
 所需要的协议，作为WindowManagerGlobal中大部分的内部实现
```

- View的绘制
```
private void performTraversals() { 
        ...... 
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); 
        ......
        performLayout(lp, desiredWindowWidth, desiredWindowHeight);
        ...... 
        performDraw(); 
    
        } 
        ...... 

    }

```
View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure,layout,draw三个流程才能最终将一个View绘制出来。

- 事件分发



##### DecorView
DecorView是PhoneWindow的内部类，DecorView是FrameLayout的子类。

```
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {


    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
    
     @Override
     protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
     ......
     }
    
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    ......
    }
    
    @Override
    public void draw(Canvas canvas) {
    ......
    }
    
    public void setWindowBackground(Drawable drawable) {
    ......
    }
    ......
}
```


##### View的工作流程

View的工作流程主要是指measure、layout、draw这三大流程，即测量、布局、绘制，其中measure确定View的测量宽/高，layout确定View的最终宽/高和四个顶点的位置，而draw则将View绘制到屏幕上。

###### MesureSpec

MeasureSpec代表一个32位的int值，前俩位代表SpecMode，后30位代表SpecSize.其中：SpecMode代表测量的模式，SpecSize值在某种测量模式下的规格大小。

注释如下：

A MeasureSpec encapsulates the layout requirements passed from parent to child. Each MeasureSpec represents a requirement for either the width or the height. A MeasureSpec is comprised of a size and a mode. There are three possible modes:

- UNSPECIFIED

The parent has not imposed any constraint on the child. It can be whatever size it wants.
- EXACTLY

The parent has determined an exact size for the child. The child is going to be given those bounds regardless of how big it wants to be.
- AT_MOST

The child can be as large as it wants up to the specified size.
MeasureSpecs are implemented as ints to reduce object allocation. This class is provided to pack and unpack the <size, mode> tuple into the int.

```

    public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /** @hide */
        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;

        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;
        
        public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }
        
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }
        
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
    }
```

###### MeasureSpec 与 LayoutParams的关系

View在测量的时候,系统会将LayoutParams在父容器的约束下转换成对应的MeasureSpec,然后根据这个MeasureSpec来确定View测量之后的高和宽

需要注意的是：view的MeasureSpec不是由LayoutParams唯一确定的,而是父容器的MeasureSpec和自身的LayoutParams一起来确定的，从而进一步决定View的宽高.而且普通的View和页面的顶级View(DecorView)的MeasureSpec的转换过程略有不同.
- DecorView的MeasureSpec是由窗口的尺寸和DecorView其自身的LayoutParams来共同决定的.
- 普通View的MeasureSpec是由父容器的MeasureSpec和普通View自身的LayoutParams来共同决定的.


1. 当View 是固定宽高的时候,不管父容器是什么模式,View都是精确模式(EXACTLY),并遵循LayoutParams里面的大小.
2. 当View 的宽高是match parent,父容器是精确模式(EXACTLY),View也是精确模式(EXACTLY);父容器是最大模式(AT_MOST),View也是最大模式(AT_MOST).而且这2种情况View 的大小都是父容器剩余大小空间
3. 当View 的宽高是wrap content,不管父容器是精确模式(EXACTLY)还是最大模式(AT_MOST),View始终都是最大模式(AT_MOST).并且大小都是父容器剩余的大小空间

###### mesure
measure过程要分情况来看，如果是一个原始的View，那么通过measure方法就完成了其测量过程，如果是一个ViewGroup，除了完成自己的测量过程外，还会遍历去调用所有子元素的measure方法，各个子元素再递归去执行这个流程。

- View的Mesure

measure方法是一个final类型的方法，这意味着子类不能重写此方法，在View的measure方法中会去调用View的onMeasure方法，所以只需看onMeasure方法即可。

```
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```
setMeasuredDimension方法会设置View的宽/高的测量值。getDefaultSize方法返回的大小就是MeasureSpec中的specSize，而这个specSize就是View测量后的大小，但View的最终大小是在layout阶段确定的，所以这里必须要加以区分，但是几乎所有情况下的View的测量大小和最终大小是相等的。

直接继承View的自定义控件需要重写onMeasure方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content就相当于使用match_parent。为什么呢，如果View在布局中使用wrap_content，那么它的specMode是AT_MOST模式，在这种模式下，它的宽/高等于specSize，也就是说，这种情况下的View的specSize是parentSize，而parentSize是父容器中目前当前剩余使用的大小，也就是父容器当前剩余的空间大小。解决办法比如给wrap_content设置一个默认值，比如都是宽/高都是200px。

- ViewGroup的measure过程

对于ViewGroup来说，除了完成自己的measure过程以外，还会遍历去调用所有子元素的measure方法，各个子元素再去递归执行这个过程。和View不同的是，ViewGroup是一个抽象类，因此它没有重写View的onMeasure方法，但是它提供了一个叫measureChildren的方法。 


```
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```
ViewGroup在measure时，会对每一个子元素进行measure。具体measureChild这个方法的实现也很好理解，如下所示：

```
protected void measureChild(View child, int parentWidthMeasureSpec,int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
很明显，measureChild的思想就是取出子元素的LayoutParams，然后再通过getChildMeasureSpec来创建子元素的MeasureSpec，接着将MeasureSpec直接传递给View的measure方法进行测量。


###### layout
Layout的作用是ViewGroup用来确定子元素的位置，当ViewGroup的位置被确定后，它在onLayout中会遍历所有的子元素并调用其layout方法，在layout方法中onLayout方法又会被调用。


```
 public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);

            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }

            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        final boolean wasLayoutValid = isLayoutValid();

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

        if (!wasLayoutValid && isFocused()) {
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
            if (canTakeFocus()) {
                // We have a robust focus, so parents should no longer be wanting focus.
                clearParentsWantFocus();
            } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
                // This is a weird case. Most-likely the user, rather than ViewRootImpl, called
                // layout. In this case, there's no guarantee that parent layouts will be evaluated
                // and thus the safest action is to clear focus here.
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                clearParentsWantFocus();
            } else if (!hasParentWantsFocus()) {
                // original requestFocus was likely on this view directly, so just clear focus
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            }
            // otherwise, we let parents handle re-assigning focus during their layout passes.
        } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
            View focused = findFocus();
            if (focused != null) {
                // Try to restore focus as close as possible to our starting focus.
                if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
                    // Give up and clear focus once we've reached the top-most parent which wants
                    // focus.
                    focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                }
            }
        }

        if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
            mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
            notifyEnterOrExitForAutoFillIfNeeded(true);
        }
    }

```


layout方法的大致流程：首先会通过setFrame方法来设定View的四个顶点的位置，即初始化mLeft、mRight、mTop和mBottom这四个值，View的四个顶点一旦确定，那么View在父容器中的位置也就确定了，接着就会调用onLayout方法，这个方法用途是父容器确定子元素的位置，和onMeasure方法类似，onLayout的具体实现同样和具体的布局有关，所以View和ViewGroup均没有真正实现onLayout方法。


###### draw
draw过程就比较简单了，它的作用是将View绘制到屏幕上面，View的绘制过程循序以下几步：

1. 绘制背景background.draw(canvas)
2. 绘制自己(onDraw)
3. 绘制children(dispatchDraw)
4. 绘制装饰(onDrawScrollBars)


```
 /**
     * Manually render this view (and all of its children) to the given Canvas.
     * The view must have already done a full layout before this function is
     * called.  When implementing a view, implement
     * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
     * If you do need to override this method, call the superclass version.
     *
     * @param canvas The Canvas to which the View is rendered.
     */
    @CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
        }

        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */

        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;

        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;

        // Step 2, save the canvas' layers
        int paddingLeft = mPaddingLeft;

        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }

        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);

        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }

        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;

        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }

        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }

        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }

        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }

        saveCount = canvas.getSaveCount();

        int solidColor = getSolidColor();
        if (solidColor == 0) {
            if (drawTop) {
                canvas.saveUnclippedLayer(left, top, right, top + length);
            }

            if (drawBottom) {
                canvas.saveUnclippedLayer(left, bottom - length, right, bottom);
            }

            if (drawLeft) {
                canvas.saveUnclippedLayer(left, top, left + length, bottom);
            }

            if (drawRight) {
                canvas.saveUnclippedLayer(right - length, top, right, bottom);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }

        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }

        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, left + length, bottom, p);
        }

        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(right - length, top, right, bottom, p);
        }

        canvas.restoreToCount(saveCount);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }
    }

```



### View 事件体系

##### 位置参数
坐标系：X,Y轴正方向分别是向右和向下。

View的位置主要由他的四个顶点来决定，分别对应于View的四个属性：top,left,right,bottom。top与left对应于左上角顶点的纵坐标和横坐标，right和bottom对应于右下角的横坐标和纵坐标，这些坐标都是相对于View的父容器来说的，因此是相对坐标。

从Android3.0开始，View增加了额外的几个参数：x,y,translationX,translationY。其中x和y是左上角的坐标，translationX,translationY是View左上角相对于父容器的偏移量。这几个参数也是相对于父容器的坐标，并且translationX,translationY的默认值是0。

这些参数都有对应的get/set方法。换算关系如下：

```
x = left + translationX
y = top + translationY
```
需要注意的是View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会变化，此时改变的是x,y,translationX,translationY这四个参数。

##### MotionEvent和TouchSlop

通过MotionEvent对象我们可以得到点击事件发生的x和y坐标，
getX/getY:相对于当前View左上角的x,y坐标
getRawX/getRawY:相对于当前手机屏幕左上角的x和y坐标。

TouchSlop:是系统能识别出来的滑动最小距离，两次滑动之间如果距离小于这个值系统就不认为是滑动。这是个常量和设备有关，可以如下获得这个值：ViewConfiguration.get(getContext()).getScaledTouchSlop()。
在源码中定义在frameworks/base/core/res/values/config.xml文件中

```
<!-- Base "touch slop"  value used by ViewConfiguration as a movement threshold where scrolling should begin .-->
<dimen name ="config_viewConfigurationTouchSlop">8dp</dimen>
```

##### VelocityTracker
X/Y方向的速度相关的帮助类

公式： 

```
速度 = （终点速度 - 起点速度）/ 时间段
```
当手指逆着坐标系正方向滑动，速度就是负值

使用方法：

```
VelocityTracker mVelocityTracker = VelocityTracker.obtain();
//设置追踪的事件
mVelocityTracker.addMovement(event);

//计算速度，参数是时间间隔（ms）,速度就是在这个时间间隔里手指在水平或竖直方向上划过的像素数
mVelocityTracker.computeCurrentVelocity(1000);
//然后获得当前的滑动速度
float xVelocity = mVelocityTracker.getXVelocity();
float yVelocity = mVelocityTracker.getYVelocity();

//不需要使用的时候进行回收内存
mVelocityTracker.clear();
mVelocityTracker.recycle();

```

##### Scroller
用于实现View的弹性滑动，可以代替View的scrollTo/scrollBy实现有过度效果的滑动。Scroller本身无法让View弹性滑动，它需要和View的computeScroll方法配合使用才能完成弹性滑动。

startScroll并不能让View滑动，仅仅是保存了我们传入的参数，invalidate()才是让View滑动的关键，invalidate()会导致View的重绘，在View的draw方法中又会去调用computeScroll方法，computeScroll在View中是个空实现，需要我们自己去实现，computeScroll会去向Scroller获取当前的scrollX和scrollY,然后通过scrollTo方法进行滑动，接着又调用postInvalidate方法导致第二次重绘，和第一次重绘一样还是会导致computeScroll被调用，如此反复，直到滑动结束。

典型代码如下：

```
    Scroller scroller = new Scroller(context);

    private void smoothScrollTo(int y) {
        int scrollY = getScrollY();
        int dy = y - scrollY;
        scroller.startScroll(0, scrollY, 0, dy, 500);
        invalidate();
    }

    @Override
    public void computeScroll() {
        super.computeScroll();
        if (scroller.computeScrollOffset()) {
            scrollTo(scroller.getCurrX(), scroller.getCurrY());
            postInvalidate();
        }
    }
```
computeScrollOffset会根据时间流逝来计算出当前的scrollX和scrollY的值
```
    /**
     * Call this when you want to know the new location.  If it returns true,
     * the animation is not yet finished.
     */ 
    public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    
        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE:
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                final float t = (float) timePassed / mDuration;
                final int index = (int) (NB_SAMPLES * t);
                float distanceCoef = 1.f;
                float velocityCoef = 0.f;
                if (index < NB_SAMPLES) {
                    final float t_inf = (float) index / NB_SAMPLES;
                    final float t_sup = (float) (index + 1) / NB_SAMPLES;
                    final float d_inf = SPLINE_POSITION[index];
                    final float d_sup = SPLINE_POSITION[index + 1];
                    velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                    distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                }

                mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
                
                mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
                // Pin to mMinX <= mCurrX <= mMaxX
                mCurrX = Math.min(mCurrX, mMaxX);
                mCurrX = Math.max(mCurrX, mMinX);
                
                mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
                // Pin to mMinY <= mCurrY <= mMaxY
                mCurrY = Math.min(mCurrY, mMaxY);
                mCurrY = Math.max(mCurrY, mMinY);

                if (mCurrX == mFinalX && mCurrY == mFinalY) {
                    mFinished = true;
                }

                break;
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
```


##### View的滑动
1. 使用scrollTo/scrollBy

scrollTo实现了基于所传参数的绝对滑动，scrollBy实际上也是调用了scrollTo方法实现了基于当前位置的相对滑动。
scrollTo和scrollBy只能改变View内容的位置而不能改变View在布局中的位置。不影响内部元素的点击事件。

通过getScrollX和getScrollY可以得到View内部的两个属性mScrollX和mScrollY.
其单位为像素

- mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，当View的左边缘在View内容的右边时，为正值，反之为负值
- mScrollY的值总是等于View上边缘和View内容上边缘在竖直方向的距离，当View的上边缘在View的内容的上边缘下边时为正值，反之为负值

2. 使用动画给View施加平移效果

使用动画来移动View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画，如果采用属性动画的话为了兼容3.0以下的版本，需要采用开源动画库nineoldandroids。

```
ObjectAnimator.ofFloat(myObject, "translationY", -myObject.getHeight()).start();
```
使用View动画并不能改变View的位置，他的位置信息（四个顶点和宽高）并不会随着动画而改变，在系统眼里他还是在原来的位置。点击事件还是只能在原始位置触发。
从Android3.0开始使用属性动画可以解决上面的问题。

3. 改变View的LayoutParams使View重新布局实现滑动

###### 实现View的弹性滑动
1. Scroller
2. 动画
3. 使用延时策略（handler,View的postDelayed,线程sleep）

#### 事件分发机制
伪代码：

```
public boolean dispatchTouchEvent(MotionEvent ev){
    boolean consume = false;
    if(onInterceptTouchEvent(ev)){
        consume = onTouchEvent(ev);
    }else{
        consume = child.dispatchTouchEvent(ev);
    }
    
    return consume;
}
```
给View设置的OnTouchListener优先级要比onTouchEvent高，如果onTouch的返回值返回false,则当前View的onTouchEvent方法会被调用；如果返回true,那么onTouchEvent方法将不会被调用。

我们平时常用的OnclickListener，其优先级最低，即处于事件传递的尾端。

当一个事件产生后，它的传递过程遵循如下顺序：Activity->Window->View.

如果一个View的onTouchEvent返回false,那么它的父容器的onTouchEvent将会被调用，以此类推。如果所有元素都不处理这个事件，那么这个事件将会最终传递给Activity处理，即Activity的onTouchEvent方法会被调用。


### 动画
##### 补间动画（View动画）

##### 帧动画
容易引起oom

##### 属性动画
API11（3.0）引入，可使用nineoldandroids,兼容以前的版本

要使属性动画生效的两个条件：

- 属性动画要求对象的该属性有set方法和get方法（可选，如果没有传递初始值，系统就需要通过getXXX方法去获取XXX属性的初始值）（不满足会导致crash）
- object的setXXX方法对属性的改变必须能通过某种方法反映出来，比如带来UI的改变之类。(不满足导致无效果，但不会crash)

查看PropertyValuesHolder源码可知get/set方法都是通过反射进行调用的。


自定义插值器需要实现Interpolator或者TimeInterpolator,自定义估值算法需要实现TypeEvaluator.如果要对其他类型（非int,float,Color）做动画，必须自定义类型估值算法。

注意点：

1. 如果属性动画是无限循环的动画，需要在activity退出时及时停止，否则会导致activity无法释放而内存泄漏，View动画不存在此问题。
2. 动画在3.0以下系统做好适配工作
3. View动画是对View的影像做动画，并不是正真改变View的状态。因此有时候会出现动画完成setVisibility(View.GONE)失效的问题，可以调用view.clearAnimation清除View动画即可。
4. 将View移动后，在3.0以前的系统上，不管是View动画还是属性动画新位置都无法触发单击事件，老位置依然可以触发单击事件，尽管View在视觉上已经不存在了。从3.0开始属性动画单击事件触发位置在移动后的位置，但是View动画依然在原位置。
5. 尽量使用dp，统一不同设备的显示效果
6. 帧动画注意OOM问题
7. 建议开启硬件加速，提高流畅性


# Style

[样式和主题](https://developer.android.google.cn/guide/topics/ui/themes)

**样式**  是指为 View 或窗口指定外观和格式的属性集合。样式可以指定高度、填充、字体颜色、字号、背景色等许多属性。 样式是在与指定布局的 XML 不同的 XML 资源中进行定义。

Android 中的样式与网页设计中层叠样式表的原理类似 — 您可以通过它将设计与内容分离。

**主题**  是指对整个 Activity 或应用而不是对单个 View（如上例所示）应用的样式。 以主题形式应用样式时，Activity 或应用中的每个视图都将应用其支持的每个样式属性。 例如，您可以 Activity 主题形式应用同一 CodeFont 样式，之后该 Activity 内的所有文本都将具有绿色固定宽度字体。

## 继承
1. 可以通过 <style> 元素中的 parent 属性指定应作为您的样式所继承属性来源的样式。您可以利用它来继承现有样式的属性，然后只定义您想要更改或添加的属性。 您可以从自行创建的样式或平台内建的样式继承属性。 

2. 如果您想从自行定义的样式继承属性，则不必使用 parent 属性， 而是只需将您想继承的样式的名称以前缀形式添加到新样式的名称之中，并以句点进行分隔。可以通过使用句点链接名称继续进行这样的继承，次数不限。


这种通过将名称链接起来的继承方法只适用于由您自己的资源定义的样式。 您无法通过这种方法继承 Android 内建样式。 要引用内建样式（例如 TextAppearance），您必须使用 parent 属性。


## 根据平台版本选择主题
例如，以下这个声明所对应的自定义主题就是标准的平台默认明亮主题。 它位于 res/values 之下的一个 XML 文件（通常是 res/values/styles.xml）中：


```
<style name="LightThemeSelector" parent="android:Theme.Light">
    ...
</style>
```

为了让该主题在应用运行在 Android 3.0（API 级别 11）或更高版本系统上时使用更新的全息主题，您可以在 res/values-v11 下的 XML 文件中加入一个替代主题声明，但将父主题设置为全息主题：


```
<style name="LightThemeSelector" parent="android:Theme.Holo.Light">
    ...
</style>
```

现在像您使用任何其他主题那样使用该主题，您的应用将在其运行于 Android 3.0 或更高版本的系统上时自动切换到全息主题。

