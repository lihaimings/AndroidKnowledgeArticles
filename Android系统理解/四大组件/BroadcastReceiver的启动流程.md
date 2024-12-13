> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/core/java/android/content/ContextWrapper.java
/frameworks/base/core/java/android/app/ContextImpl.java
 /frameworks/base/core/java/android/app/LoadedApk.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
/frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
/frameworks/base/core/java/android/app/ActivityThread.java
  ```
## 广播的注册
广播的注册分为：**静态注册和动态注册**。静态注册是在AndroidManifest文件中注册广播接收器BroadcastReceiver和intent-filter,然后在安装的时候通过PKMS进行解析。动态注册则通过Context.registerReceiver向AMS注册广播接收器BroadcastReceiver和IntentFilter。
- 通过继承BroadcastReceiver创建一个广播接收器


  ```
class MyBroadcastReceiver : BroadcastReceiver() {

    override fun onReceive(p0: Context?, p1: Intent?) {
    }

}
  ```
- 静态注册广播接收器
  ```
        <receiver
            android:name=".MyBroadcastReceiver"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="com.example.blogBroadcasts.MY_BROADCAS" />
            </intent-filter>
        </receiver>
  ```
- 动态注册广播接收器


  ```
// 动态注册广播
val intentFilter = IntentFilter(ACTION)
val broadcastReceiver = MyBroadcastReceiver()
registerReceiver(broadcastReceiver, intentFilter)
  ```
以下讲讲动态注册广播接收器的过程：

### 动态注册广播接收器
动态注册是把BroadcastReceiver包装成一个Binder对象，然后创建一个包含BroadcastReceiver和IntentFilter的`BroadcastFilter`对象，然后把`BroadcastFilter`对象加入到AMS的`mReceiverResolver`变量中。动态的注册的大致流程图如下：

![](https://upload-images.jianshu.io/upload_images/22650779-16cccdc2d8084d12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 通过Context的注册处理：
  ```
public class ContextWrapper extends Context {

    Context mBase;

    @Override
    public Intent registerReceiver(
            BroadcastReceiver receiver, IntentFilter filter) {
        // 调用ContextImpl.registerReceiver()方法
        return mBase.registerReceiver(receiver, filter);
    }

    @Override
    public Intent registerReceiver(
            BroadcastReceiver receiver, IntentFilter filter, int flags) {
        return mBase.registerReceiver(receiver, filter, flags);
    }

    @Override
    public Intent registerReceiver(
            BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
        return mBase.registerReceiver(receiver, filter, broadcastPermission,
                scheduler);
    }

    @Override
    public Intent registerReceiver(
            BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler, int flags) {
        return mBase.registerReceiver(receiver, filter, broadcastPermission,
                scheduler, flags);
    }

}

class ContextImpl extends Context {

