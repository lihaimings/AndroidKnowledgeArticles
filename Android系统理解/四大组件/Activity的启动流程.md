> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/core/java/android/app/Activity.java
/frameworks/base/core/java/android/app/Instrumentation.java
/frameworks/base/core/java/android/app/ActivityTaskManager.java
/frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
/frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
/frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
/frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java
/frameworks/base/services/core/java/com/android/server/wm/ActivityStackSupervisor.java
/frameworks/base/services/core/java/com/android/server/wm/ClientLifecycleManager.java
/frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java
/frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java
/frameworks/base/core/java/android/app/ActivityThread.java
/frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java
/frameworks/base/core/java/android/app/LoadedApk.java
  ```
## 概述
调用`startActivity`或`startActivityForResult`来启动Activity。那么启动的Activity有两种情况：第一种是启动同进程内的Activity; 第二种是启动不同进程的根Activity,比如在桌面点击启动App,就是启动不同进程的Activity。这两种情况的Activity的启动流程大致相同，其流程大致是以下三个过程：
1. 调用进程的Activity收集好信息后，向system_server进程的ActivityTaskManagerServer服务发起请求。
2. ATMS向PKMS寻找启动的Activity信息和进程信息，如果启动的Activity进程没有被创建，则[创建新进程](https://www.jianshu.com/p/00e2ce96123d),之后管理Activity的堆栈，并回调启动Activity所在进程的ApplicationThread类
3. ApplicationThread通过调用ActivityThread来反射调用启动Activity。

以下逐一讲解这三大过程：

## 1. 向ATMS发起请求(system_server进程)
下图是调用进程向system_server进程发起请求的过程：

![](https://upload-images.jianshu.io/upload_images/22650779-342c30aee77528ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
无论是使用`startActivity`还是`startActivityForResult`最终都会调用到`Activity.startActivityForResult`:
  ```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {

    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                       @Nullable Bundle options) {
        // mParent 是Activity类型，是当前Activity的父类
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //主要看这里mInstrumentation为Instrumentation对象
            Instrumentation.ActivityResult ar =
                    mInstrumentation.execStartActivity(
                            this, mMainThread.getApplicationThread(), mToken, this,
                            intent, requestCode, options);
            //...
        } else {
            //...
        }
    }

}
  ```
startActivityForResult方法继续调用`Instrumentation.execStartActivity`方法。而Instrumentation类主要用来监控应用程序和系统的交互。
  ```
// Instrumentation主要用来监控应用程序和系统的交互
public class Instrumentation {

    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            //核心在这一句 调用ActivityTaskManagerService.startActivity（通过Binder向System_server进程通信）
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            // 根据返回的状态，检查是否能启动Activity，不能启动则抛出不同的异常
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

    // 根据不同的异常状态，抛出不同的异常
    public static void checkStartActivityResult(int res, Object intent) {
        if (!ActivityManager.isStartResultFatalError(res)) {
            return;
        }

        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent) intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                                    + ((Intent) intent).getComponent().toShortString()
                                    + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                        + intent);
            case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
                throw new AndroidRuntimeException(
                        "FORWARD_RESULT_FLAG used while also requesting a result");
            case ActivityManager.START_NOT_ACTIVITY:
                throw new IllegalArgumentException(
                        "PendingIntent is not an activity");
            case ActivityManager.START_NOT_VOICE_COMPATIBLE:
                throw new SecurityException(
                        "Starting under voice control not allowed for: " + intent);
            case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                        "Session calling startVoiceActivity does not match active session");
            case ActivityManager.START_VOICE_HIDDEN_SESSION:
                throw new IllegalStateException(
                        "Cannot start voice activity on a hidden session");
            case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                        "Session calling startAssistantActivity does not match active session");
            case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
                throw new IllegalStateException(
                        "Cannot start assistant activity on a hidden session");
            case ActivityManager.START_CANCELED:
                throw new AndroidRuntimeException("Activity could not be started for "
                        + intent);
            default:
                throw new AndroidRuntimeException("Unknown error code "
                        + res + " when starting " + intent);
        }
    }
}

