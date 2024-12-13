> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/core/java/android/content/ContextWrapper.java
/frameworks/base/core/java/android/app/ContextImpl.java
/frameworks/base/core/java/android/content/ContentResolver.java
/frameworks/base/core/java/android/app/ActivityThread.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  ```
![](https://upload-images.jianshu.io/upload_images/22650779-619a519e3c1b2e0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## ContentProvider的简单使用
1. 继承`ContentProvider`并重写相关方法，在`AndroidManifest`文件中注册ContentProvider并指定唯一的`URI`。


```
public class GameProvider extends ContentProvider {
    public static final String AUTHORITY = "com.contentprovidertest";
    public static final Uri GAME_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/book");
    private static final UriMatcher mUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    private SQLiteDatabase mDb;
    private Context mContext;
    private String table;

    static {
        mUriMatcher.addURI(AUTHORITY, "book", 0);
    }

    @Override
    public boolean onCreate() {
        table = DbOpenHelper.GAME_TABLE_NAME;
        mContext = getContext();
        initProvider();
        return false;
    }

    private void initProvider() {
        mDb = new DbOpenHelper(mContext).getWritableDatabase();
        new Thread(new Runnable() {
            @Override
            public void run() {
                mDb.execSQL("delete from " + DbOpenHelper.GAME_TABLE_NAME);
                mDb.execSQL("insert into book values(1,'毛传\','伟大的一生');");
            }
        }).start();
    }

    @Override
    public String getType(Uri uri) {
        return null;
    }


    // 增
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        mDb.insert(table, null, values);
        return null;
    }

    // 查
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        String table = DbOpenHelper.GAME_TABLE_NAME;
        Cursor mCursor = mDb.query(table, projection, selection, selectionArgs, null, sortOrder, null);
        return mCursor;
    }

    // 删
    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        return 0;
    }

    // 更
    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        return 0;
    }
}
```

```
  <provider
            android:name=".GameProvider"
            android:authorities="com.contentprovidertest"
            android:exported="true"
            android:process=":provider" />
```
```
public class DbOpenHelper extends SQLiteOpenHelper {
    private static final String DB_NAME = "book_provider.db";
    static final String GAME_TABLE_NAME = "book";
    private static final int DB_VERSION = 1;

    private String CREATE_GAME_TABLE = "create table if not exists " + GAME_TABLE_NAME + "(_id integer primary key," + "name TEXT, " + "describe TEXT)";

    public DbOpenHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(CREATE_GAME_TABLE);
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

    }
}
```
2. 通过`Context`的获取到`contentResolver`对象，然后通过Uri对相应的ContentProvider进行`增删改查`操作。
```
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        insertAndQueryCP()
    }
    
    private fun insertAndQueryCP() {
        // 通过uri识别ContentProvider
        val uri = Uri.parse("content://com.contentprovidertest")

        // 增加
        val mContentValues = ContentValues()
        mContentValues.put("_id", 2)
        mContentValues.put("name", "毛选")
        mContentValues.put("describe", "实事求是")
        contentResolver.insert(uri, mContentValues)

        // 查询
        val gameCursor: Cursor? =
            contentResolver.query(uri, arrayOf("name", "describe"), null, null, null)
        while (gameCursor?.moveToNext() == true) {
            val mGame = Game(gameCursor.getString(0), gameCursor.getString(1))
            Log.d("MainActivity", mGame.gameName.toString() + "---" + mGame.gameDescribe)
        }
    }

}
```
`ContentProvider`对应着一个唯一的`Uri`，`ContentResolver`通过Uri找到对应的ContentProvider，并对相应的ContentProvider进行`增删改查`操作。

以下则已`contentResolver.query`的查询为例了解整个流程。

## ContentProvider的启动流程
#### 1. query方法到AMS的调用过程
  ```
  private fun insertAndQueryCP() {
        // 通过uri识别ContentProvider
        val uri = Uri.parse("content://com.contentprovidertest")

        // 增加
        val mContentValues = ContentValues()
        mContentValues.put("_id", 2)
        mContentValues.put("name", "毛选")
        mContentValues.put("describe", "实事求是")
        contentResolver.insert(uri, mContentValues)

        // 1. 查询
        val gameCursor: Cursor? =
            contentResolver.query(uri, arrayOf("name", "describe"), null, null, null)
     
       while (gameCursor?.moveToNext() == true) {
            val mGame = Game(gameCursor.getString(0), gameCursor.getString(1))
            Log.d("MainActivity", mGame.gameName.toString() + "---" + mGame.gameDescribe)
        }

    }
  ```
通过`contentResolver.query()`去查询数据。其中`contentProvider`返回的是`ApplicationContentResolver`对象。
```
class ContextImpl extends Context {

