> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/core/java/android/content/ContextWrapper.java
/frameworks/base/core/java/android/app/ContextImpl.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
/frameworks/base/core/java/android/app/ActivityThread.java
  ```
启动Service有两种方式，一种是 `startService`，一种是`bindService`。其中startService和bindService会分别执行不同的方法。如startService会执行`onStartCommand`方法，而bindService会执行`onBind、onUnbind`方法。

![](https://upload-images.jianshu.io/upload_images/22650779-446f2ff193879b55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中`startService`的启动方式是不能进行通信的，而且其Service的生命周期不跟调用方同步。`binderService`启动方式则可以通过Binder进行通信。一下是startService的流程图：
![](https://upload-images.jianshu.io/upload_images/22650779-28bbd35780960fd7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 1. startService发起进程端
### 1.1 ContextWrapper.startService
  ```
public class ContextWrapper extends Context {

    Context mBase;

    @Override
    public ComponentName startService(Intent service) {
        // mBase的实例是ContextImpl
        return mBase.startService(service);
    }
}
  ```
### 1.2 ContextImpl.startService
  ```
class ContextImpl extends Context {
    @Override
    public ComponentName startService(Intent service) {
        // 检测是否为system_server进程
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, false, mUser);
    }

    private ComponentName startServiceCommon(Intent service, boolean requireForeground,
                                             UserHandle user) {
        try {
            //检测Intent的包名是否为空,throw异常
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);


            // ActivityManager.getService()通过ServiceManager得到AMS的BinderProxy,
            // 并调用AMS的BinderProxy其startService方法。
            ComponentName cn = ActivityManager.getService().startService(
                    mMainThread.getApplicationThread(), service,
                    service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
                    getOpPackageName(), getAttributionTag(), user.getIdentifier());



            // 对返回的ComponentName进行检查，异常抛throw
            if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException(
                            "Not allowed to start service " + service
                                    + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException(
                            "Unable to start service " + service
                                    + ": " + cn.getClassName());
                } else if (cn.getPackageName().equals("?")) {
                    throw new IllegalStateException(
                            "Not allowed to start service " + service + ": " + cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
  ```
  ```
// 获取ASM的BinderProxy
public class ActivityManager {
    @UnsupportedAppUsage
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    @UnsupportedAppUsage
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    // Context.ACTIVITY_SERVICE是activity
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
}
  ```
`startServiceCommon`方法先检查Intent，然后通过Binder通信调用`AMS.startService`方法，其中参数` mMainThread.getApplicationThread()`是一个Binder，AMS通过其和调用端进行通信,其过程如下所示：
![](https://upload-images.jianshu.io/upload_images/22650779-bcbaaefe96db2547.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. system_server进程
在startService的过程中`ActivityManagerServer`的作用大致是找到对应的Servive并调用其生命周期。
### 2.1 AMS.startService
  ```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    final ActiveServices mServices;

    @Override
    public ComponentName startService(IApplicationThread caller, Intent service,
                                      String resolvedType, boolean requireForeground, String callingPackage,
                                      String callingFeatureId, int userId)
            throws TransactionTooLargeException {
        //当调用者是孤立进程，则抛出异常
        enforceNotIsolatedCaller("startService");
        // 拒绝可能泄露的文件描述符
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        if (callingPackage == null) {
            throw new IllegalArgumentException("callingPackage cannot be null");
        }

        synchronized (this) {
            final int callingPid = Binder.getCallingPid(); //调用者pid
            final int callingUid = Binder.getCallingUid(); //调用者uid
            final long origId = Binder.clearCallingIdentity();
            ComponentName res;
            try {

                // 调用ActiveServices.startServiceLocked方法
                res = mServices.startServiceLocked(caller, service,
                        resolvedType, callingPid, callingUid,
                        requireForeground, callingPackage, callingFeatureId, userId);

            } finally {
                Binder.restoreCallingIdentity(origId);
            }
            return res;
        }
    }

  ```
`AMS.startService`方法作了一下调用端进程的检查后，直接调用`ActiveServices.startServiceLocked`方法，并把参数全部传递给`ActiveServices`类。

### 2.2 ActiveServices.startServiceLocked
  ```
public final class ActiveServices {

    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
                                     int callingPid, int callingUid, boolean fgRequired, String callingPackage,
                                     @Nullable String callingFeatureId, final int userId)
            throws TransactionTooLargeException {
        //  继续调用startServiceLocked方法
        return startServiceLocked(caller, service, resolvedType, callingPid, callingUid, fgRequired,
                callingPackage, callingFeatureId, userId, false);
    }

  ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
                                     int callingPid, int callingUid, boolean fgRequired, String callingPackage,
                                     @Nullable String callingFeatureId, final int userId,
                                     boolean allowBackgroundActivityStarts) throws TransactionTooLargeException {

        // 检查是否为前台进程
        final boolean callerFg;
        // .....
     
        //根据Intent检索对应的服务信息
        ServiceLookupResult res =
                retrieveServiceLocked(service, null, resolvedType, callingPackage,
                        callingPid, callingUid, userId, true, callerFg, false, false);
       
        //对于非前台进程的调度
        if (!callerFg && !fgRequired && r.app == null
                && mAm.mUserController.hasStartedUserState(r.userId)) {
            // 获取启动Service的所在进程信息 ProcessRecord
            ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
        }

         
            // ...
            // 调用startServiceInnerLocked
            ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);

            return cmp;
        }
    }

  ```
`ActiveServices.startServiceLocked `方法会获取调用端的ProcessRecord，和获取被调用端的`ServiceRecord、ProcessRecord`查看此次启动的Service是否要加入延迟队列。最终调用`startServiceInnerLocked`方法继续执行逻辑。
  ```
 ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
                                          boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        //。。。
        // 调用bringUpServiceLocked方法
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        //。。。
        return r.name;
    }
    
    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
                                        boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        if (r.app != null && r.app.thread != null) {
            // 如果service已经启动，那么多次启动Service时会多次调用service.onStartCommand()方法
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        final boolean isolated = (r.serviceInfo.flags & ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;
        if (!isolated) {
            //根据进程名和uid，查询ProcessRecord
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) {
                // 目标进程已存在
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    // 启动服务，调用Service对应方法
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortInstanceName, e);
                }

            }
        } else {
            app = r.isolatedProc;
        }
 
       //... 对Service进程的做一些空判断和延迟操作
        return null;
    }
  ```
获取Service所对应的进程，如果进程创建成功而且不延迟启动，则调用`realStartServiceLocked`方法调用Service的一些方法。


### 2.3 ActiveServices.realStartServiceLocked
  ```
    private final void realStartServiceLocked(ServiceRecord r,
                                              ProcessRecord app, boolean execInFg) throws RemoteException {
     
        // handle发送延迟消息，如果在规定时间还没有被取消，则证明方法执行时间长，则抛ANR异常。
        bumpServiceExecutingLocked(r, execInFg, "create");
        //...
        try {
            //调用Service对应onCreate()方法
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
     
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app, "Died when creating service");
            throw e;
        } finally {
           //...
        }

        //服务 进入onStartCommand()
        sendServiceArgsLocked(r, execInFg, true);

        // 
        if (r.delayedStop) {
            r.delayedStop = false;
            if (r.startRequested) {
                //停止服务
                stopServiceLocked(r);
            }
        }
    }

