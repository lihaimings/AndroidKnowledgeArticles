## 1. 内存泄漏
为什么会出现内存泄漏？因为在GC垃圾回收时会利用GC Root`可达性`分析算法去遍历哪些对象正在被引用。如果一个对象该销毁时却被另一个更长生命周期的对象引用，则会发生该销毁的对象无法被回收，导致内存泄漏。
在Java中，有四种对象引用：`强引用`可达性分析算法中此引用不会被回收；`软引用`可达性分析算法如果此时内存溢出时，这种引用的对象会被回收；`弱引用`可达性分析算法，对于这种引用对象会将不在引用链之内则会将其回收；`虚引用`则是一种标志作用，在回收时会调用finalize()方法。

## 2. 内存泄漏常见场景
#### 2.1 单例模式中使用Context
当在一个单例对象中持有Activity的Context引用时，该引用会持有整个应用程序的生命周期，这可能会导致内存泄漏。

```
public class MySingleton {
    private Context mContext;
    private static MySingleton sInstance;

    private MySingleton(Context context) {
        mContext = context;
    }

    public static MySingleton getInstance(Context context) {
        if (sInstance == null) {
            sInstance = new MySingleton(context);
        }
        return sInstance;
    }

    // ... some other methods ...
}
```
正确代码：
```
public class MySingleton {
    private Context mContext;
    private static MySingleton sInstance;

    private MySingleton(Context context) {
        mContext = context.getApplicationContext(); //使用ApplicationContext代替Activity或Application的Context
    }

    public static MySingleton getInstance(Context context) {
        if (sInstance == null) {
            sInstance = new MySingleton(context);
        }
        return sInstance;
    }

    // ... some other methods ...
}
```
#### 2.2 非静态内部类持有外部类引用
当一个非静态内部类实例化时，它会持有一个对外部类实例的引用，如果该内部类的实例长时间存在，则可能导致外部类实例的生命周期过长，从而导致内存泄漏。

示例代码：
```
public class MyActivity extends Activity {
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // ... handle message ...
        }
    };

    // ... some other methods ...
}
```
正确代码：
```
public class MyActivity extends Activity {
    private static class MyHandler extends Handler {
        private WeakReference<MyActivity> mActivity;

        public MyHandler(MyActivity activity) {
            mActivity = new WeakReference<MyActivity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MyActivity activity = mActivity.get();
            if (activity != null) {
                // ... handle message ...
            }
        }
    }

    private MyHandler mHandler = new MyHandler(this);

    // ... some other methods ...
}
```

#### 2.3 注册广播接收器未注销
当应用程序注册广播接收器时，如果不及时注销，它将一直存在，从而导致内存泄漏。

正确代码：
```
public class MyActivity extends Activity {
    private BroadcastReceiver mReceiver;
    private boolean mIsReceiverRegistered = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    mReceiver = new MyBroadcastReceiver();
    IntentFilter filter = new IntentFilter();
    filter.addAction(Intent.ACTION_SCREEN_ON);
    filter.addAction(Intent.ACTION_SCREEN_OFF);
    mIsReceiverRegistered = true;
    registerReceiver(mReceiver, filter); //注册广播接收器
}

 @Override
protected void onDestroy() {
    super.onDestroy();
    if (mIsReceiverRegistered) { //判断广播接收器是否已注册
        unregisterReceiver(mReceiver); //注销广播接收器
        mIsReceiverRegistered = false;
    }
}

// ... some other methods ...
}
```

#### 2.4 匿名内部类持有外部类引用
当一个匿名内部类实例化时，它会持有一个对外部类实例的引用，如果该内部类的实例长时间存在，则可能导致外部类实例的生命周期过长，从而导致内存泄漏。

示例代码：

```java
public class MyActivity extends Activity {
    private Button mButton;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread().start();
    }

    // ... some other methods ...
}
```

#### 2.5 Handler导致的内存泄漏
当使用Handler时，如果在处理消息时，持有Activity或Fragment的引用，则可能导致内存泄漏。

