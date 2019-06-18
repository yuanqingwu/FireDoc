https://www.jianshu.com/p/f8526b791ebc

https://juejin.im/post/5b1fbd796fb9a01e8c5fd847



## 观察者模式中四个重要的角色：
- 抽象主题：定义添加和删除观察者的功能，也就是注册和解除注册的功能
- 抽象观察者：定义观察者收到主题通知后要做什么事情
- 具体主题：抽象主题的实现
- 具体观察者：抽象观察者的实现


```
public class TestObservePattern {

    public static void main(String[] args) {
        // 创建主题(被观察者)
        ConcreteSubject concreteSubject = new ConcreteSubject();
        // 创建观察者
        ObserverOne observerOne=new ObserverOne();
        // 为主题添加观察者
        concreteSubject.addObserver(observerOne);        
        //主题通知所有的观察者
        concreteSubject.notifyAllObserver("wake up,wake up");
    }

}
```


#### 观察者实例化
- 调用create操作符创建Observable
```
Observable observable = Observable.create(new ObservableOnSubscribe<Object>() {
            @Override
            public void subscribe(ObservableEmitter<Object> emitter) throws Exception {

            }
        });
        
//支持背压
         Flowable flowable = Flowable.create(new FlowableOnSubscribe<Object>() {
          @Override
          public void subscribe(FlowableEmitter<Object> emitter) throws Exception {
              
          }
      },BackpressureStrategy.DROP);
```

- 调用new ObservableCreate<T>(source)
```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```


```
    @NonNull
    public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```

- 通过ObservableCreate创建，构造方法需要ObservableOnSubscribe对象
```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        ......
    }

 static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
    ......
    }
    
    
    static final class SerializedEmitter<T>
    extends AtomicInteger
    implements ObservableEmitter<T> {
    ......
    }


}
```

- ObservableOnSubscribe

```
public interface ObservableOnSubscribe<T> {

    /**
     * Called for each Observer that subscribes.
     * @param emitter the safe emitter instance, never null
     * @throws Exception on error
     */
    void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception;
}
```

- ObservableEmitter继承于Emitter接口添加了Disposable相关方法与tryOnError接口

```
public interface ObservableEmitter<T> extends Emitter<T> {

    void setDisposable(@Nullable Disposable d);

    void setCancellable(@Nullable Cancellable c);

    boolean isDisposed();

    @NonNull
    ObservableEmitter<T> serialize();

    /**
     * Attempts to emit the specified {@code Throwable} error if the downstream
     * hasn't cancelled the sequence or is otherwise terminated, returning false
     * if the emission is not allowed to happen due to lifecycle restrictions.
     * <p>
     * Unlike {@link #onError(Throwable)}, the {@code RxJavaPlugins.onError} is not called
     * if the error could not be delivered.
     * @param t the throwable error to signal if possible
     * @return true if successful, false if the downstream is not able to accept further
     * events
     * @since 2.1.1 - experimental
     */
    @Experimental
    boolean tryOnError(@NonNull Throwable t);
}
```
- Emitter接口中定义了onNext，onError，onComplete方法

```
public interface Emitter<T> {

    /**
     * Signal a normal value.
     * @param value the value to signal, not null
     */
    void onNext(@NonNull T value);

    /**
     * Signal a Throwable exception.
     * @param error the Throwable to signal, not null
     */
    void onError(@NonNull Throwable error);

    /**
     * Signal a completion.
     */
    void onComplete();
}
```



#### 订阅

```
mObservable.subscribe(mObserver);
```
注意这里是是观察者（下游）订阅了主题（上游），虽然从代码上看好像了前者订阅了后者，不要搞混了。

- subscribe内部调用subscribeActual方法
```
    @SchedulerSupport(SchedulerSupport.NONE)
    @Override
    public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);

            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            // can't call onError because no way to know if a Disposable has been set or not
            // can't call onSubscribe because the call might have set a Subscription already
            RxJavaPlugins.onError(e);

            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }
```

