> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/services/java/com/android/server/SystemServer.java
/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

  ```

## 1. WMS的启动过程
`WindowManagerService`服务在`system_server`进程中被启动，在其`startOtherServices()`方法中调用`WMS.main()`方法开始实例化WMS,如下是WMS的整体启动流程：

![wms的启动过程](https://upload-images.jianshu.io/upload_images/22650779-65b528085a6086f2.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ```
public final class SystemServer {

    private void run() {
        try {
            // 启动服务
            startBootstrapServices(t);
            startCoreServices(t);
            startOtherServices(t);
        } catch (Throwable ex) {
            throw ex;
        }
    }

    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        try {
            //...
            // 1. WindowManagerService启动
            wm = WindowManagerService.main(context, inputManager, !mFirstBoot, mOnlyCore,
                    new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
            // 2. 调用WindowManagerService的onInitReady()方法
            wm.onInitReady();
            //...
            // 3.调用WindowManagerService的displayReady()方法
            wm.displayReady();
            // 4.调用WindowManagerService的systemReady()方法
            wm.systemReady(
        }
    }
}

  ```
首先调用`WMS.main()`实例化WindowManagerService对象，之后分别调用`WMS.onInitReady()`、`WMS.displayReady()`、`WMS.systemReady()`方法做后续的初始化操作。

### 1.2 WMS.main()
  ```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {

    @VisibleForTesting
    public static WindowManagerService main(final Context context, final InputManagerService im,
                                            final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy,
                                            ActivityTaskManagerService atm, Supplier<SurfaceControl.Transaction> transactionFactory,
                                            Supplier<Surface> surfaceFactory,
                                            Function<SurfaceSession, SurfaceControl.Builder> surfaceControlFactory) {
        // DisplayThread线程的创建WindowManagerService
        DisplayThread.getHandler().runWithScissors(() ->
                // 创建WindowManagerService()实例，
                sInstance = new WindowManagerService(context, im, showBootMsgs, onlyCore, policy,
                        atm, transactionFactory, surfaceFactory, surfaceControlFactory), 0);
        return sInstance;
    }

  private WindowManagerService(Context context, InputManagerService inputManager,
                                 boolean showBootMsgs, boolean onlyCore, WindowManagerPolicy policy,
                                 ActivityTaskManagerService atm, Supplier<SurfaceControl.Transaction> transactionFactory,
                                 Supplier<Surface> surfaceFactory,
                                 Function<SurfaceSession, SurfaceControl.Builder> surfaceControlFactory) {
        installLock(this, INDEX_WINDOW);
        mGlobalLock = atm.getGlobalLock();
        mAtmService = atm;
        mContext = context;
        mIsPc = mContext.getPackageManager().hasSystemFeature(FEATURE_PC);
        mAllowBootMessages = showBootMsgs;
        mOnlyCore = onlyCore;
        mLimitedAlphaCompositing = context.getResources().getBoolean(
                com.android.internal.R.bool.config_sf_limitedAlpha);
        mHasPermanentDpad = context.getResources().getBoolean(
                com.android.internal.R.bool.config_hasPermanentDpad);
        mInTouchMode = context.getResources().getBoolean(
                com.android.internal.R.bool.config_defaultInTouchMode);
        inputManager.setInTouchMode(mInTouchMode);
        mDrawLockTimeoutMillis = context.getResources().getInteger(
                com.android.internal.R.integer.config_drawLockTimeoutMillis);
        mAllowAnimationsInLowPowerMode = context.getResources().getBoolean(
                com.android.internal.R.bool.config_allowAnimationsInLowPowerMode);
        mMaxUiWidth = context.getResources().getInteger(
                com.android.internal.R.integer.config_maxUiWidth);
        mDisableTransitionAnimation = context.getResources().getBoolean(
                com.android.internal.R.bool.config_disableTransitionAnimation);
        mPerDisplayFocusEnabled = context.getResources().getBoolean(
                com.android.internal.R.bool.config_perDisplayFocusEnabled);
        mAssistantOnTopOfDream = context.getResources().getBoolean(
                com.android.internal.R.bool.config_assistantOnTopOfDream);
        mInputManager = inputManager; // Must be before createDisplayContentLocked.
        mDisplayManagerInternal = LocalServices.getService(DisplayManagerInternal.class);

        mSurfaceControlFactory = surfaceControlFactory;
        mTransactionFactory = transactionFactory;
        mSurfaceFactory = surfaceFactory;
        mTransaction = mTransactionFactory.get();

        mDisplayWindowSettings = new DisplayWindowSettings(this);
        mPolicy = policy;
        mAnimator = new WindowAnimator(this);
        mRoot = new RootWindowContainer(this);

        mUseBLAST = DeviceConfig.getBoolean(
                DeviceConfig.NAMESPACE_WINDOW_MANAGER_NATIVE_BOOT,
                WM_USE_BLAST_ADAPTER_FLAG, false);

        mWindowPlacerLocked = new WindowSurfacePlacer(this);
        mTaskSnapshotController = new TaskSnapshotController(this);

        mWindowTracing = WindowTracing.createDefaultAndStartLooper(this,
                Choreographer.getInstance());

        LocalServices.addService(WindowManagerPolicy.class, mPolicy);

        mDisplayManager = (DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);

        mKeyguardDisableHandler = KeyguardDisableHandler.create(mContext, mPolicy, mH);

        mPowerManager = (PowerManager) context.getSystemService(Context.POWER_SERVICE);
        mPowerManagerInternal = LocalServices.getService(PowerManagerInternal.class);

        if (mPowerManagerInternal != null) {
            mPowerManagerInternal.registerLowPowerModeObserver(
                    new PowerManagerInternal.LowPowerModeListener() {
                        @Override
                        public int getServiceType() {
                            return ServiceType.ANIMATION;
                        }

                        @Override
                        public void onLowPowerModeChanged(PowerSaveState result) {
                            synchronized (mGlobalLock) {
                                final boolean enabled = result.batterySaverEnabled;
                                if (mAnimationsDisabled != enabled && !mAllowAnimationsInLowPowerMode) {
                                    mAnimationsDisabled = enabled;
                                    dispatchNewAnimatorScaleLocked(null);
                                }
                            }
                        }
                    });
            mAnimationsDisabled = mPowerManagerInternal
                    .getLowPowerState(ServiceType.ANIMATION).batterySaverEnabled;
        }
        mScreenFrozenLock = mPowerManager.newWakeLock(
                PowerManager.PARTIAL_WAKE_LOCK, "SCREEN_FROZEN");
        mScreenFrozenLock.setReferenceCounted(false);

        mDisplayNotificationController = new DisplayWindowListenerController(this);

        mActivityManager = ActivityManager.getService();
        mActivityTaskManager = ActivityTaskManager.getService();
        mAmInternal = LocalServices.getService(ActivityManagerInternal.class);
        mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
        mAppOps = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);
        AppOpsManager.OnOpChangedInternalListener opListener =
                new AppOpsManager.OnOpChangedInternalListener() {
                    @Override
                    public void onOpChanged(int op, String packageName) {
                        updateAppOpsState();
                    }
                };
        mAppOps.startWatchingMode(OP_SYSTEM_ALERT_WINDOW, null, opListener);
        mAppOps.startWatchingMode(AppOpsManager.OP_TOAST_WINDOW, null, opListener);

        mPmInternal = LocalServices.getService(PackageManagerInternal.class);
        final IntentFilter suspendPackagesFilter = new IntentFilter();
        suspendPackagesFilter.addAction(Intent.ACTION_PACKAGES_SUSPENDED);
        suspendPackagesFilter.addAction(Intent.ACTION_PACKAGES_UNSUSPENDED);
        context.registerReceiverAsUser(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                final String[] affectedPackages =
                        intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
                final boolean suspended =
                        Intent.ACTION_PACKAGES_SUSPENDED.equals(intent.getAction());
                updateHiddenWhileSuspendedState(new ArraySet<>(Arrays.asList(affectedPackages)),
                        suspended);
            }
        }, UserHandle.ALL, suspendPackagesFilter, null, null);

        final ContentResolver resolver = context.getContentResolver();
        // Get persisted window scale setting
        mWindowAnimationScaleSetting = Settings.Global.getFloat(resolver,
                Settings.Global.WINDOW_ANIMATION_SCALE, mWindowAnimationScaleSetting);
        mTransitionAnimationScaleSetting = Settings.Global.getFloat(resolver,
                Settings.Global.TRANSITION_ANIMATION_SCALE,
                context.getResources().getFloat(
                        R.dimen.config_appTransitionAnimationDurationScaleDefault));

        setAnimatorDurationScale(Settings.Global.getFloat(resolver,
                Settings.Global.ANIMATOR_DURATION_SCALE, mAnimatorDurationScaleSetting));

        mForceDesktopModeOnExternalDisplays = Settings.Global.getInt(resolver,
                DEVELOPMENT_FORCE_DESKTOP_MODE_ON_EXTERNAL_DISPLAYS, 0) != 0;

        IntentFilter filter = new IntentFilter();
        // Track changes to DevicePolicyManager state so we can enable/disable keyguard.
        filter.addAction(ACTION_DEVICE_POLICY_MANAGER_STATE_CHANGED);
        mContext.registerReceiverAsUser(mBroadcastReceiver, UserHandle.ALL, filter, null, null);

        mLatencyTracker = LatencyTracker.getInstance(context);

        mSettingsObserver = new SettingsObserver();

        mHoldingScreenWakeLock = mPowerManager.newWakeLock(
                PowerManager.SCREEN_BRIGHT_WAKE_LOCK | PowerManager.ON_AFTER_RELEASE, TAG_WM);
        mHoldingScreenWakeLock.setReferenceCounted(false);

        mSurfaceAnimationRunner = new SurfaceAnimationRunner(mTransactionFactory,
                mPowerManagerInternal);

        mAllowTheaterModeWakeFromLayout = context.getResources().getBoolean(
                com.android.internal.R.bool.config_allowTheaterModeWakeFromWindowLayout);

        mTaskPositioningController = new TaskPositioningController(
                this, mInputManager, mActivityTaskManager, mH.getLooper());
        mDragDropController = new DragDropController(this, mH.getLooper());

        mHighRefreshRateBlacklist = HighRefreshRateBlacklist.create(context.getResources());

        mConstants = new WindowManagerConstants(this, DeviceConfigInterface.REAL);
        mConstants.start(new HandlerExecutor(mH));

        LocalServices.addService(WindowManagerInternal.class, new LocalService());
        mEmbeddedWindowController = new EmbeddedWindowController(mAtmService);

        mDisplayAreaPolicyProvider = DisplayAreaPolicy.Provider.fromResources(
                mContext.getResources());

        setGlobalShadowSettings();
    }

}

  ```
通过DisplayThread线程的Handler,并调用其`runWithScissors()`方法，该方法会将当前的执行线程阻塞，并直到要执行的任务完成后，在重新唤醒被阻塞的线程：
  ```