    final LoadedApk mPackageInfo;

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        return registerReceiver(receiver, filter, null, null);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
                                   int flags) {
        return registerReceiver(receiver, filter, null, null, flags);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
                                   String broadcastPermission, Handler scheduler) {
        // 最后调用registerReceiverInternal()方法
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext(), 0);
    }

    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
                                   String broadcastPermission, Handler scheduler, int flags) {
        // 最后调用registerReceiverInternal()方法
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext(), flags);
    }

    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
                                            IntentFilter filter, String broadcastPermission,
                                            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        // 将BroadcastReceiver存储到LoadedApk.ReceiverDispatcher对象里，
        // 并通过LoadedApk.ReceiverDispatcher.InnerReceiver的Binder对象和AMS进行通信
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                // 获取IntentReceiver Binder对象
                rd = mPackageInfo.getReceiverDispatcher(
                        receiver, context, scheduler,
                        mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                // 获取IntentReceiver Binder对象
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            // 调用AMS.registerReceiverWithFeature
            final Intent intent = ActivityManager.getService().registerReceiverWithFeature(
                    mMainThread.getApplicationThread(), mBasePackageName, getAttributionTag(), rd,
                    filter, broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

}

public final class LoadedApk {

    public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
                                                 Context context, Handler handler,
                                                 Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            // 先尝试从缓存中获取，缓存中没有则new一个
            LoadedApk.ReceiverDispatcher rd = null;

            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            // 获取内部类Binder对象
            return rd.getIIntentReceiver();
        }
    }

    static final class ReceiverDispatcher {

        final IIntentReceiver.Stub mIIntentReceiver;
        final BroadcastReceiver mReceiver;
        @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 115609023)
        final Context mContext;
        final Handler mActivityThread;
        final Instrumentation mInstrumentation;
        final boolean mRegistered;
        final IntentReceiverLeaked mLocation;
        RuntimeException mUnregisterLocation;
        boolean mForgotten;

        ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                           Handler activityThread, Instrumentation instrumentation,
                           boolean registered) {
            if (activityThread == null) {
                throw new NullPointerException("Handler must not be null");
            }

            mIIntentReceiver = new InnerReceiver(this, !registered);
            mReceiver = receiver;
            mContext = context;
            mActivityThread = activityThread;
            mInstrumentation = instrumentation;
            mRegistered = registered;
            mLocation = new IntentReceiverLeaked(null);
            mLocation.fillInStackTrace();
        }

        // 获取InnerReceiver的Binder类
        IIntentReceiver getIIntentReceiver() {
            return mIIntentReceiver;
        }

        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }

            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                                       Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                final LoadedApk.ReceiverDispatcher rd;
                //...
                // 调用ReceiverDispatcher.performReceive方法
                if (rd != null) {
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                }
            }
        }


    }


}
  ```
将`BroadcastReceiver`包装成一个Binder对象，用于与AMS的回调通信，最后调用`AMS.registerReceiverWithFeature`方法。

#### AMS处理动态注册广播
  ```
public class ActivityManagerService extends IActivityManager.Stub {

    public Intent registerReceiverWithFeature(IApplicationThread caller, String callerPackage,
                                              String callerFeatureId, IIntentReceiver receiver, IntentFilter filter,
                                              String permission, int userId, int flags) {
        //...
        synchronized (this) {
            // 根据IntentReceiver查看缓存中是否有ReceiverList
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            // 创建一个新的ReceiverList，并把IntentReceiver传入其中
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                mRegisteredReceivers.put(receiver.asBinder(), rl); // 增加到缓存中
            }
            // 创建一个BroadcastFilter对象，并把ReceiverList对象和IntentFilter对象传入其中
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage, callerFeatureId,
                    permission, callingUid, userId, instantApp, visibleToInstantApps);
            // 向ReceiverList对象增加BroadcastFilter对象
            rl.add(bf);
            // 向mReceiverResolver变量增加BroadcastFilter对象
            mReceiverResolver.addFilter(bf);
            
            // ...
            return sticky;
        }
    }

}
  ```
在`registerReceiverWithFeature`方法中省略了很多代码，主要代码中就是创建一个有`IntentFilter和IntentReceiver`的`BroadcastFilter`对象，然后将`BroadcastFilter`对象加入到AMS的变量`mReceiverResolver`中。

## 广播的发送
广播的发送是通过Activity/Service的Context的`sendBroadcast、sendOrderedBroadcast`方法将Intent发送广播:
  ```
    val intent = Intent(this, MyBroadcastReceiver::class.java)
    intent.action = ACTION
    sendBroadcast(intent)
  ```

![](https://upload-images.jianshu.io/upload_images/22650779-7d3789fc8cdc652f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Context处理广播发送
  ```
public class ContextWrapper extends Context {                                                             
                                                                                                          
    Context mBase;                                                                                        
                                                                                                          
    @Override                                                                                             
    public void sendBroadcast(Intent intent) {                                                            
        mBase.sendBroadcast(intent);                                                                      
    }                                                                                                     
                                                                                                          
    @Override                                                                                             
    public void sendBroadcast(Intent intent, String receiverPermission) {                                 
        mBase.sendBroadcast(intent, receiverPermission);                                                  
    }                                                                                                     
}                                                                                                         
                                                                                                          
class ContextImpl extends Context {                                                                       
                                                                                                          
