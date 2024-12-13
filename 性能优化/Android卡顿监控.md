## 实现思路
卡顿监控主要监控：`慢方法的监控、ANR的监控、掉帧的监控`。其实现方案主要有三种：
- 1. Looper的Printer在消息执行前后的打印，计算出消息执行时间。
- 2. 利用Choreographer向其注册CALL_BACK, 监听Vsync的开始从而得到上一帧的执行时间。
- 3. 利用插桩的方法计算每个方法的执行时间。

比如对`慢方法、ANR`的监控，则是对主线程的Looper的消息的监听，对`掉帧`的监听则是对Choreographer注册CALL_BACK。下面则分段实现卡顿的监控。

## Looper的监控
利用Looper的Printer监听消息的开始和结束，如果对Handler源码这块不熟悉的建议先看看Handler的源码。
这里要注意的一点是，在设置Printer时即不能覆盖原本的Printer又要检查本次的Pritner是否被后续的Printer覆盖。
下面的`LooperMonitor`工具类就是实现Printer的兼容和检查，同时把事件通过Listener分发出去
```
object LooperMonitor {

    /**
     * 主线程的Looper
     */
    private var defaultMainLooper = Looper.getMainLooper()
    private var listeners: MutableList<LooperDispatchListener?> = mutableListOf()
    private var isReflectLoggingError = false
    private var printer: LooperPrinter? = null
    private var isStarted = false
    private val TAG = "LooperMonitor"
    private var lastCheckPrinterTime: Long = 0
    private const val CHECK_TIME = 60 * 1000L

    /**
     * 通过监听去分发的Looper的消息的开始和结束
     * 注册监听，如果有监听则自动给Looper设置Printer
     */
    fun addDispatchListener(listener: LooperDispatchListener?) {
        listener ?: return
        if (!listeners.contains(listener)) {
            listeners.add(listener)
        }
        // 如果监听>0 ,则调用 start() 方法
        if (listeners.size > 0 && !isStarted) {
            start()
        }
    }

    /**
     * 移除监听，没有监听自动结束对移除Looper的Printer
     */
    fun removeDispatchListener(listener: LooperDispatchListener?) {
        listener ?: return
        if (listeners.contains(listener)) {
            listeners.remove(listener)
        }
        // 没有监听自动结束对移除Looper的Printer
        if (listeners.size <= 0 && isStarted) {
            stop()
        }
    }

    private fun start() {
        isStarted = true
        // 设置Looper的Printer
        resetPrinter()
        // 增加IdleHandler,避免Printer被覆盖
        addIdleHandler()
    }

    private fun stop() {
        isStarted = false
        if (printer != null) {
            // 移除监听
            synchronized(listeners) { listeners.clear() }
            // Printer设置成原来的Printer
            defaultMainLooper.setMessageLogging(printer?.origin)
            removeIdleHandler()
            defaultMainLooper = null
            printer = null
        }
    }

    /**
     * 检测Looper原本是否有Printer，如果有不要将原本的Printer覆盖,然后设置本次的Printer
     */
    private fun resetPrinter() {
        var originPrinter: Printer? = null
        try {
            if (!isReflectLoggingError) {
                // 通过反射，拿到Lopper的mLogging变量
                originPrinter =
                    ReflectUtils.get(defaultMainLooper::class.java, "mLogging", defaultMainLooper)
                // 检测Printer是否被后续的Printer覆盖
                if (originPrinter == printer && printer != null) {
                    return
                }
                if (originPrinter != null && printer != null) {
                    if (originPrinter.javaClass.name == printer?.javaClass?.name) {
                        return
                    }
                }
            }
        } catch (e: Exception) {
            isReflectLoggingError = true
            e.printStackTrace()
        }

        // 设置本次的Printer，并适配原本的Printer
        printer = LooperPrinter(originPrinter)
        defaultMainLooper.setMessageLogging(printer)
    }

    /**
     * 在MessageQueue空闲的时候,执行Looper的Printer的检查，避免被后续的Printer覆盖
     */
    private fun addIdleHandler() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            defaultMainLooper.queue.addIdleHandler(idleHandler)
        } else {
            try {
                val queue: MessageQueue? =
                    ReflectUtils.get(defaultMainLooper::class.java, "mQueue", defaultMainLooper)
                queue?.addIdleHandler(idleHandler)
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }
    }

    private fun removeIdleHandler() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            defaultMainLooper.queue.removeIdleHandler(idleHandler)
        } else {
            try {
                val queue: MessageQueue? =
                    ReflectUtils.get(defaultMainLooper::class.java, "mQueue", defaultMainLooper)
                queue?.removeIdleHandler(idleHandler)
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }
    }

    /**
     * 每隔60s，检测一下本次的Printer是否被别的Printer覆盖
     */
    private val idleHandler = MessageQueue.IdleHandler {
        if (SystemClock.uptimeMillis() - lastCheckPrinterTime > CHECK_TIME) {
            resetPrinter()
            lastCheckPrinterTime = SystemClock.uptimeMillis()
        }
        true
    }


    /**
     * 新的Printer，要适配原本的已有的Printer
     */
    class LooperPrinter(var origin: Printer?) : Printer {
        private var isValid = false

        override fun println(x: String?) {
            // 给原本的Printer调用
            if (origin != null) {
                origin?.println(x)
                if (origin == this) {
                    throw RuntimeException("$TAG origin == this")
                }
            }

            // 分发给Listener
            isValid = x?.getOrNull(0) == '>' || x?.getOrNull(0) == '<'
            if (isValid) {
                dispatch(x?.getOrNull(0) == '>', x ?: "")
            }

        }

    }

    /**
     * 分发给Listener
     */
    private fun dispatch(isBegin: Boolean, log: String) {
        synchronized(listeners) {
            listeners.forEach {
                it ?: return
                if (isBegin) {
                    if (!it.isHasDispatchStart) {
                        it.onDispatchStart(log)
                    }
                } else {
                    if (it.isHasDispatchStart) {
                        it.onDispatchEnd(log)
                    }
                }
            }
        }
    }


    /**
     * 分发的接口
     */
    abstract class LooperDispatchListener {
        var isHasDispatchStart = false

        abstract fun dispatchStart()

        abstract fun dispatchEnd()

        fun onDispatchStart(x: String?) {
            isHasDispatchStart = true
            dispatchStart()
        }

        fun onDispatchEnd(x: String?) {
            isHasDispatchStart = false
            dispatchEnd()
        }
    }

}
```
调用`LooperMonitor. addDispatchListener()`方法就可以得到

