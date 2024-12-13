> 本次源码基于Android 12.0分析

## Looper源码分析
**作用：**每个线程中只有一个`Looper`，`Looper`在创建的时候同时创建了一个`MessageQueue`,所以每个线程中也只有一个`MessageQueue`。在`ActivityThread`类的`main()`方法中已经在Looper声明当前线程为主线程，并开启了`Looper.loop()`循环。所以在主线程中为什么能一直循环等待工作，是`Looper.loop()`一直执行死循环的原因，同时在`ActivityThread`类中创建了一个Handler叫`H`，它定义和处理了：Activity启动，生命周期，更新UI，控件事件等消息方法，它们也是通过消息机制进行处理。

### **源码分析**：
#### 1. 向主线程和子线程提供的创建`Looper`的方法

  ```
 static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    // 向主线程提供创建Looper的方法
   @Deprecated
    public static void prepareMainLooper()
    {
        // 检查是否有创建Looper，没有则创建一个Looper和MessageQueue，false表示MessageQueue不可以退出
        prepare(false);
        synchronized(Looper.class) {
            // 如果已经创建主线程Looper再创建会抛异常，只能创建一次
            if (sMainLooper != null)
            {
                throw new IllegalStateException ("The main Looper has already been prepared.");
            }
            // 保存在sMainLooper变量
            sMainLooper = myLooper();
        }
    }

    // 为子线程提供创建Looper的方法
    public static void prepare()
    {
        // 检查是否有创建Looper，没有则创建一个Looper和MessageQueue，true表示MessageQueue可以退出
        prepare(true);

    }

    
    private static void prepare(boolean quitAllowed)
    {
        // 如果当前线程有Looper则会抛异常
        if (sThreadLocal.get() != null) {
            throw new RuntimeException ("Only one Looper may be created per thread");
        }
        // 通过ThreadLocal，向当前线程添加一个新的Looper
        sThreadLocal.set(new Looper (quitAllowed));
    }

    // Looper的构造方法
    private Looper(boolean quitAllowed)
    {
        // 创建一个MessageQueue，quitAllowed表示是否允许MessageQueue退出
        mQueue = new MessageQueue (quitAllowed);
        mThread = Thread.currentThread();
    }

  ```
每个线程只能创建一次Looper，否则会抛出异常
> `prepareMainLooper()`是为主线程创建Looper，并且创建一个不可以退出的MessageQueue,这也是为什么主线程一直不会销毁退出的原因。其中在`ActivityThread`中已经调用此方法创建主线程的Looper，所以我们不要在主线程调用此方法，不然会抛异常。
`prepare()`方法是提供给子线程创建Looper，并且创建一个可以退出的MessageQueue对象与之绑定，所以子线程在完成任务后是可以销毁退出的。

关于`ThreadLocal`会在分析完Hanlder后进行讲解。

#### 2. 进入`Looper.loop()`方法
  ```
    public static void loop() {
        // 获取当前线程的Looper
        final Looper me = myLooper();
        // Looper为null抛出异常
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
    
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        me.mSlowDeliveryDetected = false;
        // 开启死循环 ，除非 loopOnce()方法返回false ，才结束跳出死循环
        for (; ; ) {
            if (!loopOnce(me, ident, thresholdOverride)) {
                return;
            }
        }
    }

    private static boolean loopOnce(final Looper me,
                                    final long ident, final int thresholdOverride) {
         // MessageQueue.next()，可能会阻塞
        Message msg = me.mQueue.next();
        // mQueue.next()返回为null，则退出死循环
        if (msg == null) {
            return false;
        }

        // 设置此回调可以知道msg的开始和结束时间
        final Printer logging = me.mLogging;
        if (logging != null) {
            // 开始前调用接口方法
            logging.println(">>>>> Dispatching to " + msg.target + " "
                    + msg.callback + ": " + msg.what);
        }

        try {
            // 发给Handler.dispatchMessage(msg) 处理消息
            msg.target.dispatchMessage(msg);
        } catch (Exception exception) {
            throw exception;
        } finally {

        }
        if (logging != null) {
            // 结束调用接口方法
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // msg回收到消息池循环利用
        msg.recycleUnchecked();

        return true;
    }
  ```