public class Handler {

    public final boolean runWithScissors(@NonNull Runnable r, long timeout) {
        // 对runnable和time进行检查
        if (r == null) {
            throw new IllegalArgumentException("runnable must not be null");
        }
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout must be non-negative");
        }

        // 检查当前线程的Looper是否等于此mLooper
        if (Looper.myLooper() == mLooper) {
            r.run(); // 直接运行Runnable.run
            return true;
        }

        BlockingRunnable br = new BlockingRunnable(r);
        return br.postAndWait(this, timeout);
    }

    private static final class BlockingRunnable implements Runnable {
        private final Runnable mTask;
        private boolean mDone;

        public BlockingRunnable(Runnable task) {
            mTask = task;
        }

        @Override
        public void run() {
            try {
                // 执行Runnable创建WMS
                mTask.run();
            } finally {
                synchronized (this) {
                    mDone = true;
                    notifyAll();
                }
            }
        }

        public boolean postAndWait(Handler handler, long timeout) {
            // 将BlockingRunnable加入到handler中
            if (!handler.post(this)) {
                return false;
            }

            // 没执行完run()方法则调用wait()让调用的线程等待
            synchronized (this) {
                if (timeout > 0) {
                    final long expirationTime = SystemClock.uptimeMillis() + timeout;
                    while (!mDone) {
                        long delay = expirationTime - SystemClock.uptimeMillis();
                        if (delay <= 0) {
                            return false; // timeout
                        }
                        try {
                            wait(delay);
                        } catch (InterruptedException ex) {
                        }
                    }
                } else {
                    // 没执行完run()方法则调用
                    while (!mDone) {
                        try {
                            wait();
                        } catch (InterruptedException ex) {
                        }
                    }
                }
            }
            return true;
        }
    }
}
  ```
会判断当前执行的线程跟调用的Handler线程是否相等，相等则执行Runnable。如果不想等，则在封装一个Runnable，并在这个Runnable类中判断是否执行了`run()`方法，如果没有则阻塞当前的执行线程，直到执行完Run方法后才重新唤醒。
所以在DisplayThread线程中创建了`WindowManagerService`实例对象。

### 1.3 WMS.onInitReady()
  ```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {


    public void onInitReady() {
        // 调用initPolicy()方法
        initPolicy();
        //...
    }

    private void initPolicy() {
        UiThread.getHandler().runWithScissors(new Runnable() {
            @Override
            public void run() {
                WindowManagerPolicyThread.set(Thread.currentThread(), Looper.myLooper());
                // 此mPolicy在构造函数中赋值，由SystemServer类中传入，为PhoneWindowManager()
                mPolicy.init(mContext, WindowManagerService.this, WindowManagerService.this);
            }
        }, 0);
    }

}


