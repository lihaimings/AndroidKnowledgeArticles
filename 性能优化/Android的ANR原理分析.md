## 大概
Android的ANR主要有两种方式：
`1. 通过handler的延迟机制触发ANR`
`2. Input事件触发ANR`
**Service、BroadcastReceiver、ContentProvider**都是通过Hander机制触发ANR。

ANR的发生的场景有：
- service timeout:前台服务在20s未执行完，后台服务200s未执行完。
- BroadcastQueue timeout:前台广播在10s未执行完，后台广播在60s未执行完。
- ContentProvider timeout: ContentProvider在发布时超过10s未执行完。
- InputDispatching Timeout：输入分发事件超过5s未执行完。

ANR的过程总体就是：装炸弹、拆炸弹、引爆炸弹

## 1. Service Timeout
在文章[startService启动流程](https://www.jianshu.com/p/17cdbb502f34)可以知道Service的生命周期和启动流程。
![Service的生命周期](https://upload-images.jianshu.io/upload_images/22650779-5c29d79ff6c34108.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![启动流程](https://upload-images.jianshu.io/upload_images/22650779-305324aad8892e9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 装弹
在`ActiveServices.realStartServiceLocked()`方法中开始真正执行Service的生命周期方法，并开始装炸弹的开始。
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

  }

// 通过Handler发送延迟时间，到时间内没被取消则抛ANR异常
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    ....
    // 发送Handler
    scheduleServiceTimeoutLocked(r.app);
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
  static final int SERVICE_TIMEOUT = 20 * 1000;
  static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;

```
在执行生命周期的方法前会通过`bumpServiceExecutingLocked()`方法进行装炸弹，通过Handler机制发送一个标志为`SERVICE_TIMEOUT_MSG`的延迟消息，如果是前台则20s,是后台则200s执行。

### 拆弹
在执行完Service的生命周期方法后就会执行拆弹，比如`onCreate()`方法在`Application.handleCreateService()`执行完毕：
```
 private class ApplicationThread extends IApplicationThread.Stub {

  // 创建对应的Service并执行onCreate()方法
  private void handleCreateService(CreateServiceData data) {
   
      try {
          ....
          // 调用service.attach绑定资源文件
          service.attach(context, this, data.info.name, data.token, app,
                  ActivityManager.getService());
          // 调用 service.onCreate()方法
          service.onCreate();
        
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


   // AMS
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized (this) {
            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.serviceDoneExecutingLocked((ServiceRecord) token, type, startId, res);
        }
    }


    // ActiveServices
    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
                                            boolean finishing) {
        //...
        // 取消SERVICE_TIMEOUT_MSG消息，拆出炸弹
        mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
        //...       
    }

```
如果在生命周期方法内执行完，会回调AMS的方法，解除`SERVICE_TIMEOUT_MSG`延迟发送的消息。

### 引爆弹
如果Service方法在规定时间内没有执行完成，则会执行`AMS.MainHandler`的`SERVICE_TIMEOUT_MSG`类型的消息：
```
    // AMS
    final class MainHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case SERVICE_TIMEOUT_MSG: {
                    mServices.serviceTimeout((ProcessRecord) msg.obj);
                }
                break;
            }
        }
    }

    // ActiveServices
    void serviceTimeout(ProcessRecord proc) {
        // 如果超时，调用appNotResponding()方法
        if (anrMessage != null) {
            mAm.mAnrHelper.appNotResponding(proc, anrMessage);
        }
    }

```
如果超时则会触发ANR,调用` mAm.mAnrHelper.appNotResponding()`方法。

## 2. BroadcastReceiver Timeout
![image.png](https://upload-images.jianshu.io/upload_images/22650779-befe1b7302e3ee99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 装弹
在文章[广播的注册、发送原理流程](https://www.jianshu.com/p/6fbc1a43c837)可以了解，发送广播后调用之行对应的广播接收器的方法，对应的方法在`BroadcastQueue.processNextBroadcast()`：
```
  public final class BroadcastQueue {

        final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
            BroadcastRecord r;

            // 处理当前有序广播
            do {
                // 获取BroadcastRecord
                final long now = SystemClock.uptimeMillis();
                r = mDispatcher.getNextBroadcastLocked(now);

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

                    //拆炸弹
                    cancelBroadcastTimeoutLocked();
                    r = null;
                    looped = true;
                    continue;
                }
                //..
            } while (r == null);


            //获取下一个receiver的index
            int recIdx = r.nextReceiver++;
            if (!mPendingBroadcastTimeoutMessage) {
                long timeoutTime = r.receiverTime + mTimeoutPeriod;
                //装炸弹，设置广播超时时间，发送BROADCAST_TIMEOUT_MSG
                setBroadcastTimeoutLocked(timeoutTime);
            }

            //...
        }

        // 装弹
        final void setBroadcastTimeoutLocked(long timeoutTime) {
            if (!mPendingBroadcastTimeoutMessage) {
                Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
                mHandler.sendMessageAtTime(msg, timeoutTime);
                mPendingBroadcastTimeoutMessage = true;
            }
        }

    }