## 主线程的监控
```
/**
 * 主线程的监控
 * 即监听Looper中的printer，又监听Choreographer中的input animation traversal
 * Looper和Choreographer结合计算主线程的：方法耗时和帧耗时
 */
object UIThreadMonitor {

    @Volatile
    var isAlive = false  // 是否已经开启监测

    // 注册UI线程监控的订阅者集合
    private val observers: HashSet<UIThreadMonitorObserver?> = HashSet<UIThreadMonitorObserver?>()

    private val TAG = "UIThreadMonitor"

    /**
     * Choreographer
     */
    // 注册的三种类型
    private const val CALLBACK_INPUT = 0
    private const val CALLBACK_ANIMATION = 1
    private const val CALLBACK_TRAVERSAL = 2
    private val DO_QUEUE_END_ERROR: Long = -100
    private val CALLBACK_LAST: Int = CALLBACK_TRAVERSAL

    private var queueStatus = IntArray(CALLBACK_LAST + 1)  // 记录类型的执行状态
    private var callbackExist = BooleanArray(CALLBACK_LAST + 1) // 记录此类型是否已经添加
    private var queueCost = LongArray(CALLBACK_LAST + 1) // 记录各个类型花费的时间
    private const val DO_QUEUE_DEFAULT = 0
    private const val DO_QUEUE_BEGIN = 1
    private const val DO_QUEUE_END = 2

    // 主线程的Choreographer实例
    private var choreographer: Choreographer? = null

    // 利用反射获取Choreographer的变量和方法
    private var callbackQueueLock: Any? = null
    private var callbackQueues: Array<Any>? = null
    private var addTraversalQueue: Method? = null  // Traversal类型的注册方法
    private var addInputQueue: Method? = null // Input类型的注册方法
    private var addAnimationQueue: Method? = null // Animation类型的注册方法
    private var vsyncReceiver: Any? = null
    var frameIntervalNanos: Long = 16666666

    private var isVsyncFrame = false // 是否为Vsync信号回调

    /**
     * Looper
     */
    // 记录Looper的开始和结束的时间
    private var dispatchTimeMs = LongArray(4)

    // 记录 Looper分发的消息开始的时间
    @Volatile
    private var token = 0L

    /***
     * 开始监测主线程
     */
    fun startMonitor() {
        if (Thread.currentThread() != Looper.getMainLooper().thread) {
            throw AssertionError("must be init in main thread!")
        }

        // 添加主线程的Looper的分发
        LooperMonitor.addDispatchListener(looperDispatch)

        // 初始化并向Choreographer注册Callback
        initAddCallbackByChoreographer()

    }

    /***
     * 结束监测主线程
     */
    fun stopMonitor() {
        if (isAlive) {
            isAlive = false
            observers.clear()
            LooperMonitor.removeDispatchListener(looperDispatch)

        }
    }

    /**
     * 增加订阅者，会自动开始监控
     */
    fun addObserver(observer: UIThreadMonitorObserver) {
        if (!isAlive) {
            startMonitor()
        }
        synchronized(observers) {
            observers.add(observer)
        }
    }

    /**
     * 删除订阅者，会自动结束监控
     */
    fun removeObserver(observer: UIThreadMonitorObserver) {
        synchronized(observers) {
            observers.remove(observer)
            if (observers.size <= 0) {
                stopMonitor()
            }
        }
    }

    /**
     * Looper分发的具体处理逻辑
     */
    private val looperDispatch = object : LooperMonitor.LooperDispatchListener() {
        override fun dispatchStart() {
            this@UIThreadMonitor.dispatchBegin()
        }

        override fun dispatchEnd() {
            this@UIThreadMonitor.dispatchEnd()
        }
    }

    /**
     * 初始化并向Choreographer注册Callback
     */
    private fun initAddCallbackByChoreographer() {
        // 反射获取Choreographer的变量和方法
        reflectByChoreographer()

        if (!isAlive) {
            this.isAlive = true
            synchronized(this) {
                // 初始化三个类型的数组
                callbackExist = BooleanArray(CALLBACK_LAST + 1) // 此类型是否已经注册
                queueStatus = IntArray(CALLBACK_LAST + 1) // 此类型执行的状态
                queueCost = LongArray(CALLBACK_LAST + 1) // 此类型的花费时间
                // 向Choreographer添加CALLBACK_INPUT类型的Callback
                addFrameCallback(CALLBACK_INPUT, runnable, true)
            }
        }
    }

    /**
     * 反射获取Choreographer的变量和方法
     */
    private fun reflectByChoreographer() {
        choreographer = Choreographer.getInstance()
        frameIntervalNanos =
            ReflectUtils.reflectObject(choreographer, "mFrameIntervalNanos", frameIntervalNanos)
        callbackQueueLock = ReflectUtils.reflectObject(choreographer, "mLock", Any())
        callbackQueues = ReflectUtils.reflectObject(choreographer, "mCallbackQueues", null)
        callbackQueues?.let {
            addInputQueue = ReflectUtils.reflectMethod(
                it[CALLBACK_INPUT], "addCallbackLocked",
                Long::class.javaPrimitiveType,
                Any::class.java,
                Any::class.java
            )
            addAnimationQueue = ReflectUtils.reflectMethod(
                it[CALLBACK_ANIMATION], "addCallbackLocked",
                Long::class.javaPrimitiveType,
                Any::class.java,
                Any::class.java
            )
            addTraversalQueue = ReflectUtils.reflectMethod(
                it[CALLBACK_TRAVERSAL], "addCallbackLocked",
                Long::class.javaPrimitiveType,
                Any::class.java,
                Any::class.java
            )
        }
        vsyncReceiver = ReflectUtils.reflectObject(choreographer, "mDisplayEventReceiver", null)
    }


    /**
     * 向Choreographer添加CALLBACK
     */
    @Synchronized
    private fun addFrameCallback(type: Int, callback: Runnable, isAddHeader: Boolean) {
        // callbackExist判断是否已经加入Runnable
        if (callbackExist[type]) {
            return
        }
        // 判断是否结束
        if (!isAlive && type == CALLBACK_INPUT) {
            return
        }
        // 利用反射，把对应的CALLBACK添加到链表中
        try {
            callbackQueues?.let {
                synchronized(it) {
                    var method: Method? = null
                    when (type) {
                        CALLBACK_INPUT -> method = addInputQueue
                        CALLBACK_ANIMATION -> method = addAnimationQueue
                        CALLBACK_TRAVERSAL -> method = addTraversalQueue
                    }
                    if (null != method) {
                        method.invoke(
                            it[type],
                            if (!isAddHeader) SystemClock.uptimeMillis() else -1,
                            callback,
                            null
                        )
                        callbackExist[type] = true
                    }
                }
            }
        } catch (e: Exception) {
            Log.e(TAG, e.toString())
        }
    }

    /**
     * Choreographer中执行Input CALLBACK的回调
     */
    private val runnable = object : Runnable {
        override fun run() {
            val start = System.nanoTime()
            try {
                //isBelongFrame标志位为true,标志已经开始纳入统计
                doFrameBegin()

                // 设置CALLBACK_INPUT类型的queueStatus和queueCost
                doQueueBegin(CALLBACK_INPUT)

                // 注册CALLBACK_ANIMATION
                addFrameCallback(
                    CALLBACK_ANIMATION,
                    { // CALLBACK_ANIMATION执行表示，它前面的CALLBACK_INPUT已经执行完了
                        // 结束CALLBACK_INPUT统计，更新queueStatus，queueCost，callbackExist的值
                        doQueueEnd(CALLBACK_INPUT)
                        // 开始统计CALLBACK_ANIMATION
                        doQueueBegin(CALLBACK_ANIMATION)
                    },
                    true
                )

                // 注册CALLBACK_TRAVERSAL
                addFrameCallback(
                    CALLBACK_TRAVERSAL,
                    { // 它执行，表示前面的CALLBACK_ANIMATION已经执行完
                        doQueueEnd(CALLBACK_ANIMATION)
                        // 开始激励CALLBACK_TRAVERSAL
                        doQueueBegin(CALLBACK_TRAVERSAL)
                    },
                    true
                )

            } catch (e: Exception) {
                e.printStackTrace()
            }
        }
    }

    /**
     * input CallBack的回调开始
     * Frame的开始
     */
    private fun doFrameBegin() {
        this.isVsyncFrame = true
    }

    /**
     * TRAVERSAL CallBack的结束，用Looper的分发的结束做判断
     * Frame的结束
     */
    private fun doFrameEnd() {
        doQueueEnd(CALLBACK_TRAVERSAL)
        for (i in queueStatus) {
            if (i != DO_QUEUE_END) {
                queueCost[i] = DO_QUEUE_END_ERROR
                throw RuntimeException("UIThreadMonitor happens type[$i] != DO_QUEUE_END")
            }
        }
        queueStatus = IntArray(CALLBACK_LAST + 1)
        addFrameCallback(CALLBACK_INPUT, runnable, true)
    }

    /**
     * 每种类型的开始
     */
    private fun doQueueBegin(type: Int) {
        queueStatus[type] = DO_QUEUE_BEGIN
        queueCost[type] = System.nanoTime()
    }

    /**
     * 每种类型的结束
     */
    private fun doQueueEnd(type: Int) {
        queueStatus[type] = DO_QUEUE_END
        queueCost[type] = System.nanoTime() - queueCost[type]
        synchronized(this) {
            callbackExist[type] = false
        }
    }

    /**
     * Looper分发的Message开始
     */
    private fun dispatchBegin() {
        // 将开始时间分发给订阅者
        token = System.nanoTime()
        dispatchTimeMs[0] = token
        dispatchTimeMs[2] = SystemClock.currentThreadTimeMillis()

        synchronized(observers) {
            observers.forEach {
                it ?: return
                if (!it.isDispatchBegin) {
                    it.dispatchBegin(dispatchTimeMs[0], dispatchTimeMs[2], token)
                }
            }
        }
    }

    /**
     * Looper分发的Message结束
     */
    private fun dispatchEnd() {
        // message开始的时间
        val startNS = token
        var intendedFrameTimeNs = startNS
        if (isVsyncFrame) {
            // 如果是isVsyncFrame则调用doFrameEnd()表示最后一个CallBack执行已经结束
            doFrameEnd()
            // 开始时间用Choreographer记录的开始时间
            intendedFrameTimeNs = getIntendedFrameTimeNs(startNS)
        }

        //  message结束的时间
        val endNs = System.nanoTime()
        // 分发给订阅者
        synchronized(observers) {
            observers.forEach {
                it ?: return
                if (it.isDispatchBegin) {
                    it.doFrame(
                        ApmUtil.getTopActivityName(),
                        startNS,
                        endNs,
                        isVsyncFrame,
                        intendedFrameTimeNs,
                        queueCost[CALLBACK_INPUT],
                        queueCost[CALLBACK_ANIMATION],
                        queueCost[CALLBACK_TRAVERSAL]
                    )
                }
            }

            // 记录结束的时间
            dispatchTimeMs[3] = SystemClock.currentThreadTimeMillis()
            dispatchTimeMs[1] = System.nanoTime()

            // 分发给订阅者
            synchronized(observers) {
                observers.forEach {
                    it ?: return
                    if (it.isDispatchBegin) {
                        it.dispatchEnd(
                            dispatchTimeMs[0],
                            dispatchTimeMs[2],
                            dispatchTimeMs[1],
                            dispatchTimeMs[3],
                            token, isVsyncFrame
                        )
                    }
                }
            }

            isVsyncFrame = false
        }
    }

    /**
     * 反射获取Choreographer记录的Frame的开始时间：mTimestampNanos
     */
    private fun getIntendedFrameTimeNs(defaultValue: Long): Long {
        try {
            return ReflectUtils.reflectObject<Long>(
                vsyncReceiver,
                "mTimestampNanos",
                defaultValue
            )
        } catch (e: Exception) {
            e.printStackTrace()
            Log.e(TAG, e.toString())
        }
        return defaultValue
    }

    /**
     * 主线程监控的订阅者
     */
    abstract class UIThreadMonitorObserver {
        var isDispatchBegin = false

        open fun dispatchBegin(beginNs: Long, cpuBeginNs: Long, token: Long) {
            isDispatchBegin = true
        }

        open fun doFrame(
            focusedActivity: String?,
            startNs: Long,
            endNs: Long,
            isVsyncFrame: Boolean,
            intendedFrameTimeNs: Long,
            inputCostNs: Long,
            animationCostNs: Long,
            traversalCostNs: Long
        ) {
        }

        open fun dispatchEnd(
            beginNs: Long,
            cpuBeginMs: Long,
            endNs: Long,
            cpuEndMs: Long,
            token: Long,
            isVsyncFrame: Boolean
        ) {
            isDispatchBegin = false
        }
    }

}
```