    public void sendBroadcast(Intent intent) {                                                            
        warnIfCallingFromSystemProcess();                                                                 
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());                           
        try {                                                                                             
            intent.prepareToLeaveProcess(this);                                                           
            // 调用AMS的broadcastIntentWithFeature方法                                                         
            ActivityManager.getService().broadcastIntentWithFeature(                                      
                    mMainThread.getApplicationThread(), getAttributionTag(), intent, resolvedType,        
                    null, Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false,       
                    false, getUserId());                                                                  
        } catch (RemoteException e) {                                                                     
            throw e.rethrowFromSystemServer();                                                            
        }                                                                                                 
    }                                                                                                     
                                                                                                          
    @Override                                                                                             
    public void sendBroadcast(Intent intent, String receiverPermission) {                                 
        warnIfCallingFromSystemProcess();                                                                 
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());                           
        String[] receiverPermissions = receiverPermission == null ? null                                  
                : new String[]{receiverPermission};                                                       
        try {                                                                                             
            intent.prepareToLeaveProcess(this);                                                           
            ActivityManager.getService().broadcastIntentWithFeature(                                      
                    mMainThread.getApplicationThread(), getAttributionTag(), intent, resolvedType,        
                    null, Activity.RESULT_OK, null, null, receiverPermissions,                            
                    AppOpsManager.OP_NONE, null, false, false, getUserId());                              
        } catch (RemoteException e) {                                                                     
            throw e.rethrowFromSystemServer();                                                            
        }                                                                                                 
    }                                                                                                     
                                                                                                          
}                                                                                                         
                                                                                                         
  ```
最后调用`AMS`的`broadcastIntentWithFeature`方法。

#### AMS处理发送广播
我们发送广播的类型可分为：无序广播(并行)、有序广播(有序)。对于动态注册的广播接收器如果接收到的是并行广播则并行执行，如果是串行广播则串行执行。如果是静态注册的工广播接收器则无论发送的广播是否为并行还是串行，都按串行执行。

  ```
public class ActivityManagerService extends IActivityManager.Stub {

    public final int broadcastIntentWithFeature(IApplicationThread caller, String callingFeatureId,
                                                Intent intent, String resolvedType, IIntentReceiver resultTo,
                                                int resultCode, String resultData, Bundle resultExtras,
                                                String[] requiredPermissions, int appOp, Bundle bOptions,
                                                boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized (this) {
            intent = verifyBroadcastLocked(intent);

            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();

            final long origId = Binder.clearCallingIdentity();
            try {
                return broadcastIntentLocked(callerApp,
                        callerApp != null ? callerApp.info.packageName : null, callingFeatureId,
                        intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                        requiredPermissions, appOp, bOptions, serialized, sticky,
                        callingPid, callingUid, callingUid, callingPid, userId);
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
        }
    }


    @GuardedBy("this")
    final int broadcastIntentLocked(ProcessRecord callerApp,
                                    String callerPackage, String callerFeatureId, Intent intent, String
                                            resolvedType,
                                    IIntentReceiver resultTo, int resultCode, String resultData,
                                    Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
                                    boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
                                    int realCallingPid, int userId) {
        return broadcastIntentLocked(callerApp, callerPackage, callerFeatureId, intent,
                resolvedType, resultTo, resultCode, resultData, resultExtras, requiredPermissions,
                appOp, bOptions, ordered, sticky, callingPid, callingUid, realCallingUid,
                realCallingPid, userId, false /* allowBackgroundActivityStarts */,
                null /*broadcastWhitelist*/);
    }

    @GuardedBy("this")
    final int broadcastIntentLocked(ProcessRecord callerApp, String callerPackage,
                                    @Nullable String callerFeatureId, Intent intent, String resolvedType,
                                    IIntentReceiver resultTo, int resultCode, String resultData,
                                    Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
                                    boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
                                    int realCallingPid, int userId, boolean allowBackgroundActivityStarts,
                                    @Nullable int[] broadcastWhitelist) {

        intent = new Intent(intent);
        //增加该flag，则广播不会发送给已停止的package
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
        //...
        // ...广播权限验证
        // ... 处理系统相关广播
        // ...增加sticky广播

        // 通过Intent查询到对应的广播接收器
        List receivers = null; // 静态广播接收器
        List<BroadcastFilter> registeredReceivers = null; //动态广播接收器
        //当允许静态接收者处理该广播，则通过PKMS根据Intent查询相应的静态receivers
        if ((intent.getFlags() & Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        if (intent.getComponent() == null) {
            if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
             ...
            } else {
                // 查询相应的动态注册的广播
                registeredReceivers = mReceiverResolver.queryIntent(intent,
                        resolvedType, false, userId);
            }
        }

        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        // 处理并行广播，并且只给动态广播接收器并行处理
        if (!ordered && NR > 0) {
            // 根据Intent查询是前台还是后台BroadcastQueue
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            // 创建一个BroadcastRecord，参数有Intent和registeredReceivers
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp, callerPackage,
                    callerFeatureId, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
                    resultCode, resultData, resultExtras, ordered, sticky, false, userId,
                    allowBackgroundActivityStarts, timeoutExempt);
            if (!replaced) {
                // 将BroadcastRecord加入到mParallelBroadcasts队列
                queue.enqueueParallelBroadcastLocked(r);
                // 执行队列
                queue.scheduleBroadcastsLocked();
            }
            // 如果是并行广播，执行完成后动态广播接收器清空处理
            registeredReceivers = null;
            NR = 0;
        }

        //...

        //如果此时registeredReceivers不为null，则表明这是一个串行广播
        // 则将registeredReceivers合并到receivers一起执行串行处理
        int NT = receivers != null ? receivers.size() : 0;
        int it = 0;
        ResolveInfo curt = null;
        BroadcastFilter curr = null;
        while (it < NT && ir < NR) {
            if (curt == null) {
                curt = (ResolveInfo) receivers.get(it);
            }
            if (curr == null) {
                curr = registeredReceivers.get(ir);
            }
            if (curr.getPriority() >= curt.priority) {
                receivers.add(it, curr);
                ir++;
                curr = null;
                it++;
                NT++;
            } else {
                it++;
                curt = null;
            }
        }
        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }

        // receivers执行串行广播
        if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            //根据intent的flag来判断前台队列或者后台队列
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            //创建BroadcastRecord
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
            if (!replaced) {
                //将BroadcastRecord加入到有序广播队列
                queue.enqueueOrderedBroadcastLocked(r);
                //处理广播
                queue.scheduleBroadcastsLocked();
            }
        }