    private final ApplicationContentResolver mContentResolver;

    // 获取ContentResolver，实际是ApplicationContentResolver
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }

    // 构造函数
    private ContextImpl(@Nullable ContextImpl container, @NonNull ActivityThread mainThread,
                        @NonNull LoadedApk packageInfo, @Nullable String attributionTag,
                        @Nullable String splitName, @Nullable IBinder activityToken, @Nullable UserHandle user,
                        int flags, @Nullable ClassLoader classLoader, @Nullable String overrideOpPackageName) {
        // ...
        // 实例化ContentResolver对象
        mContentResolver = new ApplicationContentResolver(this, mainThread);
    }

    private static final class ApplicationContentResolver extends ContentResolver {
       //...
    }

}

```
ContextImpl在构造函数时会实例化`mContentResolver`的对象为`ApplicationContentResolver`，`ApplicationContentResolver`是`ContentResolver`的子类，也是ContextImpl的静态内部类，`query()`方法的实现在`ContentResolver`父类中。
```
public abstract class ContentResolver implements ContentInterface {

    // 查
    @Override
    public final @Nullable
    Cursor query(final @RequiresPermission.Read @NonNull Uri uri,
                 @Nullable String[] projection, @Nullable Bundle queryArgs,
                 @Nullable CancellationSignal cancellationSignal) {
        //...
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            //...
            try {
                qCursor = unstableProvider.query(mPackageName, mAttributionTag, uri, projection,
                        queryArgs, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                unstableProviderDied(unstableProvider);
                //unstable类型死亡后，再创建stable类型的provider
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                //再次执行查询操作
                qCursor = stableProvider.query(mPackageName, mAttributionTag, uri, projection,
                        queryArgs, remoteCancellationSignal);
            }

            if (qCursor == null) {
                return null;
            }

            //强制执行查询操作，可能会失败并跑出RuntimeException
            qCursor.getCount();
            //创建对象CursorWrapperInner
            final CursorWrapperInner wrapper = new CursorWrapperInner(qCursor, provider);
            stableProvider = null;
            qCursor = null;
            return wrapper;
        } catch (RemoteException e) {
            return null;
        } finally {
            //...
        }
    }
    
    // 增
    @Override
    public final @Nullable
    Uri insert(@RequiresPermission.Write @NonNull Uri url,
               @Nullable ContentValues values, @Nullable Bundle extras) {
        //...
        IContentProvider provider = acquireProvider(url);
        if (provider == null) {
            throw new IllegalArgumentException("Unknown URL " + url);
        }
        //...
        Uri createdRow = provider.insert(mPackageName, mAttributionTag, url, values, extras);
        return createdRow;
        //...
    }

    // 删
    @Override
    public final int delete(@RequiresPermission.Write @NonNull Uri url, @Nullable Bundle extras) {
        //...
        IContentProvider provider = acquireProvider(url);
        if (provider == null) {
            throw new IllegalArgumentException("Unknown URL " + url);
        }
        //...
        int rowsDeleted = provider.delete(mPackageName, mAttributionTag, url, extras);
        return rowsDeleted;
        //...
    }

    // 改
    @Override
    public final int update(@RequiresPermission.Write @NonNull Uri uri,
                            @Nullable ContentValues values, @Nullable Bundle extras) {

        //...
        IContentProvider provider = acquireProvider(uri);
        if (provider == null) {
            throw new IllegalArgumentException("Unknown URI " + uri);
        }
        //...
        int rowsUpdated = provider.update(mPackageName, mAttributionTag, uri, values, extras);
        return rowsUpdated;
        //...
    }

}