> 调用`Looper.loop()`线程开始进入死循环，然后不断的调用`MessageQueue.next()`方法获取msg，如果没有msg或msg不到执行时间就会阻塞，如果MessageQueue要退出则返回null。
`loopOnce()`的msg方法的执行是同步，只有前面的一个msg方法执行完成才能执行下一个msg方法。msg.target就是Handler。
这里执行死循环为什么不卡死，其实对于CPU来说是没有卡死的概念的，应用在UI线程执行耗时任务会ANR是因为对其进行了时间监听超出一定时间后抛出ANR。
`msg.recycleUnchecked()`: 是对msg进行回收，在下面的Message中会分析得到。

#### 3. Looper.`quit()/quitSafely()`

  ```
    final MessageQueue mQueue;

    public void quit()
    {
       // 调用MessageQueue.quit方法，参数false表示不是安全删除
        mQueue.quit(false);
    }

    public void quitSafely()
    {
        // 调用MessageQueue.quit方法，参数true表示安全删除
        mQueue.quit(true);
    }
  ```
> Looper.`quit()/quitSafely()`方法的调用代表要退出MessageQueue队列，都是调用MessageQueue.quit方法。
`quit()`不是安全的退出，它会把MessageQueue的全部消息清空并回收。
`quitSafely()`方法则是，把还没有发送出去的消息进行清空回收。

### ThreadLocal
**作用**：根据`ThreadLocal`作为`key`值，在当前线程中存储一个数据或获取一个数据。这个数据的作用域是当前的线程，所以以线程作为作用域的数据可以通过`ThreadLocal`来存储，或在代码中有很深的回调参数也可以通过ThreadLocal在当前线程中共享。

#### 源码和原理
基本原理：在`Thread`线程类中，有一个`ThreadLocal. ThreadLocalMap`的哈希变量，这个`ThreadLocal. ThreadLocalMap`变量以`ThreadLocal`为`key`，所以ThreadLocal就是在当前线程中去给这个哈希变量去设置数据或获取数据。
  ```
public class Thread implements Runnable {
    // ThreadLocal.ThreadLocalMap变量
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
  ```

**ThreadLocal的核心代码：**

  ```
public class ThreadLocal<T> {

    //   获取Thread的threadLocals变量
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    // 获取当前线程的哈希变量，查找以此`ThreadLoca`为`key`的数据，没有则返回null
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T) e.value;
                return result;

            }

        }
        return setInitialValue();

    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;

    }

    protected T initialValue() {
        return null;
    }

    // 给当前线程的哈希变量添加以此`ThreadLocal`为`key`，value来增加结点。
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

   // 每个线程都持有一个此哈希变量类
    static class ThreadLocalMap {

        private static final int INITIAL_CAPACITY = 16; // 初始化空间大小
        private Entry[] table; // 存储结点
        private int threshold; // 应该扩展的大小

// 结点
 static class Entry extends WeakReference<ThreadLocal<?>> {
            /**
             * The value associated with this ThreadLocal.
             */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        // 第一次创建ThreadLocalMap
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
     
        // 添加结点
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len - 1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        // 查找结点
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            // Android-changed: Use refersTo()
            if (e != null && e.refersTo(key))
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }


    }
}
  ```
> `ThreadLocal`内部又一个哈希类`ThreadLocalMap`,而每个线程都定义了这个`ThreadLocal.ThreadLocalMap`变量。`ThreadLocal`的`set(value)`方法就是往当前线程的map变量以此ThreadLocal为key添加结点，`get()`方法则是以此ThreadLocal为key向当前线程的map变量查询是否有数据并返回。

## MessageQueue源码分析
**作用**：`MessageQueue`是消息队列，但其内部实现并不是队列，而是一个单向链表组成，这个单向链表把`Message`按其执行的先后顺序排列起来，`MessageQueue`的主要作用就是向单列表`插入和获取Message`,如`Handler`向MessageQueue插入消息，`Looper`死循环向MessageQueue获取消息。`MessageQueue`除了处理`Message`信息外，还定义了`IdleHandler`接口在Message空闲的时候去执行此接口实例。

