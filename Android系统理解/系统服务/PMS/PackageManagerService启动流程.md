> 本次源码基于Android11分析

## PKMS的启动流程
packageManagerService作为系统的核心服务，其作用是：对应用进行**安装、卸载和信息查询**。此篇文章分析PKMS的启动流程，其启动流程大致如下：
![](https://upload-images.jianshu.io/upload_images/22650779-dbde1765f5b8df13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
关于PackageManagerService的类结构关系如下图所示：
![](https://upload-images.jianshu.io/upload_images/22650779-31659f11eb3f58c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### PKMS的启动
PKMS作为系统的核心服务，也是从[SystemServer进程](https://www.jianshu.com/p/987a43c1a708)中创建启动的，在其`startBootstrapServices、startOtherServices`两个方法中启动PKMS的服务。

  ```
public final class SystemServer implements Dumpable {

    private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        // 1. 启动Installer服务
        Installer installer = mSystemServiceManager.startService(Installer.class);

        // 2. 获取设备是否加密，加密则仅运行“核心”应用程序。
        String cryptState = VoldProperties.decrypt().orElse("");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            mOnlyCore = true;
        }

        // 3. 调用PKMS.main ，实例化PKMS构造 (重点)
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                domainVerificationService, mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF,
                mOnlyCore);

        // 4. 如果设备没有加密，操作它。管理A/B OTA dexopting
        if (!mOnlyCore) {
            OtaDexoptService.main(mSystemContext, mPackageManagerService);
        }

    }

    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        if (!mOnlyCore) {
            // 5. dex优化操作
            mPackageManagerService.updatePackagesIfNeeded();
        }

        // 6. 磁盘优化维护操作
        mPackageManagerService.performFstrimIfNeeded();

        // 7. PKMS准备就绪
        mPackageManagerService.systemReady();

    }
}
  ```
在`startBootstrapServices`方法中关于PKMS主要做了四件事：
> 1.启动Installer服务
2.获取设备是否加密，加密则仅运行“核心”应用程序。
3.调用PKMS.main ，实例化PKMS构造 (重点)
4.如果设备没有加密，操作它。管理A/B OTA dexopting
其中第三点：调用**PKMS.main**会创建PKMS实例，并调用PKMS的构造函数，这也是耗时比较久的地方，也是重点分析的地方

在`startOtherServices`方法关于PKMS主要做了三件事：
> 1.dex优化操作
2.磁盘优化维护操作
3.PKMS准备就绪
其中dex优化操作也是比较耗时的操作

以上便是`PKMS`出生的启动的地方，其中最主要的是调用`PKMS.main`方法：
  ```
 public static PackageManagerService main(Context context, Installer installer,
                                             boolean factoryTest, boolean onlyCore) {
        // 检查Package编译相关系统属性
        PackageManagerServiceCompilerMapping.checkProperties();
        .....
        //执行pkms构造方法
        PackageManagerService m = new PackageManagerService(injector, onlyCore, factoryTest);
        //启动部分应用服务于多用户场景
        m.installWhitelistedSystemPackages();
        //往ServiceManager中注册"package"和"package_native"
        ServiceManager.addService("package", m);
        final PackageManagerNative pmn = m.new PackageManagerNative();
        ServiceManager.addService("package_native", pmn);
        return m;
    }
  ```
> main()方法通过调用PackageManagerService的构造函数创建实例，并把PKMS实例向`ServiceManager`中注册。PKMS的构造函数的代码很长，通过下小节分析它。

### PKMS的构造函数
构造函数分为五个阶段，如下代码简易概括每个阶段的重点执行的事情：
  ```
 public PackageManagerService(Injector injector, boolean onlyCore, boolean factoryTest) {
        ......
        // 阶段一： 开始阶段
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());
        // 1. 构造 DisplayMetrics，保存分辨率相关信息
        // 2. 创建Installer对象，与installd交互
        // 3. 创建mPermissionManager对象，进行权限管理
        // 4. 构造Settings类，保存安装包信息，清除路径不存在的孤立应用，主要涉及/data/system/目录的packages.xml, packages-backup.xml,packages.list,packages-stopped.xml,packages-stopped-backup.xml等文件。
        // 5. 构造PackageDexOptimizer及DexManager类，处理dex优化
        // 6. 创建SystemConfig实例，获取系统配置信息，配置共享lib库
        // 7. 创建PackageManager的Handler线程，循环处理外部安装相关消息。
        ......

        // 阶段二：系统扫描阶段
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                startTime);
        // 1. 从init.rc中获取环境变量BOOTCLASSPATH和SYSTEMSERVERCLASSPATH；
        // 2. 对于旧版本升级的情况，将安装时获取权限变更为运行时申请权限
        // 3. 扫描system/vendor/product/odm/oem等目录的priv-app、app、overlay包
        // 4. 清除安装时临时文件以及其他不必要的信息。
        //扫描各个系统分区的的App
        //解析系统App信息
        //把解析结果存储起来，存储在PMS的相关属性和mSettings里
        ......

        if (!mOnlyCore) {
            // 阶段三： Data扫描阶段
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                    SystemClock.uptimeMillis());

            // 1. 扫描/data/app下的App，也就是用户安装的App
            scanDirTracedLI(sAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0,
                    packageParser, executorService);
        }
        ......

        // 阶段四，扫描结束阶段
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                SystemClock.uptimeMillis());
        // 1.sdk版本变更，更新权限
        // 2.OTA升级后首次启动，清除不必要的缓存数据
        // 3.权限等默认项更新完后，清理相关数据
        // 4.更新package.xml
        ......

        // 阶段五：就绪阶段
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                SystemClock.uptimeMillis());
        // 1. 创建PackageInstallerService对象
        // 2. GC回收内存

    }

  ```
> `阶段一`：主要是解析了package.xml，获取已经安装的App信息，存储到Settings的mPackages中。并创建了工作线程和hanlder。
`阶段二`：扫描解析系统App,并把系统App信息存储在PKMS的相关属性和mSettings里
`阶段三`：对对/data/app路径扫描，此路径是用户安装的App
`阶段四`：对SDK版本进行检查更新，为系统核心服务准备存储空间、把更新后的信息写回对应的xml文件中。
`阶段五`：初始化PackageInstallerService

其中阶段三的扫描用户安装的app,是应该重点关注的，其调用`scanDirTracedLI`方法对App进行扫描解析。

### PKMS的扫描流程
PKMS会扫描如下路径的App
> // 系统App
/vendor/overlay
/product/overlay
/product_services/overlay
/odm/overlay
/oem/overlay
/system/framework
/system/priv-app
/system/app
/vendor/priv-app
/vendor/app
/odm/priv-app
/odm/app
/oem/app
/oem/priv-app
/product/priv-app
/product/app
/product_services/priv-app
/product_services/app
/product_services/priv-app
// 用户App
/data/app


  ```
  private void scanDirTracedLI(File scanDir, final int parseFlags, int scanFlags,
                                 long currentTime, PackageParser2 packageParser, ExecutorService executorService) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir [" + scanDir.getAbsolutePath() + "]");
        try {
            // 调用scanDirLI
            scanDirLI(scanDir, parseFlags, scanFlags, currentTime, packageParser, executorService);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }

    private void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime,
                           PackageParser2 packageParser, ExecutorService executorService) {
        final File[] files = scanDir.listFiles();
        if (ArrayUtils.isEmpty(files)) {
            Log.d(TAG, "No files in app dir " + scanDir);
            return;
        }

        if (DEBUG_PACKAGE_SCANNING) {
            Log.d(TAG, "Scanning app dir " + scanDir + " scanFlags=" + scanFlags
                    + " flags=0x" + Integer.toHexString(parseFlags));
        }

        // ParallelPackageParser是一个队列，收集系统 apk 文件，
        // 然后从这个队列里面一个个取出apk，调用PackageParser解析
        ParallelPackageParser parallelPackageParser =
                new ParallelPackageParser(packageParser, executorService);

        // Submit files for parsing in parallel
        int fileCount = 0;
        for (File file : files) {
            // 是 apk文件，或者是目录
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            // 过滤掉非 apk 文件，如果不是则跳过继续扫描
            if (!isPackage) {
                // Ignore entries which are not packages
                continue;
            }
            // 把 apk 信息存入ParallelPackageParser中的对象mQueue，
            // parsePackage()函数赋予给了队列中的parsedPackage成员
            parallelPackageParser.submit(file, parseFlags);
            fileCount++;
        }

        // Process results one by one
        for (; fileCount > 0; fileCount--) {
            ParallelPackageParser.ParseResult parseResult = parallelPackageParser.take();
            Throwable throwable = parseResult.throwable;
            int errorCode = PackageManager.INSTALL_SUCCEEDED;

            if (throwable == null) {
                // TODO(toddke): move lower in the scan chain
                // Static shared libraries have synthetic package names
                if (parseResult.parsedPackage.isStaticSharedLibrary()) {
                    renameStaticSharedLibraryPackage(parseResult.parsedPackage);
                }
                try {
                     // 更新mSettings中相关包的数据
                     // 把包的信息存储到mPackages里面
                    addForInitLI(parseResult.parsedPackage, parseFlags, scanFlags,
                            currentTime, null);
                } catch (PackageManagerException e) {
                    errorCode = e.error;
                    Slog.w(TAG, "Failed to scan " + parseResult.scanFile + ": " + e.getMessage());
                }
            } else if (throwable instanceof PackageParserException) {
                PackageParserException e = (PackageParserException)
                        throwable;
                errorCode = e.error;
                Slog.w(TAG, "Failed to parse " + parseResult.scanFile + ": " + e.getMessage());
            } else {
                throw new IllegalStateException("Unexpected exception occurred while parsing "
                        + parseResult.scanFile, throwable);
            }

            if ((scanFlags & SCAN_AS_APK_IN_APEX) != 0 && errorCode != INSTALL_SUCCEEDED) {
                mApexManager.reportErrorWithApkInApex(scanDir.getAbsolutePath());
            }

            // 如果是非系统 apk 并且解析失败
            if ((scanFlags & SCAN_AS_SYSTEM) == 0
                    && errorCode != PackageManager.INSTALL_SUCCEEDED) {
                logCriticalInfo(Log.WARN,
                        "Deleting invalid package at " + parseResult.scanFile);
                // 非系统 Package 扫描失败，删除文件
                removeCodePathLI(parseResult.scanFile);
            }
        }
    }
  ```
> `scanDirLI`方法：遍历目录下所有文件，如果文件不是Apk或目录则不扫描，之后调用`ParallelPackageParser.submit`。如果非系统的App没有解析失败将被删除。
 ```
    public void submit(File scanFile, int parseFlags) {
        mExecutorService.submit(() -> {
            ParseResult pr = new ParseResult();
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parallel parsePackage [" + scanFile + "]");
            try {
                pr.scanFile = scanFile;
                // 通过parsePackage()方法解析
                pr.parsedPackage = parsePackage(scanFile, parseFlags);
            } catch (Throwable e) {
                pr.throwable = e;
            } finally {
                Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
            }
            try {
                // 将ParseResult加入到mQueue队列
                mQueue.put(pr);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                // Propagate result to callers of take().
                // This is helpful to prevent main thread from getting stuck waiting on
                // ParallelPackageParser to finish in case of interruption
                mInterruptedInThread = Thread.currentThread().getName();
            }
        });
    }

    protected ParsedPackage parsePackage(File scanFile, int parseFlags)
            throws PackageParser.PackageParserException {
        // 调用 PackageParser2.parsePackage方法
        return mPackageParser.parsePackage(scanFile, parseFlags, true);
    }
  ```
> 通过`PackageParser2.parsePackage`解析Apk文件
  ```
     // PackageParser2
    public ParsedPackage parsePackage(File packageFile, int flags, boolean useCaches)
            throws PackageParserException {
        ......
        // 调用ParsingPackageUtils.parsePackage进行解析
        ParseResult<ParsingPackage> result = parsingUtils.parsePackage(input, packageFile, flags);
        ......
        ParsedPackage parsed = (ParsedPackage) result.getResult().hideAsParsed();
        return parsed;
    }

    //ParsingPackageUtils
    public ParseResult<ParsingPackage> parsePackage(ParseInput input, File packageFile,
                                                    int flags)
            throws PackageParserException {
        // 如果是目录，调用parseClusterPackage
        if (packageFile.isDirectory()) {
            return parseClusterPackage(input, packageFile, flags);
        } else {
            // 如果是apk，调用parseMonolithicPackage
            return parseMonolithicPackage(input, packageFile, flags);
        }
    }
  ```
> 如果是目录调用`parseClusterPackage`方法,如果是apk文件调用`parseMonolithicPackage`方法
  ```
    private ParseResult<ParsingPackage> parseClusterPackage(ParseInput input, File packageDir,
                                                            int flags) {
        // 获取应用目录的PackageLite对象，这个对象分开保存了目录下的核心应用和非核心应用的名称
        ParseResult<PackageParser.PackageLite> liteResult =
                ApkLiteParseUtils.parseClusterPackageLite(input, packageDir, 0);
        if (liteResult.isError()) {
            return input.error(liteResult);
        }

        final PackageParser.PackageLite lite = liteResult.getResult();
        // 如果lite中没有核心应用，退出
        if (mOnlyCoreApps && !lite.coreApp) {
            return input.error(INSTALL_PARSE_FAILED_ONLY_COREAPP_ALLOWED,
                    "Not a coreApp: " + packageDir);
        }

        // Build the split dependency tree.
        SparseArray<int[]> splitDependencies = null;
        final SplitAssetLoader assetLoader;
        if (lite.isolatedSplits && !ArrayUtils.isEmpty(lite.splitNames)) {
            try {
                splitDependencies = SplitAssetDependencyLoader.createDependenciesFromPackage(lite);
                assetLoader = new SplitAssetDependencyLoader(lite, splitDependencies, flags);
            } catch (SplitAssetDependencyLoader.IllegalDependencyException e) {
                return input.error(INSTALL_PARSE_FAILED_BAD_MANIFEST, e.getMessage());
            }
        } else {
            assetLoader = new DefaultSplitAssetLoader(lite, flags);
        }

        try {
            final AssetManager assets = assetLoader.getBaseAssetManager();
            final File baseApk = new File(lite.baseCodePath);
            // 对核心应用进行解析
            ParseResult<ParsingPackage> result = parseBaseApk(input, baseApk,
                    lite.codePath, assets, flags);
            if (result.isError()) {
                return input.error(result);
            }

            ParsingPackage pkg = result.getResult();
            if (!ArrayUtils.isEmpty(lite.splitNames)) {
                pkg.asSplit(
                        lite.splitNames,
                        lite.splitCodePaths,
                        lite.splitRevisionCodes,
                        splitDependencies
                );
                final int num = lite.splitNames.length;

                for (int i = 0; i < num; i++) {
                    final AssetManager splitAssets = assetLoader.getSplitAssetManager(i);
                    // 对非核心应用的处理
                    parseSplitApk(input, pkg, i, splitAssets, flags);
                }
            }

            pkg.setUse32BitAbi(lite.use32bitAbi);
            return input.success(pkg);
        } catch (PackageParserException e) {
            return input.error(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to load assets: " + lite.baseCodePath, e);
        } finally {
            IoUtils.closeQuietly(assetLoader);
        }
    }


    private ParseResult<ParsingPackage> parseMonolithicPackage(ParseInput input, File apkFile,
                                                               int flags) throws PackageParserException {
        ParseResult<PackageParser.PackageLite> liteResult =
                ApkLiteParseUtils.parseMonolithicPackageLite(input, apkFile, flags);
        if (liteResult.isError()) {
            return input.error(liteResult);
        }

        final PackageParser.PackageLite lite = liteResult.getResult();
        if (mOnlyCoreApps && !lite.coreApp) {
            return input.error(INSTALL_PARSE_FAILED_ONLY_COREAPP_ALLOWED,
                    "Not a coreApp: " + apkFile);
        }

        final SplitAssetLoader assetLoader = new DefaultSplitAssetLoader(lite, flags);
        try {
            // 对核心应用解析
            ParseResult<ParsingPackage> result = parseBaseApk(input,
                    apkFile,
                    apkFile.getCanonicalPath(),
                    assetLoader.getBaseAssetManager(), flags);
            if (result.isError()) {
                return input.error(result);
            }

            return input.success(result.getResult()
                    .setUse32BitAbi(lite.use32bitAbi));
        } catch (IOException e) {
            return input.error(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to get path: " + apkFile, e);
        } finally {
            IoUtils.closeQuietly(assetLoader);
        }
    }
  ```
> 无论文件是目录还是apk，都会调用 `parseBaseApk`方法,从parseBaseApk方法开始解析AndroidManifest.xml文件

  ```
  private ParseResult<ParsingPackage> parseBaseApk(ParseInput input, File apkFile,
                                                     String codePath, AssetManager assets, int flags) {
        final String apkPath = apkFile.getAbsolutePath();

        String volumeUuid = null;
        if (apkPath.startsWith(PackageParser.MNT_EXPAND)) {
            final int end = apkPath.indexOf('/', PackageParser.MNT_EXPAND.length());
            volumeUuid = apkPath.substring(PackageParser.MNT_EXPAND.length(), end);
        }

        if (PackageParser.DEBUG_JAR) Slog.d(TAG, "Scanning base APK: " + apkPath);

        final int cookie = assets.findCookieForPath(apkPath);
        if (cookie == 0) {
            return input.error(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                    "Failed adding asset path: " + apkPath);
        }

        // assets.openXmlResourceParser(...)。获得一个 XML 资源解析对象，
        try (XmlResourceParser parser = assets.openXmlResourceParser(cookie,
                PackageParser.ANDROID_MANIFEST_FILENAME)) {
            final Resources res = new Resources(assets, mDisplayMetrics, null);
            // 再调用重载函数parseBaseApk
            ParseResult<ParsingPackage> result = parseBaseApk(input, apkPath, codePath, res,
                    parser, flags);
            if (result.isError()) {
                return input.error(result.getErrorCode(),
                        apkPath + " (at " + parser.getPositionDescription() + "): "
                                + result.getErrorMessage());
            }

            final ParsingPackage pkg = result.getResult();
            if (assets.containsAllocatedTable()) {
                final ParseResult<?> deferResult = input.deferError(
                        "Targeting R+ (version " + Build.VERSION_CODES.R + " and above) requires"
                                + " the resources.arsc of installed APKs to be stored uncompressed"
                                + " and aligned on a 4-byte boundary",
                        DeferredError.RESOURCES_ARSC_COMPRESSED);
                if (deferResult.isError()) {
                    return input.error(INSTALL_PARSE_FAILED_RESOURCES_ARSC_COMPRESSED,
                            deferResult.getErrorMessage());
                }
            }

            ApkAssets apkAssets = assets.getApkAssets()[0];
            if (apkAssets.definesOverlayable()) {
                SparseArray<String> packageNames = assets.getAssignedPackageIdentifiers();
                int size = packageNames.size();
                for (int index = 0; index < size; index++) {
                    String packageName = packageNames.get(index);
                    Map<String, String> overlayableToActor = assets.getOverlayableMap(packageName);
                    if (overlayableToActor != null && !overlayableToActor.isEmpty()) {
                        for (String overlayable : overlayableToActor.keySet()) {
                            pkg.addOverlayable(overlayable, overlayableToActor.get(overlayable));
                        }
                    }
                }
            }

            pkg.setVolumeUuid(volumeUuid);

            if ((flags & PackageParser.PARSE_COLLECT_CERTIFICATES) != 0) {
                pkg.setSigningDetails(getSigningDetails(pkg, false));
            } else {
                pkg.setSigningDetails(SigningDetails.UNKNOWN);
            }

            return input.success(pkg);
        } catch (Exception e) {
            return input.error(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to read manifest from " + apkPath, e);
        }
    }


    private ParseResult<ParsingPackage> parseBaseApk(ParseInput input, String apkPath,
                                                     String codePath, Resources res, XmlResourceParser parser, int flags)
            throws XmlPullParserException, IOException, PackageParserException {
        final String splitName;
        final String pkgName;

        ParseResult<Pair<String, String>> packageSplitResult =
                ApkLiteParseUtils.parsePackageSplitNames(input, parser, parser);
        if (packageSplitResult.isError()) {
            return input.error(packageSplitResult);
        }

        Pair<String, String> packageSplit = packageSplitResult.getResult();
        pkgName = packageSplit.first;
        splitName = packageSplit.second;

        if (!TextUtils.isEmpty(splitName)) {
            return input.error(
                    PackageManager.INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME,
                    "Expected base APK, but found split " + splitName
            );
        }

        // AndroidManifest
        final TypedArray manifestArray = res.obtainAttributes(parser, R.styleable.AndroidManifest);
        try {
            final boolean isCoreApp =
                    parser.getAttributeBooleanValue(null, "coreApp", false);
            final ParsingPackage pkg = mCallback.startParsingPackage(
                    pkgName, apkPath, codePath, manifestArray, isCoreApp);
            // 调用parseBaseApkTags()
            final ParseResult<ParsingPackage> result =
                    parseBaseApkTags(input, pkg, manifestArray, res, parser, flags);
            if (result.isError()) {
                return result;
            }

            return input.success(pkg);
        } finally {
            manifestArray.recycle();
        }
    }


    private ParseResult<ParsingPackage> parseBaseApkTags(ParseInput input, ParsingPackage pkg,
                                                         TypedArray sa, Resources res, XmlResourceParser parser, int flags)
            throws XmlPullParserException, IOException {
        ParseResult<ParsingPackage> sharedUserResult = parseSharedUser(input, pkg, sa);
        if (sharedUserResult.isError()) {
            return sharedUserResult;
        }

        pkg.setInstallLocation(anInteger(PackageParser.PARSE_DEFAULT_INSTALL_LOCATION,
                R.styleable.AndroidManifest_installLocation, sa))
                .setTargetSandboxVersion(anInteger(PackageParser.PARSE_DEFAULT_TARGET_SANDBOX,
                        R.styleable.AndroidManifest_targetSandboxVersion, sa))
                /* Set the global "on SD card" flag */
                .setExternalStorage((flags & PackageParser.PARSE_EXTERNAL_STORAGE) != 0);

        boolean foundApp = false;
        final int depth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG
                || parser.getDepth() > depth)) {
            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            String tagName = parser.getName();
            final ParseResult result;

            // TODO(b/135203078): Convert to instance methods to share variables
            // <application> has special logic, so it's handled outside the general method
            // 判断是是否是Application标签
            if (PackageParser.TAG_APPLICATION.equals(tagName)) {
                if (foundApp) {
                    if (PackageParser.RIGID_PARSER) {
                        result = input.error("<manifest> has more than one <application>");
                    } else {
                        Slog.w(TAG, "<manifest> has more than one <application>");
                        result = input.success(null);
                    }
                } else {
                    foundApp = true;
                    // 是application标签，调用parseBaseApplication方法，解析application
                    result = parseBaseApplication(input, pkg, res, parser, flags);
                }
            } else {
                // 如果不是application标签，调用parseBaseApkTag方法，解析核心应用的所有tag
                result = parseBaseApkTag(tagName, input, pkg, res, parser, flags);
            }

            if (result.isError()) {
                return input.error(result);
            }
        }

        if (!foundApp && ArrayUtils.size(pkg.getInstrumentations()) == 0) {
            ParseResult<?> deferResult = input.deferError(
                    "<manifest> does not contain an <application> or <instrumentation>",
                    DeferredError.MISSING_APP_TAG);
            if (deferResult.isError()) {
                return input.error(deferResult);
            }
        }

        if (!ParsedAttribution.isCombinationValid(pkg.getAttributions())) {
            return input.error(
                    INSTALL_PARSE_FAILED_BAD_MANIFEST,
                    "Combination <feature> tags are not valid"
            );
        }

        convertNewPermissions(pkg);

        convertSplitPermissions(pkg);

        // At this point we can check if an application is not supporting densities and hence
        // cannot be windowed / resized. Note that an SDK version of 0 is common for
        // pre-Doughnut applications.
        if (pkg.getTargetSdkVersion() < DONUT
                || (!pkg.isSupportsSmallScreens()
                && !pkg.isSupportsNormalScreens()
                && !pkg.isSupportsLargeScreens()
                && !pkg.isSupportsExtraLargeScreens()
                && !pkg.isResizeable()
                && !pkg.isAnyDensity())) {
            adjustPackageToBeUnresizeableAndUnpipable(pkg);
        }

        return input.success(pkg);
    }

    private ParseResult<ParsingPackage> parseBaseApplication(ParseInput input,
                                                             ParsingPackage pkg, Resources res, XmlResourceParser parser, int flags)
            throws XmlPullParserException, IOException {

        ......
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG
                || parser.getDepth() > depth)) {
            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final ParseResult result;
            // 获取 "application" 子标签的标签内容
            String tagName = parser.getName();
            boolean isActivity = false;
            switch (tagName) {
                // 如果标签是 "activity"
                case "activity":
                    isActivity = true;
                    // fall-through
                case "receiver":
                    // 如果标签是 "receiver"，获取receiver信息
                    ParseResult<ParsedActivity> activityResult =
                            ParsedActivityUtils.parseActivityOrReceiver(mSeparateProcesses, pkg,
                                    res, parser, flags, PackageParser.sUseRoundIcon, input);

                    if (activityResult.isSuccess()) {
                        ParsedActivity activity = activityResult.getResult();
                        if (isActivity) {
                            hasActivityOrder |= (activity.getOrder() != 0);
                            pkg.addActivity(activity);
                        } else {
                            hasReceiverOrder |= (activity.getOrder() != 0);
                            pkg.addReceiver(activity);
                        }
                    }

                    result = activityResult;
                    break;
                case "service":
                    // 解析 "service"
                    ParseResult<ParsedService> serviceResult =
                            ParsedServiceUtils.parseService(mSeparateProcesses, pkg, res, parser,
                                    flags, PackageParser.sUseRoundIcon, input);
                    if (serviceResult.isSuccess()) {
                        ParsedService service = serviceResult.getResult();
                        hasServiceOrder |= (service.getOrder() != 0);
                        pkg.addService(service);
                    }

                    result = serviceResult;
                    break;
                case "provider":
                    // 解析 "provider"
                    ParseResult<ParsedProvider> providerResult =
                            ParsedProviderUtils.parseProvider(mSeparateProcesses, pkg, res, parser,
                                    flags, PackageParser.sUseRoundIcon, input);
                    if (providerResult.isSuccess()) {
                        pkg.addProvider(providerResult.getResult());
                    }

                    result = providerResult;
                    break;
                case "activity-alias":
                    // 解析 "activity-alias"
                    activityResult = ParsedActivityUtils.parseActivityAlias(pkg, res,
                            parser, PackageParser.sUseRoundIcon, input);
                    if (activityResult.isSuccess()) {
                        ParsedActivity activity = activityResult.getResult();
                        hasActivityOrder |= (activity.getOrder() != 0);
                        pkg.addActivity(activity);
                    }

                    result = activityResult;
                    break;
                default:
                    result = parseBaseAppChildTag(input, tagName, pkg, res, parser, flags);
                    break;
            }

            if (result.isError()) {
                return input.error(result);
            }
        }

        if (TextUtils.isEmpty(pkg.getStaticSharedLibName())) {
            // Add a hidden app detail activity to normal apps which forwards user to App Details
            // page.
            ParseResult<ParsedActivity> a = generateAppDetailsHiddenActivity(input, pkg);
            if (a.isError()) {
                // Error should be impossible here, as the only failure case as of SDK R is a
                // string validation error on a constant ":app_details" string passed in by the
                // parsing code itself. For this reason, this is just a hard failure instead of
                // deferred.
                return input.error(a);
            }

            pkg.addActivity(a.getResult());
        }

        if (hasActivityOrder) {
            pkg.sortActivities();
        }
        if (hasReceiverOrder) {
            pkg.sortReceivers();
        }
        if (hasServiceOrder) {
            pkg.sortServices();
        }

        // Must be run after the entire {@link ApplicationInfo} has been fully processed and after
        // every activity info has had a chance to set it from its attributes.
        setMaxAspectRatio(pkg);
        setMinAspectRatio(pkg);
        setSupportsSizeChanges(pkg);

        pkg.setHasDomainUrls(hasDomainURLs(pkg));

        return input.success(pkg);
    }

  ```
> `parseBaseApk`方法开始解析AndroidManifest.xml文件，`parseBaseApkTags`方法中将application标签和其他标签分开处理，如果是application标签则里面定义了四大组件，通过`parseBaseApplication`方法对application标签进行解析。

App通过解析后，要对信息进行保存，则调用`PKMS. addForInitLI`
  ```
private AndroidPackage addForInitLI(ParsedPackage parsedPackage,
            @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
            @Nullable UserHandle user)
                    throws PackageManagerException {
        
        // 判断系统应用是否需要更新
        synchronized (mLock) {
            if (scanSystemPartition) {
                if (isSystemPkgUpdated) {
                   //......
                }
            }
        }
 
        if (isSystemPkgBetter) {
            // 更新安装包到 system 分区中
            synchronized (mLock) {
                // just remove the loaded entries from package lists
                mPackages.remove(pkgSetting.name);
            }
 
            // 创建安装参数 InstallArgs
            final InstallArgs args = createInstallArgsForExisting(
                    pkgSetting.codePathString,
                    pkgSetting.resourcePathString, getAppDexInstructionSets(
                            pkgSetting.primaryCpuAbiString, pkgSetting.secondaryCpuAbiString));
            args.cleanUpResourcesLI();
            synchronized (mLock) {
                mSettings.enableSystemPackageLPw(pkgSetting.name);
            }
        }

        // 安装包校验
        collectCertificatesLI(pkgSetting, parsedPackage, forceCollect, skipVerify);
                try (@SuppressWarnings("unused") PackageFreezer freezer = freezePackage(
                        parsedPackage.getPackageName(),
                        "scanPackageInternalLI")) {
                  // 如果两个apk签名不匹配，则调用 deletePackageLIF 方法清除 apk 文件及其数据
                    deletePackageLIF(parsedPackage.getPackageName(), null, true, null, 0, null,
                            false, null);
                }
                pkgSetting = null;
            } else if (newPkgVersionGreater) {
                // 更新系统 apk 程序
                InstallArgs args = createInstallArgsForExisting(
                        pkgSetting.codePathString,
                        pkgSetting.resourcePathString, getAppDexInstructionSets(
                                pkgSetting.primaryCpuAbiString, pkgSetting.secondaryCpuAbiString));
                synchronized (mInstallLock) {
                    args.cleanUpResourcesLI();
                }
            } 
        }
 
        // 如果新安装的系统 app 会被旧的 APP 数据覆盖，所以需要隐藏系统应用程序，并重新扫描 /data/app 目录
        if (shouldHideSystemApp) {
            synchronized (mLock) {
                mSettings.disableSystemPackageLPw(parsedPackage.getPackageName(), true);
            }
        }
        return scanResult.pkgSetting.pkg;
    }
  ```

### 总结：
PKMS的启动流程：在`SystemServer`进程中启动 `PKMS`，通过调用PKMS的构造函数遍历系统App和非系统App进行解析对AndroidManifest.xml的结点进行解析，然后将解析结果进行保存更新。