// 通过Handler发送延迟时间，到时间内没被取消则抛ANR异常
 private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        long now = SystemClock.uptimeMillis();
        if (r.executeNesting == 0) {
            //...
            if (r.app != null) {
                r.app.executingServices.add(r);
                r.app.execServicesFg |= fg;
                if (timeoutNeeded && r.app.executingServices.size() == 1) {
                    scheduleServiceTimeoutLocked(r.app);
                }
            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {
            r.app.execServicesFg = true;
            if (timeoutNeeded) {
                // 发送Handler
                scheduleServiceTimeoutLocked(r.app);
            }
        }
        r.executeFg |= fg;
        r.executeNesting++;
        r.executingStart = now;
    }

   //  发送延迟消息
    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        //当超时后仍没有remove该SERVICE_TIMEOUT_MSG消息，则执行service Timeout流程
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }

   // 执行Service的onStartCommand方法
   private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
                                             boolean oomAdjusted) throws TransactionTooLargeException {
        while (r.pendingStarts.size() > 0) {
            //...
            // 装弹，延迟发送一条消息，如果执行onStartCommand方法超过
            bumpServiceExecutingLocked(r, execInFg, "start");
            //...
            try {
                // 调用AT.scheduleServiceArgs,最终调用onStartCommand
                r.app.thread.scheduleServiceArgs(r, slice);
            }
        }

    }

  ```
`realStartServiceLocked`方法，分别会调用`app.thread`所对应的scheduleCreateService、scheduleServiceArgs，此app.thread是Service所在进程的ApplicationThread类。在调用方法前，还会调用`bumpServiceExecutingLocked`方法发送一个延迟消息，如果这个消息没被取消，则会弹ANR。

## 3. Service进程

### 3.1 ApplicationThread.scheduleCreateService

  ```
    private class ApplicationThread extends IApplicationThread.Stub {

        public final void scheduleCreateService(IBinder token,
                                                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            // 通过Handler发送消息执行
            sendMessage(H.CREATE_SERVICE, s);
        }

    }

    public void handleMessage(Message msg) {
        switch (msg.what) {
            case CREATE_SERVICE:
                handleCreateService((CreateServiceData) msg.obj);
                break;
        }
    }

    // 创建对应的Service并执行onCreate()方法
    private void handleCreateService(CreateServiceData data) {
        //当应用处于后台即将进行GC，而此时被调回到活动状态，则跳过本次gc
        unscheduleGcIdler();
        // 获取LoadedApk
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        Service service = null;
        try {
            // 根据LoadedApk创建ContextImpl
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            //创建Application对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            // 获取ClassLoader
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            // 通过反射创建Service对象
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
            context.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            context.setOuterContext(service);
            // 调用service.attach绑定资源文件
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            // 调用 service.onCreate()方法
            service.onCreate();
            mServices.put(data.token, service);
            try {
                // onCreate()执行完成，拆弹过程，最终调用到ActiveServices.serviceDoneExecutingLocked方法
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
          //...
        }
    }

  ```
先会创建一个`ContextImpl`对象，在通过反射创建`Service对象`，通过调用`Service.attach`方法将资源传给Service对象,最后调用`Service.onCreate`对象，并向AMS发送取消延迟发送消息。

### 3.2 ApplicationThread. scheduleServiceArgs

  ```
    private class ApplicationThread extends IApplicationThread.Stub {

        public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
            List<ServiceStartArgs> list = args.getList();

            for (int i = 0; i < list.size(); i++) {
                ServiceStartArgs ssa = list.get(i);
                ServiceArgsData s = new ServiceArgsData();
                s.token = token;
                s.taskRemoved = ssa.taskRemoved;
                s.startId = ssa.startId;
                s.flags = ssa.flags;
                s.args = ssa.args;

                sendMessage(H.SERVICE_ARGS, s);
            }
        }
    }

    public void handleMessage(Message msg) {
        switch (msg.what) {
            case SERVICE_ARGS:
                handleServiceArgs((ServiceArgsData) msg.obj);
                break;
        }
    }

    // 创建对应的Service并执行onCreate()方法
    private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                // 调用Service的onStartCommand方法
                int res;
                if (!data.taskRemoved) {
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                }

                try {
                    // 向AMS取消Handler发送ANR消息
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
    }

  ```
根据缓存找到对应的`Service`，并执行`Service. onStartCommand()`方法，并向AMS发送了取消ANR的延迟消息。

在来看看stopService会发生什么：
  ```
   private void handleStopService(IBinder token) {
        Service s = mServices.remove(token);
        if (s != null) {
            try {
                s.onDestroy();
                s.detachAndCleanUp();
                Context context = s.getBaseContext();
                if (context instanceof ContextImpl) {
                    final String who = s.getClassName();
                    ((ContextImpl) context).scheduleFinalCleanup(who, "Service");
                }

                QueuedWork.waitToFinish();

                try {
                    ActivityManager.getService().serviceDoneExecuting(
                            token, SERVICE_DONE_EXECUTING_STOP, 0, 0);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
                
            }
        }
  ```
执行`Service.onDestroy`方法，并对Service和对应ContextImpl进行clear操作。startService启动的Service的生命周期并不会跟调用端同步，所以要自己手动调用停止操作：`stopSelf()或stopService()`






> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/core/java/android/content/ContextWrapper.java
/frameworks/base/core/java/android/app/ContextImpl.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
/frameworks/base/core/java/android/app/ActivityThread.java
  ```
通过`bindService`启动的Service,会执行Service的`onCreate、onBind、onUnbind、onDestroy`方法，可以通过onBind方法返回的Binder对象和调用端进行通信，并且Service的生命周期和调用端同步。
如下是启动bindService的代码
  ```
var stu: Student? = null
val connection = object : ServiceConnection {
    override fun onServiceConnected(p0: ComponentName?, p1: IBinder?) {
        stu = Student.Stub.asInterface(p1)
    }
    override fun onServiceDisconnected(p0: ComponentName?) {
    }
}
val intent = Intent(this, Student::class.java)
bindService(intent, connection, BIND_AUTO_CREATE)

  ```

如下是bindService的启动流程：

![](https://upload-images.jianshu.io/upload_images/22650779-cbe2246e6f0c5afd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1.bindService 发起端进程

### 1.1 ContextWrapper.bindService
  ```
public class ContextWrapper extends Context {

    public boolean bindService(Intent service, ServiceConnection conn,
                               int flags) {
        //mBase为ContentImpl
        return mBase.bindService(service, conn, flags);
    }
}
  ```
继续调用`ContentImpl.bindService`方法：
  ```
class ContextImpl extends Context {
    
    public boolean bindService(Intent service, ServiceConnection conn, int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
                getUser());
    }

   final @NonNull LoadedApk mPackageInfo;
    private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
                                      String instanceName, Handler handler, Executor executor, UserHandle user) {
        // 将ServiceConnection转换成Binder对象变量，用于进程间通信
        IServiceConnection sd;
        //...
        if (mPackageInfo != null) {
            // 将ServiceConnection转换成可跨进程
            if (executor != null) {
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
            } else {
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
            }
        } else {
            throw new RuntimeException("Not supported in system context");
        }
        validateServiceIntent(service);
        try {
            //...
            // 调用AMS.bindIsolatedService方法
            int res = ActivityManager.getService().bindIsolatedService(
                    mMainThread.getApplicationThread(), getActivityToken(), service,
                    service.resolveTypeIfNeeded(getContentResolver()),
                    sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
  ```
首先将ServiceConnection变量存储到可跨进程通信的Binder对象并赋值到`sd`变量，并把`sd`变量传给AMS中，后续AMS通过这个Binder通信。最后调用`AMS.bindIsolatedService`跨进程调用方法。在分析另一个AMS进程前，先分析本进程如何
  ```
public final class LoadedApk {

    @UnsupportedAppUsage
    public final IServiceConnection getServiceDispatcher(ServiceConnection c,
                                                         Context context, Handler handler, int flags) {
        return getServiceDispatcherCommon(c, context, handler, null, flags);
    }

    private IServiceConnection getServiceDispatcherCommon(ServiceConnection c,
                                                          Context context, Handler handler, Executor executor, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            //...
            // 创建ServiceDispatcher对象，并把ServiceConnection参数作为其变量之一
            sd = new ServiceDispatcher(c, context, executor, flags);
            //...
            // 返回ServiceDispatcher.InnerConnection内部类，其继承IServiceConnection.Stub，是个binder
            return sd.getIServiceConnection();
        }
    }
}

static final class ServiceDispatcher {
    // 返回的是此对象
    private final ServiceDispatcher.InnerConnection mIServiceConnection;
    // 存储ServiceConnection的变量
    private final ServiceConnection mConnection;
    private final Context mContext;
    private final Handler mActivityThread;
    private final Executor mActivityExecutor;
    private final ServiceConnectionLeaked mLocation;

    ServiceDispatcher(ServiceConnection conn,
                      Context context, Handler activityThread, int flags) {
        //创建InnerConnection对象,等会会返回该对象
        mIServiceConnection = new InnerConnection(this);
        //用户定义的ServiceConnection
        mConnection = conn;
        mContext = context;
        mActivityThread = activityThread;
        mActivityExecutor = null;
        mLocation = new ServiceConnectionLeaked(null);
        mLocation.fillInStackTrace();
        mFlags = flags;
    }

    // 返回的是mIServiceConnection变量，是Binder类
    IServiceConnection getIServiceConnection() {
        return mIServiceConnection;
    }

    //内部的Binder类
    private static class InnerConnection extends IServiceConnection.Stub {

        final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

        // 通过构造函数弱引用ServiceDispatcher对象，此对象有ServiceConnection变量
        InnerConnection(LoadedApk.ServiceDispatcher sd) {
            mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
        }

        // 调用ServiceDispatcher.connected()方法
        public void connected(ComponentName name, IBinder service, boolean dead)
                throws RemoteException {
            LoadedApk.ServiceDispatcher sd = mDispatcher.get();
            if (sd != null) {
                sd.connected(name, service, dead);
            }
        }
    }

    // ServiceDispatcher.connected()的方法
    public void connected(ComponentName name, IBinder service, boolean dead) {
        if (mActivityExecutor != null) {
            // 在线程池执行一个任务
            mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
        } else if (mActivityThread != null) {
            // 给主线程发送一个post一个任务
            mActivityThread.post(new RunConnection(name, service, 0, dead));
        } else {
            // 如果上述两个都为空，则执行doConnected方法
            doConnected(name, service, dead);
        }
    }

    private final class RunConnection implements Runnable {

        RunConnection(ComponentName name, IBinder service, int command, boolean dead) {
            mName = name;
            mService = service;
            mCommand = command;
            mDead = dead;
        }

        public void run() {
            if (mCommand == 0) {
                // mCommand为0 ,进入doConnected方法
                doConnected(mName, mService, mDead);
            } else if (mCommand == 1) {
                doDeath(mName, mService);
            }
        }

        final ComponentName mName;
        final IBinder mService;
        final int mCommand;
        final boolean mDead;
    }

    // 调用ServiceConnection.onServiceConnected()方法
    public void doConnected(ComponentName name, IBinder service, boolean dead) {
        ServiceDispatcher.ConnectionInfo old;
        ServiceDispatcher.ConnectionInfo info;

        synchronized (this) {
            if (mForgotten) {
                return;
            }
            old = mActiveConnections.get(name);
            if (old != null && old.binder == service) {
                return;
            }

            if (service != null) {
                info = new ConnectionInfo();
                info.binder = service;
                //创建死亡监听对象
                info.deathMonitor = new DeathMonitor(name, service);
                try {
                    //建立死亡通知
                    service.linkToDeath(info.deathMonitor, 0);
                    mActiveConnections.put(name, info);
                } catch (RemoteException e) {
                    mActiveConnections.remove(name);
                    return;
                }

            } else {
                mActiveConnections.remove(name);
            }

            if (old != null) {
                old.binder.unlinkToDeath(old.deathMonitor, 0);
            }
        }

        // 如果有旧服务，它现在已断开连接。
        if (old != null) {
            mConnection.onServiceDisconnected(name);
        }
        if (dead) {
            mConnection.onBindingDied(name);
        }
        //如果有新的可行服务，它现在已连接。
        if (service != null) {
            // 回调用户定义的ServiceConnection()
            mConnection.onServiceConnected(name, service);
        } else {
            // The binding machinery worked, but the remote returned null from onBind().
            mConnection.onNullBinding(name);
        }
    }

}
  ```
创建`LoadedApk.ServiceDispatcher`类的实例化对象，对象里面包含了一个Binder对象`LoadedApk.ServiceDispatcher.InnerConnection`,方法最后就是返回这个Binder对象。AMS也通过这个Binder对象通信调用`ServiceConnection.onServiceConnected()`方法。

## AMS处理bindService请求
首先调用AMS的`bindIsolatedService`方法
  ```
public class ActivityManagerService extends IActivityManager.Stub {

    final ActiveServices mServices;

    public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
                                   String resolvedType, IServiceConnection connection, int flags, String instanceName,
                                   String callingPackage, int userId) throws TransactionTooLargeException {
        // 空判断 ...
        // 调用ActiveServices.bindServiceLocked方法
        synchronized (this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, instanceName, callingPackage, userId);
        }
    }
}
  ```
AMS方法首先进行空判断，然后调用`ActiveServices`类的`bindServiceLocked()`方法：
  ```
public final class ActiveServices {

    int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
                          String resolvedType, final IServiceConnection connection, int flags,
                          String instanceName, String callingPackage, final int userId)
            throws TransactionTooLargeException {

        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        //查询发起端所对应的进程记录结构
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                            + " (pid=" + callingPid
                            + ") when binding service " + service);
        }

        ActivityServiceConnectionsHolder<ConnectionRecord> activity = null;
        //token不为空, 代表着发起方具有activity上下文
        if (token != null) {
            activity = mAm.mAtmInternal.getServiceConnectionsHolder(token);
            if (activity == null) {
                return 0;
            }
        }

        int clientLabel = 0;
        PendingIntent clientIntent = null;
        final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;

        ...
        //根据发送端所在进程的SchedGroup来决定是否为前台service
        final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;

        // 1. 根据传递进来Intent来检索相对应的服务,service变量就是Intent
        ServiceLookupResult res =
                retrieveServiceLocked(service, instanceName, resolvedType, callingPackage,
                        callingPid, callingUid, userId, true,
                        callerFg, isBindExternal, allowInstant);
        // 空检查
        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }

        //2. 查询到相应的Service记录
        ServiceRecord s = res.record;

        final long origId = Binder.clearCallingIdentity();
        try {

            // 调用ServiceRecord.retrieveAppBindingLocked方法
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            //创建对象ConnectionRecord,此处connection来自发起方
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent,
                    callerApp.uid, callerApp.processName, callingPackage);

            IBinder binder = connection.asBinder();
            ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
            if (clist == null) {
                clist = new ArrayList<>();
                mServiceConnections.put(binder, clist);
            }
            clist.add(c); // clist是ServiceRecord.connections的成员变量

            if ((flags & Context.BIND_AUTO_CREATE) != 0) {
                //更新当前service活动时间
                s.lastActivity = SystemClock.uptimeMillis();
                //3. 启动service，这个过程跟startService过程一致
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {
                    return 0;
                }
            }
          //.....
        return 1;
    }
  ```
bindServiceLocked方法主要做了三件事：
1. 调用`retrieveServiceLocked`方法，根据Intent解析寻找要启动的Service，并得到ServiceLookupResult对象实例。
2. 根据ServiceLookupResult对象实例得到`ServiceRecord`对象。
3. 调用`bringUpServiceLocked`方法，开始启动Service。

#### 1. `retrieveServiceLocked`根据Intent解析要启动的ServiceRecord
  ```
    private ServiceLookupResult retrieveServiceLocked(Intent service,
                                                      String instanceName, String resolvedType, String callingPackage,
                                                      int callingPid, int callingUid, int userId,
                                                      boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal,
                                                      boolean allowInstant) {
        ServiceRecord r = null;

        userId = mAm.mUserController.handleIncomingUser(callingPid, callingUid, userId,
                /* allowAll= */false, getAllowMode(service, callingPackage),
                /* name= */ "service", callingPackage);

        ServiceMap smap = getServiceMapLocked(userId);
        // 1. 根据Intent获取全类名
        final ComponentName comp;
        if (instanceName == null) {
            comp = service.getComponent();
        } else {
            final ComponentName realComp = service.getComponent();
            if (realComp == null) {
                throw new IllegalArgumentException("Can't use custom instance name '" + instanceName
                        + "' without expicit component in Intent");
            }
            comp = new ComponentName(realComp.getPackageName(),
                    realComp.getClassName() + ":" + instanceName);
        }

        // 2. 根据全类名在缓存中查找相应的ServiceRecord
        if (comp != null) {
            r = smap.mServicesByInstanceName.get(comp);
        }

        // ServiceRecord为空
        if (r == null) {
            try {
                // 3. 通过PKMS来查询相应的ResolveInfo
                ResolveInfo rInfo = mAm.getPackageManagerInternalLocked().resolveService(service,
                        resolvedType, flags, userId, callingUid);
                ServiceInfo sInfo = rInfo != null ? rInfo.serviceInfo : null;
                if (sInfo == null) {
                    return null;
                }
                //获取组件名
                ComponentName className = new ComponentName(
                        sInfo.applicationInfo.packageName, sInfo.name);
                if (userId > 0) {
                    //服务是否属于单例模式
                    if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                            sInfo.name, sInfo.flags)
                            && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
                        userId = 0;
                        smap = getServiceMapLocked(0);
                    }
                    sInfo = new ServiceInfo(sInfo);
                    sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
                }
                r = smap.mServicesByInstanceName.get(name);
                if (r == null && createIfNeeded) {
                    final Intent.FilterComparison filter
                            = new Intent.FilterComparison(service.cloneFilter());
                    //创建Restarter对象
                    final ServiceRestarter res = new ServiceRestarter();
                    final BatteryStatsImpl.Uid.Pkg.Serv ss;
                    final BatteryStatsImpl stats = mAm.mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        ss = stats.getServiceStatsLocked(
                                sInfo.applicationInfo.uid, name.getPackageName(),
                                name.getClassName());
                    }
                    
                    // 3.2 创建ServiceRecord对象
                    r = new ServiceRecord(mAm, ss, className, name, definingPackageName,
                            definingUid, filter, sInfo, callingFromFg, res);
                    r.mRecentCallingPackage = callingPackage;
                    res.setService(r);
                    smap.mServicesByInstanceName.put(name, r);
                    smap.mServicesByIntent.put(filter, r);

                    //确保该组件不再位于pending队列
                    for (int i = mPendingServices.size() - 1; i >= 0; i--) {
                        final ServiceRecord pr = mPendingServices.get(i);
                        if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid
                                && pr.instanceName.equals(name)) {
                            mPendingServices.remove(i);
                        }
                    }
                }
            } catch (RemoteException ex) {
                //运行在同一个进程，不会发生RemoteException
            }
        }
        if (r != null) {
            //各种权限检查，不满足条件则返回为null的service
            //...
            //4. 创建Service查询结果对象
            return new ServiceLookupResult(r, null);
        }
        return null;
    }

  ```
`retrieveServiceLocked`方法主要做了四件事：
1. 根据Intent数据获取Service的全类名
2. 根据全类名在缓存查找是否有`ServiceRecord`
3. 如果缓存中没有`ServiceRecord`，则根据PKMS查找对应的`ResolveInfo`,并创建一个`ServiceRecord`对象
4. 最后如果`ServiceRecord`对象不为空，则根据ServiceRecord创建一个`ServiceLookupResult`对象实例并返回。

#### 2. `bringUpServiceLocked`启动Service
  ```
    // 启动Service的方法
    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
                                        boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        if (r.app != null && r.app.thread != null) {
            // 如果service已经启动，那么多次启动Service时会多次调用service.onStartCommand()方法
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && mRestartingServices.contains(r)) {
            return null; //等待延迟重启的过程，则直接返回
        }

        // 启动service前，把service从重启服务队列中移除
        if (mRestartingServices.remove(r)) {
            clearRestartingIfNeededLocked(r);
        }

        // service正在启动，将delayed设置为false
        if (r.delayed) {
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        //确保拥有该服务的user已经启动，否则停止；
        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            bringDownServiceLocked(r);
            return msg;
        }

        //服务正在启动，设置package停止状态为false
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags & ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;
        if (!isolated) {
            // 1. 根据进程名和uid，查询ProcessRecord
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) {
                // 目标进程已存在
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    // 2. 调用realStartServiceLocked方法启动Service
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortInstanceName, e);
                }

            }
        } else {
            app = r.isolatedProc;
        }

        // 对于进程没有启动的情况
        if (app == null && !permissionsReviewRequired) {
            // 3. 通过AMS启动service所要运行的进程
            if ((app = mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                // 进程启动失败
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            r.delayedStop = false;
            if (r.startRequested) {
                //停止服务
                stopServiceLocked(r);
            }
        }

        return null;
    }

  ```
`bringUpServiceLocked`方法主要做了两件事：
1. 检查进程是否启动，如果启动了，则调用`realStartServiceLocked`方法开始真正启动Service。
2. 如果进程没启动，则先创建进程，进程创建完成后才调用`realStartServiceLocked`方法开始真正启动Service。 
继续查看`realStartServiceLocked`方法
  ```
  // 真是启动Service方法
    private final void realStartServiceLocked(ServiceRecord r,
                                              ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        //...
        // 1. 发送delay消息，若onCreate方法超时执行未取消，则引发ANR
        bumpServiceExecutingLocked(r, execInFg, "create");
        try {
            //...
            // 2. 通过ApplicationThread调用Service.onCreate方法
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app, "Died when creating service");
            throw e;
        } finally {
            //...
        }

        // 3. 检查是否需要执行onBind()方法
        requestServiceBindingsLocked(r, execInFg);
        //...
        //4. 服务 进入onStartCommand()
        sendServiceArgsLocked(r, execInFg, true);
        //...
    }

  ```
`realStartServiceLocked`方法主要做了四件事：
1. 调用`bumpServiceExecutingLocked()`方法发送一个延迟消息，为Service的onCreate执行时间装弹。
2. 调用`app.thread.scheduleCreateService()`方法，通过Service所在进程的`ApplicationThread`类，去创建Service并调用Service的`onCreate()`方法。
3. 调用`requestServiceBindingsLocked()`方法，检查是否需要执行Service的`onBind()`方法。
4. 调用`sendServiceArgsLocked()`方法，检查是否需要执行Service的`onStartCommand()`方法

##### bumpServiceExecutingLocked延迟发送ANR消息：
  ```
    static final int SERVICE_TIMEOUT = 20 * 1000;

    static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;

    // 发送一个延迟消息，消息没被取消，则执行ANR。fg参数为是否为前台Service
    private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        long now = SystemClock.uptimeMillis();
        if (r.executeNesting == 0) {
            //...
            if (r.app != null) {
                r.app.executingServices.add(r);
                r.app.execServicesFg |= fg;
                if (timeoutNeeded && r.app.executingServices.size() == 1) {
                    scheduleServiceTimeoutLocked(r.app);
                }
            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {
            r.app.execServicesFg = true;
            if (timeoutNeeded) {
                // 发送Handler
                scheduleServiceTimeoutLocked(r.app);
            }
        }
        r.executeFg |= fg;
        r.executeNesting++;
        r.executingStart = now;
    }

    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        //当超时后仍没有remove该SERVICE_TIMEOUT_MSG消息，则执行service Timeout流程
        // 在前台延迟20s,否则延迟200s
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }

  ```
如果是前台Service则延迟20s发送ANR消息，如果是后台Service则延迟200s发送ANR消息。

##### requestServiceBindingsLocked检查是否需要执行onBind()方法：
  ```
    private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
        // 通过bindService的启动方式，bindings一定不为空
        for (int i = r.bindings.size() - 1; i >= 0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }

    private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
                                                      boolean execInFg, boolean rebind) throws TransactionTooLargeException {
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {
                // 1. ANR装炸弹
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                // 2. 通过ApplicationThread调用Service的onBind()方法
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.getReportedProcState());
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            }
            //...
        }
        return true;
    }
  ```
如果`ServiceRecord.bindings`大小>0，则调用`requestServiceBindingLocked`方法，先通过`bumpServiceExecutingLocked`方法给onBind()方法发送延迟的ANR消息，然后调用` r.app.thread.scheduleBindService()`方法通过ApplicationThread类调用Service的`onBind()`方法。

**sendServiceArgsLocked检查是否需要执行onStartCommand方法**
  ```
    private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
                                             boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }

        ArrayList<ServiceStartArgs> args = new ArrayList<>();

        while (r.pendingStarts.size() > 0) {
            //...
            // 装弹，延迟发送一条消息，如果执行onStartCommand方法超过
            bumpServiceExecutingLocked(r, execInFg, "start");
            //...
            try {
                // 调用AT.scheduleServiceArgs,最终调用onStartCommand
                r.app.thread.scheduleServiceArgs(r, slice);
            }
        }

    }
  ```
如果`ServiceRecord.pendingStarts`大小>0则通过`r.app.thread.scheduleServiceArgs`调用Service的`onStartCommand`方法。

## ActivityThread创建并执行Service
由于`bindService`启动的Service并不会执行onStartCommand()方法，所以这里只分析`onCreate()、onBind()`方法：
#### 执行onCreate()
AMS通过调用`app.thread.scheduleCreateService()`方法去调用Service的`onCreate()`方法的执行。
  ```
 private class ApplicationThread extends IApplicationThread.Stub {

        public final void scheduleCreateService(IBinder token,
                                                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            // 通过Handler发送消息执行
            sendMessage(H.CREATE_SERVICE, s);
        }

    }

    public void handleMessage(Message msg) {
        switch (msg.what) {
            case CREATE_SERVICE:
                handleCreateService((CreateServiceData) msg.obj);
                break;
        }
    }

    // 创建对应的Service并执行onCreate()方法
    private void handleCreateService(CreateServiceData data) {
        //当应用处于后台即将进行GC，而此时被调回到活动状态，则跳过本次gc
        unscheduleGcIdler();
        // 1.1 获取LoadedApk
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        Service service = null;
        try {
            // 1.2 根据LoadedApk创建ContextImpl
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            // 2. 创建Application对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            // 获取ClassLoader
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            // 3. 通过反射创建Service对象
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
            context.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            context.setOuterContext(service);
            // 4. 调用service.attach绑定资源文件
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            // 5. 调用 service.onCreate()方法
            service.onCreate();
            mServices.put(data.token, service);
            try {
                // 6. onCreate()执行完成，拆弹过程，最终调用到ActiveServices.serviceDoneExecutingLocked方法
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            //...
        }
    }
  ```
通过Handler通信，最终会调用`ActivityThread. handleCreateService`方法，该方法主要6件事：
1. 通过`getPackageInfoNoCheck`方法创建一个`LoadedApk`实例对象。并通过LoadedApk对象创建一个`ContextImpl`对象。
2. 调用`packageInfo.makeApplication`创建一个`Application`对象
3. 通过反射创建对应的Service实例对象
4. 调用`service.attach`方法绑定资源文件
5. 调用`service.onCreate()`方法
6. 调用`ActivityManager.getService().serviceDoneExecuting`向AMS取消延迟发送的ANR消息。

#### 执行onBind()
AMS通过调用`r.app.thread.scheduleBindService()`方法去执行Service的`onBind()`方法。
  ```
public final class ActivityThread extends ClientTransactionHandler {

    private class ApplicationThread extends IApplicationThread.Stub {

        public final void scheduleBindService(IBinder token, Intent intent,
                                              boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            sendMessage(H.BIND_SERVICE, s);
        }

    }

    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BIND_SERVICE:
                handleBindService((BindServiceData) msg.obj);
                break;
        }
    }


    private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {

                try {
                    if (!data.rebind) {
                        // 1. 执行Service.onBind()回调方法
                        IBinder binder = s.onBind(data.intent);
                        // 2. 将onBind返回值传递回给AMS，
                        // 让其回调ServiceConnection方法，并取消ANR消息
                        ActivityManager.getService().publishService(
                                data.token, data.intent, binder);
                    } else {
                        // 取消ANR消息
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                }
                //...
            }
        }

    }
  ```
通过Handler通信最终调用到`ActivityThread. handleBindService`方法，该方法主要做两件事：
1. 调用`Service.onBind()`方法
2. 调用`ActivityManager.getService().publishService`通过AMS向发起方的ServiceConnection进行回调，并取消ANR的延迟消息。
  ```
    public class ActivityManagerService extends IActivityManager.Stub
            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

        final ActiveServices mServices;

        public void publishService(IBinder token, Intent intent, IBinder service) {
            // Refuse possible leaked file descriptors
            if (intent != null && intent.hasFileDescriptors() == true) {
                throw new IllegalArgumentException("File descriptors passed in Intent");
            }

            synchronized (this) {
                if (!(token instanceof ServiceRecord)) {
                    throw new IllegalArgumentException("Invalid service token");
                }
                mServices.publishServiceLocked((ServiceRecord) token, intent, service);
            }
        }
    }

    public final class ActiveServices {

        void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
            final long origId = Binder.clearCallingIdentity();
            try {
                if (r != null) {
                    Intent.FilterComparison filter
                            = new Intent.FilterComparison(intent);
                    IntentBindRecord b = r.bindings.get(filter);
                    if (b != null && !b.received) {
                        b.binder = service;
                        b.requested = true;
                        b.received = true;
                        ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                        for (int conni = connections.size() - 1; conni >= 0; conni--) {
                            ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
                            for (int i = 0; i < clist.size(); i++) {
                                ConnectionRecord c = clist.get(i);
                                try {
                                    //1. 调用发起方进程的LoadedApk.ServiceDispatcher.InnerConnection.connected方法
                                    c.conn.connected(r.name, service, false);
                                } catch (Exception e) {
                                }
                            }
                        }
                    }
                    // 2. 取消ANR消息
                    serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
                }
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
        }
    }

  ```
最终调用到`ActiveServices.publishServiceLocked()`方法，向发起方的进行回调，并取消的ANR的延迟消息。

## 总结：
`bingService`会执行的方法有 `onCreate、onBind、onUnBind、onDestory`,其中`onBind()`方法会返回一个`Binder`对象。
- 发起方在调用bingService时要传入一个`ServiceConnection`实例对象，并把这个对象封装在一个Binder对象中，便于AMS回调。
- AMS则通过`Intent`找到对应的Service记录，如果Service的进程没启动则先创建进程，进程启动后依此通过进程的`ActivityThread`创建并调用Service的`onCreate、onBind`方法，并在调用方法前发送一个延迟的ANR消息
- Service进程的ActivityThread在`onCreate()`时先通过反射创建Service，并调用Service的`attach、onCreate`方法。在onBind()时先调用Service的`onBinder`方法，并向AMS发送i个给发起方的`ServiceConnection`的回调。在执行完方法后发送一个取消执行ANR的延迟消息。