## 卡顿的监控
```
/**
 * 卡顿监控，包括：ANR、慢函数、掉帧
 */
class BlockPlugin : Plugin {

    private var mainHandler: Handler? = null
    private var application: Application? = null
    private var apmWindowManager: ApmWindowManager? = null

    /**
     * 慢方法,耗时超过500ms
     */
    var SLOW_TIME_BLOCK = 500  // 定义慢方法的时间
    private var slowMethodHandlerThread: HandlerThread? = null
    private var slowMethodStackCollectHandler: Handler? = null
    private var slowMethodBlockStackTraceMap = HashMap<String, BlockStackTraceInfo>()
    private val SLOW_COLLECT_TIME: Long = 500L // 收集堆栈的时间

    /**
     * ANR,耗时超过5S
     */
    var ANR_TIME_BLOCK = 5_000  // 定义ANR的时间
    private var anrHandlerThread: HandlerThread? = null
    private var anrStackCollectHandler: Handler? = null
    private var anrBlockStackTraceMap = HashMap<String, BlockStackTraceInfo>()
    private val ANR_COLLECT_TIME: Long = 1000L // 收集堆栈的时间

    /**
     * 帧：掉帧、帧率
     */
    var dropFrameListenerThreshold = 3  // 掉帧上报阀值
    private var frameHandlerThread: HandlerThread? = null
    private var frameStackCollectHandler: Handler? = null
    private var frameBlockStackTraceMap = HashMap<String, BlockStackTraceInfo>()
    private var frame_collect_time: Long = 16L // 收集堆栈的时间
    private var collectCount = 0

    // 帧率
    private var sumFrameCost: Long = 0  // 帧数花费的总时间
    private var lastCost = LongArray(1)  // 上一次使用的时间
    private var sumFrames: Long = 0   // 总共的帧数
    private var lastFrames = LongArray(1)   // 上一次掉帧的数量
    private var maxFps = 0f  // 1s中最大的帧率

    /**
     * 开始
     */
    override fun start(application: Application?) {
        val sdkInt = Build.VERSION.SDK_INT
        if (sdkInt < Build.VERSION_CODES.JELLY_BEAN) {
            return
        }
        this.application = application
        // 在主线程中执行Runnable开始订阅主线程监控
        mainHandler = Handler(Looper.getMainLooper())
        mainHandler?.post(runnable)
    }

    /**
     * 结束
     */
    override fun stop() {
        mainHandler?.post {
            UIThreadMonitor.removeObserver(looperListener)
        }
        mainHandler?.removeCallbacksAndMessages(null)
        apmWindowManager?.dismiss()
    }

    /**
     * 开始的Runnable
     */
    private val runnable = Runnable {
        try {

            // 初始化子线程handler，用于收集堆栈
            initStackHandler()

            // 开始主线程监控
            UIThreadMonitor.addObserver(looperListener)
            if (!UIThreadMonitor.isAlive) {
                UIThreadMonitor.startMonitor()
            }
            frame_collect_time = UIThreadMonitor.frameIntervalNanos / Constants.TIME_MILLIS_TO_NANO
            maxFps = (1000f / frame_collect_time.toFloat()).roundToInt().toFloat()

            // 初始化显示信息的window
            apmWindowManager = ApmWindowManager().apply {
                init(application)
                show()
            }

        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

    /**
     * 初始化收集：慢函数、ANR、掉帧的堆栈的子线程Handler
     */
    private fun initStackHandler() {
        slowMethodHandlerThread = HandlerThread("SlowMethodMonitor")
        slowMethodHandlerThread?.start()
        slowMethodStackCollectHandler = Handler(slowMethodHandlerThread!!.looper)

        anrHandlerThread = HandlerThread("ANRMonitor")
        anrHandlerThread?.start()
        anrStackCollectHandler = Handler(anrHandlerThread!!.looper)

        frameHandlerThread = HandlerThread("FrameMonitor")
        frameHandlerThread?.start()
        frameStackCollectHandler = Handler(frameHandlerThread!!.looper)
    }


    /**
     * 主线程监控的订阅者
     */
    private val looperListener = object : UIThreadMonitor.UIThreadMonitorObserver() {
        override fun dispatchBegin(beginNs: Long, cpuBeginNs: Long, token: Long) {
            super.dispatchBegin(beginNs, cpuBeginNs, token)
            //装炸弹 ，子线程延迟去收集堆栈
            slowMethodStackCollectHandler?.postDelayed(
                slowMethodStackCollectTask,
                SLOW_COLLECT_TIME
            )
            anrStackCollectHandler?.postDelayed(anrStackCollectTask, ANR_COLLECT_TIME)
            frameStackCollectHandler?.postDelayed(frameStackCollectTask, frame_collect_time)
        }

        override fun dispatchEnd(
            beginNs: Long,
            cpuBeginMs: Long,
            endNs: Long,
            cpuEndMs: Long,
            token: Long,
            isVsyncFrame: Boolean
        ) {
            super.dispatchEnd(beginNs, cpuBeginMs, endNs, cpuEndMs, token, isVsyncFrame)
            // 拆炸弹
            slowMethodStackCollectHandler?.removeCallbacks(slowMethodStackCollectTask)
            anrStackCollectHandler?.removeCallbacks(anrStackCollectTask)

            val costTime = (endNs - beginNs) / Constants.TIME_MILLIS_TO_NANO

            // 慢方法
            if (costTime >= SLOW_TIME_BLOCK && slowMethodBlockStackTraceMap.isNotEmpty()) {
                val traceList = slowMethodBlockStackTraceMap.values.toList()
                traceList.forEach {
                    Log.e(SLOW_METHOD, "慢函数 -> 耗时=${costTime}ms \n ${it.stackTrace}")
                }
                // 更新显示到window中
                apmWindowManager?.insertSlowMethodCount()
            }
            slowMethodBlockStackTraceMap.clear()

            // ANR
            if (costTime >= ANR_TIME_BLOCK && anrBlockStackTraceMap.isNotEmpty()) {
                val traceList = anrBlockStackTraceMap.values.toList()
                traceList.forEach {
                    Log.e(ANR, "ANR函数-> 耗时=${costTime}ms \n ${it.stackTrace}")
                }
                // 更新显示到window中
                apmWindowManager?.insertANRCount()
            }
            anrBlockStackTraceMap.clear()

        }

        override fun doFrame(
            focusedActivity: String?,
            startNs: Long,
            endNs: Long,
            isVsyncFrame: Boolean,
            intendedFrameTimeNs: Long,
            inputCostNs: Long,
            animationCostNs: Long,
            traversalCostNs: Long
        ) {
            // 拆炸弹
            frameStackCollectHandler?.removeCallbacks(frameStackCollectTask)
            application ?: return
            // App在前台
            if (ApmUtil.isAppRunningInForground(application!!, application!!.packageName)) {
                // 执行时间
                val jitter = endNs - intendedFrameTimeNs
                // 跳过的帧数
                val dropFrame = (jitter / UIThreadMonitor.frameIntervalNanos)

                if (apmWindowManager?.isShowing == false) {
                    apmWindowManager?.show()
                }

                // 跳帧
                if (dropFrame > dropFrameListenerThreshold && frameBlockStackTraceMap.isNotEmpty()) {
                    val traceList = frameBlockStackTraceMap.values.toMutableList()
                    val string = StringBuffer()
                    traceList.forEach {
                        string.append(it.stackTrace.toString())
                    }
                    Log.e(DROP_FRAME, "掉帧函数 = ${dropFrame}次 \n ${string}")
                    // 更新显示到window中
                    apmWindowManager?.dropFrameCount(dropFrame.toInt())
                }
                frameBlockStackTraceMap.clear()

                // 帧率
                sumFrameCost += ((dropFrame + 1) * frame_collect_time)// 总时间相加
                sumFrames += 1  // 总帧数+1
                val duration = (sumFrameCost - lastCost[0]).toFloat()  // 这次帧数的时间
                val collectFrame = sumFrames - lastFrames[0]  // 总帧数 - 上次的帧数
                if (duration >= 200) {
                    val fps = maxFps.coerceAtMost(1000f * collectFrame / duration)
                    // 更新显示到window中
                    apmWindowManager?.updateFrameRate(fps.toInt())
                    lastCost[0] = sumFrameCost  // 上一次的时间 等于总时间
                    lastFrames[0] = sumFrames   // 上一次的总帧率
                }

            } else {
                if (apmWindowManager?.isShowing == true) {
                    apmWindowManager?.dismiss()
                }
            }
        }
    }


    /**
     * 慢函数定时收集堆栈任务
     */
    private val slowMethodStackCollectTask = object : Runnable {
        override fun run() {
            val info =
                BlockStackTraceInfo(ThreadManager.getJavaStackByName(Looper.getMainLooper().thread.name))
            val key = info.getMapKey()
            val value = slowMethodBlockStackTraceMap[key]
            if (value == null) {
                slowMethodBlockStackTraceMap[key] = info
            }
            slowMethodStackCollectHandler?.postDelayed(this, SLOW_COLLECT_TIME)
        }
    }

    /**
     * ANR定时收集堆栈任务
     */
    private val anrStackCollectTask = object : Runnable {
        override fun run() {
            val info =
                BlockStackTraceInfo(ThreadManager.getJavaStackByName(Looper.getMainLooper().thread.name))
            val key = info.getMapKey()
            val value = anrBlockStackTraceMap[key]
            if (value == null) {
                anrBlockStackTraceMap[key] = info
            }
            anrStackCollectHandler?.postDelayed(this, SLOW_COLLECT_TIME)
        }
    }

    /**
     * 掉帧定时收集堆栈任务
     */
    private val frameStackCollectTask = object : Runnable {
        override fun run() {
            collectCount++
            if (collectCount > 1) {
                val info =
                    BlockStackTraceInfo(ThreadManager.getJavaStackByName(Looper.getMainLooper().thread.name))
                val key = info.getMapKey()
                val value = frameBlockStackTraceMap[key]
                if (value == null) {
                    frameBlockStackTraceMap[key] = info
                } else {
                    value.collectCount++
                }
            }
            frameStackCollectHandler?.postDelayed(this, frame_collect_time)
        }
    }

    companion object {
        const val SLOW_METHOD = "SlowMethod"  // 慢方法堆栈的Log
        const val ANR = "ANR"  // ANR堆栈的Log
        const val DROP_FRAME = "DropFrame"   // 掉帧堆栈的Log
    }

}
```