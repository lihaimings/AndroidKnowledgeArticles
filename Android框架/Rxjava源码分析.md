  从最简单的使用开始分析起：
  ```
 Observable.just("Url")
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        
                    }

                    @Override
                    public void onNext(String value) {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onComplete() {

                    }
                });
  ```
先看一下**just("Url")**f方法的去向：
  ```
#Observable:
 @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Observable<T> just(T item) {
        ObjectHelper.requireNonNull(item, "The item is null");
        return RxJavaPlugins.onAssembly(new ObservableJust<T>(item));
    }
  ```
看一下**RxJavaPlugins.onAssembly()**
  ```
 public static <T> Observable<T> onAssembly(Observable<T> source) {
        Function<Observable, Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
  ```
从return可以看出，它返回就是我们传给它的参数值，**所以just()方法返回的就是new ObservableJust<T>(item),ObservableJust类**，看看这个方法：
  ```
//可以看到这个类继承了Observable
public final class ObservableJust<T> extends Observable<T> implements ScalarCallable<T> {

    private final T value;
    public ObservableJust(final T value) {
        this.value = value;
    }

    @Override
    protected void subscribeActual(Observer<? super T> s) {
        ScalarDisposable<T> sd = new ScalarDisposable<T>(s, value);
        s.onSubscribe(sd);
        sd.run();
    }

    @Override
    public T call() {
        return value;
    }
}
  ```
**just()的时候就是通过一个泛型，把参数值传递进来保存ObservableJust类中。**

**第二，订阅方法subscribe()源码**
  ```
 @Override
    public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);

            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
            //重点方法
            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e)；
            RxJavaPlugins.onError(e);
            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(e);
            throw npe;
        }
    }

 protected abstract void subscribeActual(Observer<? super T> observer);

  ```
可以看到，**把参数Observer传给了subscribeActual(observer),但是这个方法是Observable的抽象类**，所以我们在其子类Observable中查看此方法实现：
  ```
public final class ObservableJust<T> extends Observable<T> implements ScalarCallable<T> {

    private final T value;
    public ObservableJust(final T value) {
        this.value = value;
    }

    @Override
    protected void subscribeActual(Observer<? super T> s) {
        ScalarDisposable<T> sd = new ScalarDisposable<T>(s, value);
       //首先调用Observer.onSubscribe方法，并把sd传进去
        s.onSubscribe(sd);
       //随后，调用sd的run方法
        sd.run();
    }

    @Override
    public T call() {
        return value;
    }
}
  ```
