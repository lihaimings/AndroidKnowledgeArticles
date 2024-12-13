> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/core/java/android/app/ActivityThread.java
/frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
/frameworks/base/core/java/android/view/WindowManagerImpl.java
/frameworks/base/core/java/android/view/WindowManagerGlobal.java
/frameworks/base/core/java/android/view/ViewRootImpl.java
/frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
/frameworks/base/services/core/java/com/android/server/wm/Session.java
/frameworks/base/services/core/java/com/android/server/wm/WindowState.java

  ```
本文以`startActvity()`为启点分析`Window`窗口的创建过程。

# 窗口的创建过程

## 1. Activity的onCreate阶段
  ```
public final class ActivityThread extends ClientTransactionHandler {

    public Activity handleLaunchActivity(ActivityClientRecord r,
                                         PendingTransactionActions pendingActions, Intent customIntent) {

        // 1.WindowManagerGlobal的初始化
        WindowManagerGlobal.initialize();
        // 2.启动Activity
        final Activity a = performLaunchActivity(r, customIntent);
        //...
        return a;
    }

    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        try {
            //...
            // 执行activity.attach()，初始化如PhoneWindow等
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);
            //...
            // 1. 执行Activity.onCreate()方法
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            //...

        } catch (SuperNotCalledException e) {
            throw e;
        }
        //...
        return activity;
    }

}
  ```
启动`Activity`的时候会通过反射创建Activity的实例，并执行Activity的`attach()`方法对一些变量进行初始化，比如PhoneWindow变量。
  ```
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {

    final void attach(Context context, ActivityThread aThread,
                      Instrumentation instr, IBinder token, int ident,
                      Application application, Intent intent, ActivityInfo info,
                      CharSequence title, Activity parent, String id,
                      NonConfigurationInstances lastNonConfigurationInstances,
                      Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                      Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
        attachBaseContext(context);
        mFragments.attachHost(null /*parent*/);

        //创建PhoneWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        //...
        mUiThread = Thread.currentThread(); //获取UI线程
        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token; //远程ActivityRecord的appToken的代理端
        mAssistToken = assistToken;
        mIdent = ident;
        mApplication = application; //所属的Appplication
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        // ...
        //设置WindowManagerImpl对象
        mWindow.setWindowManager(
                (WindowManager) context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        // 获取WindowManagerImpl对象
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
        mWindow.setPreferMinimalPostProcessing(
                (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

        setAutofillOptions(application.getAutofillOptions());
        setContentCaptureOptions(application.getContentCaptureOptions());
    }

}
  ```
在`Activity.attach()`执行在`onCreate()`方法之前，会实例化一个`PhoneWindow`。

## 2. Activity的onResume阶段
  ```
class ActivityThread {

    // Activity的onResume
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                                     String reason) {
        //执行到onResume方法()
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);

        final Activity a = r.activity;

        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            // ...
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                // 添加视图
                r.activity.makeVisible();
            }
        }
        //...
    }
}

public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {

    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            // 向WindowManagerImpl添加DecorView
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
}

public final class WindowManagerImpl implements WindowManager {

    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        // 调用WMG.addView方法
        mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
                mContext.getUserId());
    }
}

public final class WindowManagerGlobal {

    @UnsupportedAppUsage
    private final ArrayList<View> mViews = new ArrayList<View>();
    @UnsupportedAppUsage
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    @UnsupportedAppUsage
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();