        return ActivityManager.BROADCAST_SUCCESS;
    }

}                                                                    
                                                                                                                                  
  ```
在`AMS`的`broadcastIntentLocked`方法中
- 通过`Intent`去`PKMS`和`AMS. mReceiverResolver`变量查询到对应的广播接收器，其中变量receivers存储静态接收器，registeredReceivers变量存储动态接收器。
- 如果发送的是并行广播，则查看是否有对应的动态广播接收器，并创建一个拥有Intent和registeredReceivers的`BroadcastRecord`对象，并保存在BroadcastQueue.mParallelBroadcasts变量中，最终执行`queue.scheduleBroadcastsLocked()`处理广播，并把registeredReceivers信息置null。
- 如果此时registeredReceivers不为null，说明发送的是串行广播，则把registeredReceivers合并到receivers变量，一起串行执行。
- 对receivers进行串行执行，创建一个拥有Intent和receivers的`BroadcastRecord`对象，并保存到队列的mOrderedBroadcasts变量中，最终执行`queue.scheduleBroadcastsLocked()`处理广播。

#### BroadcastQueue处理广播
  ```
public final class BroadcastQueue {


    public void scheduleBroadcastsLocked() {
        if (mBroadcastsScheduled) {
            return;
        }
        // 发送BROADCAST_INTENT_MSG类型消息
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;

    }