public class PhoneWindowManager implements WindowManagerPolicy {

    // 在UiThread线程中执行
    public void init(Context context, IWindowManager windowManager,
                     WindowManagerFuncs windowManagerFuncs) {
        mContext = context;
        mWindowManager = windowManager;
        mWindowManagerFuncs = windowManagerFuncs;
        mWindowManagerInternal = LocalServices.getService(WindowManagerInternal.class);
        mActivityManagerInternal = LocalServices.getService(ActivityManagerInternal.class);
        mActivityTaskManagerInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
        mInputManagerInternal = LocalServices.getService(InputManagerInternal.class);
        mDreamManagerInternal = LocalServices.getService(DreamManagerInternal.class);
        mPowerManagerInternal = LocalServices.getService(PowerManagerInternal.class);
        mAppOpsManager = mContext.getSystemService(AppOpsManager.class);
        ...

        mHandler = new PolicyHandler();
        mWakeGestureListener = new MyWakeGestureListener(mContext, mHandler);
        ...

        mPowerManager = (PowerManager) context.getSystemService(Context.POWER_SERVICE);
        mBroadcastWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                "PhoneWindowManager.mBroadcastWakeLock");
        mPowerKeyWakeLock = mPowerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                "PhoneWindowManager.mPowerKeyWakeLock");
        ...

        mGlobalKeyManager = new GlobalKeyManager(mContext);


        if (!mPowerManager.isInteractive()) {
            startedGoingToSleep(WindowManagerPolicy.OFF_BECAUSE_OF_USER);
            finishedGoingToSleep(WindowManagerPolicy.OFF_BECAUSE_OF_USER);
        }

        mWindowManagerInternal.registerAppTransitionListener(new AppTransitionListener() {
            @Override
            public int onAppTransitionStartingLocked(int transit, long duration,
                                                     long statusBarAnimationStartTime, long statusBarAnimationDuration) {
                return handleStartTransitionForKeyguardLw(transit, duration);
            }

            @Override
            public void onAppTransitionCancelledLocked(int transit) {
                handleStartTransitionForKeyguardLw(transit, 0 /* duration */);
            }
        });
    }

}
  ```
在UiThread线程的Handler中执行`PhoneWindowManager.init()`方法，进行一些变量的的初始化操作。

### 1.4 WMS.displayReady()、systemReady()
  ```