```
通过`setBroadcastTimeoutLocked()`方法进行装弹。

### 拆弹
通过上面的`processNextBroadcastLocked()`方法可知，调用`cancelBroadcastTimeoutLocked()`进行拆弹。
```
    // 拆弹
    final void cancelBroadcastTimeoutLocked() {
        if (mPendingBroadcastTimeoutMessage) {
            mHandler.removeMessages(BROADCAST_TIMEOUT_MSG, this);
            mPendingBroadcastTimeoutMessage = false;
        }
    }
```

### 引爆弹
通过`BroadcastHandler`的`BROADCAST_TIMEOUT_MSG`类型的消息的执行，进行引爆炸弹。
```
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case BROADCAST_TIMEOUT_MSG: {
                synchronized (mService) {
                    // 调用broadcastTimeoutLocked()方法
                    broadcastTimeoutLocked(true);
                }
            }
            break;
        }
    }


    final void broadcastTimeoutLocked(boolean fromMsg) {
        ...
        // 如果超时，调用AMS.mAnrHelper.appNotResponding
        if (!debugging && anrMessage != null) {
            mService.mAnrHelper.appNotResponding(app, anrMessage);
        }
    }

```
最后也是调用`AMS.mAnrHelper.appNotResponding()`方法

## 3. ContentProvider Timeout
### 装弹
ContentProvider的注册在启动进程的时候就开始执行，在注册的过程中会向AMS绑定Application，如果有ContentProvider就装弹，方法在`AMS.attachApplicationLocked()`方法中：
```
    // AMS
    private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
                                            int pid, int callingUid, long startSeq) {

        //...
        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            // 装弹
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg,
                    ContentResolver.CONTENT_PROVIDER_PUBLISH_TIMEOUT_MILLIS);
        }
    }

    public static final int CONTENT_PROVIDER_PUBLISH_TIMEOUT_MILLIS = 10 * 1000;

```
在绑定Application时，会判断是否有ContentProvider时会装炸弹进行一个`CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG`消息的延迟发送。
### 拆弹
在`AT.installContentProviders()`安装完后会调用`AMS.publishContentProviders()`方法进行拆弹。
```
    public final void publishContentProviders(IApplicationThread caller,
                                              List<ContentProviderHolder> providers) {
        ...
        // 拆弹，移除CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG消息
        mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r)

    }
```

### 引爆弹
如果消息没被移除则引爆炸弹,`CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG`的handler在`AMS.MainHandler`中，
```
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG: {
                ProcessRecord app = (ProcessRecord) msg.obj;
                synchronized (ActivityManagerService.this) {
                    processContentProviderPublishTimedOutLocked(app);
                }
            }
            break;
        }
    }

    private final void processContentProviderPublishTimedOutLocked(ProcessRecord app) {
        //移除死亡的provider
        cleanupAppInLaunchingProvidersLocked(app, true);
        //移除mProcessNames中的相应对象
        mProcessList.removeProcessLocked(app, false, true,
                ApplicationExitInfo.REASON_INITIALIZATION_FAILURE,
                ApplicationExitInfo.SUBREASON_UNKNOWN,
                "timeout publishing content providers");
    }
```

## 4. InputDispatching  Timeout
`InputReader` 不断的从`EventHub`中监听是否有Input事件，InputReader把事件分发给`InputDispatcher`。
InputDispatcher调用`dispatchOnce()`方法开始把事件分发给对应的View,就从InputDispatcher的分发开始监控ANR，InputDispatcher的ANR区间是查找窗口`findFocusedWindowTargetsLocked()`方法到`resetANRTimeoutsLocked()`重置方法。


```
    void InputDispatcher::dispatchOnce() {
        ...
        // 调用dispatchOnceInnerLocked进行分析
        dispatchOnceInnerLocked(&nextWakeupTime);
    }

    void InputDispatcher::dispatchOnceInnerLocked(nsecs_t*nextWakeupTime) {
        nsecs_t currentTime = now();
        // ...
        // 重置标记
        resetANRTimeoutsLocked();

        switch (mPendingEvent -> type) {

            case EventEntry::TYPE_KEY: {
                ...
                // key类型
                done = dispatchKeyLocked(currentTime, typedEntry, & dropReason, nextWakeupTime);
                break;
            }

            case EventEntry::TYPE_MOTION: {
                ...
                done = dispatchMotionLocked(currentTime, typedEntry,
                        & dropReason, nextWakeupTime);
                break;
            }

            default:
                ALOG_ASSERT(false);
                break;
        }
    }


    void InputDispatcher::resetANRTimeoutsLocked() {
        // 将mInputTargetWaitCause设置为INPUT_TARGET_WAIT_CAUSE_NONE
       　mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_NONE;
        mInputTargetWaitApplicationToken.clear();
    }