    private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                // 执行此消息
                case BROADCAST_INTENT_MSG: {
                    processNextBroadcast(true);
                }
                break;
            }
        }
    }


    final void processNextBroadcast(boolean fromMsg) {
        synchronized (mService) {
            processNextBroadcastLocked(fromMsg, false);
        }
    }

    final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        BroadcastRecord r;
        //...
        //处理并行广播
        while (mParallelBroadcasts.size() > 0) {
            //把mParallelBroadcasts队列的BroadcastRecord执行完
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();
            // r.receivers就是AMS的registeredReceivers变量
            final int N = r.receivers.size();

            for (int i = 0; i < N; i++) {
                Object target = r.receivers.get(i);
                //分发广播给已注册的receiver
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter) target, false, i);
            }
            addBroadcastToHistoryLocked(r);
        }

        // 处理当前有序广播
        do {
            // 获取BroadcastRecord
            final long now = SystemClock.uptimeMillis();
            r = mDispatcher.getNextBroadcastLocked(now);

            //没有更多的广播等待处理
            if (r == null) {
                mDispatcher.scheduleDeferralCheckLocked(false);
                mService.scheduleAppGcsLocked();
                if (looped) {
                    mService.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_START_RECEIVER);
                }
                if (mService.mUserController.mBootCompleted && mLogLatencyMetrics) {
                    mLogLatencyMetrics = false;
                }
                return;
            }

            boolean forceReceive = false;
            // 获取Receivers的大小
            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;

            //当广播处理时间超时，则强制结束这条广播
            if (mService.mProcessesReady && !r.timeoutExempt && r.dispatchTime > 0) {
                if ((numReceivers > 0) &&
                        (now > r.dispatchTime + (2 * mConstants.TIMEOUT * numReceivers))) {
                    broadcastTimeoutLocked(false); // forcibly finish this broadcast
                    forceReceive = true;
                    r.state = BroadcastRecord.IDLE;
                }
            }
            //...
            if (r.receivers == null || r.nextReceiver >= numReceivers
                    || r.resultAbort || forceReceive) {
                if (r.resultTo != null) {
                    //...
                    //处理广播消息消息，调用到onReceive()
                    performReceiveLocked(r.callerApp, r.resultTo,
                            new Intent(r.intent), r.resultCode,
                            r.resultData, r.resultExtras, false, false, r.userId);
                    //...
                }

                //...
                mDispatcher.retireBroadcastLocked(r);
                r = null;
                looped = true;
                continue;
            }
            //..
        } while (r == null);


        //获取下一个receiver的index
        int recIdx = r.nextReceiver++;

        r.receiverTime = SystemClock.uptimeMillis();
        if (recIdx == 0) {
            r.dispatchTime = r.receiverTime;
            r.dispatchClockTime = System.currentTimeMillis();
        }
        if (!mPendingBroadcastTimeoutMessage) {
            long timeoutTime = r.receiverTime + mTimeoutPeriod;
            //设置广播超时时间，发送BROADCAST_TIMEOUT_MSG
            setBroadcastTimeoutLocked(timeoutTime);
        }

        final BroadcastOptions brOptions = r.options;
         //获取下一个广播接收者
        final Object nextReceiver = r.receivers.get(recIdx);

        if (nextReceiver instanceof BroadcastFilter) {
            //对于动态注册的广播接收者，deliverToRegisteredReceiverLocked处理广播
            BroadcastFilter filter = (BroadcastFilter) nextReceiver;
            deliverToRegisteredReceiverLocked(r, filter, r.ordered);
            if (r.receiver == null || !r.ordered) {
                r.state = BroadcastRecord.IDLE;
                scheduleBroadcastsLocked();
            } else {
                ...
            }
            return;
        }

        //对于静态注册的广播接收者
        ResolveInfo info = (ResolveInfo) nextReceiver;
        ComponentName component = new ComponentName(
                info.activityInfo.applicationInfo.packageName,
                info.activityInfo.name);
           ...
         //执行各种权限检测，此处省略，当权限不满足时skip=true

        if (skip) {
            r.receiver = null;
            r.curFilter = null;
            r.state = BroadcastRecord.IDLE;
            scheduleBroadcastsLocked();
            return;
        }

        r.state = BroadcastRecord.APP_RECEIVE;
        String targetProcess = info.activityInfo.processName;
        r.curComponent = component;
        final int receiverUid = info.activityInfo.applicationInfo.uid;
        if (r.callingUid != Process.SYSTEM_UID && isSingleton
                && mService.isValidSingletonCall(r.callingUid, receiverUid)) {
            info.activityInfo = mService.getActivityInfoForUser(info.activityInfo, 0);
        }
        r.curReceiver = info.activityInfo;
        ...
        //Broadcast正在执行中，stopped状态设置成false
        AppGlobals.getPackageManager().setPackageStoppedState(
                r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
        //该receiver所对应的进程已经运行，则直接处理
        ProcessRecord app = mService.getProcessRecordLocked(targetProcess,
                info.activityInfo.applicationInfo.uid, false);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(info.activityInfo.packageName,
                        info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                processCurBroadcastLocked(r, app);
                return;
            } catch (RemoteException e) {
            } catch (RuntimeException e) {
                finishReceiverLocked(r, r.resultCode, r.resultData, r.resultExtras, r.resultAbort, false);
                scheduleBroadcastsLocked();
                r.state = BroadcastRecord.IDLE; //启动receiver失败则重置状态
                return;
            }
        }

        //该receiver所对应的进程尚未启动，则创建该进程
        if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                == null) {
            //创建失败，则结束该receiver
            finishReceiverLocked(r, r.resultCode, r.resultData,
                    r.resultExtras, r.resultAbort, false);
            scheduleBroadcastsLocked();
            r.state = BroadcastRecord.IDLE;
            return;
        }
        mPendingBroadcast = r;
        mPendingBroadcastRecvIndex = recIdx;
        //...
    }

    private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
                                                   BroadcastFilter filter, boolean ordered, int index) {
        //...
        // 调用performReceiveLocked方法
        performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                new Intent(r.intent), r.resultCode, r.resultData,
                r.resultExtras, r.ordered, r.initialSticky, r.userId);

        //...
    }

    void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
                              Intent intent, int resultCode, String data, Bundle extras,
                              boolean ordered, boolean sticky, int sendingUser)
            throws RemoteException {
        if (app != null) {
            // 如果ProcessRecord != null，且ApplicationThread不为空
            if (app.thread != null) {
                try {
                    // 调用ApplicationThread.scheduleRegisteredReceiver
                    app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                            data, extras, ordered, sticky, sendingUser, app.getReportedProcState());
                } catch (RemoteException ex) {
                    synchronized (mService) {
                        app.scheduleCrash("can't deliver broadcast");
                    }
                    throw ex;
                }
            } else {
                // Application has died. Receiver doesn't exist.
                throw new RemoteException("app.thread must not be null");
            }
        } else {
            // 如果ProcessRecord为空
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }

}
  ```
BroadcastQueue就是将队列中列表拿出来执行，最后调用`app.thread.scheduleRegisteredReceiver`对广播进行回调。

#### 回调广播
  ```