    public void addView(View view, ViewGroup.LayoutParams params,
                        Display display, Window parentWindow, int userId) {
        ...
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;

        ViewRootImpl root;
        View panelParentView = null;
        //1. 创建ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        try {
            // 2. 调用ViewRoot的setView()方法
            root.setView(view, wparams, panelParentView, userId);
        } catch (RuntimeException e) {
            //...
        }
    }
}

  ```
调用`Activity.makeVisible()`方法添加窗口视图，继而调用`WindowManagerImpl.addView()`->`WindowManagerGlobal.addView()`方法，在WMG中首先创建了`ViewRootImpl`的实例，并调用了`ViewRootImpl.setView()`方法。
  ```
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {

    public final Context mContext;
    final IWindowSession mWindowSession;
    Display mDisplay;
    final DisplayManager mDisplayManager;
    final String mBasePackageName;
    final int[] mTmpLocation = new int[2];
    final TypedValue mTmpValue = new TypedValue();
    final Thread mThread;
    final WindowLeaked mLocation;
    public final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();
    final W mWindow;
    final Choreographer mChoreographer;

    // 构造函数，获取IWindowSession对象
    public ViewRootImpl(Context context, Display display) {
        // 1. 调用WindowManagerGlobal.getWindowSession()方法获取IWindowSession
        this(context, display, WindowManagerGlobal.getWindowSession(),
                false /* useSfChoreographer */);
    }

    public ViewRootImpl(Context context, Display display, IWindowSession session,
                        boolean useSfChoreographer) {
        mContext = context;
        //获取IWindowSession的代理类
        mWindowSession = session;
        mDisplay = display;
        mThread = Thread.currentThread();
        // 创建W对象
        mWindow = new W(this);
        mChoreographer = useSfChoreographer
                ? Choreographer.getSfInstance() : Choreographer.getInstance();
        ...
    }

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
                        int userId) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                requestLayout();
                try {
                    // 1. 调用Session.addToDisplayAsUser()方法
                    res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mDisplayCutout, inputChannel,
                            mTempInsets, mTempControls);
                    setFrame(mTmpFrame);
                } catch (RemoteException e) {
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                }
            }
        }
    }
}

  ```
在ViewRootImpl的构造函数会调用`WindowManagerGlobal.getWindowSession()`方法获取`Session`实例，在ViewRootImpl的setView()方法中，会调用`Session.addToDisplayAsUser()`方法。
  ```
public final class WindowManagerGlobal {

    public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager.ensureDefaultInstanceForDefaultDisplayIfNecessary();
                    //获取WMS的代理类
                    IWindowManager windowManager = getWindowManagerService();
                    //经过Binder调用，最终调用WMS.openSession()方法
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            });
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }

}

