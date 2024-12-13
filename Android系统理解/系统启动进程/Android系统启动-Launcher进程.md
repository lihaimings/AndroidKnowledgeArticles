
> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
/frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
/frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
/packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
/packages/apps/Launcher3/src/com/android/launcher3/LauncherModel.java
/packages/apps/Launcher3/src/com/android/launcher3/model/LoaderTask.java

  ```

在前面文章中分别讲解了在[SystemServer进程](https://www.jianshu.com/p/987a43c1a708)的**startBootstrapServices()、startOtherServices()**方法中启动了[ActivityManagerService(ActivityTaskManagerService)](https://www.jianshu.com/p/c9a371475a16)和[PackageManagerService](https://www.jianshu.com/p/05093ce7a4ad)系统服务。
1. ActivityManagerService：主要负责四大组件的创建和管理。
2. PackageManagerService:主要负责安装包的查询、安装、卸载。

系统的**桌面进程Launcher**就是在[ActivityManagerService服务](https://www.jianshu.com/p/c9a371475a16)启动完成后开始启动。

## Launcher进程启动
`ActivityManagerService.systemReady`方法中开始了Launcher进程的启动
  ```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    // mAtmInternal为ActivityTaskManagerService.LocalService类
    public ActivityTaskManagerInternal mAtmInternal;

    // ActivityManagerService准备完成
    public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {

        // 调用启动Launcher进程
        mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
    }
}

public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    RootWindowContainer mRootWindowContainer;

    final class LocalService extends ActivityTaskManagerInternal {
        @Override
        public boolean startHomeOnAllDisplays(int userId, String reason) {
            synchronized (mGlobalLock) {
                return mRootWindowContainer.startHomeOnAllDisplays(userId, reason);
            }
        }

    }
}
  ```
1. 在AMS的systemReady方法中启动Launcher进程，但并没有真正的启动进程，这只是一个开始，而后给`ActivityTaskManagerService.LocalService`类的startHomeOnAllDisplays方法继续执行。
2. 在ActivityTaskManagerService中并没有做处理，而是调用了`RootWindowContainer.startHomeOnAllDisplays`方法处理。

`RootWindowContainer.startHomeOnAllDisplays`通过一系列的调用最终调用到`RootWindowContainer.startHomeOnTaskDisplayArea`方法：
  ```
class RootWindowContainer extends WindowContainer<DisplayContent>
        implements DisplayManager.DisplayListener {

    ActivityTaskManagerService mService;

    boolean startHomeOnAllDisplays(int userId, String reason) {
        boolean homeStarted = false;
        // getChildCount获取显示设备数目，这个主要从mChildren参数中获取对应的数量
        // mChildren是一个WindowList的一个对象，其包含的数据是在setWindowManager函数被调用时，从DisplayManagerService中获取到的Display的数目
        for (int i = getChildCount() - 1; i >= 0; i--) {
            // 获取到对应新建的DisplayContent的displayId
            final int displayId = getChildAt(i).mDisplayId;
            // 调用startHomeOnDisplay函数
            homeStarted |= startHomeOnDisplay(userId, reason, displayId);

        }
        return homeStarted;

    }
    //一系列调用最终调用startHomeOnTaskDisplayArea.......

    boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,
                                       boolean allowInstrumenting, boolean fromHomeKey) {
        // Fallback to top focused display area if the provided one is invalid.
        if (taskDisplayArea == null) {
            final ActivityStack stack = getTopDisplayFocusedStack();
            taskDisplayArea = stack != null ? stack.getDisplayArea()
                    : getDefaultTaskDisplayArea();
        }

        Intent homeIntent = null;
        ActivityInfo aInfo = null;
        if (taskDisplayArea == getDefaultTaskDisplayArea()) {
            // 1.向ActivityTaskManagerService获取 Launcher 的启动意图
            homeIntent = mService.getHomeIntent();
            //2. 向PackageManagerService通过意图解析到 ActivityInfo
            aInfo = resolveHomeActivity(userId, homeIntent);
        } else if (shouldPlaceSecondaryHomeOnDisplayArea(taskDisplayArea)) {
            Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, taskDisplayArea);
            aInfo = info.first;
            homeIntent = info.second;
        }
        if (aInfo == null || homeIntent == null) {
            return false;
        }

        if (!canStartHomeOnDisplayArea(aInfo, taskDisplayArea, allowInstrumenting)) {
            return false;
        }

        // Updates the home component of the intent.
        homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
        // Updates the extra information of the intent.
        if (fromHomeKey) {
            homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
            mWindowManager.cancelRecentsAnimation(REORDER_KEEP_IN_PLACE, "startHomeActivity");
        }
        // Update the reason for ANR debugging to verify if the user activity is the one that
        // actually launched.
        final String myReason = reason + ":" + userId + ":" + UserHandle.getUserId(
                aInfo.applicationInfo.uid) + ":" + taskDisplayArea.getDisplayId();

        // 根据homeIntent、aInfo，调用 startHomeActivity 方法去启动和创建 Launcher
        mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
                taskDisplayArea);
        return true;
    }

    // 根据Intent的Component向PackageManagerService找到对应的ActivityInfo
    ActivityInfo resolveHomeActivity(int userId, Intent homeIntent) {
        final int flags = ActivityManagerService.STOCK_PM_FLAGS;
        final ComponentName comp = homeIntent.getComponent();
        ActivityInfo aInfo = null;
        try {
            if (comp != null) {
                // Factory test.
                aInfo = AppGlobals.getPackageManager().getActivityInfo(comp, flags, userId);
            } else {
                final String resolvedType =
                        homeIntent.resolveTypeIfNeeded(mService.mContext.getContentResolver());
                final ResolveInfo info = AppGlobals.getPackageManager()
                        .resolveIntent(homeIntent, resolvedType, flags, userId);
                if (info != null) {
                    aInfo = info.activityInfo;
                }
            }
        } catch (RemoteException e) {
            // ignore
        }

        if (aInfo == null) {
            Slog.wtf(TAG, "No home screen found for " + homeIntent, new Throwable());
            return null;
        }

        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = mService.getAppInfoForUser(aInfo.applicationInfo, userId);
        return aInfo;
    }

}