```
大致通过`acquireProvider()或acquireUnstableProvider()`方法获取到`ContentProvider`对象,获取到ContentProvider对象后调用相应的`query、insert、delete、update`方法。`acquireUnstableProvider()`方法的实现在`ApplicationContentResolver`类中:
```
 private static final class ApplicationContentResolver extends ContentResolver {
        @UnsupportedAppUsage
        private final ActivityThread mMainThread;

        public ApplicationContentResolver(Context context, ActivityThread mainThread) {
            super(context);
            mMainThread = Objects.requireNonNull(mainThread);
        }

        @Override
        @UnsupportedAppUsage
        protected IContentProvider acquireProvider(Context context, String auth) {
            // provider进程死亡后关联其的进程也会死亡
            return mMainThread.acquireProvider(context,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), true);
        }

        @Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            // provider进程死亡后关联其的进程也不会死亡
            return mMainThread.acquireProvider(c,
                    ContentProvider.getAuthorityWithoutUserId(auth),
                    resolveUserIdFromAuthority(auth), false);
        }

        //...
    }
```
后续通过`ActivityThread. acquireProvider()`或`acquireUnstableProvider()`方法查找到对应的`ContentProvider`对象。acquireProvider和acquireUnstableProvider区别在于provider进程死亡时关联其的进程是否也会受影响死亡。
```
public final class ActivityThread extends ClientTransactionHandler {

    public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        // 1. 先从本地中的mProviderMap中获取IContentProvider
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        ContentProviderHolder holder = null;
        try {
            synchronized (getGetProviderLock(auth, userId)) {
                // 2. 本地没有，则从AMS获取到ContentProviderHolder
                holder = ActivityManager.getService().getContentProvider(
                        getApplicationThread(), c.getOpPackageName(), auth, userId, stable);
            }
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }

        if (holder == null) {
            return null;
        }

        // 3. 安装provider，存储到mProviderMap中
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);

        // 返回ContentProviderHolder的provider
        return holder.provider;
    }

}
```
1. 在ActivityThread中首先从调用`acquireExistingProvider`从mProviderMap中本地获取`ContentProvider`。
2. 如果mProviderMap中没有对应的ContentProvider，则调用`AMS.getContentProvider()`去查找对应的`ContentProvider`。
3. 在AMS获取到对应的ContentProvider后，会调用`installProvider()`方法，将ContentProvider存储到`mProviderMap`变量中。

**1. 从ActivityThread本地中查询：**
```
public final class ActivityThread extends ClientTransactionHandler {

    final ArrayMap<ProviderKey, ProviderClientRecord> mProviderMap
            = new ArrayMap<ProviderKey, ProviderClientRecord>();
    
    public final IContentProvider acquireExistingProvider(
            Context c, String auth, int userId, boolean stable) {
        synchronized (mProviderMap) {
            //从AT.mProviderMap查询是否存在相对应的provider
            final ProviderKey key = new ProviderKey(auth, userId);
            // 从mProviderMap变量中查询对应ProviderClientRecord
            final ProviderClientRecord pr = mProviderMap.get(key);
            if (pr == null) {
                return null;
            }

            IContentProvider provider = pr.mProvider;
            IBinder jBinder = provider.asBinder();
            //当provider所在进程已经死亡则返回
            if (!jBinder.isBinderAlive()) {
                //清理provider信息
                handleUnstableProviderDiedLocked(jBinder, true);
                return null;
            }
            ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
            if (prc != null) {
                //增加引用计数
                incProviderRefLocked(prc, stable);
            }
            return provider;
        }
    }
}
```

#### 2. AMS查找ContentProvider的过程
```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    @Override
    public final ContentProviderHolder getContentProvider(
            IApplicationThread caller, String callingPackage, String name, int userId,
            boolean stable) {
        if (caller == null) {
            throw new SecurityException(msg);
        }
        //...
        // 实际调用getContentProviderImpl
        return getContentProviderImpl(caller, name, null, callingUid, callingPackage,
                null, stable, userId);
    }