public class ActivityTaskManager {

    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    @UnsupportedAppUsage(trackingBug = 129726065)
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    // Context.ACTIVITY_TASK_SERVICE这个是ATMS
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
}
  ```
1. 通过`ActivityTaskManager.getService()`获取到ATMS的代理类，通过Binder跨进程通信，向ATMS发起startActivity请求。
2. 通过向ATMS发起启动Activity请求，从而得到的返回结果result，再根据result判断能否启动Activity,不能则抛异常，比如Activity没有在AndroidManifest中注册等。

## 2. ATMS到ApplicationThread调用过程
下图是ATMS处理startActivity过程，并回调启动进程的ApplicationThread

![](https://upload-images.jianshu.io/upload_images/22650779-eabf22046e2f33d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**ATMS. startActivity:**
  ```
// system_server进程
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {

    public final int startActivity(IApplicationThread caller, String callingPackage,
                                   String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
                                   String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
                                   Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                   String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
                                   String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
                                   Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                    @Nullable String callingFeatureId, Intent intent, String resolvedType,
                                    IBinder resultTo, String resultWho, int requestCode, int startFlags,
                                    ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
        //判断调用者进程是否被隔离
        enforceNotIsolatedCaller("startActivityAsUser");
        //检查调用者权限
        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // getActivityStartController().obtainStarter 返回 ActivityStarter，
        // 把参数设置到ActivityStarter.Request类中
        // 最后调用ActivityStarter.execute()，
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId)
                .execute();
    }
}
  ```
ATMS类通过一系列方法调用最终来到`startActivityAsUser`的方法，该方法先检查调用进程的权限，然后通过`getActivityStartController().obtainStarter`创建`ActivityStarter`类，并把参数设置到`ActivityStarter.Request`类中，最后执行`ActivityStarter.execute()`方法。

**ActivityStarter准备堆栈**
  ```
class ActivityStarter {

    int execute() {
        try {
            // ...
            int res;
            synchronized (mService.mGlobalLock) {
                //...
                res = executeRequest(mRequest);
                //...
                return getExternalResult(mRequest.waitResult == null ? res
                        : waitForResult(res, mLastStartActivityRecord));
            }
        } finally {
            onExecutionComplete();
        }
    }