### 源码分析
对`MessageQueue`的核心变量进行解释说明，便于我们后续分析:
  ```
public final class MessageQueue {

    // 表示消息队列是否可以被关闭，通过构造函数传入，主线程传false表示不可以关闭
    private final boolean mQuitAllowed;

    // native层代码的MessageQueue指针
    private long mPtr; // used by native code

    // 存储的单向链表
    Message mMessages;

    // 保存线程空闲时需要处理的事务
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    // 即将要执行的IdleHandler
    private IdleHandler[] mPendingIdleHandlers;

    // 表识MessageQueue是否正在关闭
    private boolean mQuitting;

    // 是否阻塞
    private boolean mBlocked;
    
    // 构造函数
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed; // 表识是否可以退出消息队列
        mPtr = nativeInit(); // native初始化创建NativeMessageQueue
    }
}
  ```

#### 1. 插入消息 enqueueMessage
`MessageQueue.enqueueMessage`代码很长，我们分多段分析:
  ```
    boolean enqueueMessage(Message msg, long when) {
        // msg.target就是handler没有设置则抛异常
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
  ```
> msg.target == null则是没有分发的目标，是不能被添加到MessageQueue，除非此msg是屏障，但是屏障只能在`MessageQueue`对象的`postSyncBarrier()`方法中创建生成而且此方法是不对应用开发者开放的。
  ```
// 加入同步锁
 synchronized (this) {
    // 如果该msg.isInUse()为true，则表明该msg已经入MessageQueue队列，则跑异常
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use."
    }
    // 如果消息队列正在退出，则没必要添加msg，并回收msg
    if (mQuitting) {
        IllegalStateException e = new IllegalStateException(
                msg.target + " sending message to a Handler on a dead thread");
        Log.w(TAG, e.getMessage(), e);
        msg.recycle();
        return false;
    }
    // 设置msg的InUse、when
    msg.markInUse();
    msg.when = when;
    // 获取消息链表
    Message p = mMessages;
    // 是否需要唤醒
    boolean needWake;
    // 如果消息的链表为null，或when为0表示加到链头 ，或msg.when小于链头的when，则把msg加入到链头
    if (p == null || when == 0 || when < p.when) {
        // New head, wake up the event queue if blocked.
        msg.next = p;
        mMessages = msg;
        // 如果正在被阻塞则需要被唤醒
        needWake = mBlocked;
    } 
  ```
> 开始进行同步锁，并检查msg是否`InUse`和消息队列是否正在退出`mQuitting`，如果正常则继续往下执行。
给msg设置`InUse、when`，并判断当前msg是否可以加入到链头，如果条件满足则把此msg加入到链头，并根据是否阻塞判断是否需要唤醒。

  ```
          // 否则当前msg不会插入到链头
          else {
            // 阻塞 与 链头是屏障 与  msg是异步 则唤醒否则不唤醒
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            // 按时间找到msg应该插入的位置
            for (; ; ) {
                prev = p;
                p = p.next;
                // 到最后 或 找到比msg.when大的链表结点，则退出循环
                if (p == null || when < p.when) {
                    break;
                }
                // 如果msg时异步，在它前面也有msg是异常则不需要唤醒
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            // msg插入到中间
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        // 如果需要唤醒，调用native层进行唤醒
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
  ```
> 如果msg不是链表的第一个，则按照时间排序找到msg插入的位置，其中如果链头是屏障而且在它插入的位置之前没有其他异步msg则需要唤醒，否则不需要唤醒。

#### 2. 获取消息 next()
`MessageQueue.next()`主要由`Looper.loop()`方法调用，`looper`方法主要流程为：
> 获取`Looper`对象
根据Looper获取对应的`MessageQueue`对象
 死循环遍历
通过`MessageQueue.next()`获取`Message msg`对象
调用`msg.target.dispatchMessage(msg)`分发对象
调用`msg. recycleUnchecked()`方法回收对象

下面分多段分析`MessgeQueue.next()`方法：
  ```
Message next() {
    // native层的MessageQueue指针
    final long ptr = mPtr;
    if (ptr == 0) {
        // 为0 代表MessageQueue已经退出或还没初始化，返回null，让Looper.loop()退出死循环
        return null;
    }
    // 准备要执行的IdleHandler数量，默认为-1
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    // native层需要用到的变量，默认为0，>0表示有消息未处理，-1代表阻塞
    int nextPollTimeoutMillis = 0;
  ```