public class WindowManagerService extends IWindowManager.Stub
        implements Watchdog.Monitor, WindowManagerPolicy.WindowManagerFuncs {

    @Override
    public IWindowSession openSession(IWindowSessionCallback callback) {
        // this为WindowManagerService对象
        return new Session(this, callback);
    }

}
  ```
`WindowManagerGlobal.getWindowSession()`方法通过`WMS. openSession()`方法创建一个`Session`对象并返回。                

  ```
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {

    final WindowManagerService mService;

    public int addToDisplayAsUser(IWindow window, int seq, WindowManager.LayoutParams attrs,
                                  int viewVisibility, int displayId, int userId, Rect outFrame,
                                  Rect outContentInsets, Rect outStableInsets,
                                  DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
                                  InsetsState outInsetsState, InsetsSourceControl[] outActiveControls) {
        // 1. 调用WindowManagerService的addWindow(）方法
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,
                outContentInsets, outStableInsets, outDisplayCutout, outInputChannel,
                outInsetsState, outActiveControls, userId);
    }
}
  ```
最后`ViewRootImpl.setView()`方法通过`WindowManagerService.addWindow()`去处理

## 2.1 WMS.addView添加窗口
  ```
    public int addWindow(Session session, IWindow client, int seq,
                         LayoutParams attrs, int viewVisibility, int displayId, Rect outFrame,
                         Rect outContentInsets, Rect outStableInsets,
                         DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel,
                         InsetsState outInsetsState, InsetsSourceControl[] outActiveControls,
                         int requestUserId) {
        int[] appOp = new int[1];

        // 1. 权限检查，调用phoneWindow.checkAddPermission()方法中
        int res = mPolicy.checkAddPermission(attrs.type, isRoundedCornerOverlay, attrs.packageName,
                appOp);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res; // 检查不通过返回
        }
        ...
        synchronized (mGlobalLock) {
            if (!mDisplayReady) {
                throw new IllegalStateException("Display has not been initialialized");
            }

            // 2. 通过displayId来查找对应的DisplayContent
            final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);

            // DisplayContent为null则返回
            if (displayContent == null) {
                ProtoLog.w(WM_ERROR, "Attempted to add window to a display that does "
                        + "not exist: %d. Aborting.", displayId);
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }
            //...

            if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
                //  通过attrs.token获取到父窗口
                parentWindow = windowForClientLocked(null, attrs.token, false);
                // 父窗口为空则返回
                if (parentWindow == null) {
                    ProtoLog.w(WM_ERROR, "Attempted to add window with token that is not a window: "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
                if (parentWindow.mAttrs.type >= FIRST_SUB_WINDOW
                        && parentWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                    ProtoLog.w(WM_ERROR, "Attempted to add window with token that is a sub-window: "
                            + "%s.  Aborting.", attrs.token);
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
            }

            if (type == TYPE_PRIVATE_PRESENTATION && !displayContent.isPrivate()) {
                ProtoLog.w(WM_ERROR,
                        "Attempted to add private presentation window to a non-private display.  "
                                + "Aborting.");
                return WindowManagerGlobal.ADD_PERMISSION_DENIED;
            }

            //...

            ActivityRecord activity = null;
            final boolean hasParent = parentWindow != null;
            // 3. 通过displayContent拿到WindowToken
            WindowToken token = displayContent.getWindowToken(
                    hasParent ? parentWindow.mAttrs.token : attrs.token);
            // 拿到父窗口的type
            final int rootType = hasParent ? parentWindow.mAttrs.type : type;

            boolean addToastWindowRequiresToken = false;

            if (token == null) {
                // 4. 有父窗口用父窗口的windowToken,没有则new一个
                if (hasParent) {
                    // Use existing parent window token for child windows.
                    token = parentWindow.mToken;
                } else {
                    final IBinder binder = attrs.token != null ? attrs.token : client.asBinder();
                    token = new WindowToken(this, binder, type, false, displayContent,
                            session.mCanAddInternalSystemWindow, isRoundedCornerOverlay);
                }
            } else if (rootType >= FIRST_APPLICATION_WINDOW
                    && rootType <= LAST_APPLICATION_WINDOW) {
                // 获取ActivityRecord
                activity = token.asActivityRecord();
            }
            //....

            // 5. 创建WindowState对象，包含窗口的状态信息
            final WindowState win = new WindowState(this, session, client, token, parentWindow,
                    appOp[0], seq, attrs, viewVisibility, session.mUid, userId,
                    session.mCanAddInternalSystemWindow);
            // 6. 判断请求添加窗口的客户端是否死亡
            if (win.mDeathRecipient == null) {
                ProtoLog.w(WM_ERROR, "Adding window client %s"
                        + " that is dead, aborting.", client.asBinder());
                return WindowManagerGlobal.ADD_APP_EXITING;
            }
            if (win.getDisplayContent() == null) {
                ProtoLog.w(WM_ERROR, "Adding window to Display that has been removed.");
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }

            final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
            // 根据窗口的type对窗口的LayoutParams的一些成员变量进行修改
            displayPolicy.adjustWindowParamsLw(win, win.mAttrs, callingPid, callingUid);
            // 准备将窗口添加到系统中
            res = displayPolicy.validateAddingWindowLw(attrs, callingPid, callingUid);
            if (res != WindowManagerGlobal.ADD_OKAY) {
                return res;
            }
            //...

            // 7. 执行windowState.attach()方法
            win.attach();
            // 8. 将WindowState加入到mWindowMap
            mWindowMap.put(client.asBinder(), win);
            // 9. 将WindowState加入到相应的WindowToken
            win.mToken.addWindow(win);
            return res;
        }
    }
  ```
`addWindow()`方法很长，大体上可以分为首先做一些权限的判断，然后拿到`WindowToken`,之后对token进行一些判断，再创建`WindowState`，并执行`WindowState.attach()`方法，并把WindowState加入到对应的WindowToken。