    private int executeRequest(Request request) {
        //判断启动的理由不为空
        if (TextUtils.isEmpty(request.reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
        ...

        //获取调用的进程
        WindowProcessController callerApp = null;
        if (caller != null) {
            callerApp = mService.getProcessController(caller);
            if (callerApp != null) {
                //获取调用进程的pid和uid并赋值
                callingPid = callerApp.getPid();
                callingUid = callerApp.mInfo.uid;
            } else {
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        final int userId = aInfo != null && aInfo.applicationInfo != null
                ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            //获取调用者所在的ActivityRecord
            sourceRecord = mRootWindowContainer.isInAnyStack(resultTo);
            if (sourceRecord != null) {
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    //requestCode = -1 则不进入
                    resultRecord = sourceRecord;
                }
            }
        }

        final int launchFlags = intent.getFlags();
        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
           ... // activity执行结果的返回由源Activity转换到新Activity, 不需要返回结果则不会进入该分支
        }

        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
            //从Intent中无法找到相应的Component
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            //从Intent中无法找到相应的ActivityInfo
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }
        ...

        //执行后resultStack = null
        final ActivityStack resultStack = resultRecord == null
                ? null : resultRecord.getRootTask();

        ... //权限检查

        // ActivityController不为空的情况，比如monkey测试过程
        if (mService.mController != null) {
            try {
                Intent watchIntent = intent.cloneFilter();
                abort |= !mService.mController.activityStarting(watchIntent,
                        aInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                mService.mController = null;
            }
        }


        if (abort) {
            ... //权限检查不满足,才进入该分支则直接返回；
            return START_ABORTED;
        }

        if (aInfo != null) {
            if (mService.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                    aInfo.packageName, userId)) {
                ...
                //向PKMS获取启动Activity的ResolveInfo
                rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId, 0,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, request.filterCallingUid));
                //向PKMS获取启动Activity的ActivityInfo
                aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags,
                        null /*profilerInfo*/);

            }
        }
        ...
        // 创建即将要启动的Activity的描述类ActivityRecord
        final ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, callingFeatureId, intent, resolvedType, aInfo,
                mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode,
                request.componentSpecified, voiceSession != null, mSupervisor, checkedOptions,
                sourceRecord);
        mLastStartActivityRecord = r;

        ...
        // 调用 startActivityUnchecked
        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
    }

}
  ```
在`ActivityStarter. executeRequest`方法中先做一系列的检查，包括调用进程的检查、Intent的检查、权限的检查、向PKMS获取启动Activity的ActivityInfo等信息，然后调用`startActivityUnchecked`方法开始对要启动的Activity做堆栈管理。

  ```

    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                                       IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                       int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                                       boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.deferWindowLayout();
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
        } finally {
            //...
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }


    ActivityRecord mStartActivity;

    private ActivityStack mSourceStack;
    private ActivityStack mTargetStack;
    private Task mTargetTask;


    // 主要处理栈管理相关的逻辑
    int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
                           IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                           int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                           boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        // 初始化启动Activity的各种配置，在初始化前会重置各种配置再进行配置，
        // 这些配置包括：ActivityRecord、Intent、Task和LaunchFlags（启动的FLAG）等等
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor, restrictedBgActivity);

        // 给不同的启动模式计算出mLaunchFlags
        computeLaunchingTaskFlags();

        // 主要作用是设置ActivityStack
        computeSourceStack();

        // 将mLaunchFlags设置给Intent
        mIntent.setFlags(mLaunchFlags);

        // 确定是否应将新活动插入现有任务。如果不是，则返回null，
        // 或者返回带有应将新活动添加到其中的任务的ActivityRecord。
        final Task reusedTask = getReusableTask();

        //...

        // 如果reusedTask为null，则计算是否存在可以使用的任务栈
        final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
        final boolean newTask = targetTask == null; // 启动Activity是否需要新创建栈
        mTargetTask = targetTask;

        computeLaunchParams(r, sourceRecord, targetTask);

        // 检查是否允许在给定任务或新任务上启动活动。
        int startResult = isAllowedToStart(r, newTask, targetTask);
        if (startResult != START_SUCCESS) {
            return startResult;
        }


        final ActivityStack topStack = mRootWindowContainer.getTopDisplayFocusedStack();
        if (topStack != null) {
            // 检查正在启动的活动是否与当前位于顶部的活动相同，并且应该只启动一次
            startResult = deliverToCurrentTopIfNeeded(topStack, intentGrants);
            if (startResult != START_SUCCESS) {
                return startResult;
            }
        }

        if (mTargetStack == null) {
            // 复用或者创建堆栈
            mTargetStack = getLaunchStack(mStartActivity, mLaunchFlags, targetTask, mOptions);
        }
        if (newTask) {
            // 新建一个task
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
            setNewTask(taskToAffiliate);
            if (mService.getLockTaskController().isLockTaskModeViolation(
                    mStartActivity.getTask())) {
                Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
        } else if (mAddingToTask) {
            // 复用之前的task
            addOrReparentStartingActivity(targetTask, "adding to task");
        }

        ...

        // 检查是否需要触发过渡动画和开始窗口
        mTargetStack.startActivityLocked(mStartActivity,
                topStack != null ? topStack.getTopNonFinishingActivity() : null, newTask,
                mKeepCurTransition, mOptions);


        if (mDoResume) {

            //...
            // 调用RootWindowContainer的resumeFocusedStacksTopActivities方法
            mRootWindowContainer.resumeFocusedStacksTopActivities(
                    mTargetStack, mStartActivity, mOptions);
        }

        // ...

        return START_SUCCESS;
    }

    // 初始化状态
    private void setInitialState(ActivityRecord r, ActivityOptions options, Task inTask,
                                 boolean doResume, int startFlags, ActivityRecord sourceRecord,
                                 IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                                 boolean restrictedBgActivity) {

        reset(false /* clearRequest */); //重置状态
        mStartActivity = r; // 被启动的Activity
        mIntent = r.intent; //启动的intent
        mSourceRecord = sourceRecord; //发起启动的Activity
        mLaunchMode = r.launchMode; // 启动模式
        // 启动Flags
        mLaunchFlags = adjustLaunchFlagsToDocumentMode(
                r, LAUNCH_SINGLE_INSTANCE == mLaunchMode,
                LAUNCH_SINGLE_TASK == mLaunchMode, mIntent.getFlags());
        mInTask = inTask;
        // ...
    }

    // 根据不同启动模式计算出不同的mLaunchFlags
    private void computeLaunchingTaskFlags() {
        if (mInTask == null) {
            if (mSourceRecord == null) {
                if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
                    mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            } else if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
        }
    }

    // 设置ActivityStack
    private void computeSourceStack() {
        if (mSourceRecord == null) {
            mSourceStack = null;
            return;
        }
        if (!mSourceRecord.finishing) {
            mSourceStack = mSourceRecord.getRootTask();
            return;
        }

        if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0) {
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            mNewTaskInfo = mSourceRecord.info;

            final Task sourceTask = mSourceRecord.getTask();
            mNewTaskIntent = sourceTask != null ? sourceTask.intent : null;
        }
        mSourceRecord = null;
        mSourceStack = null;
    }

    private Task getReusableTask() {
        // If a target task is specified, try to reuse that one
        if (mOptions != null && mOptions.getLaunchTaskId() != INVALID_TASK_ID) {
            Task launchTask = mRootWindowContainer.anyTaskForId(mOptions.getLaunchTaskId());
            if (launchTask != null) {
                return launchTask;
            }
            return null;
        }

        //标志位，如果为true，说明要放入已经存在的栈，
        // 可以看出，如果是设置了FLAG_ACTIVITY_NEW_TASK 而没有设置 FLAG_ACTIVITY_MULTIPLE_TASK，
        // 或者设置了singleTask以及singleInstance
        boolean putIntoExistingTask = ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0 &&
                (mLaunchFlags & FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK);
        // 重新检验
        putIntoExistingTask &= mInTask == null && mStartActivity.resultTo == null;
        ActivityRecord intentActivity = null;
        if (putIntoExistingTask) {
            if (LAUNCH_SINGLE_INSTANCE == mLaunchMode) {
                //如果是 singleInstance，那么就找看看之前存在的该实例，找不到就为null
                intentActivity = mRootWindowContainer.findActivity(mIntent, mStartActivity.info,
                        mStartActivity.isActivityTypeHome());
            } else if ((mLaunchFlags & FLAG_ACTIVITY_LAUNCH_ADJACENT) != 0) {
                // For the launch adjacent case we only want to put the activity in an existing
                // task if the activity already exists in the history.
                intentActivity = mRootWindowContainer.findActivity(mIntent, mStartActivity.info,
                        !(LAUNCH_SINGLE_TASK == mLaunchMode));
            } else {
                // Otherwise find the best task to put the activity in.
                intentActivity =
                        mRootWindowContainer.findTask(mStartActivity, mPreferredTaskDisplayArea);
            }
        }

        if (intentActivity != null
                && (mStartActivity.isActivityTypeHome() || intentActivity.isActivityTypeHome())
                && intentActivity.getDisplayArea() != mPreferredTaskDisplayArea) {
            // Do not reuse home activity on other display areas.
            intentActivity = null;
        }

        return intentActivity != null ? intentActivity.getTask() : null;
    }


    // 计算启动的Activity的栈
    private Task computeTargetTask() {
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            // 返回null，应该新创建一个Task，而不是使用现有的Task
            return null;
        } else if (mSourceRecord != null) {
            // 使用源Activity的task
            return mSourceRecord.getTask();
        } else if (mInTask != null) {
            // 使用启动时传递的task
            return mInTask;
        } else {
            // 理论上的可能，不可能走到这里
            final ActivityStack stack = getLaunchStack(mStartActivity, mLaunchFlags,
                    null /* task */, mOptions);
            final ActivityRecord top = stack.getTopNonFinishingActivity();
            if (top != null) {
                return top.getTask();
            } else {
                // Remove the stack if no activity in the stack.
                stack.removeIfPossible();
            }
        }
        return null;
    }
  ```
在`startActivityInner`方法中，根据启动模式计算出flag，在根据flag等条件启动的Activity的ActivityRecord是加入现有的Task栈中，还是创建新Task栈。为Activity准备好堆栈后，调用`RootWindowContainer.resumeFocusedStacksTopActivities`方法

**ActivityStack对栈的管理：**
  ```
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener {

    boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        //...
        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackInDisplayArea()
                || getTopDisplayFocusedStack() == targetStack)) {
            // 调用ActivityStack.resumeTopActivityUncheckedLocked
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        //...
        return result;
    }

}