> 变量`nextPollTimeoutMillis`默认为0，表示到下一个msg执行时还需要的时间，如果>0表示还有消息没到执行时间，-1则表示没有消息要处理那就进入阻塞
  ```
// 开启死循环
for (; ; ) {
    if (nextPollTimeoutMillis != 0) {
        Binder.flushPendingCommands();
    }
    // native层代码，根据nextPollTimeoutMillis值判断是否阻塞
    nativePollOnce(ptr, nextPollTimeoutMillis);
    // 同步锁
    synchronized (this) {
        // 开机到现在的时间
        final long now = SystemClock.uptimeMillis();
        Message prevMsg = null;
        // 获取消息链表链头
        Message msg = mMessages;
        // msg.target == null 表示msg是屏障
        if (msg != null && msg.target == null) {
            // 向后循环，直到查到第一个异步的msg
            do {
                prevMsg = msg;
                msg = msg.next;
            } while (msg != null && !msg.isAsynchronous());
        }
  ```
> 开启死循环后，首先调用`native`层的`nativePollOnce`方法并传入`nextPollTimeoutMillis`作为参数，如果值为-1表示没有消息就阻塞。
拿到消息链头后，首先会判断根据 `msg.target == null`判断链头是否是消息屏障，如果是，那么这次拿取的Message就是链表的第一个`异步Message`。注意这里的`Messsge`的同步异步并不是线程所谓的多任务执行，Handler处理消息都是同步的一个消息处理完成后再到另一个消息处理，`异步Message`是表示这个Message需要及时处理，所以消息屏障就会找到这几需要及时处理的`异步Message`给它先处理。需要注意的是，如果有消息屏障而没有异步消息，那么此msg会永远为null。

  ```
        //  获取msg != null
        if (msg != null) {
             // msg还没到执行时间
            if (now < msg.when) {
                // nextPollTimeoutMillis 为还需要等待的时间
                nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MA
            } else {
                //  msg到执行时间
                // 不阻塞
                mBlocked = false;
                // 如果是消息屏障，prevMsg != null，
                if (prevMsg != null) {
                    // 删除msg在链表结点
                    prevMsg.next = msg.next;
                } else {
                    // 删除msg在链头结点
                    mMessages = msg.next;
                }
                msg.next = null;
                if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                msg.markInUse();
                // 退出死循环，返回msg给Looper
                return msg;
            }
        } else {
            // msg  为 null ，nextPollTimeoutMillis = -1进入阻塞
            nextPollTimeoutMillis = -1;
        }
 
  ```
> 找到`Message msg`后，先判断是否`msg为null`，为null则` nextPollTimeoutMillis = -1`进入阻塞状态，`msg不为null`则判断是否到执行时间，还没有到则`nextPollTimeoutMillis等于还要等待的时间`,如果到了执行时间则把此`msg`从链表中移除。

如果还能继续往下执行代码表示并没有`return msg`，所以要么是链表中没有消息，要么是还没有消息的执行时间，所以此时Message是空闲的。
  ```
        // 如果有调用quit(boolean safe)方法则会把mQuitting赋值为true，表示MessageQueue正在退出
        if (mQuitting) {
            // 销毁mPtr的地址 置0，并返回null，让Looper也退出死循环
            dispose();
            return null;
        }
      
        // 刚进来的时候pendingIdleHandlerCount为-1，且判断Message没有执行
        if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
            // mIdleHandlers数组大小
            pendingIdleHandlerCount = mIdleHandlers.size();
        }
        // 没有IdleHandler，则结束这个查询，如果有在往下
        if (pendingIdleHandlerCount <= 0) {
            // 阻塞
            mBlocked = true;
            continue;
        }
        // 将mIdleHandlers的数组转到准备执行的数组
        if (mPendingIdleHandlers == null) {
            mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
        }
        mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
    }
  ```
> 在不同执行`Message`的时候，会查询是否需要处理IdleHandler接口实例数组，没有IdleHandler则返回这次查询，如果有在往下执行代码

  ```
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            // 遍历执行IdleHandler.queueIdle()方法
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            // 根据返回的布尔值，判断是否需要重复执行，false则把此IdleHandler实例从数组中移除
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        // pendingIdleHandlerCount重置为0
        pendingIdleHandlerCount = 0;
        // 执行完IdleHandler后，可能已经有Message要执行
        nextPollTimeoutMillis = 0;
    }
  ```