```
实际调用`getContentProviderImpl()`方法去完成查询操作：
```
   final ProviderMap mProviderMap;

    private ContentProviderHolder getContentProviderImpl(IApplicationThread caller,
                                                         String name, IBinder token, int callingUid, String callingPackage, String callingTag,
                                                         boolean stable, int userId) {
        ContentProviderRecord cpr;
        ContentProviderConnection conn = null;
        ProviderInfo cpi = null;

        synchronized (this) {
            //...
            // 1. 从AMS本地中查询相应的ContentProviderRecord
            cpr = mProviderMap.getProviderByName(name, userId);

            //...
            boolean providerRunning = false;
            if (cpr != null && cpr.proc != null) {
                // 判断目标provider是否已存在
                providerRunning = !cpr.proc.killed;
                //...
            }
            // 目标provider已存在的情况
            if (providerRunning) {
                //...
            }

            // 目标provider不存在的情况
            if (!providerRunning) {
                // 获取目标ContentProvider的应用程序进程信息
                ProcessRecord proc = getProcessRecordLocked(
                        cpi.processName, cpr.appInfo.uid, false);

                if (proc != null && proc.thread != null && !proc.killed) {
                    if (!proc.pubProviders.containsKey(cpi.name)) {
                        proc.pubProviders.put(cpi.name, cpr);
                        try {
                            //2. provider进程已启动但未发布
                            proc.thread.scheduleInstallProvider(cpi);
                        } catch (RemoteException e) {
                        }
                    }
                } else {
                    // 3. Provider进程尚未启动，则启动新进程
                    proc = startProcessLocked(cpi.processName,
                            cpr.appInfo, false, 0,
                            new HostingRecord("content provider",
                                    new ComponentName(cpi.applicationInfo.packageName,
                                            cpi.name)),
                            ZYGOTE_POLICY_FLAG_EMPTY, false, false, false);
                    if (proc == null) {
                        return null;
                    }
                }
            }
            //...
            return cpr.newHolder(conn);
        }

```
第一：从AMS的`mProviderMap`变量中查找是否已经存在对应`ContentProviderRecord`，有则从AMS中返回。
第二：对于provider进程已启动但未发布，就调用`ActivityThread.scheduleInstallProvider()`方法发布Provider
第三：对于provider进程未启动，则调用`startProcessLocked()`方法启动进程。

**1. provider进程已启动但未发布**
调用ActivityThread.scheduleInstallProvider()方法：
```
public final class ActivityThread extends ClientTransactionHandler {

    public void scheduleInstallProvider(ProviderInfo provider) {
        // INSTALL_PROVIDER消息会调用对应的handleInstallProvider()方法
        sendMessage(H.INSTALL_PROVIDER, provider);
    }

    public void handleInstallProvider(ProviderInfo info) {
        try {
            // 调用installContentProviders()方法
            installContentProviders(mInitialApplication, Arrays.asList(info));
        } finally {
            StrictMode.setThreadPolicy(oldPolicy);
        }
    }

    private void installContentProviders(
            Context context, List<ProviderInfo> providers) {

        final ArrayList<ContentProviderHolder> results = new ArrayList<>();

        // 循环providers生成对应的ContentProviderHolder
        for (ProviderInfo cpi : providers) {
            if (DEBUG_PROVIDER) {
                StringBuilder buf = new StringBuilder(128);
                buf.append("Pub ");
                buf.append(cpi.authority);
                buf.append(": ");
                buf.append(cpi.name);
                Log.i(TAG, buf.toString());
            }
            //  调用installProvider将
            ContentProviderHolder cph = installProvider(context, null, cpi,
                    false /*noisy*/, true /*noReleaseNeeded*/, true /*stable*/);
            if (cph != null) {
                cph.noReleaseNeeded = true;
                results.add(cph);
            }

        }

        try {
            // 将ContentProviderHolder的List发布到AMS中
            ActivityManager.getService().publishContentProviders(
                    getApplicationThread(), results);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```
scheduleInstallProvider()方法会将Providers遍历执行`installProvider()`方法生成对应的`ContentProviderHolder`对象，installProvider()会将ContentProvider保存在mProviderMap中，并将之发布到AMS中。
```
    // AMS
    public final void publishContentProviders(IApplicationThread caller,
                                              List<ContentProviderHolder> providers) {
        if (providers == null) {
            return;
        }
        synchronized (this) {
            //...
            final int N = providers.size();
            for (int i = 0; i < N; i++) {
                ContentProviderHolder src = providers.get(i);
                if (src == null || src.info == null || src.provider == null) {
                    continue;
                }
                ContentProviderRecord dst = r.pubProviders.get(src.info.name);
                if (dst != null) {
                    ComponentName comp = new ComponentName(dst.info.packageName, dst.info.name);
                    // put到mProviderMap中
                    mProviderMap.putProviderByClass(comp, dst);
                    String names[] = dst.info.authority.split(";");
                    for (int j = 0; j < names.length; j++) {
                        mProviderMap.putProviderByName(names[j], dst);
                    }
                    //...
                }
            }
        }
    }

```
AMS将ContentProviderHolder转换成ContentProviderRecord并存储到AMS的mProviderMap变量中。

**2. provider进程未启动**
`AMS.startProcessLocked`会通过socket通知`Zygote`进程去fork()一个进程出来，之后调用该进程的`ActivityThread.main()`方法，并开始启动`Application`,在启动Application中会调用`installContentProviders()`方法发布ContentProvider。
启动Application方法为：`handleBindApplication`方法
```
 private void handleBindApplication(AppBindData data) {

    
        // 创建LoadedApk和ContextImpl
        final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                appContext.getClassLoader(), false, true, false);
        final ContextImpl instrContext = ContextImpl.createAppContext(this, pi,
                appContext.getOpPackageName());
        // 反射创建Application
        try {
            final ClassLoader cl = instrContext.getClassLoader();
            mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                            + data.instrumentationName + ": " + e.toString(), e);
        }

        final ComponentName component = new ComponentName(ii.packageName, ii.name);
        mInstrumentation.init(this, instrContext, appContext, component,
                data.instrumentationWatcher, data.instrumentationUiAutomationConnection);

        try {
            // 安装ContentProvider
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    installContentProviders(app, data.providers);
                }
            }
            
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            } catch (Exception e) {
                throw new RuntimeException(
                        "Exception thrown in onCreate() of "
                                + data.instrumentationName + ": " + e.toString(), e);
            }
            try {
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                //...
            }
        } finally {
            //...
        }

    }