// 单个活动堆栈的状态和管理。
class ActivityStack extends Task {

    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recurscheduleTransactionsing.
            return false;
        }

        boolean result = false;
        try {
            mInResumeTopActivity = true;
            // 继续调用resumeTopActivityInnerLocked
            result = resumeTopActivityInnerLocked(prev, options);

            final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }

    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {

        //在此堆栈中找到下一个要恢复的最顶层活动，该活动未完成且可聚焦。如果它不是可聚焦的，我们将陷入下面的情况以在下一个可聚焦任务中恢复顶部活动。
        ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        final boolean hasRunningActivity = next != null;

        if (next.attachedToProcess()) {
            ...
        } else {
            ...
            // 调用StackSupervisor.startSpecificActivity
            mStackSupervisor.startSpecificActivity(next, true, true);
        }
        return true;
    }

}
  ```
对于启动Activity对应的`ActivityStack`来说，是管理其栈中Activity的显示逻辑。而后继续调用`ActivityStackSupervisor.startSpecificActivity `

**ActivityStackSupervisor检查启动进程和回调ApplicationThread**
  ```
public class ActivityStackSupervisor implements RecentTasks.Callbacks {

    // 检查启动Activity所在进程是否有启动，没有则先启动进程
    void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // 根据processName和Uid查找启动Activity的所在进程
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;