> 所以在`MessageQueue`没有`Message`要执行的时候，它会执行IdleHandler的方法，通过`Looper.myQueue().addIdleHandler`去添加IdleHandler实例做一下不跟UI抢资源的操作。

### 消息屏障
**作用**：消息屏障用于把标记为`异步Message`揪出来，然后让异步message先得到Handler的执行，为什么要这么设计，因为我们应用的UI事件也是通过Handler处理的，UI事件的响应优先级是很高的，如果它按照顺序排队可能排很久都没到它，所以有一个`没有设置targer的message`作为消息屏障，在next()方法中只要识别出此消息是消息屏障就会找到第一个异步Message交给Handler执行。

  ```
public final class MessageQueue {

    // 自增作为消息屏障的token
    private int mNextBarrierToken;

    /**
     * @hide
     */
    // 这个方法是不对应用者开发者提供，所以我们无法设置消息屏障
    public int postSyncBarrier() {
        // when为开机到现在的时间，返回消息屏障的token
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // 同步锁
        synchronized (this) {
            //  mNextBarrierToken++为token
            final int token = mNextBarrierToken++;
            // 获取一个Message
            final Message msg = Message.obtain();
            // 没有设置target
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;

            // 拿到链头
            Message prev = null;
            Message p = mMessages;
            // 找到msg的下一个next结点
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }

            }
            // 将msg插入对应的位置
            if (prev != null) {
                msg.next = p;
                prev.next = msg;

            } else {
                msg.next = p;
                mMessages = msg;
            }
            //返回token
            return token;

        }
    }

    /**
     * @hide
     */
    // 移除消息屏障
    public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
}
  ```
> 消息屏障的添加和移除都不对应用开发者提供，而且`移除`也是手动移除。消息屏障在没有异步消息的时候一定要及时移除不然会造成一直阻塞，可能这也是系统不对外提供的原因。


## Handler源码分析
**作用**：Handler主要有三个作用：
> 1.通过构造函数绑定`Looper`
2.给`MessageQueue`发送插入消息，即调用`MessageQueue.enqueueMessage`方法
3.通过`handleMessage`处理消息

#### 1. 构造函数绑定`Looper`
  ```
    @Deprecated
    public Handler() {
        this(null, false);
    }
    @Deprecated
    public Handler(@Nullable Callback callback) {
        this(callback, false);
    }
    public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }
    public Handler(@NonNull Looper looper, @Nullable Callback callback) {
        this(looper, callback, false);
    }
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
    public Handler(boolean async) {
        this(null, async);
    }

  ```
创建Handler的构造函数有很多，但最终都是回调到这两个构造函数：
**如果没传Looper**：
  ```
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                        klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                    "Can't create handler inside thread " + Thread.currentThread()
                            + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
  ```
> 没有传Looper就使用当前线程的Looper，并从Looper中获取`MessageQueue`

**有传Looper**:
  ```
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;

    }
  ```
> 使用传进来的Looper