- Observable的实例是一个ObservableCreate对象
```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        //关键点：onSubscribe并不是由主题（上游）主动发送的事件，而是有观察者（下游）自己调用的一个事件，只是为了方便获取Emitter的实例对象，准确的说应该是Disposable的实例对象，这样下游就可以控制上游了。
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

 static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
    ......
    }
    
    
    static final class SerializedEmitter<T>
    extends AtomicInteger
    implements ObservableEmitter<T> {
    ......
    }


}
```


#### map操作符

- 创建ObservableMap对象
```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```

- ObservableMap继承AbstractObservableWithUpstream

```
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
       
    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
          ......

            U v;

            try {
            //将T类型的原始数据t变换为类型U的新数据v
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            actual.onNext(v);
        }

       ......
    }
       
}
```

我们在订阅的时候map操作符将原始的observer包装成了一个 MapObserver 对象，在这个 MapObserver 内部的 onNext 方法中首先会将原始数据 1 通过我们传入的变换规则变换为 result : 1，在变换之后才会调用我们编写的原始observer来处理新的 String 类型的数据。


#### 线程切换

##### subscribeOn(Schedulers.io())

返回一个ObservableSubscribeOn对象
```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.CUSTOM)
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
    }
```

- ObservableSubscribeOn中scheduler.scheduleDirect传入一个SubscribeTask 对象
```
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> s) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

        s.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

......

    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }

}
```

- Scheduler的scheduleDirect中创建一个worker执行任务

```
public abstract class Scheduler {

    ......

    @NonNull
    public Disposable scheduleDirect(@NonNull Runnable run) {
        return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
    }
    
    @NonNull
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        //createWorker是一个抽象方法，不同的线程调度器有不同的实现
        final Worker w = createWorker();

        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        DisposeTask task = new DisposeTask(decoratedRun, w);

        //线程调度
        w.schedule(task, delay, unit);

        return task;
    }
    
    ......
    
}
```

- IoSchedule 这个调度器来说，它最终是执行这个方法，将任务提交到线程池中
```
public class NewThreadWorker extends Scheduler.Worker implements Disposable {

    @NonNull
    public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

        if (parent != null) {
            if (!parent.add(sr)) {
                return sr;
            }
        }

        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            if (parent != null) {
                parent.remove(sr);
            }
            RxJavaPlugins.onError(ex);
        }

        return sr;
    }
    
}
```


subscribeOn(Schedulers.io()) 本质上是通过线程池开启了一个线程，在这个新的线程中还是通过调用 source.subscribe(observer) 来订阅事件。


##### subscribeOn(AndroidSchedulers.mainThread())


```
public final class AndroidSchedulers {

    private static final class MainHolder {

        static final Scheduler DEFAULT = new HandlerScheduler(new Handler(Looper.getMainLooper()));
    }

    private static final Scheduler MAIN_THREAD = RxAndroidPlugins.initMainThreadScheduler(
            new Callable<Scheduler>() {
                @Override public Scheduler call() throws Exception {
                    return MainHolder.DEFAULT;
                }
            });

    /** A {@link Scheduler} which executes actions on the Android main thread. */
    public static Scheduler mainThread() {
        return RxAndroidPlugins.onMainThreadScheduler(MAIN_THREAD);
    }

    /** A {@link Scheduler} which executes actions on {@code looper}. */
    public static Scheduler from(Looper looper) {
        if (looper == null) throw new NullPointerException("looper == null");
        return new HandlerScheduler(new Handler(looper));
    }

    private AndroidSchedulers() {
        throw new AssertionError("No instances.");
    }
}
```


```
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            Scheduler.Worker w = scheduler.createWorker();

            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }

    static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

        ......

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }

        ......
        
        void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }
    }
    
}
```




可以看到 subscribeOn 与 observeOn 不同的是二者线程切换的地方。 subscribeOn 是将 source.subscribe(observer) 整个部分切换到了相应的线程，而 observeOn 则是在 ObserveOnObserver 这个包装类中的 onNext 、 onError 方法中做了线程的切换，具体是在 onNext 等方法中创建了相关的 Worker 工作类，并调用了这个类的 schedule 方法，这个跟我们讲 subscribeOn 是一样的。