从代码中得知**subscribeActual()方法，先把Observer的实例和参数value创建ScalarDisposable对象，然后调用Observer的onSubscribe方法，然后执行sd.run(）方法**

最后我们查看一下ScalarDisposable的run方法整个流程就走完了：
  ```
 public static final class ScalarDisposable<T>
    extends AtomicInteger
    implements QueueDisposable<T>, Runnable {
        ....
        final Observer<? super T> observer;

        final T value;
        
        public ScalarDisposable(Observer<? super T> observer, T value) {
            this.observer = observer;
            this.value = value;
        }

      .....
       
        @Override
        public void run() {
            if (get() == START && compareAndSet(START, ON_NEXT)) {
               //调用observer接口的onNext()方法，并把我们传的参数写进去。
               observer.onNext(value);
                if (get() == ON_NEXT) {
                    lazySet(ON_COMPLETE);
                    observer.onComplete();//调用observer接口的onComplete方法
                }
            }
        }
    }
  ```
所以，在Obdervable类中subscribe()方法中没有直接调用Observer的方法，而是通过两个类中去调用。

**(二)有时我们会使用new Consumer这种简单的写法**
我们来分析一下:
  ```
首先Consumer是一个接口，里面有一个accept方法。
public interface Consumer<T> {
    void accept(T t) throws Exception;
}

接着我们从subscribe(Consumer)中点进去看一源码：

public final Disposable subscribe(Consumer<? super T> onNext) {
        return subscribe(onNext, Functions.ERROR_CONSUMER, Functions.EMPTY_ACTION, Functions.emptyConsumer());
    }
我们发现它把我们实例化的Consumer作为参数，传到另一个subscribe中。
其中还有Funtions类中静态量，点进去发现两个是Consumer接口实例类，还有一个是Action。
 public static final Consumer<Throwable> ERROR_CONSUMER = new Consumer<Throwable>() {
        @Override
        public void accept(Throwable error) {
            RxJavaPlugins.onError(error);
        }
    };
 public static final Action EMPTY_ACTION = new Action() {
        @Override
        public void run() { }

        @Override
        public String toString() {
            return "EmptyAction";
        }
    };
  static final Consumer<Object> EMPTY_CONSUMER = new Consumer<Object>() {
        @Override
        public void accept(Object v) { }

        @Override
        public String toString() {
            return "EmptyConsumer";
        }
    };

   
我们看看return的subscribe方法：

 public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete, Consumer<? super Disposable> onSubscribe) {
        ObjectHelper.requireNonNull(onNext, "onNext is null");
        ObjectHelper.requireNonNull(onError, "onError is null");
        ObjectHelper.requireNonNull(onComplete, "onComplete is null");
        ObjectHelper.requireNonNull(onSubscribe, "onSubscribe is null");
        //我们发现这个类实现了Observer接口，并传进去我们的Consumer实现类。
        LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);
       //这里才是真正订阅我们的Obsever
        subscribe(ls);

        return ls;
    }
  ```
从上面的分析我们知道，**LambadaObserver类中的onNext等四个方法会接收我们传递的参数，然后我们这个参数通过Consumer实例传给方法。**
  ```
public final class LambdaObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

    private static final long serialVersionUID = -7251123623727029452L;
    final Consumer<? super T> onNext;
    final Consumer<? super Throwable> onError;
    final Action onComplete;
    final Consumer<? super Disposable> onSubscribe;

    public LambdaObserver(Consumer<? super T> onNext, Consumer<? super Throwable> onError,
            Action onComplete,
            Consumer<? super Disposable> onSubscribe) {
        super();
        this.onNext = onNext;
        this.onError = onError;
        this.onComplete = onComplete;
        this.onSubscribe = onSubscribe;
    }

    @Override
    public void onSubscribe(Disposable s) {
        if (DisposableHelper.setOnce(this, s)) {
            try {
                onSubscribe.accept(this);
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                s.dispose();
                RxJavaPlugins.onError(ex);
            }
        }
    }

    @Override
    public void onNext(T t) {
        try {
            onNext.accept(t);
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            onError(e);
        }
    }

    @Override
    public void onError(Throwable t) {
        dispose();
        try {
            onError.accept(t);
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            RxJavaPlugins.onError(e);
            RxJavaPlugins.onError(t);
        }
    }

    @Override
    public void onComplete() {
        dispose();
        try {
            onComplete.run();
        } catch (Throwable e) {
            Exceptions.throwIfFatal(e);
            RxJavaPlugins.onError(e);
        }
    }

    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return get() == DisposableHelper.DISPOSED;
    }
}
  ```
##变换操作符map：
这是经过两次变换的map代码
  ```
.map(new Function<String, Bitmap>() {
                    @Override
                    public Bitmap apply(String s) throws Exception {
                        URL url= new URL(s);
                        HttpURLConnection urlConnection = (HttpURLConnection) url.openConnection();
                        InputStream inputStream= urlConnection.getInputStream();
                        Bitmap bitmap= BitmapFactory.decodeStream(inputStream);
                        return bitmap;
                    }
                })
                .map(new Function<Bitmap, Bitmap>() {
                    @Override
                    public Bitmap apply(Bitmap bitmap) throws Exception {
                        return createWatermark(bitmap,"Rxjava");
                    }
                })
  ```
事件流，只关心上游和下游。
定义了两个泛型的接口：
  ```
public interface Function<T, R> {
   
    R apply(T t) throws Exception;
}
  ```
  ```
    泛型方法前面要<R>,并传入一个Function实例，返回一个ObservableMap
    public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
       // onAeesmbly()方法前面分析过返回它的参数，这里传了一个this:就是调用方法上一个Observable，还有function实例
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
  ```
  ```

public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    //这里保存上一个调用方法的Observable,和此次function实例
    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

   //subscribeActual方法的功能就是把我们传递的参数给Observer的onNext方法。
   //如果有多个map操作，就会遍历这个这个方法，
   //一直到找到一个source(如ObservableJust)可以在subscribe()里的subscribeActual能真正把我们传递的参数给Observer的onNext方法
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
//正向:ObservableJust.subscribe(new MapObserver<T, U>(t, function)) :我们手动的url传给了下一个的MapObserver.onNext()参数。



    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

       //这里将订阅的Observer,和当前的function实例传进来
        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }
            if (sourceMode != NONE) {
                actual.onNext(null);
                return;
            }
            //做事件的变换赋值
            U v;
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
           //给订阅的Observer传递此次变换后的数据
            actual.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Override
        public U poll() throws Exception {
            T t = qs.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
}
  ```
![大体流程图](https://upload-images.jianshu.io/upload_images/7730759-670d744e52194553.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##Subscribe.on()源码：
  ```
.subscribeOn(Schedulers.io())// 上面之前的执行在子线程中
.observeOn(AndroidSchedulers.mainThread())// 下面之后的执行在主线程中
  ```
我们直接看重点部分：
  ```
// 创建了一个 ObservableSubscribeOn 
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}

// 主要看 subscribeActual 方法
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> s) {
        // 创建了一个 SubscribeOnObserver ，也就是把 SubscribeOnObserver 进行了一层包装
        // 代理设计模式
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);
        // 调用代理的 Observer 的 onSubscribe 方法
        s.onSubscribe(parent);
        // 把下面这个代码变为两行，容易看懂一点
        // parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
        Disposable disposable = scheduler.scheduleDirect(new SubscribeTask(parent));
        parent.setDisposable(disposable);
    }
}
// 看到这里差不多要明白了 implements Runnable 看样子要开线程的节奏
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

// Schedulers.io() 返回 IoHolder.DEFAULT
static final class IOTask implements Callable<Scheduler> {
        @Override
        public Scheduler call() throws Exception {
            return IoHolder.DEFAULT;
        }
}
// 单例设计模式 - 静态内部类
static final class IoHolder {
    static final Scheduler DEFAULT = new IoScheduler();
}

//scheduler.scheduleDirect最终调用这个方法生成Disposable
// 线程池 + 线程 + Runnable
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    // 创建 线程池
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
    // 代理
    DisposeTask task = new DisposeTask(decoratedRun, w);
    // 利用线程池去执行任务
    w.schedule(task, delay, unit);

    return task;
}
// 在子线程中执行 source.subscribe(parent); // 又要往上走，这是子线程处理的逻辑
  ```
**observerOn源码**
如果你不调用observeOn()方法，那么最后的一个Observer实例也是运行在上一个的线程中
直接看重点：
  ```
// 创建了一个 ObservableObserveOn 
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}

// 主要还是看 subscribeActual 方法
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
       // ...... 省略部分代码
       // 创建一个 Scheduler.Worker(HandlerWorker) 
       Scheduler.Worker w = scheduler.createWorker();
       // Worker对象和下一个observer传进去
       source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
// 最后的 onNext 是 schedule() 
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

    void schedule() {
        if (getAndIncrement() == 0) {
          //通过Handler切换到主线程中执行任务
            worker.schedule(this);
        }
    }
       //handlerMessage要处理的内容。
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }

  1. void drainNormal() {
            int missed = 1;

            final SimpleQueue<T> q = queue;
            final Observer<? super T> a = actual;

            for (;;) {
                if (checkTerminated(done, q.isEmpty(), a)) {
                    return;
                }

                for (;;) {
                    boolean d = done;
                    T v;

                    try {
                        v = q.poll();
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        s.dispose();
                        q.clear();
                        a.onError(ex);
                        return;
                    }
                    boolean empty = v == null;

                    if (checkTerminated(d, empty, a)) {
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(v);
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        2.void drainFused() {
            int missed = 1;

            for (;;) {
                if (cancelled) {
                    return;
                }

                boolean d = done;
                Throwable ex = error;

                if (!delayError && d && ex != null) {
                    actual.onError(error);
                    worker.dispose();
                    return;
                }

                actual.onNext(null);

                if (d) {
                    ex = error;
                    if (ex != null) {
                        actual.onError(ex);
                    } else {
                        actual.onComplete();
                    }
                    worker.dispose();
                    return;
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

}
// MAIN_THREAD 的 Scheduler 
public final class AndroidSchedulers {
    private static final class MainHolder {
        // new Handler(Looper.getMainLooper()) 创建一个主线程的 Handler 对象
        static final Scheduler MAIN_THREAD= new HandlerScheduler(new Handler(Looper.getMainLooper()));
    }
    //所以AndroidSchedulers.mainThread()最后返回是HandlerScheduler对象
   public static Scheduler mainThread() {
        return RxAndroidPlugins.onMainThreadScheduler(MAIN_THREAD);
    }
}
final class HandlerScheduler extends Scheduler {
    private final Handler handler;
    HandlerScheduler(Handler handler) {
        this.handler = handler;
    }
    @Override
    public Worker createWorker() {
        return new HandlerWorker(handler);
    }
}

// Handler 切换到主线程
private static final class HandlerWorker extends Worker {
    @Override
    public Disposable schedule(Runnable run, long delay, TimeUnit unit) {
      //这种写法在内部执行了scheduled.run
        Message message = Message.obtain(handler, scheduled);
        message.obj = this; // Used as token for batch disposal of this worker's runnables.
        // 但是 handler 并没有复写 handleMessage 方法，那是怎么调用了方法？一切都在 Handler 源码中
        handler.sendMessageDelayed(message, Math.max(0L, unit.toMillis(delay)));
    }
}
  ```