```
在分发之前会调用`resetANRTimeoutsLocked()`方法，重置mInputTargetWaitCause标记为：`INPUT_TARGET_WAIT_CAUSE_NONE`。接着根据下发的类型，寻找对应的窗口，比如KEY类型，则调用`dispatchKeyLocked()`方法。
```
    bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry*entry,
                      DropReason*dropReason, nsecs_t*nextWakeupTime) {

        // 寻找目标窗口
        int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime);

        // 给目标窗口分发事件
        dispatchEventLocked(currentTime, entry, inputTargets);
        return true;
    }

    int32_t InputDispatcher::findFocusedWindowTargetsLocked(nsecs_t currentTime,
        const EventEntry*entry, std::vector<InputTarget>&inputTargets, nsecs_t*nextWakeupTime) {
        ...
        // 检查窗口不能input的原因
        reason = checkWindowReadyForMoreInputLocked(currentTime,
                focusedWindowHandle, entry, "focused");
        if (!reason.empty()) {
            // 调用handleTargetsNotReadyLocked()方法
            injectionResult = handleTargetsNotReadyLocked(currentTime, entry,
                    focusedApplicationHandle, focusedWindowHandle, nextWakeupTime, reason.c_str());
         goto Unresponsive;
        }

        ...
        return injectionResult;
    }

    int32_t InputDispatcher::handleTargetsNotReadyLocked(nsecs_t currentTime,const EventEntry*entry,const sp<InputApplicationHandle>&applicationHandle,const sp<InputWindowHandle>&windowHandle,
                                nsecs_t*nextWakeupTime, const char*reason) {
        // 在resetANRTimeoutsLocked方法中，mInputTargetWaitCause为INPUT_TARGET_WAIT_CAUSE_NONE
        if (mInputTargetWaitCause != INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY) {
            // DEFAULT_INPUT_DISPATCHING_TIMEOUT为5s
            nsecs_t timeout;
            if (windowHandle != nullptr) {
                timeout = windowHandle -> getDispatchingTimeout(DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            } else if (applicationHandle != nullptr) {
                timeout = applicationHandle -> getDispatchingTimeout(
                        DEFAULT_INPUT_DISPATCHING_TIMEOUT);
            } else {
                timeout = DEFAULT_INPUT_DISPATCHING_TIMEOUT;
            }
            // 要等到下次调用resetANRTimeoutsLocked时才能进
            mInputTargetWaitCause = INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY;
            // 当前时间加上5s
            mInputTargetWaitTimeoutTime = currentTime + timeout;
            mInputTargetWaitTimeoutExpired = false;
            mInputTargetWaitApplicationToken.clear();

        }

        if (mInputTargetWaitTimeoutExpired) {
            return INPUT_EVENT_INJECTION_TIMED_OUT;
        }

        if (currentTime >= mInputTargetWaitTimeoutTime) {
            // 当前时间超过设定的5s，后执行onANRLocked()的ANR方法
            onANRLocked(currentTime, applicationHandle, windowHandle,
                    entry -> eventTime, mInputTargetWaitStartTime, reason);
            return INPUT_EVENT_INJECTION_PENDING;
        } else {
            // Force poll loop to wake up when timeout is due.
            if (mInputTargetWaitTimeoutTime < *nextWakeupTime){
                     *nextWakeupTime = mInputTargetWaitTimeoutTime;
            }
            return INPUT_EVENT_INJECTION_PENDING;
        }
    }
```
在分发一次事件时，会调用`resetANRTimeoutsLocked`将标记为`INPUT_TARGET_WAIT_CAUSE_NONE`,所以第一次事件会设置一个5s后的超时时间，并把标记设置为`INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY`，如果下次事件来临时当前的时间超过上次设置的5s时间就会调用`onANRLocked()`方法产生ANR。
```
    void InputDispatcher::onANRLocked(
            nsecs_t currentTime, const sp<InputApplicationHandle>&applicationHandle,
            const sp<InputWindowHandle>&windowHandle,
            nsecs_t eventTime, nsecs_t waitStartTime, const char*reason) {
        float dispatchLatency = (currentTime - eventTime) * 0.000001f;
        float waitDuration = (currentTime - waitStartTime) * 0.000001f;

        // 收集ANR现场信息
        time_t t = time(nullptr);
        struct tm tm;
        localtime_r( & t, &tm);
        char timestr[ 64];
        strftime(timestr, sizeof(timestr), "%F %T", & tm);
        mLastANRState.clear();
        mLastANRState += INDENT "ANR:\n";
        mLastANRState += StringPrintf(INDENT2"Time: %s\n", timestr);
        mLastANRState += StringPrintf(INDENT2"Window: %s\n",
                getApplicationWindowLabel(applicationHandle, windowHandle).c_str());
        mLastANRState += StringPrintf(INDENT2"DispatchLatency: %0.1fms\n", dispatchLatency);
        mLastANRState += StringPrintf(INDENT2"WaitDuration: %0.1fms\n", waitDuration);
        mLastANRState += StringPrintf(INDENT2"Reason: %s\n", reason);
        //dump信息
        dumpDispatchStateLocked(mLastANRState);
        //将ANR命令加入commandQueue
        CommandEntry * commandEntry = postCommandLocked(
                & InputDispatcher::doNotifyANRLockedInterruptible);
        commandEntry -> inputApplicationHandle = applicationHandle;
        commandEntry -> inputChannel = windowHandle != nullptr ?
                getInputChannelLocked(windowHandle -> getToken()) : nullptr;
        commandEntry -> reason = reason;
    }
```
在下次执行`InputDispatcher.dispatchOnce`时会先执行commandQueue的队列命令，这里把`InputDispatcher::doNotifyANRLockedInterruptible`放入到了队列。
```
    void InputDispatcher::doNotifyANRLockedInterruptible(CommandEntry*commandEntry) {
        mLock.unlock();
        //mPolicy是指NativeInputManager
        nsecs_t newTimeout = mPolicy -> notifyANR(
                commandEntry -> inputApplicationHandle,
                commandEntry -> inputChannel ? commandEntry -> inputChannel -> getToken() : nullptr,
                commandEntry -> reason);

        mLock.lock();

        resumeAfterTargetsNotReadyTimeoutLocked(newTimeout,
                commandEntry -> inputChannel);
    }

```
`mPolicy -> notifyANR`通过JNI最终调用到`InputManagerService.notifyANR()`方法：
```
   // InputManagerService
    private long notifyANR(InputApplicationHandle inputApplicationHandle, IBinder token,
                           String reason) {
        return mWindowManagerCallbacks.notifyANR(inputApplicationHandle,
                token, reason);
    }
```
这里的`mWindowManagerCallbacks`是InputManagerCallback对象
```
//InputManagerCallback
    public long notifyANR(InputApplicationHandle inputApplicationHandle, IBinder token,
                          String reason) {
        final long startTime = SystemClock.uptimeMillis();
        try {
            return notifyANRInner(inputApplicationHandle, token, reason);
        } finally {
        }
    }

    private long notifyANRInner(InputApplicationHandle inputApplicationHandle, IBinder token,
                                String reason) {
        ...
        // 调用AMS的inputDispatchingTimedOut()方法
        long timeout = mService.mAmInternal.inputDispatchingTimedOut(windowPid, aboveSystem,
                reason);

        return 0; // abort dispatching
    }

```
最终调用到`AMS.inputDispatchingTimedOut()`方法

```
    // AMS
    long inputDispatchingTimedOut(int pid, final boolean aboveSystem, String reason) {
        if (checkCallingPermission(FILTER_EVENTS) != PackageManager.PERMISSION_GRANTED) {
            throw new SecurityException("Requires permission " + FILTER_EVENTS);
        }
        ProcessRecord proc;
        long timeout;
        synchronized (this) {
            synchronized (mPidsSelfLocked) {
                proc = mPidsSelfLocked.get(pid);
            }
            timeout = proc != null ? proc.getInputDispatchingTimeout() : KEY_DISPATCHING_TIMEOUT_MS;
        }
        // 调用inputDispatchingTimedOut
        if (inputDispatchingTimedOut(proc, null, null, null, null, aboveSystem, reason)) {
            return -1;
        }

        return timeout;
    }


    boolean inputDispatchingTimedOut(ProcessRecord proc, String activityShortComponentName,
                                     ApplicationInfo aInfo, String parentShortComponentName,
                                     WindowProcessController parentProcess, boolean aboveSystem, String reason) {
            // 调用appNotResponding方法
            mAnrHelper.appNotResponding(proc, activityShortComponentName, aInfo,
                    parentShortComponentName, parentProcess, aboveSystem, annotation);

        return true;
    }
```
还是执行`appNotResponding()`方法。





