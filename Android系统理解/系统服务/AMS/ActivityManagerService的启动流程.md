> 本篇文章基于Android 12分析

相关源码：
  ```
/frameworks/base/services/java/com/android/server/SystemServer.java
  /frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  /frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
  /frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
 /frameworks/base/core/java/android/app/ActivityThread.java
  ```
AMS对三大组件(service、broadcast、contentProvider)进行管理和调度，在Android 10 之后增加ATMS对Activity进行单独的管理和调度。

## AMS启动之前
系统启动Zygote进程后第一个fork就是StsyemServer进程，通过执行`SystemServer.run()`方法来启动SystemServer。关于SystemServer详细启动过程看:[Android系统启动-SystemServer进程](https://www.jianshu.com/p/987a43c1a708)
  ```
/frameworks/base/services/java/com/android/server/SystemServer.java
private void run() {
    try {
        Looper.prepareMainLooper();
        // Initialize the system context.
        createSystemContext();
        // Start services.
        try {
            startBootstrapServices(t);
            startCoreServices(t);
            startOtherServices(t);
        }
        // Loop forever.
        Looper.loop();
    }

    private void createSystemContext () {
        ActivityThread activityThread = ActivityThread.systemMain();
        // 初始化系统context，并设置主题
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
        // 初始化SystemUi context，并设置主题
        final Context systemUiContext = activityThread.getSystemUiContext();
        systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
  ```
 `createSystemContext()`创建 `ActivityThread`:为当前进程(SystemServer)的主线程，和两个context：`一个系统Context，一个SystemUi Context`。

下面看看**ActivityThread.systemMain()**如何执行：
  ```
/frameworks/base/core/java/android/app/ActivityThread.java
 public final class ActivityThread extends ClientTransactionHandler{

public static ActivityThread systemMain () {
    ThreadedRenderer.initForSystemProcess();
    // 创建一个ActivityThread实例
    ActivityThread thread = new ActivityThread();
    // 调用ActivityThread的attach方法
    thread.attach(true, 0);
    return thread;
}

ActivityThread() {
    // 获取ResourcesManager单例对象
    mResourcesManager = ResourcesManager.getInstance();
}

// system：是否为systemServer进程调用，这里为true
private void attach ( boolean system, long startSeq){
    // 把当前的实例赋值到sCurrentActivityThread变量
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ......
    } else {
       ......
        try {
            // 创建Instrumentation实例
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
            // 创建Context
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            // 创建Application
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            // 调用Application.onCreate()
            mInitialApplication.onCreate();
        }
        ......
    }
    ......
}

}
  ```
`systemMain ()`：
1. 创建了一个`ActivityThread`类，
2. 调用其`attach`方法，创建几个重要变量Instrumentation、ContextImpl、Application

## AMS启动流程
AMS的启动流程在SystemServer的startBootstrapServices(t)方法,下面看看跟ActivityManagerService相关的代码：
  ```
/frameworks/base/services/java/com/android/server/SystemServer.java
private void startBootstrapServices (@NonNull TimingsTraceAndSlog t){
    ......
    //part 0
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    //part 1
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    // part 2
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    // part 3
    mActivityManagerService.setInstaller(installer);
    // part 4
    mActivityManagerService.initPowerManagement();
    // part 5
    mActivityManagerService.setSystemProcess();
}
  ```

#### part 0: mSystemServiceManager.startService( ActivityTaskManagerService.Lifecycle.class).getService();
先看`SystemServiceManager.startService`方法：
  ```
/frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
public <T extends SystemService > T startService(Class < T > serviceClass) {
    try {
        final String name = serviceClass.getName();
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            // 通过构造函数实例化Class对象
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        }......
        startService(service);
        return service;
    }
}
public void startService ( @NonNull final SystemService service){
    // Register it.
    mServices.add(service);
    // Start it.
    long time = SystemClock.elapsedRealtime();
    try {
        // 调用对象的onStart()方法
        service.onStart();
    } catch (RuntimeException ex) {
       ......
    }
    warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
}

  ```
通过反射构造函数实例化Class对象，并调用对象的`onStart()`方法。
ATMS是Android 10 引入的，它把之前AMS对activity组件的管理和调度转移到ATMS类中。
  ```
/frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {

    public ActivityTaskManagerService(Context context) {
        mContext = context;
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();
        mInternal = new LocalService();
        ......
    }

    private void start() {
        // 这是一个hashmap，将一个LocalService存入
        LocalServices.addService(ActivityTaskManagerInternal.class, mInternal);
    }

    public static final class Lifecycle extends SystemService {
        private final ActivityTaskManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            // 实例化 ATMS
            mService = new ActivityTaskManagerService(context);
        }

        @Override
        public void onStart() {
            // 将name = activity_task的ATMS，向ServiceManager注册
            publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
            mService.start();
        }

        public ActivityTaskManagerService getService() {
            // 返回 ATMS
            return mService;
        }
    }
}
  ```
实际上就是实例化`ActivityTaskManagerService`，并调用其`start()`方法。

#### part 1：ActivityManagerService.Lifecycle.startService()
  ```
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;
    private static ActivityTaskManagerService sAtm;

    // 实例化ActivityManagerService对象,并把ATMS传入
    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context, sAtm);
    }
     
    public static ActivityManagerService startService(
            SystemServiceManager ssm, ActivityTaskManagerService atm) {
        // 保存ATMS对象
        sAtm = atm;
        // 调用SystemServiceManager.startService方法
        return ssm.startService(ActivityManagerService.Lifecycle.class).getServi
    }
    @Override
    public void onStart() {
        // 调用 ATM.start()方法
        mService.start();
    }
    public ActivityManagerService getService() {
        return mService;
    }
}
  ```
首先在AMS中保存了`ATMS实例`，然后调用`SystemServiceManager.startService`,在上一节中分析了通过反射创建对象，并调用onStart()方法，最终创建`ActivityManagerService`实例，并调用`ActivityManagerService.start()`方法：

##### new ActivityManagerService()
  ```
public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
        mInjector = new Injector(systemContext);
        // 系统上下文，这个systemContext是在SystemService进程创建的
        mContext = systemContext;

        mFactoryTest = FactoryTest.getMode();
        // 系统进程的主线程，是在SystemService时创建的ActivityThread
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

        // 创建HandlerThread子线程
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        // 处理 AMS 的Handler
        mHandler = new MainHandler(mHandlerThread.getLooper());
        // 对应UI线程
        mUiHandler = mInjector.getUiHandler(this);

        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
        mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

        mConstants = new ActivityManagerConstants(mContext, this, mHandler);
        final ActiveUids activeUids = new ActiveUids(this, true /* postChangesToAtm */);
        mPlatformCompat = (PlatformCompat) ServiceManager.getService(
                Context.PLATFORM_COMPAT_SERVICE);
        mProcessList = mInjector.getProcessList(this);
        mProcessList.init(this, activeUids, mPlatformCompat);
        mAppProfiler = new AppProfiler(this, BackgroundThread.getHandler().getLooper(),
                new LowMemDetector(this));
        mPhantomProcessList = new PhantomProcessList(this);
        mOomAdjuster = new OomAdjuster(this, mProcessList, activeUids);

        // Broadcast policy parameters
        final BroadcastConstants foreConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_FG_CONSTANTS);
        foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

        final BroadcastConstants backConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_BG_CONSTANTS);
        backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;

        final BroadcastConstants offloadConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
        offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
        // by default, no "slow" policy in this queue
        offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

        mEnableOffloadQueue = SystemProperties.getBoolean(
                "persist.device_config.activity_manager_native_boot.offload_queue_enabled", false);
        // 创建几种广播相关对象，前台广播、后台广播、offload广播
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", foreConstants, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", backConstants, true);
        mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload", offloadConstants, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        mBroadcastQueues[2] = mOffloadBroadcastQueue;

        // 创建ActiveServices对象，管理ServiceRecord
        mServices = new ActiveServices(this);
        // 创建ContentProviderHelper对象
        mCpHelper = new ContentProviderHelper(this, true);
        mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
        mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);
        mUidObserverController = new UidObserverController(mUiHandler);

        final File systemDir = SystemServiceManager.ensureSystemDir();

        // TODO: Move creation of battery stats service outside of activity manager service.
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir,
                BackgroundThread.get().getHandler());
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);
        mOomAdjProfiler.batteryPowerChanged(mOnBattery);

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);

        mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);

        mUserController = new UserController(this);

        mPendingIntentController = new PendingIntentController(
                mHandlerThread.getLooper(), mUserController, mConstants);

        mUseFifoUiScheduling = SystemProperties.getInt("sys.use_fifo_ui", 0) != 0;

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);

        // 保存ATMS对象，并调用initialize方法
        mActivityTaskManager = atm;
        mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());
        mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);

        mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);

        // 加入Watchdog的监控
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);

        // bind background threads to little cores
        // this is expected to fail inside of framework tests because apps can't touch cpusets directly
        // make sure we've already adjusted system_server's internal view of itself first
        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        try {
            Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
            Process.setThreadGroupAndCpuset(
                    mOomAdjuster.mCachedAppOptimizer.mCachedAppOptimizerThread.getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
        } catch (Exception e) {
            Slog.w(TAG, "Setting background thread cpuset failed");
        }

        // 实例化LocalService
        mInternal = new LocalService();
        mPendingStartActivityUids = new PendingStartActivityUids(mContext);
        mTraceErrorLogger = new TraceErrorLogger();
    }
  ```
在ActivityManagerService构造函数中：
- 首先创建处理消息机制的`Handler`
- 三大组件：`broad、service、contentProvider`相关变量的初始化。保存ATMS变量并调用其initialize方法。
- 监控内存、电池、权限、性能相关的对象或变量初始化

##### AMS的start()方法：
  ```
private void start() {
        // 移除所有进程组
        removeAllProcessGroups();
        // 注册电池、权限管理服务
        mBatteryStatsService.publish();
        mAppOpsService.publish();
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, mInternal);
        LocalManagerRegistry.addManager(ActivityManagerLocal.class,
                (ActivityManagerLocal) mInternal);
        mActivityTaskManager.onActivityManagerInternalAdded();
        mPendingIntentController.onActivityManagerInternalAdded();
        mAppProfiler.onActivityManagerInternalAdded();
    }
  ```
移除进程组，注册服务，增加LocalService

#### part 2:mActivityManagerService.setSystemServiceManager(mSystemServiceManager)
  ```
    public void setSystemServiceManager(SystemServiceManager mgr) {
        mSystemServiceManager = mgr;
    }
  ```
将SystemService进程的`SystemServiceManager`对象实例传给AMS。

#### part 3:mActivityManagerService.setInstaller(installer)
  ```
    public void setInstaller(Installer installer) {
        mInstaller = installer;
    }
  ```
将SystemService进程的`Installer`对象实例传给AMS。

#### part 4:mActivityManagerService.initPowerManagement()
  ```
    public void initPowerManagement() {
        mActivityTaskManager.onInitPowerManagement();
        mBatteryStatsService.initPowerManagement();
        mLocalPowerManager = LocalServices.getService(PowerManagerInternal.class);
    }
  ```
初始化电源管理的功能
#### part 5:mActivityManagerService.setSystemProcess()
  ```
    public void setSystemProcess() {
        try {
            // 注册 name = activity 的 AMS服务
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
            // 注册 name = procstats 的 ProcessStats进程状态服务
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            // 注册 name = meminfo 的 MemBinder内存信息服务
            ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH);
            // 注册 name = gfxinfo 的 GraphicsBinder图像信息服务
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            // 注册 name = dbinfo 的 DbBinder数据库信息服务
            ServiceManager.addService("dbinfo", new DbBinder(this));
            // 注册 name = cpuinfo 的 CpuBinder cpu信息服务
            mAppProfiler.setCpuInfoService();
            // 注册 name = permission 的 PermissionController权限信息服务
            ServiceManager.addService("permission", new PermissionController(this));
            // 注册 name = processinfo 的 ProcessInfoService 进程信息服务
            ServiceManager.addService("processinfo", new ProcessInfoService(this));
            // 注册 name = cacheinfo 的 CacheBinder服务
            ServiceManager.addService("cacheinfo", new CacheBinder(this));

            // 获取包名为"android"的ApplicationInfo
            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            // 装载到mSystemThread
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

            synchronized (this) {
                //创建ProcessRecord维护进程的相关信息
                ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
                        false,
                        0,
                        new HostingRecord("system"));
                app.setPersistent(true);
                app.setPid(MY_PID);
                app.mState.setMaxAdj(ProcessList.SYSTEM_ADJ);
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                addPidLocked(app);
                updateLruProcessLocked(app, false, null);
                updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }

        // Start watching app ops after we and the package manager are up and running.
        mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
                new IAppOpsCallback.Stub() {
                    @Override
                    public void opChanged(int op, int uid, String packageName) {
                        if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                            if (getAppOpsManager().checkOpNoThrow(op, uid, packageName)
                                    != AppOpsManager.MODE_ALLOWED) {
                                runInBackgroundDisabled(uid);
                            }
                        }
                    }
                });

        final int[] cameraOp = {AppOpsManager.OP_CAMERA};
        mAppOpsService.startWatchingActive(cameraOp, new IAppOpsActiveCallback.Stub() {
            @Override
            public void opActiveChanged(int op, int uid, String packageName, String attributionTag,
                                        boolean active, @AttributionFlags int attributionFlags,
                                        int attributionChainId) {
                cameraActiveChanged(uid, active);
            }
        });
    }
  ```
把AMS等一系例服务注册到ServiceManager中，把ApplicationInfo安装到系统主线程mSystemThread中。创建ProcessRecord维护进程的相关信息。

**AMS启动完成**：
到这里AMS已经启动完成，但在SystemServer进程启动过程中，还会在`startOtherServices`方法，调用`AMS.systemReady`方法，这个方法也很重要，所以也把这个启动之后的方法也分析一下：
  ```
    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        ......
        mActivityManagerService.systemReady(() -> {
            ......
        }, t)
    }
  ```

## AMS.systemReady
`/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
`
  ```
    public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
        t.traceBegin("PhaseActivityManagerReady");
        mSystemServiceManager.preSystemReady();
        synchronized (this) {
            // 第一次不走这里
            if (mSystemReady) {
               ......
            }

            //关键服务等待systemReady，继续完成一些初始化或进一步的工作
            mLocalDeviceIdleController =
                    LocalServices.getService(DeviceIdleInternal.class);
            mActivityTaskManager.onSystemReady();
            // Make sure we have the current profile info, since it is needed for security checks.
            mUserController.onSystemReady();
            mAppOpsService.systemReady();
            mProcessList.onSystemReady();
            mSystemReady = true;
            t.traceEnd();
        }
        ......
        // mPidsSelfLocked中保留了当前正在运行的所有进程信息
        ArrayList<ProcessRecord> procsToKill = null;
        synchronized (mPidsSelfLocked) {
            for (int i = mPidsSelfLocked.size() - 1; i >= 0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);
                //已启动的进程，若进程没有FLAG_PERSISTENT标志，则会被加入到procsToKill中
                if (!isAllowedWhileBooting(proc.info)) {
                    if (procsToKill == null) {
                        procsToKill = new ArrayList<ProcessRecord>();
                    }
                    procsToKill.add(proc);
                }
            }
        }

        // 关闭procsToKill中所有进程
        synchronized (this) {
            if (procsToKill != null) {
                for (int i = procsToKill.size() - 1; i >= 0; i--) {
                    ProcessRecord proc = procsToKill.get(i);
                    Slog.i(TAG, "Removing system update proc: " + proc);
                    mProcessList.removeProcessLocked(proc, true, false,
                            ApplicationExitInfo.REASON_OTHER,
                            ApplicationExitInfo.SUBREASON_SYSTEM_UPDATE_DONE,
                            "system update done");
                }
            }

            // 系统准备完成
            mProcessesReady = true;
        }
        ......

        // 执行传进来的goingCallback
        if (goingCallback != null) goingCallback.run();

        final int currentUserId = mUserController.getCurrentUserId();
        ......
        final boolean bootingSystemUser = currentUserId == UserHandle.USER_SYSTEM;

        if (bootingSystemUser) {
            mSystemServiceManager.onUserStarting(t, currentUserId);
        }

        synchronized (this) {
            // Only start up encryption-aware persistent apps; once user is
            // unlocked we'll come back around and start unaware apps
            startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

            // Start up initial activity.
            mBooting = true;
            // Enable home activity for system user, so that the system can always boot. We don't
            // do this when the system user is not setup since the setup wizard should be the one
            // to handle home activity in this case.
            if (UserManager.isSplitSystemUser() &&
                    Settings.Secure.getInt(mContext.getContentResolver(),
                            Settings.Secure.USER_SETUP_COMPLETE, 0) != 0
                    || SystemProperties.getBoolean(SYSTEM_USER_HOME_NEEDED, false)) {
                ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
                try {
                    AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                            UserHandle.USER_SYSTEM);
                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }
            }

            if (bootingSystemUser) {
                //启动launcher的Activity
                mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
            }

            //发送一些广播ACTION_USER_STARTED  ACTION_USER_STARTING
            if (bootingSystemUser) {
                t.traceBegin("sendUserStartBroadcast");
                final int callingUid = Binder.getCallingUid();
                final int callingPid = Binder.getCallingPid();
                final long ident = Binder.clearCallingIdentity();
                try {
                    Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                            | Intent.FLAG_RECEIVER_FOREGROUND);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                    broadcastIntentLocked(null, null, null, intent,
                            null, null, 0, null, null, null, null, OP_NONE,
                            null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                            currentUserId);
                    intent = new Intent(Intent.ACTION_USER_STARTING);
                    intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
                    intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                    broadcastIntentLocked(null, null, null, intent, null,
                            new IIntentReceiver.Stub() {
                                @Override
                                public void performReceive(Intent intent, int resultCode,
                                                           String data, Bundle extras, boolean ordered, boolean sticky,
                                                           int sendingUser) {
                                }
                            }, 0, null, null, new String[]{INTERACT_ACROSS_USERS}, null, OP_NONE,
                            null, true, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                            UserHandle.USER_ALL);
                } catch (Throwable e) {
                    Slog.wtf(TAG, "Failed sending first user broadcasts", e);
                } finally {
                    Binder.restoreCallingIdentity(ident);
                }
                t.traceEnd();
            } else {
                Slog.i(TAG, "Not sending multi-user broadcasts for non-system user "
                        + currentUserId);
            }


            t.traceEnd(); // ActivityManagerStartApps
            t.traceEnd(); // PhaseActivityManagerReady
        }
    }

  ```
- 对一些关键服务进一步初始化
- 已启动的进程，若没有`FLAG_PERSISTENT`则杀掉
- 执行`goingCallback.run`代码
- 启动launcher的activity,就是桌面应用
- 发送一些广播

## goingCallback.run
`/frameworks/base/services/java/com/android/server/SystemServer.java`
  ```
    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        ......
        mActivityManagerService.systemReady(() -> {
            // 启动阶段 500
            mSystemServiceManager.startBootPhase(t, SystemService.PHASE_ACTIVITY_MANAGER_READY);
            try {
                // 监测 native Crash
                mActivityManagerService.startObservingNativeCrashes();
            } catch (Throwable e) {
                reportWtf("observing native crashes", e);
            }
            // No dependency on Webview preparation in system server. But this should
            // be completed before allowing 3rd party
            final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
            Future<?> webviewPrep = null;
            if (!mOnlyCore && mWebViewUpdateService != null) {
                webviewPrep = SystemServerInitThreadPool.submit(() -> {
                    Slog.i(TAG, WEBVIEW_PREPARATION);
                    TimingsTraceAndSlog traceLog = TimingsTraceAndSlog.newAsyncLog();
                    traceLog.traceBegin(WEBVIEW_PREPARATION);
                    ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygote preload");
                    mZygotePreload = null;
                    mWebViewUpdateService.prepareWebViewInSystemServer();
                    traceLog.traceEnd();
                }, WEBVIEW_PREPARATION);
            }

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_AUTOMOTIVE)) {
                t.traceBegin("StartCarServiceHelperService");
                final SystemService cshs = mSystemServiceManager
                        .startService(CAR_SERVICE_HELPER_SERVICE_CLASS);
                if (cshs instanceof Dumpable) {
                    mDumper.addDumpable((Dumpable) cshs);
                }
                if (cshs instanceof DevicePolicySafetyChecker) {
                    dpms.setDevicePolicySafetyChecker((DevicePolicySafetyChecker) cshs);
                }
                t.traceEnd();
            }

            // Enable airplane mode in safe mode. setAirplaneMode() cannot be called
            // earlier as it sends broadcasts to other services.
            // TODO: This may actually be too late if radio firmware already started leaking
            // RF before the respective services start. However, fixing this requires changes
            // to radio firmware and interfaces.
            if (safeMode) {
                t.traceBegin("EnableAirplaneModeInSafeMode");
                try {
                    connectivityF.setAirplaneMode(true);
                } catch (Throwable e) {
                    reportWtf("enabling Airplane Mode during Safe Mode bootup", e);
                }
                t.traceEnd();
            }
            t.traceBegin("MakeNetworkManagementServiceReady");
            try {
                if (networkManagementF != null) {
                    networkManagementF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making Network Managment Service ready", e);
            }
            CountDownLatch networkPolicyInitReadySignal = null;
            if (networkPolicyF != null) {
                networkPolicyInitReadySignal = networkPolicyF
                        .networkScoreAndNetworkManagementServiceReady();
            }
            t.traceEnd();
            t.traceBegin("MakeIpSecServiceReady");
            try {
                if (ipSecServiceF != null) {
                    ipSecServiceF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making IpSec Service ready", e);
            }
            t.traceEnd();
            t.traceBegin("MakeNetworkStatsServiceReady");
            try {
                if (networkStatsF != null) {
                    networkStatsF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making Network Stats Service ready", e);
            }
            t.traceEnd();
            t.traceBegin("MakeConnectivityServiceReady");
            try {
                if (connectivityF != null) {
                    connectivityF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making Connectivity Service ready", e);
            }
            t.traceEnd();
            t.traceBegin("MakeVpnManagerServiceReady");
            try {
                if (vpnManagerF != null) {
                    vpnManagerF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making VpnManagerService ready", e);
            }
            t.traceEnd();
            t.traceBegin("MakeVcnManagementServiceReady");
            try {
                if (vcnManagementF != null) {
                    vcnManagementF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making VcnManagementService ready", e);
            }
            t.traceEnd();
            t.traceBegin("MakeNetworkPolicyServiceReady");
            try {
                if (networkPolicyF != null) {
                    networkPolicyF.systemReady(networkPolicyInitReadySignal);
                }
            } catch (Throwable e) {
                reportWtf("making Network Policy Service ready", e);
            }
            t.traceEnd();

            // Wait for all packages to be prepared
            mPackageManagerService.waitForAppDataPrepared();

            // It is now okay to let the various system services start their
            // third party code...
            t.traceBegin("PhaseThirdPartyAppsCanStart");
            // confirm webview completion before starting 3rd party
            if (webviewPrep != null) {
                ConcurrentUtils.waitForFutureNoInterrupt(webviewPrep, WEBVIEW_PREPARATION);
            }
            // 设置启动阶段 600
            mSystemServiceManager.startBootPhase(t, SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
            t.traceEnd();

            t.traceBegin("StartNetworkStack");
            try {
                // Note : the network stack is creating on-demand objects that need to send
                // broadcasts, which means it currently depends on being started after
                // ActivityManagerService.mSystemReady and ActivityManagerService.mProcessesReady
                // are set to true. Be careful if moving this to a different place in the
                // startup sequence.
                NetworkStackClient.getInstance().start();
            } catch (Throwable e) {
                reportWtf("starting Network Stack", e);
            }
            t.traceEnd();

            t.traceBegin("StartTethering");
            try {
                // TODO: hide implementation details, b/146312721.
                ConnectivityModuleConnector.getInstance().startModuleService(
                        TETHERING_CONNECTOR_CLASS,
                        PERMISSION_MAINLINE_NETWORK_STACK, service -> {
                            ServiceManager.addService(Context.TETHERING_SERVICE, service,
                                    false /* allowIsolated */,
                                    DUMP_FLAG_PRIORITY_HIGH | DUMP_FLAG_PRIORITY_NORMAL);
                        });
            } catch (Throwable e) {
                reportWtf("starting Tethering", e);
            }
            t.traceEnd();

            t.traceBegin("MakeCountryDetectionServiceReady");
            try {
                if (countryDetectorF != null) {
                    countryDetectorF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying CountryDetectorService running", e);
            }
            t.traceEnd();
            t.traceBegin("MakeNetworkTimeUpdateReady");
            try {
                if (networkTimeUpdaterF != null) {
                    networkTimeUpdaterF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying NetworkTimeService running", e);
            }
            t.traceEnd();
            t.traceBegin("MakeInputManagerServiceReady");
            try {
                // TODO(BT) Pass parameter to input manager
                if (inputManagerF != null) {
                    inputManagerF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying InputManagerService running", e);
            }
            t.traceEnd();
            t.traceBegin("MakeTelephonyRegistryReady");
            try {
                if (telephonyRegistryF != null) {
                    telephonyRegistryF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying TelephonyRegistry running", e);
            }
            t.traceEnd();
            t.traceBegin("MakeMediaRouterServiceReady");
            try {
                if (mediaRouterF != null) {
                    mediaRouterF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying MediaRouterService running", e);
            }
            t.traceEnd();
            t.traceBegin("MakeMmsServiceReady");
            try {
                if (mmsServiceF != null) {
                    mmsServiceF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying MmsService running", e);
            }
            t.traceEnd();

            t.traceBegin("IncidentDaemonReady");
            try {
                // TODO: Switch from checkService to getService once it's always
                // in the build and should reliably be there.
                final IIncidentManager incident = IIncidentManager.Stub.asInterface(
                        ServiceManager.getService(Context.INCIDENT_SERVICE));
                if (incident != null) {
                    incident.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying incident daemon running", e);
            }
            t.traceEnd();

            if (mIncrementalServiceHandle != 0) {
                t.traceBegin("MakeIncrementalServiceReady");
                setIncrementalServiceSystemReady(mIncrementalServiceHandle);
                t.traceEnd();
            }
        }, t);
    }

  ```
`goingCallback.run`其实就是执行SystemServer在`startOtherServices`方法中对AMS设置systemReady方法的run实现，比如设置启动阶段的值 500、600。

ActivityManagerService的启动流程就分析到这里。