public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {

    // 继续在system_server进程中执行WMS.displayReady()方法
    public void displayReady() {
        synchronized (mGlobalLock) {
            if (mMaxUiWidth > 0) {
                mRoot.forAllDisplays(displayContent -> displayContent.setMaxUiWidth(mMaxUiWidth));
            }
            applyForcedPropertiesForDefaultDisplay();
            mAnimator.ready();
            mDisplayReady = true;
            // Reconfigure all displays to make sure that forced properties and
            // DisplayWindowSettings are applied.
            mRoot.forAllDisplays(DisplayContent::reconfigureDisplayLocked);
            mIsTouchDevice = mContext.getPackageManager().hasSystemFeature(
                    PackageManager.FEATURE_TOUCHSCREEN);
        }

        try {
            mActivityTaskManager.updateConfiguration(null);
        } catch (RemoteException e) {
        }
    }

    // 继续在system_server进程中执行WMS.systemReady()方法
    public void systemReady() {
        mSystemReady = true;
        // 执行PhoneWindowManager.systemReady()方法
        mPolicy.systemReady();
        mRoot.forAllDisplayPolicies(DisplayPolicy::systemReady);
        mTaskSnapshotController.systemReady();
        mHasWideColorGamutSupport = queryWideColorGamutSupport();
        mHasHdrSupport = queryHdrSupport();
        UiThread.getHandler().post(mSettingsObserver::loadSettings);
        IVrManager vrManager = IVrManager.Stub.asInterface(
                ServiceManager.getService(Context.VR_SERVICE));
        if (vrManager != null) {
            try {
                final boolean vrModeEnabled = vrManager.getVrModeState();
                synchronized (mGlobalLock) {
                    vrManager.registerListener(mVrStateCallbacks);
                    if (vrModeEnabled) {
                        mVrModeEnabled = vrModeEnabled;
                        mVrStateCallbacks.onVrStateChanged(vrModeEnabled);
                    }
                }
            } catch (RemoteException e) {
                // Ignore, we cannot do anything if we failed to register VR mode listener
            }
        }
    }
}
  ```
`WMS.systemReady()`方法会调用`PhoneWindowManager.systemReady()`方法：
  ```
