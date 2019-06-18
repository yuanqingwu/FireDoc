## Synchronized原理 
https://blog.csdn.net/javazejian/article/details/72828483
## Volatile实现原理 
[深入分析Volatile的实现原理](http://ifeve.com/volatile/)

[内存屏障和volatile语义](https://blog.csdn.net/kuangzhanshatian/article/details/81738599)

Volatile是轻量级的synchronized，它在多处理器开发中保证了共享变量的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。它在某些情况下比synchronized的开销更小

Java语言规范第三版中对volatile的定义如下：java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致的更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了volatile，在某些情况下比锁更加方便。如果一个字段被声明成volatile，java线程内存模型确保所有线程看到这个变量的值是一致的


**对volatile变量运算操作在多线程环境并不保证安全性**
比如对于i++方法需要用synchronize修饰


## 线程池核心线程数一般定义多少，为什么
IO密集型=2Ncpu（常出现于线程中：数据库数据交互、文件上传下载、网络数据传输等等）

计算密集型=Ncpu（常出现于线程中：复杂算法）

java中：Ncpu=Runtime.getRuntime().availableProcessors()


一般使用线程池，按照如下顺序依次考虑（只有前者不满足场景需求，才考虑后者）：

newCachedThreadPool-->newFixedThreadPool(int threadSize)-->ThreadPoolExecutor

newCachedThreadPool不需要指定任何参数
newFixedThreadPool需要指定线程池数（核心线程数==最大线程数）
ThreadPoolExecutor需要指定核心线程数、最大线程数、闲置超时时间、队列、队列容量，甚至还有回绝策略和线程工厂

### wait() 和 sleep() 的区别
##### wait
wait是Object类的方法，也就是说，所有的对象都有wait方法，而且都是Object中的wait方法因为wait方法被标为final无法被重写，源码如下：

```
 public final void wait(long millis) throws InterruptedException {
        wait(millis, 0);
    }

 public final native void wait(long millis, int nanos) throws InterruptedException;
```
已知wait有释放同步锁的效果，即当处于synchronized方法中的线程进入wait后，synchronized锁将会自动被释放，其他线程可以申请锁、持有锁，然后进入这个synchronized方法进行运行。

##### sleep

```
   /**
     * Causes the currently executing thread to sleep (temporarily cease
     * execution) for the specified number of milliseconds, subject to
     * the precision and accuracy of system timers and schedulers. The thread
     * does not lose ownership of any monitors.
     *
     * @param  millis
     *         the length of time to sleep in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
 public static void sleep(long millis) throws InterruptedException {
        Thread.sleep(millis, 0);
    }
    
 private static native void sleep(Object lock, long millis, int nanos)
        throws InterruptedException;

```

sleep是Thread类的方法，导致此线程暂停执行指定时间，给其他线程执行机会，但是依然保持着监控状态，过了指定时间会自动恢复，调用sleep方法不会释放锁对象。

当调用sleep方法后，当前线程进入阻塞状态。目的是让出CPU给其他线程运行的机会。但是由于sleep方法不会释放锁对象，所以在一个同步代码块中调用这个方法后，线程虽然休眠了，但其他线程无法访问它的锁对象。这是因为sleep方法拥有CPU的执行权，它可以自动醒来无需唤醒。而当sleep()结束指定休眠时间后，这个线程不一定立即执行，因为此时其他线程可能正在运行。

###### 是否可以传入参数
sleep()方法必须传入参数，参数就是休眠时间，时间到了就会自动醒来。

wait()方法可以传入参数也可以不传入参数，传入参数就是在参数结束的时间后开始等待，不穿如参数就是直接等待。

###### 是否需要捕获异常
sleep方法必须要捕获异常，而wait方法不需要捕获异常。

###### 作用范围
wait、notify和notifyAll方法只能在同步方法或者同步代码块中使用，而sleep方法可以在任何地方使用。但是注意sleep是静态方法，也就是说它只对当前对象有效。通过对象名.sleep()想让该对象线程进入休眠是无效的，它只会让当前线程进入休眠。

###### 调用者的区别
为什么wait、notify和notifyAll方法要和synchronized关键字一起使用？

因为wait方法是使一个线程进入等待状态，并且释放其所持有的锁对象，notify方法是通知等待该锁对象的线程重新获得锁对象，然而如果没有获得锁对象，wait方法和notify方法都是没有意义的，因此必须先获得锁对象再对锁对象进行进一步操作于是才要把wait方法和notify方法写到同步方法和同步代码块中了。

wait和notify、notifyAll方法是由确定的对象即锁对象来调用的，锁对象就像一个传话的人，他对某个线程说停下来等待，然后对另一个线程说你可以执行了（实质上是被捕获了），这一过程是线程通信。sleep方法是让某个线程暂停运行一段时间，其控制范围是由当前线程决定，运行的主动权是由当前线程来控制（拥有CPU的执行权）。


### Runnable，Callable、Future和FutureTask
##### Runnable
Runnable，它是一个接口，在它里面只声明了一个run()方法,由于run()方法返回值为void类型，所以在执行完任务之后无法返回任何结果。
```
public interface Runnable {
    public abstract void run();
}
```
##### Callable
Callable位于java.util.concurrent包下，它也是一个接口，在它里面也只声明了一个方法，只不过这个方法叫做call()。这是一个泛型接口，该接口声明了一个名称为call()的方法，同时这个方法可以有返回值V，也可以抛出异常。call()方法返回的类型就是传递进来的V类型。

```
@FunctionalInterface
public interface Callable<V> {
   V call() throws Exception;
}
```
一般情况下是配合ExecutorService来使用的，在ExecutorService接口中声明了若干个submit方法的重载版本：

```
<T> Future<T> submit(Callable<T> task);
```

##### Future
Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。

```
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException, 
            ExecutionException, TimeoutException;
}
```

##### FutureTask

FutureTask类实现了RunnableFuture接口，RunnableFuture继承了Runnable接口和Future接口，而FutureTask实现了RunnableFuture接口，所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。
```
public class FutureTask<V> implements RunnableFuture<V> {
```

```
public interface RunnableFuture<V> extends Runnable, Future<V> {
   void run();
}
```

##### 实现Runnable接口和实现Callable接口的区别：

1、Runnable是自从java1.1就有了，而Callable是1.5之后才加上去的。

2、Callable规定的方法是call(),Runnable规定的方法是run()。

3、Callable的任务执行后可返回值，而Runnable的任务是不能返回值(是void)。

4、call方法可以抛出异常，run方法不可以。

5、运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。

6、加入线程池运行，Runnable使用ExecutorService的execute方法，Callable使用submit方法。


### 读写锁
ReentrantReadWriteLock
读写锁是一种特殊的自旋锁，它把对共享资源对访问者划分成了读者和写者，读者只对共享资源进行访问，写者则是对共享资源进行写操作。读写锁在ReentrantLock上进行了拓展使得该锁更适合读操作远远大于写操作对场景。一个读写锁同时只能存在一个写锁但是可以存在多个读锁，但不能同时存在写锁和读锁。

　　如果读写锁当前没有读者，也没有写者，那么写者可以立刻获的读写锁，否则必须自旋，直到没有任何的写锁或者读锁存在。如果读写锁没有写锁，那么读锁可以立马获取，否则必须等待写锁释放。(但是有一个例外，就是读写锁中的锁降级操作，当同一个线程获取写锁后，在写锁没有释放的情况下可以获取读锁再释放读锁这就是锁降级的一个过程)
　　
　　