示例代码：
```
public class MyActivity extends Activity {
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // ... handle message ...
        }
    };

    // ... some other methods ...
}
```
正确代码：
```
public class MyActivity extends Activity {
    private static class MyHandler extends Handler {
        private WeakReference<MyActivity> mActivity;

        public MyHandler(MyActivity activity) {
            mActivity = new WeakReference<MyActivity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MyActivity activity = mActivity.get();
            if (activity != null) {
                // ... handle message ...
            }
        }
    }

    private MyHandler mHandler = new MyHandler(this);

    // ... some other methods ...
}
```

#### 2.6 资源没有正确释放导致的内存泄漏
当使用资源（如Bitmap、Cursor等）时，如果没有正确释放，则可能导致内存泄漏。

示例代码：
```
public class MyActivity extends Activity {
    private Bitmap mBitmap;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.my_image);
    }

    // ... some other methods ...
}
```
正确代码：
```
public class MyActivity extends Activity {
    private Bitmap mBitmap;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.my_image);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mBitmap != null) {
            mBitmap.recycle();
            mBitmap = null;
        }
    }

    // ... some other methods ...
}
```

## 3. 内存泄漏监控
虽然对常见的内存泄漏场景有认识，但是还是需要对内存泄漏进行自动监控，目前有的检测工具有：[leakcanary](https://github.com/square/leakcanary)和[matrix的Resource Canary](https://github.com/Tencent/matrix)。其使用过程不再讲述。

###  3.1 检测内存泄漏
无论是使用Leakcanary还是Matrix工具，它们检测代码是否内存泄露都是一样的思路：
通过 registerActivityLifecycleCallbacks在每个Activity销毁onDestroy的时候，通过弱引用WeakReference去持有Activity，然后通过间隔预定的时间手动调用GC，并通过弱引用的get()方法去查看Activity在内存中是否被回收，如果没有被回收则判断为内存泄露。当然手动调用GC并一定会做回收操作，像Matrix就通过多次的GC判断，才认为内存泄露。
```
 class ActivityRefWatcher(val application: Application?) {

    private var handlerThread: HandlerThread? = null
    private var handler: Handler? = null
    private var lastTriggeredTime: Long = 0
    private val maxRedetectTimes = 2
    private val lock = java.lang.Object()

    private val destroyedActivityInfos: ConcurrentLinkedQueue<DestroyedActivityInfo> by lazy { ConcurrentLinkedQueue() }

    private val retryableTaskExecutor: RetryableTaskExecutor by lazy {
        RetryableTaskExecutor(GC_TIME, handlerThread)
    }

    init {
        handlerThread =
            HandlerThreadUtil.getNewHandlerThread("ActivityRefWatcher", Thread.NORM_PRIORITY)
        handler = HandlerThreadUtil.getDefaultHandler()
    }

    fun start() {
        stopDetect()
        application?.registerActivityLifecycleCallbacks(removedActivityMonitor)
        scheduleDetectProcedure()
    }

    fun stop() {
        stopDetect()
        handler?.removeCallbacksAndMessages(null)
    }

    private val removedActivityMonitor: ActivityLifecycleCallbacks =
        object : EmptyActivityLifecycleCallback() {
            override fun onActivityDestroyed(activity: Activity) {
                // 弱引用Activity，并收集相关信息
                pushDestroyedActivityInfo(activity)
                // 2s 后开始触发gc
                handler?.postDelayed({ triggerGc() }, delayTime)
            }
        }


    private val scanDestroyedActivitiesTask: RetryableTaskExecutor.RetryableTask =
        object : RetryableTaskExecutor.RetryableTask {

            override fun execute(): RetryableTaskExecutor.RetryableTask.Status {
                return checkDestroyedActivities()
            }
        }

    private fun scheduleDetectProcedure() {
        retryableTaskExecutor.executeInBackground(scanDestroyedActivitiesTask)
    }

    private fun stopDetect() {
        application?.unregisterActivityLifecycleCallbacks(removedActivityMonitor)
        unscheduleDetectProcedure()
    }

    private fun unscheduleDetectProcedure() {
        retryableTaskExecutor.clearTasks()
        destroyedActivityInfos.clear()
    }


    private fun pushDestroyedActivityInfo(activity: Activity) {
        val activityName = activity.javaClass.name
        val uuid = UUID.randomUUID()
        val keyBuilder = java.lang.StringBuilder()
        keyBuilder.append(ACTIVITY_REFKEY_PREFIX)
            .append(activityName)
            .append("_")
            .append(java.lang.Long.toHexString(uuid.mostSignificantBits))
            .append(java.lang.Long.toHexString(uuid.leastSignificantBits))
        val key = keyBuilder.toString()
        val destroyedActivityInfo = DestroyedActivityInfo(key, activity, activityName)
        destroyedActivityInfos.add(destroyedActivityInfo)
        synchronized(lock) {
            lock.notifyAll()
        }
    }

    /**
     * 调用GC
     */
    private fun triggerGc() {
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastTriggeredTime < GC_TIME / 2 - 100) {
            Log.d(TAG, "skip triggering gc for frequency")
            return
        }
        lastTriggeredTime = currentTime
        Log.d(TAG, "triggering gc...")
        Runtime.getRuntime().gc()
        try {
            Thread.sleep(100)
        } catch (e: InterruptedException) {
            e.printStackTrace()
        }
        Runtime.getRuntime().runFinalization()
        Log.d(TAG, "gc was triggered.")
    }

    private fun checkDestroyedActivities(): RetryableTaskExecutor.RetryableTask.Status {
        if (destroyedActivityInfos.isEmpty()) {
            synchronized(lock) {
                try {
                    while (destroyedActivityInfos.isEmpty())
                        lock.wait()
                } catch (ignored: Throwable) {
                    // Ignored.
                }
            }
            return RetryableTaskExecutor.RetryableTask.Status.RETRY
        }

        triggerGc()

        val infoIt = destroyedActivityInfos.iterator()
        while (infoIt.hasNext()) {
            val destroyedActivityInfo = infoIt.next()
            triggerGc()
            if (destroyedActivityInfo.activityRef.get() == null) {
                infoIt.remove()
                continue
            }
            ++destroyedActivityInfo.detectedCount
            if (destroyedActivityInfo.detectedCount < maxRedetectTimes) {
                triggerGc()
                continue
            }
            Log.i(
                TAG,
                "the leaked activity ${destroyedActivityInfo.activityName} with key ${destroyedActivityInfo.key} has been processed. stop polling",
            )
            // 内存泄漏
            infoIt.remove()
        }
        return RetryableTaskExecutor.RetryableTask.Status.RETRY
    }

    companion object {

        private const val TAG = "ActivityRefWatcher"

        private val delayTime: Long = 2_000

        private const val ACTIVITY_REFKEY_PREFIX = "ACTIVITY_RESCANARY_REFKEY_"

        private val GC_TIME = TimeUnit.MINUTES.toMillis(1)

    }


}
```

### 3.2 dump与分析hprof
当知道有Activity内存泄漏之后，就要去分析内存泄漏的引用链。这里可以通过内存快照来分析泄漏链，hprof文件就是虚拟机在某个时刻上所有对象的内存快照，记录了对象的类名，大小和引用关系等。所以，这里关于hprof有两个操作，一是dump hprof文件，二是分析hprof文件。这两个操作都是耗时的，一定要在子线程或通过fork一个子进程，进行dump和分析hprof。

**dump hprof**
dump内存快照可以通过`Debug.dumpHprofData`方法，dump文件大小可能有几百兆，一些优化是边dump边裁剪文件，这需要涉及到native hook的技术。
下面分别列出在子线程中dump和在子进程dump的实例代码：
- 子线程dump 
```

    public void dumpHprofData() {
        final String hprofPath = "/sdcard/myapp.hprof";

        // 在异步线程中执行dumpHprofData()操作
        new Thread(new Runnable() {
            @Override
            public void run() {
                Debug.dumpHprofData(hprofPath);

                // 在主线程中提示用户操作完成
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(MyActivity.this, "hprof file generated", Toast.LENGTH_SHORT).show();
                    }
                });
            }
        }).start();
    }

            val storageDirectory = File(application?.cacheDir, "leakactivity")
            if (!storageDirectory.exists()) {
                storageDirectory.mkdir()
            }
            val fileName =
                SimpleDateFormat("yyyy-MM-dd_HH-mm-ss_SSS'.hprof'", Locale.US).format(Date())
            val file = File(storageDirectory, fileName)
            // dump 出堆转储文件
            Debug.dumpHprofData(file.absolutePath)
            Log.i(TAG, "dumpHeap: ${file.absolutePath}")
```
- 子进程dump 
```
private fun dumpHprof() {
    Thread {
        // 创建一个子进程
        val process = Runtime.getRuntime().exec(arrayOf("sh"))

        // 获取输出流和输入流
        val outputStream = process.outputStream
        val inputStream = process.inputStream

        // 向子进程写入命令
        outputStream.write("am dumpheap com.package.name /sdcard/leak.hprof\n".toByteArray())
        outputStream.flush()

        // 读取子进程输出的结果
        val reader = BufferedReader(InputStreamReader(inputStream))
        var line: String?
        while (reader.readLine().also { line = it } != null) {
            Log.d("DumpHprof", line!!)
        }

        // 关闭输出流和输入流
        outputStream.close()
        inputStream.close()

        // 等待子进程结束
        process.waitFor()

        // 处理hprof文件
        // ...
    }.start()
}
```
这段代码创建了一个子进程，并向子进程发送命令来执行Debug.dumpHprofData操作，这样就可以让主线程不卡顿了。

**分析 hprof**
分析hprof也是一种相对耗时的操作，分析hprof可以在本地也可以放到服务器上，如果之前的hprof文件没有裁剪，可以裁剪之后才分析或上传。https://blog.yorek.xyz/android/3rd-library/hprof-shrink/ 一文中讲了几种方案，这里不展开分析。提供几种开源方案，一种是shark_leakcanary库，一种是haha库，当然也可以自己实现分析hprof文件，其主要结构是header和多个record组成。
如下是使用shark_leakcanary库实现的hprof文件的分析。

```
  private fun dumpHeap() {
        handler?.post {
            val heapAnalyzer = HeapAnalyzer(OnAnalysisProgressListener { step ->
                Log.i(TAG, "Analysis in progress, working on: ${step.name}")
            })
            val heapAnalysis = heapAnalyzer.analyze(
                heapDumpFile = file,
                leakingObjectFinder = FilteringLeakingObjectFinder(
                    AndroidObjectInspectors.appLeakingObjectFilters
                ),
                referenceMatchers = AndroidReferenceMatchers.appDefaults,
                computeRetainedHeapSize = true,
                objectInspectors = AndroidObjectInspectors.appDefaults.toMutableList(),
                proguardMapping = null,
                metadataExtractor = AndroidMetadataExtractor
            )
            Log.i(TAG, "dumpHeap: \n$heapAnalysis")
        }
    }

```

## 4. Bitmap优化

### 4.1 常规Bitmap优化

> 图片占用的内存大小 = 图片宽度 × 图片高度 × 每个像素占用的字节数
例如，如果有一张 1000 × 1000 像素的 ARGB_8888 格式的图片，每个像素占用 4 个字节，则该图片占用的内存大小为：1000 × 1000 × 4 = 4,000,000 字节 = 3.81 MB

由于 Bitmap 对象可能占用大量的内存，因此在使用 Bitmap 时需要注意其优化，以避免内存问题和性能问题。以下是一些 Android Bitmap 的优化方案：

1. 使用 inSampleSize 属性来减少 Bitmap 对象的内存使用，它指定了加载图片时缩小的倍数。例如，如果将 inSampleSize 设为 2，则图片将被缩小为原始大小的 1/2。
```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = 2;
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.image, options);
```
2. 使用 RGB_565 格式来减少 Bitmap 对象的内存使用。默认情况下，Android 使用 ARGB_8888 格式来表示 Bitmap 对象。而使用 RGB_565 格式可以将每个像素的内存使用减少到一半。
```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inPreferredConfig = Bitmap.Config.RGB_565;
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.image, options);
```
3. 使用 BitmapRegionDecoder 来只加载图片的一部分，而不是整个图片。例如，如果只需要加载图片的顶部一部分，可以使用以下代码：
```
InputStream inputStream = getResources().openRawResource(R.drawable.image);
BitmapRegionDecoder decoder = BitmapRegionDecoder.newInstance(inputStream, false);
Bitmap bitmap = decoder.decodeRegion(new Rect(0, 0, decoder.getWidth(), decoder.getHeight() / 2), null);
```
4. 缓存 Bitmap 对象
使用 LruCache 来缓存 Bitmap 对象，以避免频繁地创建和销毁 Bitmap 对象。LruCache 是一个内存缓存类，可以在内存达到一定限制时自动删除最近最少使用的对象。例如，以下代码演示了如何使用 LruCache 来缓存 Bitmap 对象：
```
private LruCache<String, Bitmap> mBitmapCache;

public void initBitmapCache() {
    int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
    int cacheSize = maxMemory / 8;

    mBitmapCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap value) {
            return value.getByteCount() / 1024;
        }
    };
}

public void addBitmapToCache(String key, Bitmap bitmap) {
    if (getBitmapFromCache(key) == null) {
        mBitmapCache.put(key, bitmap);
    }
}

public Bitmap getBitmapFromCache(String key) {
    return mBitmapCache.get(key);
}
```

5. 合理释放 Bitmap 对象
当 Bitmap 对象不再使用时，需要手动调用 recycle() 方法来释放内存。例如，以下代码演示了如何在 ImageView 中加载 Bitmap 并在不再需要时释放 Bitmap：
```
ImageView imageView = findViewById(R.id.image_view);
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.image);
imageView.setImageBitmap(bitmap);

// 释放 Bitmap 对象
imageView.setImageBitmap(null);
bitmap.recycle();
```

### 4.2 大图监控
可以Plugin、Transform、ASM技术来在ImageView的方法进行插桩技术。
1. 创建一个Gradle插件，使用Transform技术修改字节码。
2. 使用ASM技术在ImageView的setImageDrawable()方法中插入代码，用于监控图片的宽高。
具体代码不表。
## 5. 内存常见优化
最后在来总结下，内存优化的有哪些常用方案：
1. 使用 SparseArray 和 ArrayMap 替代 HashMap
HashMap 存储大量的对象时会消耗很大的内存，尤其是在处理大量数据时。在 Android 开发中，我们可以使用 SparseArray 和 ArrayMap 来替代 HashMap。SparseArray 是 Android 提供的一个优化版的 Map，专门用来处理键为 int 类型的情况；而 ArrayMap 则是优化版的 Map，专门用来处理小数据集合的情况，它比 HashMap 更加高效。

2. 使用 Bitmap 配置的 ARGB_8888
Bitmap 是 Android 开发中经常使用的对象，它占用大量的内存。在使用 Bitmap 时，我们可以通过设置 Bitmap 的 Config 来减少内存消耗。ARGB_8888 是一种高质量的 Bitmap 配置，虽然会占用更多的内存，但是可以保证图片的清晰度。

3. 使用 BitmapFactory.Options 来压缩图片
在 Android 应用中，我们经常需要加载大量的图片，而这些图片的分辨率往往很高，导致内存消耗过大。为了减少内存消耗，我们可以使用 BitmapFactory.Options 来对图片进行压缩。可以通过设置 BitmapFactory.Options 中的 inSampleSize 属性来控制压缩比例。

4. 使用 LruCache 来缓存对象
LruCache 是 Android 提供的一种缓存对象的方式，它可以帮助我们减少内存消耗。LruCache 可以按照最近最少使用的原则来缓存对象，并且可以根据缓存对象的大小来自动调整缓存容量。

5. 及时释放资源
在 Android 开发中，我们需要及时释放无用的资源，以避免内存泄漏和内存溢出。例如，关闭 Cursor 对象、释放 Bitmap 对象、及时取消异步任务等。

6. 使用工具检查内存泄漏问题
使用内存分析工具，可以帮助我们检查内存问题，包括内存泄漏和内存溢出。我们可以使用这些工具来找出应用中的内存问题，并及时进行优化。

7. 优化布局和控件
在布局中，我们可以使用 FrameLayout 代替 RelativeLayout，因为 RelativeLayout 对内存消耗较大。在使用控件时，我们可以避免使用过多的控件和嵌套控件，尽量使用简单的布局方式和控件。