        if (wpc != null && wpc.hasThread()) {
            // 进程已经存在，则直接启动Activity
            try {
                // 启动Activity ,并返回
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
               ...
            }
            knownToBeDead = true;
        }

        // 所在进程没创建则调用ATMS.startProcessAsync创建进程并启动Activity
        mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
    }

    // 启动Activity的进程存在，则执行此方法 
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
                                    boolean andResume, boolean checkConfig) throws RemoteException {

        ...

        // 创建活动启动事务
        final ClientTransaction clientTransaction = ClientTransaction.obtain(
                proc.getThread(), r.appToken);

        // 为事务设置Callback，为LaunchActivityItem，在客户端时会被调用
        clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                System.identityHashCode(r), r.info,
                // TODO: Have this take the merged configuration instead of separate global
                // and override configs.
                mergedConfiguration.getGlobalConfiguration(),
                mergedConfiguration.getOverrideConfiguration(), r.compat,
                r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));

        // 设置所需的最终状态
        final ActivityLifecycleItem lifecycleItem;
        if (andResume) {
            lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
        } else {
            lifecycleItem = PauseActivityItem.obtain();
        }
        clientTransaction.setLifecycleStateRequest(lifecycleItem);

        // 执行事件，调用ClientLifecycleManager.scheduleTransaction
        mService.getLifecycleManager().scheduleTransaction(clientTransaction);

        ...
        return true;
    }

}

class ClientLifecycleManager {
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        // 反调用ClientTransaction.schedule()
        transaction.schedule();
        ...
    }
}

public class ClientTransaction implements Parcelable, ObjectPoolItem {