```
同样是调用`installContentProviders`方法，上面已经分析过了。

此时AMS已经返回一个ContentProviderHolder对面给AT了，AT拿到后调用会调用`installProvider()`方法保存到自己的变量中，然后返回给`ContentResolver`执行对应的`query`变量。

#### installProvider
```
    private ContentProviderHolder installProvider(Context context,
                                                  ContentProviderHolder holder, ProviderInfo info,
                                                  boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;

        if (holder == null || holder.provider == null) {
            try {
                // 反射创建ContentProvider
                final java.lang.ClassLoader cl = c.getClassLoader();
                LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
                localProvider = packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                provider = localProvider.getIContentProvider();
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                return null;
            }
        } else {
            provider = holder.provider;
        }

        ContentProviderHolder retHolder;

        synchronized (mProviderMap) {
            IBinder jBinder = provider.asBinder();
            // 通过反射创建localProvider != null
            if (localProvider != null) {
                ComponentName cname = new ComponentName(info.packageName, info.name);
                ProviderClientRecord pr = mLocalProvidersByName.get(cname);
                if (pr != null) {
                    provider = pr.mProvider;
                } else {
                    holder = new ContentProviderHolder(info);
                    holder.provider = provider;
                    holder.noReleaseNeeded = true;
                    pr = installProviderAuthoritiesLocked(provider, localProvider, holder);
                    mLocalProviders.put(jBinder, pr);
                    mLocalProvidersByName.put(cname, pr);
                }
                retHolder = pr.mHolder;
            } else {
                ProviderRefCount prc = mProviderRefCountMap.get(jBinder);
                //mProviderRefCountMap变量是否已经有此ProviderRefCount
                if (prc != null) {
                    //只有当需要释放引用时则进入该分支
                    if (!noReleaseNeeded) {
                        //向ams来增加引用计数
                        incProviderRefLocked(prc, stable);
                        try {
                            ActivityManager.getService().removeContentProvider(
                                    holder.connection, stable);
                        } catch (RemoteException e) {
                            //do nothing content provider object is dead any way
                        }
                    }
                } else {
                    //installProviderAuthoritiesLocked加入到mProviderMap变量，并创建一个ProviderClientRecord
                    ProviderClientRecord client = installProviderAuthoritiesLocked(
                            provider, localProvider, holder);

                    if (noReleaseNeeded) {
                        prc = new ProviderRefCount(holder, client, 1000, 1000);
                    } else {
                        prc = stable
                                ? new ProviderRefCount(holder, client, 1, 0)
                                : new ProviderRefCount(holder, client, 0, 1);
                    }
                    // 加入到mProviderRefCountMap变量中
                    mProviderRefCountMap.put(jBinder, prc);
                }
                retHolder = prc.holder;
            }
        }
        return retHolder;
    }

    private ProviderClientRecord installProviderAuthoritiesLocked(IContentProvider provider,
                                                                  ContentProvider localProvider, ContentProviderHolder holder) {
        final String auths[] = holder.info.authority.split(";");
        final int userId = UserHandle.getUserId(holder.info.applicationInfo.uid);

        if (provider != null) {
            // If this provider is hosted by the core OS and cannot be upgraded,
            // then I guess we're okay doing blocking calls to it.
            for (String auth : auths) {
                switch (auth) {
                    case ContactsContract.AUTHORITY:
                    case CallLog.AUTHORITY:
                    case CallLog.SHADOW_AUTHORITY:
                    case BlockedNumberContract.AUTHORITY:
                    case CalendarContract.AUTHORITY:
                    case Downloads.Impl.AUTHORITY:
                    case "telephony":
                        Binder.allowBlocking(provider.asBinder());
                }
            }
        }

        final ProviderClientRecord pcr = new ProviderClientRecord(
                auths, provider, localProvider, holder);
        for (String auth : auths) {
            final ProviderKey key = new ProviderKey(auth, userId);
            final ProviderClientRecord existing = mProviderMap.get(key);
            if (existing != null) {
            } else {
                // 加入到mProviderMap变量中
                mProviderMap.put(key, pcr);
            }
        }
        return pcr;
    }
}
```
installProvider()方法如果判断`ContentProvider`没有创建会通过反射创建ContentProvider。后面判断这个有没有加入到mProviderRefCountMap中，如果没有则分别加入到mProviderMap和mProviderRefCountMap中，方便下次直接获取使用。

## 总结
首先调用`contentResolver.query`等API等，其中`contentResolver`的实例是`ApplicationContentResolver`,通过在ContentImpl的构造函数完成对象的初始化。`ApplicationContentResolver`继承了`ContentResolver`,像query等增删改查的API就写在ContentResolver类中，大致流程是流程是调用`acquireProvider或acquireUnstableProvider`方法拿到对应的`ContentProvider`。
`ApplicationContentResolver`实现了`acquireProvider或acquireUnstableProvider`并且是调用了`ActivityThread. acquireProvider`查找ContentProvider。
`ActivityThread`先从本地`mProviderMap`查找对应的ContentProvider，没有在从AMS中要对应的ContentProvider，并从AMS中得到的ContentProvider存储到`mProviderMap`变量中方便下次使用
`AMS`查找ContentProvider，也是先从本地的`mProviderMap`变量中是否有对应的ContentProvider缓存，有则直接返回。没有则查看provider进程是否有运行，如果provider进程启动了但未发布，则调用相应的`ActivityThread. scheduleInstallProvider`启动ContentProvider，并将发布到AMS中。如果provider进程未启动，则启动进程，在创建Application时会发布安装ContentProvider。