public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    String mTopAction = Intent.ACTION_MAIN;
    String mTopData;

    // homeIntent.action = Intent.ACTION_MAIN   ACTION_MAIN = "android.intent.action.MAIN"
    // homeIntent的flags包含Intent.FLAG_DEBUG_TRIAGED_MISSING
    // homeIntent的category包含Intent.CATEGORY_HOME  CATEGORY_HOME = "android.intent.category.HOME";
    Intent getHomeIntent() {
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
    }

}
  ```
代码很长，但不外乎做了两件事：
- 调用ActivityTaskManagerService.getHomeIntent方法获取Intent意图
- 根据意图Intent.Component向PackageManagerService查询对应的ActivityInfo
最后，通过ActivityStartController.startHomeActivity方法,通过层层调用最后调用`Process.start`方法去启动和创建Launcher。关于后面进程的创建和Activity的启动后面文章讲解，本篇继续分析Launcher启动后如何查询所有App信息
![](https://upload-images.jianshu.io/upload_images/22650779-faa8ce060d2f0e46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## Launcher查询App信息
在`Launcher`Activity中，会在`onCreate`方法去查询所有App信息：
  ```
public class Launcher extends StatefulActivity<LauncherState> implements LauncherExterns,
        Callbacks, InvariantDeviceProfile.OnIDPChangeListener, PluginListener<OverlayPlugin> {

    @Override
    protected void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);

        // 实例化LauncherAppState
        LauncherAppState app = LauncherAppState.getInstance(this);
        // 根据LauncherAppState获取到LauncherModel
        mModel = app.getModel();

        // LauncherModel.addCallbacksAndLoad就会去查询App信息
        if (!mModel.addCallbacksAndLoad(this)) {
            if (!internalStateHandled) {
                // If we are not binding synchronously, show a fade in animation when
                // the first page bind completes.
                mDragLayer.getAlphaProperty(ALPHA_INDEX_LAUNCHER_LOAD).setValue(0);
            }
        }

    }
}
  ```
在onCreate中通过调用`LauncherModel.addCallbacksAndLoad`去查询App信息，此方法通过一个Handler去执行一个Runnable去执行查询任务。
  ```
public class LauncherModel extends LauncherApps.Callback implements InstallSessionTracker.Callback {

    // Launcher也实现了Callbacks
    public boolean addCallbacksAndLoad(Callbacks callbacks) {
        synchronized (mLock) {
            // 将Launcher加入到回调列表
            addCallbacks(callbacks);
            // 调用startLoader()
            return startLoader();
        }
    }

    public boolean startLoader() {
        synchronized (mLock) {
            //.....
            stopLoader();
            LoaderResults loaderResults = new LoaderResults(
                    mApp, mBgDataModel, mBgAllAppsList, callbacksList, mMainExecutor);
            // 开始查询App信息
            startLoaderForResults(loaderResults);
            return false;
        }
    }

    public void startLoaderForResults(LoaderResults results) {
        synchronized (mLock) {
            stopLoader();
            // LoaderTask是一个Runnable
            mLoaderTask = new LoaderTask(mApp, mBgAllAppsList, mBgDataModel, results);

            // Handler去执行LoaderTask
            MODEL_EXECUTOR.post(mLoaderTask);
        }
    }

}
  ```
通过对重点代码的查看，LauncherModel只是发送了一个`LoaderTask`的Runnable让Handler去执行。所以查询App在`LoaderTask`类的run()方法中。
  ```
public class LoaderTask implements Runnable {

    public void run() {
        //.....

        // 加载所有App
        List<LauncherActivityInfo> allActivityList = loadAllApps();

        //.....
    }
    
    // 查询所有App
    private List<LauncherActivityInfo> loadAllApps() {
        final List<UserHandle> profiles = mUserCache.getUserProfiles();
        List<LauncherActivityInfo> allActivityList = new ArrayList<>();
        // Clear the list of apps
        mBgAllAppsList.clear();
        for (UserHandle user : profiles) {
            // Query for the set of apps
            final List<LauncherActivityInfo> apps = mLauncherApps.getActivityList(null, user);
            // Fail if we don't have any apps
            // TODO: Fix this. Only fail for the current user.
            if (apps == null || apps.isEmpty()) {
                return allActivityList;
            }
            //....
            allActivityList.addAll(apps);
        }
        //.....
        return allActivityList;
    }


}

public class LauncherApps {

    public List<LauncherActivityInfo> getActivityList(String packageName, UserHandle user) {
        logErrorForInvalidProfileAccess(user);
        try {
            return convertToActivityList(mService.getLauncherActivities(mContext.getPackageName(),
                    packageName, user), user);
        } catch (RemoteException re) {
            throw re.rethrowFromSystemServer();
        }
    }
}
  ```
LoaderTask的run的方法会查询所有App的信息。

## 总结
Launcher是系统的桌面进程，它是由`ActivityManagerService`完成准备后开始启动的，启动的时候会通过向ActivityManagerService获取Intent意图和向PackageManagerService获取对应ActivityInfo。Launcher Activity启动后会向在onCreate()方法中查询所有App信息。