private class ApplicationThread extends IApplicationThread.Stub {

    public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                                           int resultCode, String dataStr, Bundle extras, boolean ordered,
                                           boolean sticky, int sendingUser, int processState) throws RemoteException {
        updateProcessState(processState, false);
        // 调用IntentReceiver的performReceive方法
        receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                sticky, sendingUser);
    }

}

public final class LoadedApk {

    static final class ReceiverDispatcher {

        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }

            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                                       Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                final LoadedApk.ReceiverDispatcher rd;
                //...

                if (rd != null) {
                    // 调用ReceiverDispatcher的performReceive方法
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                }
                //...
            }
        }

        // ReceiverDispatcher的performReceive方法
        public void performReceive(Intent intent, int resultCode, String data,
                                   Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            //...
            // 通过Handler执行args的Runnable
            if (intent == null || !mActivityThread.post(args.getRunnable())) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManager.getService();

                    args.sendFinished(mgr);
                }
            }
        }

        final class Args extends BroadcastReceiver.PendingResult {

            public final Runnable getRunnable() {
                return () -> {
                    final BroadcastReceiver receiver = mReceiver;
                    //...
                    try {
                        ClassLoader cl = mReceiver.getClass().getClassLoader();
                        intent.setExtrasClassLoader(cl);
                        intent.prepareToEnterProcess();
                        setExtrasClassLoader(cl);
                        receiver.setPendingResult(this);
                        // 调用BroadcastReceiver的onReceive方法
                        receiver.onReceive(mContext, intent);
                    } catch (Exception e) {
                        //...
                    }

                    if (receiver.getPendingResult() != null) {
                        finish();
                    }
                };
            }
        }


    }
  ```