    public void schedule() throws RemoteException {
        // 调用ApplicationThread.scheduleTransaction
        mClient.scheduleTransaction(this);
    }
}
  ```
在ActivityStackSupervisor中先检查要启动Activity的进程是否存在，不存在则创建进程，存在则调用`realStartActivityLocked`,realStartActivityLocked方法通过事务给回调`ApplicationThread. scheduleTransaction`方法。

### ATMS小结
1. 检查调用进程、Intent、权限等信息，并向PKMS查找要启动Activity的信息，符合则往下走。
2. 根据启动模式计算出flag等信息，为启动的Activity设置栈
3. 检查启动Activity对应进程是否存在，不存在则先启动进程，存在则向客户端进程`ApplicationThread`进行回调启动Activity


## 3. ActivityThread启动Activity过程
下图为ActivityThread启动Activity过程的时序图：

![](https://upload-images.jianshu.io/upload_images/22650779-72d2d3d9acda317d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ```
// 主要是处理AMS端的请求
private class ApplicationThread extends IApplicationThread.Stub {
    @Override
    public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        ActivityThread.this.scheduleTransaction(transaction);
    }
}

// ActivityThread的父类
public abstract class ClientTransactionHandler {

    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        // 发送EXECUTE_TRANSACTION消息
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }

}

// 它管理应用程序进程中主线程的执行，根据活动管理器的请求，在其上调度和执行活动、广播和其他操作。
public final class ActivityThread extends ClientTransactionHandler {

    // 用于处理消息同步，将binder线程的消息同步到主线程中
    class H extends Handler {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    // 调用TransactionExecutor.execute去处理ATMS阶段传过来的ClientTransaction
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    break;
            }
        }
    }
}

public class TransactionExecutor {
    public void execute(ClientTransaction transaction) {
        //...
        // 调用传过来的ClientTransaction事务的Callback
        executeCallbacks(transaction);

        executeLifecycleState(transaction);
    }

    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        ...
        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            ...
            // 调用LaunchActivityItem.execute
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            ...
        }
    }
}

public class LaunchActivityItem extends ClientTransactionItem {
    public void execute(ClientTransactionHandler client, IBinder token,
                        PendingTransactionActions pendingActions) {
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
        // 调用ApplicationThread.handleLaunchActivity
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    }
}
  ```
ApplicationThread类绕了一大圈，最后调用在ATMS阶段设置的ClientTransaction的CallBack的execute方法，也就是`LaunchActivityItem. execute`方法，此方法创建一个`ActivityClientRecord`对象，然后调用`ActivityThread. handleLaunchActivity`开启真正的Actvity启动。

**真正启动Activity:**
  ```
public final class ActivityThread extends ClientTransactionHandler {
    // ActivityThread启动Activity的过程
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
                                         PendingTransactionActions pendingActions, Intent customIntent) {
        // ...
        WindowManagerGlobal.initialize();
        // 启动Activity
        final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
           ...
        } else {
            // 启动失败，调用ATMS停止Activity启动
            try {
                ActivityTaskManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }

    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //获取ActivityInfo类
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            //获取APK文件的描述类LoadedApk
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        // 启动的Activity的ComponentName类
        ComponentName component = r.intent.getComponent();

        //创建要启动Activity的上下文环境
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //用类加载器来创建该Activity的实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            // ...
        } catch (Exception e) {
            // ...
        }

        try {
            // 创建Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                // 初始化Activity
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                ...
                // 调用Instrumentation的callActivityOnCreate方法来启动Activity
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
              ...
            }
          ...

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
          ...
        }

        return activity;
    }

}


public class Instrumentation {

    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        // 调用Activity的performCreate
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
}

public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {

    final void performCreate(Bundle icicle) {
        performCreate(icicle, null);
    }

    @UnsupportedAppUsage
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        ...
        // 调用onCreate方法
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        ...
    }

}
  ```

### 小结
1. 调用从system_server进程传过来的ClientTransaction.CallBack类中的execute方法，其继续调用`ActivityThread.handleLaunchActivity`方法来启动Activity。
2. `ActivityThread.handleLaunchActivity`调用`performLaunchActivity`方法启动Activity
3. `performLaunchActivity`先获取启动Activity的ActivityInfo，而后创建一个LoadedApk和上下文ContextImpl。在通过反射去创建此Activity对象，并执行Activity的onCreate方法。