#### 2. 发送消息 MessageQueue.enqueueMessage
![](https://upload-images.jianshu.io/upload_images/22650779-e196a2eceac17734.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
发送消息提供了很多API，功能大致有`当前时间执行、延迟、加入到链头`三种发送消息功能，并且核心发送消息的三个方法为`sendMessageDelayed、sendMessageAtTime、enqueueMessage`

**post相关方法**：

  ```
    public final boolean post(@NonNull Runnable r) {
        return sendMessageDelayed(getPostMessage(r), 0);
    }

    public final boolean postAtTime(@NonNull Runnable r, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r), uptimeMillis);
    }

    public final boolean postAtTime(
            @NonNull Runnable r, @Nullable Object token, long uptimeMillis) {
        return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
    }

    public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
    }

    /**
     * @hide
     */
    public final boolean postDelayed(Runnable r, int what, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r).setWhat(what), delayMillis);
    }

    public final boolean postDelayed(
            @NonNull Runnable r, @Nullable Object token, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r, token), delayMillis);
    }

    public final boolean postAtFrontOfQueue(@NonNull Runnable r) {
        return sendMessageAtFrontOfQueue(getPostMessage(r));
    }
    
    // 创建Message
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
    // 创建Message
    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }
  ```

**send相关方法**：
  ```
   public final boolean sendMessage(@NonNull Message msg) {
        return sendMessageDelayed(msg, 0);
    }

    public final boolean sendEmptyMessage(int what) {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageAtTime(msg, uptimeMillis);
    }

    public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }
  ```
 以下三个方法是核心
  ```
    public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
                                   long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
  ```
> 在`enqueueMessage`方法中会为每个msg设置`msg.target = this`把Handler实例作为以后msg执行回调对象，并调用`MessageQueue.enqueueMessage`方法

#### 3. 处理消息
  ```
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
  ```
> 其优先级为：先调用`msg.callback`,否则调用`Handler.mCallback.handleMessage`,最后为`Handler.handleMessage`

**移除消息：**
  ```
// 移除此handler的mag.what等于此what的msg
handler.removeMessages(1)
// 移除此handler的mag.what等于此what且mag.object等于此object的msg
handler.removeMessages(1, null)
// 移除此handler的mag.callBack等于此Runnable的msg
handler.removeCallbacks {}
// 移除此handler的mag.callBack等于此Runnable且mag.object等于此object的msg
handler.removeCallbacks({}, null)
// 移除此handler的msg.object等于此object，object= null，则移除此handler所有msg
handler.removeCallbacksAndMessages(null)
  ```

## Message源码分析
  ```
public final class Message implements Parcelable {

    public int what; // 表识
    // 传输的数据
    public int arg1;
    public int arg2;
    public Object obj;
    Bundle data;

    //正在使用的标志值,表示当前 Messgae 正处于使用状态,
    static final int FLAG_IN_USE = 1 << 0;
    //异步标志值 表示当前 Message 是异步的
    static final int FLAG_ASYNCHRONOUS = 1 << 1;
    int flags;
    @VisibleForTesting(visibility = VisibleForTesting.Visibility.PACKAGE)
    public long when; // msg执行的时间
    /*package*/ Handler target; // 目标handler
    /*package*/ Runnable callback;
    /*package*/ Message next;


    /**
     * @hide
     */
    public static final Object sPoolSync = new Object();
    private static Message sPool; // 消息池
    private static int sPoolSize = 0;
    private static final int MAX_POOL_SIZE = 50; // 消息池最大数量
    private static boolean gCheckRecycle = true;

    // 从消息池复用Message，没有才新创建一个
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    // 检查是否isInUse，将msg回收到消息池
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;

        }
        recycleUnchecked();
    }

    // 将msg回收到消息池(在Looper.loop()方法消息执行完后调用)
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }


}
  ```
> Message实现了Parcelable序列化，通过`obtain`方法可以从消息池复用Message，通过`recycle()、recycleUnchecked()`回收Message到消息池，这里的msg回收并不需要我们手动调用，系统已经帮我们调用了。

## HandlerThread
**作用**：主要是为子线程创建好相关的Handler机制，通过`start()`它我们可以通过其拿到子线程的`Looper`，并很好的解决了Looper未能及时初始化问题，使我们安全的拿到子线程的 Looper。

**使用**：
  ```
class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        val handlerThread = HandlerThread("test")
        handlerThread.start()
        
        
        val handler = Handler(handlerThread.looper) {
            Log.d("数据", "${Thread.currentThread().name}")
            false
        }
        val msg = Message.obtain()
        msg.what = 1
        handler.sendMessage(msg)
    }

  ```

**源码**：
  ```
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable
    Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }

   
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        boolean wasInterrupted = false;

        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                    wasInterrupted = true;
                }
            }
        }

        /*
         * We may need to restore the thread's interrupted flag, because it may
         * have been cleared above since we eat InterruptedExceptions
         */
        if (wasInterrupted) {
            Thread.currentThread().interrupt();
        }

        return mLooper;
    }
    
    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }
    
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
    
    public int getThreadId() {
        return mTid;
    }
}
  ```

## 总结
消息机制主要是为了多线程间的通信，主要是子线程对UI线程的通信，还有应用的事件也是通过Handler消息机制进行管理执行。Handler消息机制是同步，主要靠`Looper、MessageQueue`类进行实现，Handler则是对其两个的封装调用。
一个线程中可以有多个Handler，一个Looper和一个MessageQueue，所有的Handler消息的处理都是同步的，不会有多消息同时处理的情况。