public class PhoneWindowManager implements WindowManagerPolicy {

    public void systemReady() {
        // In normal flow, systemReady is called before other system services are ready.
        // So it is better not to bind keyguard here.
        mKeyguardDelegate.onSystemReady();

        mVrManagerInternal = LocalServices.getService(VrManagerInternal.class);
        if (mVrManagerInternal != null) {
            mVrManagerInternal.addPersistentVrModeStateListener(mPersistentVrModeListener);
        }

        readCameraLensCoverState();
        updateUiMode();
        mDefaultDisplayRotation.updateOrientationListener();
        synchronized (mLock) {
            mSystemReady = true;
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    updateSettings();
                }
            });
            // If this happens, for whatever reason, systemReady came later than systemBooted.
            // And keyguard should be already bound from systemBooted
            if (mSystemBooted) {
                mKeyguardDelegate.onBootCompleted();
            }
        }

        mAutofillManagerInternal = LocalServices.getService(AutofillManagerInternal.class);
    }

}

  ```

至此**WindowManagerService**在system_server进程的启动就完成了，WMS主要有四大功能：
1. Window窗口的管理
2. Surface的管理
3. Input系统的管理
4. 窗口动画

如下是WindowManagerService的关联图：
![](https://upload-images.jianshu.io/upload_images/22650779-1bce9cf1ef007a83.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



