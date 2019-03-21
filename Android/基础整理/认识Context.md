#### Context
官方释义：
Interface to global information about an application environment. This is an abstract class whose implementation is provided by the Android system. It allows access to application-specific resources and classes, as well as up-calls for application-level operations such as launching activities, broadcasting and receiving intents, etc.

Context是一个抽象类，里面定义了大量常用的抽象方法，这些方法不但与我们上面提到的四大组件密切相关，还与资源文件，文件管理，包管理，类加载，权限管理，系统级服务获取等功能相关。

#### ContextImpl
ContextImpl继承了Context，并实现了Context的抽象方法。

```
/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {

......

 static ContextImpl getImpl(Context context) {
        Context nextContext;
        while ((context instanceof ContextWrapper) &&
                (nextContext=((ContextWrapper)context).getBaseContext()) != null) {
            context = nextContext;
        }
        return (ContextImpl)context;
    }

    @Override
    public AssetManager getAssets() {
        return getResources().getAssets();
    }

    @Override
    public Resources getResources() {
        return mResources;
    }
    
    ......


}
```

#### ContextWrapper 
ContextWrapper继承Context,是一个装饰类，在ContextWrapper里面有一个Context的引用mBase,ContextWrapper的所有方法都仅仅是调用了ContextImpl中的对应方法：

```
/**
 * Proxying implementation of Context that simply delegates all of its calls to
 * another Context.  Can be subclassed to modify behavior without changing
 * the original Context.
 */
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
     /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    
    /**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }

    @Override
    public AssetManager getAssets() {
        return mBase.getAssets();
    }

    @Override
    public Resources getResources() {
        return mBase.getResources();
    }
    
    ......
}
```

### Context是何时被创建的

##### Activity
ActivityThread类中H类收到 "LAUNCH_ACTIVITY" 消息后，调用handleLaunchActivity来处理Activity启动，handleLaunchActivity中又会调用performLaunchActivity来获取一个Activity的实例：


```
  /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ......
     ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }
     try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            if (activity != null) {
            ......
            appContext.setOuterContext(activity);
            activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
            ......
            
             int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            }catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
           ...
        }
     return activity;
    }
```

通过调用createBaseContextForActivity去构造Context对象：

```
private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
......
 ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
......
 return appContext;
}
```
在createBaseContextForActivity方法中通过ContextImpl的静态方法createActivityContext获得一个ContextImpl的实例对象。

在上面performLaunchActivity方法中通过setOuterContext将Context与Activity建立关联。

##### Application
与Activity大致流程相同，ActivityThread中的H类收到 "BIND_APPLICATION"消息后调用handleBindApplication方法，然后调用LoadedApk的makeApplication方法创建一个Application实例：

```
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
             try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
        .....
        }
        ......
        return app;
}
```

###### getApplicationContext
ContextWrapper:
```
public class ContextWrapper extends Context {
    Context mBase;
    ......
     @Override
    public Context getApplicationContext() {
        return mBase.getApplicationContext();
    }
    ......
    }
```

ContextImpl:
```
class ContextImpl extends Context {
......
final @NonNull ActivityThread mMainThread;
final @NonNull LoadedApk mPackageInfo;
......
@Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
    ......
}
```

LoadedApk的getApplication：

```
public final class LoadedApk {
    ......
    private Application mApplication;
    ......
    Application getApplication() {
        return mApplication;
    }
    
    ......
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
            ......
            mApplication = app;
            ......
    }
}
```
ActivityThread的getApplication：

```
public final class ActivityThread extends ClientTransactionHandler {
    ......
    Application mInitialApplication;
    ......
    public Application getApplication() {
        return mInitialApplication;
    }
    ......
}
```


##### 一个应用中有几个Context对象
一个应用中Context对象的总数应该等于Activity对象与Service对象之和再加上一个Application。

BroadcastReciver与ContentProvider的Context都直接或间接来自于Application,Activity和Service。

当你无法确定使用某个Context是否会造成长引用导致内存泄漏时，就使用Application的Context对象，因为Application存在于整个应用的生命周期内。


