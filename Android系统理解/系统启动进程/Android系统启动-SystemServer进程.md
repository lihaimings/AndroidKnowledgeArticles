> 本篇文章基于Android6.0源码分析

相关源码文件：
  ```
/frameworks/base/core/java/com/android/internal/os/
  - ZygoteInit.java
  - RuntimeInit.java
  - Zygote.java

/frameworks/base/core/services/java/com/android/server/
  - SystemServer.java

/frameworks/base/core/jni/
  - com_android_internal_os_Zygote.cpp
  - AndroidRuntime.cpp

/frameworks/base/cmds/app_process/App_main.cpp
  ```

# SystemServer进程
根据上篇[Android系统启动-Zygote进程
](https://www.jianshu.com/p/b5fdb50a8dd3)文章，在Zygote进程启动时，会调用`ZygoteInit.main()`方法，其中分别会调用`registerZygoteSocket、preload 、startSystemServer 、runSelectLoop`来创建服务Socket、提前加载资源、创建SystemServer进程、循环创建子进程。

本篇文章讲解`startSystemServer()`方法，在startSystemServer()方法中主要完成两件事：
**· 创建SystemServer进程
· SystemServer进程启动系统服务**
  ```
    private static boolean startSystemServer(String abiList, String socketName)
    throws MethodAndArgsCaller, RuntimeException
    {
        ...
        // 创建SystemServer进程的一些参数数组
        String args [] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
             ...
            // 1. 创建SystemServer进程
            pid = Zygote.forkSystemServer(
                parsedArgs.uid, parsedArgs.gid,
                parsedArgs.gids,
                parsedArgs.debugFlags,
                null,
                parsedArgs.permittedCapabilities,
                parsedArgs.effectiveCapabilities
            );
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException (ex);
        }

        // pid为0则子进程，就是SystemServer进程
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }
            // 2. 在SystemServer进程中启动服务
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }
  ```

下图则是startSystemServer方法的创建过程，先通过`Zygote.forkSystemServe`去创建SystemServer进程，创建SystemServer进程之后,通过`handleSystemServerProcess()`在SystemServer进程中去启动服务。

![SystemServer进程启动](https://upload-images.jianshu.io/upload_images/22650779-e42f4a7a45b9f3b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 创建SystemServer进程
**Zygote.forkSystemServer**：
  ```
    package com.android.internal.os;
    public final class Zygote {
        
        public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities)
        {
            VM_HOOKS.preFork();
            // 通过nativeForkSystemServer创建进程
            int pid = nativeForkSystemServer (
                    uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
            // Enable tracing as soon as we enter the system_server.
            if (pid == 0) {
                Trace.setTracingEnabled(true);
            }
            VM_HOOKS.postForkCommon();
            return pid;
        }

         // nativeForkSystemServer是个 native方法
        // 通过: [包名]_[类名] 在AndroidRuntime查找Jni的注册的Cpp文件，然后通过：  [包名]_[类名]_[方法名]找到对应的方法。
        // 对应执行com_android_internal_os_Zygote.cpp类中的com_android_internal_os_Zygote_nativeForkAndSpecialize
        native private static int nativeForkSystemServer(int uid, int gid, int[] gids, int debugFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities);
    }


   // com_android_internal_os_Zygote.cpp
    static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
    JNIEnv* env, jclass, jint uid, jint gid, jintArray gids,
    jint debug_flags, jobjectArray rlimits,
    jint mount_external, jstring se_info, jstring se_name,
    jintArray fdsToClose, jstring instructionSet, jstring appDataDir)
    {
        // Grant CAP_WAKE_ALARM to the Bluetooth process.
        jlong capabilities = 0;
        if (uid == AID_BLUETOOTH) {
            capabilities | = (1LL << CAP_WAKE_ALARM);
        }
        // 又调用ForkAndSpecializeCommon进行进程创建
        return ForkAndSpecializeCommon(
            env, uid, gid, gids, debug_flags,
            rlimits, capabilities, capabilities, mount_external, se_info,
            se_name, false, fdsToClose, instructionSet, appDataDir
        );
    }

    static pid_t ForkAndSpecializeCommon(JNIEnv* env, uid_t uid, gid_t gid, jintArray javaGids,
    jint debug_flags, jobjectArray javaRlimits,
    jlong permittedCapabilities, jlong effectiveCapabilities,
    jint mount_external,
    jstring java_se_info, jstring java_se_name,
    bool is_system_server, jintArray fdsToClose,
    jstring instructionSet, jstring dataDir)
    {
        //设置子进程的 signal 信号处理函数
        SetSigChldHandler();
        //  fork 创建SystemServer进程
        pid_t pid = fork ();

        if (pid == 0) {
            // 子进程
           ...
        } else if (pid > 0) {
            // 父进程
        }
        return pid;
    }
  ```
创建`SystemServer`进程是通过`com.android.internal.os. Zygote`的`nativeForkSystemServer`本地方法创建的，JNI方法的注册在`AndroidRuntime`中，通过查询[包名]_[类名]可以知道对应的方法为：com_android_internal_os_Zygote.cpp类的com_android_internal_os_Zygote_nativeForkAndSpecialize()方法。最后通过`ForkAndSpecializeCommon`方法`fork()`创建SystemServer进程。

## SystemServer进程启动服务
**handleSystemServerProcess(parsedArgs)**
  ```
    public class ZygoteInit {
        // 在SystemServer进程中启动服务
        private static void handleSystemServerProcess(
        ZygoteConnection.Arguments parsedArgs)
        throws ZygoteInit.MethodAndArgsCaller
        {
            // 关闭从Zygote进程复制而来的socket
            closeServerSocket();
            ...
            if (parsedArgs.invokeWith != null) {
                ...
            } else {
                ClassLoader cl = null;
                if (systemServerClasspath != null) {
                    // 创建类加载器，并赋予当前进程
                    cl =
                        new PathClassLoader (systemServerClasspath, ClassLoader.getSystemClassLoader());
                    Thread.currentThread().setContextClassLoader(cl);
                }
                // 跳转到RuntimeInit.zygoteInit
                RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
            }

        }
    }


    public class RuntimeInit {
        public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller
        {
            // 做一些通用的初始化
            commonInit();
            // 这个方法是 native 方法，主要是打开 binder 驱动，启动 binder 线程
            nativeZygoteInit();
            // 通过异常，通过异常跳转到SystemServer.main
            applicationInit(targetSdkVersion, argv, classLoader);
        }

    }

    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
    throws ZygoteInit.MethodAndArgsCaller
    {
        ...
        final Arguments args;
        try {
            args = new Arguments (argv);
        } catch (IllegalArgumentException ex) {
            return;
        }
        ...
        // 执行此方法
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }

    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
    throws ZygoteInit.MethodAndArgsCaller
    {
        Class<?> cl;
        try {
            // 这里的className为com.android.server.SystemServer，所以是SystemServer类
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
        }
        Method m;
        try {
            // SystemServer的main方法
            m = cl.getMethod("main", new Class [] { String[].class });
        } catch (NoSuchMethodException ex) {
        } catch (SecurityException ex) {
        }

        // 通过ZygoteInit.MethodAndArgsCaller将方法带带出去
       // 那么这里的异常在那个catch处理呢？，发现由执行最初的startSystemServer()的ZygoteInit.main()方法处理此异常
        throw new ZygoteInit.MethodAndArgsCaller (m, argv);
    }

     public class ZygoteInit {
        public static void main(String argv[])
        {
            try {
                ...
                if (startSystemServer) {
                    startSystemServer(abiList, socketName);
                }
                ...
            } catch (MethodAndArgsCaller caller) {
                // 处理MethodAndArgsCaller ，并执行此run方法
                caller.run();
            } catch (RuntimeException ex) {
                Log.e(TAG, "Zygote died with exception", ex);
                closeServerSocket();
                throw ex;
            }
        }

        public static
        class MethodAndArgsCaller extends Exception
        implements Runnable
        {
            private final Method mMethod;
            private final String[] mArgs;

            public MethodAndArgsCaller (Method method, String[] args) {
            mMethod = method;
            mArgs = args;
            }

            public void run() {
                try {
                    // 通过反射执行方法
                    mMethod.invoke(null, new Object [] { mArgs });
                } catch (IllegalAccessException ex) {
                   ...
                } catch (InvocationTargetException ex) {
                  ...
                }
            }
        }
    }
  ```
通过一连串的分析得知，一是通过`nativeZygoteInit`去启动Bind,二是通过`applicationInit`方法通过异常去反射执行`SystemServer.main`方法，这里为什么需要通过异常去反射执行方法呢？为什么不直接反射执行方法？其实是为了清空栈的信息。到这里并没有启动服务，接下来的`SystemServer.main`方法是启动服务的开始。

**SystemServer.main**
  ```
public final class SystemServer {

    public static void main(String[] args)
    {
        new SystemServer ().run();
    }

    private void run()
    {
        ...
        // 主线程 looper
        Looper.prepareMainLooper();

        // 加载android_servers.so 库
        System.loadLibrary("android_servers");

        // 初始化系统上下文
        createSystemContext();

        // 创建 系统服务管理 mSystemServiceManager
        mSystemServiceManager = new SystemServiceManager (mSystemContext);
        // 将mSystemServiceManager添加到LocalServices.sLocalServiceObjects,是个map
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

        // 启动服务
        try {
            // 启动引导服务
            startBootstrapServices();
            // 启动核心服务
            startCoreServices();
            // 启动其他服务
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        }

        // 一直循环执行
        Looper.loop();
        throw new RuntimeException ("Main thread loop unexpectedly exited");
    }

    private void createSystemContext()
    {
        // 创建系统进程的上下文信息
        ActivityThread activityThread = ActivityThread.systemMain ();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
    }

    // 启动指导服务
    private void startBootstrapServices()
    {
        //阻塞等待与installd建立socket通道
        //Installer系统安装apk时的一个服务类，启动完成Installer服务之后才能启动其他的系统服务
        Installer installer = mSystemServiceManager . startService (Installer.class);

        // 启动ActivityManagerService服务
        // ActivityManagerService：负责四大组件的启动、切换、调度。
        mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        // 启动PowerManagerService服务
        // PowerManagerService：计算系统中和电池相关的计算，然后决策系统应该如何反应
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // 现在电源管理器已经启动，让活动管理器初始化电源管理功能。
        mActivityManagerService.initPowerManagement();

        // 启动LightsService服务
        // LightsService：管理 LED 和显示器背光，因此我们需要它来调出显示器。
        mSystemServiceManager.startService(LightsService.class);

        // 启动 DisplayManagerService 服务
        // 需要显示管理器在包管理器启动之前提供显示指标。
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        //Phase100: 在初始化package manager之前，我们需要默认的显示.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // 如果我们正在加密设备，则仅运行“核心”应用程序。
        String cryptState = SystemProperties . get ("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        //启动 PackageManagerService 服务
        // PackageManagerService：用来对apk进行安装、解析、删除、卸载等等操作
        Slog.i(TAG, "Package Manager");
        mPackageManagerService = PackageManagerService.main(
            mSystemContext, installer,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore
        );
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();

        // 启动 UserManagerService 服务，多用户模式管理
        ServiceManager.addService(Context.USER_SERVICE, UserManagerService.getInstance());

        // Initialize attribute cache used to cache resources from packages.
        AttributeCache.init(mSystemContext);

        // 为系统进程设置应用程序实例并开始
        mActivityManagerService.setSystemProcess();

        //启动传感器服务
        // 传感器服务需要访问包管理服务、应用操作服务和权限服务，因此我们在它们之后启动它
        startSensorService();
    }

    // 启动核心服务
    private void startCoreServices()
    {
        // 启动BatteryService 服务
        // BatteryService：管理电池相关的服务
        mSystemServiceManager.startService(BatteryService.class);

        // 启动UsageStatsService服务
        // UsageStatsService：收集用户使用每一个APP的频率、使用时常
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));
        // 在 UsageStatsService 可用后更新，在 performBootDexOpt 之前需要
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

        // 启动WebViewUpdateService服务
        // 跟踪可更新的 WebView 是否处于就绪状态并监视更新安装。
        mSystemServiceManager.startService(WebViewUpdateService.class);
    }

    private void startOtherServices()
    {
        ...
        try {
            // 启动CameraService 服务,摄像头相关服务
            mSystemServiceManager.startService(CameraService.class);
            ...
            // 启动 AlarmManagerService 服务， 全局定时器管理服务
            mSystemServiceManager.startService(AlarmManagerService.class);
            alarm = IAlarmManager.Stub.asInterface(
                ServiceManager.getService(Context.ALARM_SERVICE)
            );
            ...
            // 启动InputManagerService服务 ，管理输入事件
            inputManager = new InputManagerService (context);

            // 启动 WindowManagerService 服务， 窗口管理服务
            wm = WindowManagerService.main(
                context, inputManager,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                !mFirstBoot, mOnlyCore
            );
            ServiceManager.addService(Context.WINDOW_SERVICE, wm);
            ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
            mActivityManagerService.setWindowManager(wm);
            inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
            // 启动input
            inputManager.start();
            mDisplayManagerService.windowManagerAndInputReady();

            // 启动 BluetoothService 服务，蓝牙管理服务
            mSystemServiceManager.startService(BluetoothService.class);
            ...
            // 启动 NotificationManagerService 服务，通知管理服务
            mSystemServiceManager.startService(NotificationManagerService.class);
            notification = INotificationManager.Stub.asInterface(
                ServiceManager.getService(Context.NOTIFICATION_SERVICE)
            );
            networkPolicy.bindNotificationManager(notification);

            // 启动DeviceStorageMonitorService 服务，存储相关管理服务
            mSystemServiceManager.startService(DeviceStorageMonitorService.class);

            if (!disableLocation) {
                try {
                    Slog.i(TAG, "Location Manager");
                    // 启动 LocationManagerService 服务， 定位管理服务
                    location = new LocationManagerService (context);
                    ServiceManager.addService(Context.LOCATION_SERVICE, location);
                } catch (Throwable e) {
                    reportWtf("starting Location Manager", e);
                }
                ...
                // 准备好了 wms,  pms, ams 服务
                wm.systemReady();
                mPowerManagerService.systemReady(mActivityManagerService.getAppOpsService());
                mPackageManagerService.systemReady();
                mDisplayManagerService.systemReady(safeMode, mOnlyCore);

                // 我们现在告诉活动管理器可以运行第三方代码。一旦达到第三方代码真正可以运行的状态（但在它实际开始启动初始应用程序之前），它就会回调我们，以便我们完成初始化。
                mActivityManagerService.systemReady(new Runnable () {
                    @Override
                    public void run() {

                    }
                });
            }
        }
    }
}

  ```
SystemServer执行`main()`方法后调用`run`方法，之后分别调用`startBootstrapServices`、`startCoreServices`、`startOtherServices`开始启动很多服务，像常见的**ActivityManagerService、PackageManagerService、WindowManagerService、InputManagerServic**服务都在这里开启。
开启服务有两种方式：
  ```
1. mSystemServiceManager.startService(LightsService.class);

  2. LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

  ```
但是发现最终都是调用到LocalServices.addService这种方法。
  ```
public final class ServiceManager {

    public static void addService(String name, IBinder service)
    {
        try {
            // 调用ServiceManagerProxy的addService方法
            getIServiceManager().addService(name, service, false);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }

    private static IServiceManager getIServiceManager()
    {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }
}

public abstract class ServiceManagerNative extends Binder implements IServiceManager
{

    /**
     * 将 Binder 对象投射到服务管理器接口中，如果需要则生成代理
     */
    static public IServiceManager asInterface (IBinder obj)
    {
        if (obj == null) {
            return null;

        }
        IServiceManager in =
        (IServiceManager) obj . queryLocalInterface (descriptor);
        if ( in != null) {
        return in;

    }
        // 返回
        return new ServiceManagerProxy (obj);
    }
}

class ServiceManagerProxy implements IServiceManager {

    public ServiceManagerProxy (IBinder remote) {
        mRemote = remote;
    }

   // IPC binder 驱动
    public void addService(String name, IBinder service, boolean allowIsolated)
    throws RemoteException {
        Parcel data = Parcel.obtain ();
        Parcel reply = Parcel.obtain ();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        data.writeStrongBinder(service);
        data.writeInt(allowIsolated ? 1 : 0);
        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
        reply.recycle();
        data.recycle();
    }

}

  ```

可见启动服务是靠Binder驱动去开启的。

## SystemServer进程总结
首先通过JNI的注册方法去创建SystemServer进程，创建进程之后，开始处理SystemServer进程，通过异常反射调用SystemServer的main方法，主要完成两件事：一、启动Binder驱动线程，二、开启服务(AMS、PMS、WMS、IMS等)，最后通过Binder驱动去启动服务。


