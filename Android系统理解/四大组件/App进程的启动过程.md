> 本次源码基于Android11分析

相关源码：
  ```
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
/frameworks/base/services/core/java/com/android/server/am/ProcessList.java
/frameworks/base/core/java/android/os/Process.java
/frameworks/base/core/java/android/os/ZygoteProcess.java
/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
/frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
/frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
 /frameworks/base/core/java/com/android/internal/os/Zygote.java
 /frameworks/base/core/java/android/app/ActivityThread.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
/frameworks/base/core/java/android/app/Instrumentation.java
/frameworks/base/core/java/android/app/LoadedApk.java
  ```
## 进程的启动过程
在四大组件：Activity、Service、ContentProvider、BroadcastReceiver，启动过程中，如果其承载的进程不存在，则会调用`AMS.startProcessLocked`方法创建进程。

![](https://upload-images.jianshu.io/upload_images/22650779-d35ec9d3565e3102.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 1. AMS向Zygote请求创建进程
### AMS.startProcessLocked
调用`startProcessLocked()`方法开始启动进程
  ```
    final ProcessList mProcessList;

    final ProcessRecord startProcessLocked(String processName,
                                           ApplicationInfo info, boolean knownToBeDead, int intentFlags,
                                           HostingRecord hostingRecord, int zygotePolicyFlags, boolean allowWhileBooting,
                                           boolean isolated, boolean keepIfLarge) {
        // 将其请求创建进程逻辑交给ProcessList类处理
        return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
                hostingRecord, zygotePolicyFlags, allowWhileBooting, isolated, 0 /* isolatedUid */,
                keepIfLarge, null /* ABI override */, null /* entryPoint */,
                null /* entryPointArgs */, null /* crashHandler */);
    }

  ```
ActivityManagerService服务的startProcessLocked方法通过调用`ProcessList.startProcessLocked`方法，接下来看看ProcessList如何处理：

### ProcessList.startProcessLocked
  ```
public final class ProcessList {

    @GuardedBy("mService")
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
                                           boolean knownToBeDead, int intentFlags, HostingRecord hostingRecord,
                                           int zygotePolicyFlags, boolean allowWhileBooting, boolean isolated, int isolatedUid,
                                           boolean keepIfLarge, String abiOverride, String entryPoint, String[] entryPointArgs,
                                           Runnable crashHandler) {
        long startTime = SystemClock.uptimeMillis();
        ProcessRecord app;
        if (!isolated) {
            //根据进程名和uid检查相应的ProcessRecord
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
            if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
                //如果当前处于后台进程，检查当前进程是否处于bad进程列表，如果处于返回null
                if (mService.mAppErrors.isBadProcessLocked(info)) {
                    if (DEBUG_PROCESSES) Slog.v(TAG, "Bad process: " + info.uid
                            + "/" + info.processName);
                    return null;
                }
            } else {
                //当用户明确地启动进程，则清空crash次数，以保证其不处于bad进程直到下次再弹出crash对话框。
                mService.mAppErrors.resetProcessCrashTimeLocked(info);
                if (mService.mAppErrors.isBadProcessLocked(info)) {
                    EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                            UserHandle.getUserId(info.uid), info.uid,
                            info.processName);
                    mService.mAppErrors.clearBadProcessLocked(info);
                    if (app != null) {
                        app.bad = false;
                    }
                }
            }
        } else {
            //对于孤立进程，无法再利用已存在的进程
            app = null;
        }

        ProcessRecord precedence = null;
        //当存在ProcessRecord,且已分配pid(正在启动或者已经启动)
        if (app != null && app.pid > 0) {
            // 且caller并不认为该进程已死亡或者没有thread对象attached到该进程.则不应该清理该进程
            if ((!knownToBeDead && !app.killed) || app.thread == null) {
                //如果这是进程中新package，则添加到列表
                app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
                // 返回该ProcessRecord
                return app;
            }

            // 当ProcessRecord已经被attached到先前的一个进程，则杀死并清理该进程，并将app = null
            ProcessList.killProcessGroup(app.uid, app.pid);
            precedence = app;
            app = null;
        }

        if (app == null) {
            // 创建新的Process Record对象
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid, hostingRecord);
            if (app == null) {
                return null;
            }
            app.crashHandler = crashHandler;
            app.isolatedEntryPoint = entryPoint;
            app.isolatedEntryPointArgs = entryPointArgs;
            if (precedence != null) {
                app.mPrecedence = precedence;
                precedence.mSuccessor = app;
            }
            checkSlow(startTime, "startProcess: done creating new process record");
        } else {
            // 如果这是进程中新package，则添加到列表
            app.addPackage(info.packageName, info.longVersionCode, mService.mProcessStats);
        }

        //当系统未准备完毕，则将当前进程加入到mProcessesOnHold
        if (!mService.mProcessesReady
                && !mService.isAllowedWhileBooting(info)
                && !allowWhileBooting) {
            if (!mService.mProcessesOnHold.contains(app)) {
                mService.mProcessesOnHold.add(app);
            }
            return app;
        }

        // 启动进程
        final boolean success =
                startProcessLocked(app, hostingRecord, zygotePolicyFlags, abiOverride);
        return success ? app : null;
    }

    @GuardedBy("mService")
    final boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
                                     int zygotePolicyFlags, String abiOverride) {
        return startProcessLocked(app, hostingRecord, zygotePolicyFlags,
                false /* disableHiddenApiChecks */, false /* disableTestApiChecks */,
                false /* mountExtStorageFull */, abiOverride);
    }


    @GuardedBy("mService")
    boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
                               int zygotePolicyFlags, boolean disableHiddenApiChecks, boolean disableTestApiChecks,
                               boolean mountExtStorageFull, String abiOverride) {
        if (app.pendingStart) {
            return true;
        }
        long startTime = SystemClock.uptimeMillis();
        //当app的pid大于0且不是当前进程的pid，则从mPidsSelfLocked中移除该app.pid
        if (app.pid > 0 && app.pid != ActivityManagerService.MY_PID) {
            mService.removePidLocked(app);
            app.bindMountPending = false;
            mService.mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            app.setPid(0);
            app.startSeq = 0;
        }

        //从mProcessesOnHold移除该app
        mService.mProcessesOnHold.remove(app);
        //更新cpu统计信息
        mService.updateCpuStats();

        //当前package已被冻结,则抛出异常
        try {
            try {
                final int userId = UserHandle.getUserId(app.uid);
                AppGlobals.getPackageManager().checkPackageStartable(app.info.packageName, userId);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            int uid = app.uid;
            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;

            if (!app.isolated) {
                int[] permGids = null;
                try {
                    //通过Package Manager获取gids
                    final IPackageManager pm = AppGlobals.getPackageManager();
                    permGids = pm.getPackageGids(app.info.packageName,
                            MATCH_DIRECT_BOOT_AUTO, app.userId);
                    if (StorageManager.hasIsolatedStorage() && mountExtStorageFull) {
                        mountExternal = Zygote.MOUNT_EXTERNAL_FULL;
                    } else {
                        StorageManagerInternal storageManagerInternal = LocalServices.getService(
                                StorageManagerInternal.class);
                        mountExternal = storageManagerInternal.getExternalStorageMountMode(uid,
                                app.info.packageName);
                    }
                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }

                //添加共享app和gids，用于app直接共享资源
                if (app.processInfo != null && app.processInfo.deniedPermissions != null) {
                    for (int i = app.processInfo.deniedPermissions.size() - 1; i >= 0; i--) {
                        int[] denyGids = mService.mPackageManagerInt.getPermissionGids(
                                app.processInfo.deniedPermissions.valueAt(i), app.userId);
                        if (denyGids != null) {
                            for (int gid : denyGids) {
                                permGids = ArrayUtils.removeInt(permGids, gid);
                            }
                        }
                    }
                }

                gids = computeGidsForProcess(mountExternal, uid, permGids);
            }
            //....
            // 设置app ProcessRecord的参数
            app.gids = gids;
            app.setRequiredAbi(requiredAbi);
            app.instructionSet = instructionSet;

            // entryPoint 为 android.app.ActivityThread
            final String entryPoint = "android.app.ActivityThread";

            // 调用startProcessLocked方法，继续请求Zygote创建新进程
            return startProcessLocked(hostingRecord, entryPoint, app, uid, gids,
                    runtimeFlags, zygotePolicyFlags, mountExternal, seInfo, requiredAbi,
                    instructionSet, invokeWith, startTime);
        } catch (RuntimeException e) {
            //...
            return false;
        }
    }

    @GuardedBy("mService")
    boolean startProcessLocked(HostingRecord hostingRecord, String entryPoint, ProcessRecord app,
                               int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags, int mountExternal,
                               String seInfo, String requiredAbi, String instructionSet, String invokeWith,
                               long startTime) {
        // 最终都会调用startProcess()方法,调用Zygote进程创建进程
        final Process.ProcessStartResult startResult = startProcess(hostingRecord,
                entryPoint, app,
                uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                requiredAbi, instructionSet, invokeWith, startTime);
    }

    private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
                                                    ProcessRecord app, int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
                                                    int mountExternal, String seInfo, String requiredAbi, String instructionSet,
                                                    String invokeWith, long startTime) {
        try {
            //...
            // 通过Process.start通知Zygote进程创建进程
            startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                    isTopApp, app.mDisabledCompatChanges, pkgDataInfoMap,
                    whitelistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                    new String[]{PROC_START_SEQ_IDENT + app.startSeq});
            return startResult;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
    }
}

  ```
对于非独立App，会根据进程名和uid查找到进程的信息，如果存在进程并且不在bad进程列表并且没有thread对象attached到该进程则直接返回该进程信息。否则创建新的进程，并且通过`ProcessList. startProcessLocked`层层调用最终调用`Process.start`向Zygote进程发起请求。

### Process.start

  ```
public class Process {

    public static final ZygoteProcess ZYGOTE_PROCESS = new ZygoteProcess();

    public static ProcessStartResult start(@NonNull final String processClass,
                                           @Nullable final String niceName,
                                           int uid, int gid, @Nullable int[] gids,
                                           int runtimeFlags,
                                           int mountExternal,
                                           int targetSdkVersion,
                                           @Nullable String seInfo,
                                           @NonNull String abi,
                                           @Nullable String instructionSet,
                                           @Nullable String appDataDir,
                                           @Nullable String invokeWith,
                                           @Nullable String packageName,
                                           int zygotePolicyFlags,
                                           boolean isTopApp,
                                           @Nullable long[] disabledCompatChanges,
                                           @Nullable Map<String, Pair<String, Long>>
                                                   pkgDataInfoMap,
                                           @Nullable Map<String, Pair<String, Long>>
                                                   whitelistedDataInfoMap,
                                           boolean bindMountAppsData,
                                           boolean bindMountAppStorageDirs,
                                           @Nullable String[] zygoteArgs) {
        // 调用ZygoteProcess.start
        return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, packageName,
                zygotePolicyFlags, isTopApp, disabledCompatChanges,
                pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                bindMountAppStorageDirs, zygoteArgs);
    }
}
  ```
第一个参数processClass为：ActivityThread。
第二个参数niceName为：进程名  。
Process.start方法继续调用`ZygoteProcess. start`方法。

### ZygoteProcess. start
  ```
public class ZygoteProcess {
    public final Process.ProcessStartResult start(@NonNull final String processClass,
                                                  final String niceName,
                                                  int uid, int gid, @Nullable int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  @Nullable String seInfo,
                                                  @NonNull String abi,
                                                  @Nullable String instructionSet,
                                                  @Nullable String appDataDir,
                                                  @Nullable String invokeWith,
                                                  @Nullable String packageName,
                                                  int zygotePolicyFlags,
                                                  boolean isTopApp,
                                                  @Nullable long[] disabledCompatChanges,
                                                  @Nullable Map<String, Pair<String, Long>>
                                                          pkgDataInfoMap,
                                                  @Nullable Map<String, Pair<String, Long>>
                                                          whitelistedDataInfoMap,
                                                  boolean bindMountAppsData,
                                                  boolean bindMountAppStorageDirs,
                                                  @Nullable String[] zygoteArgs) {
        // TODO (chriswailes): Is there a better place to check this value?
        if (fetchUsapPoolEnabledPropWithMinInterval()) {
            informZygotesOfUsapPoolStatus();
        }

        try {
            // 调用startViaZygote
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, /*startChildZygote=*/ false,
                    packageName, zygotePolicyFlags, isTopApp, disabledCompatChanges,
                    pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                    bindMountAppStorageDirs, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            Log.e(LOG_TAG,
                    "Starting VM process through Zygote failed");
            throw new RuntimeException(
                    "Starting VM process through Zygote failed", ex);
        }
    }

    private Process.ProcessStartResult startViaZygote(@NonNull final String processClass,
                                                      @Nullable final String niceName,
                                                      final int uid, final int gid,
                                                      @Nullable final int[] gids,
                                                      int runtimeFlags, int mountExternal,
                                                      int targetSdkVersion,
                                                      @Nullable String seInfo,
                                                      @NonNull String abi,
                                                      @Nullable String instructionSet,
                                                      @Nullable String appDataDir,
                                                      @Nullable String invokeWith,
                                                      boolean startChildZygote,
                                                      @Nullable String packageName,
                                                      int zygotePolicyFlags,
                                                      boolean isTopApp,
                                                      @Nullable long[] disabledCompatChanges,
                                                      @Nullable Map<String, Pair<String, Long>>
                                                              pkgDataInfoMap,
                                                      @Nullable Map<String, Pair<String, Long>>
                                                              whitelistedDataInfoMap,
                                                      boolean bindMountAppsData,
                                                      boolean bindMountAppStorageDirs,
                                                      @Nullable String[] extraArgs)
            throws ZygoteStartFailedEx {

        // 往argsForZygote数组填写进程信息
        ArrayList<String> argsForZygote = new ArrayList<>();

        // --runtime-args, --setuid=, --setgid=,
        // and --setgroups= must go first
        argsForZygote.add("--runtime-args");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        argsForZygote.add("--runtime-flags=" + runtimeFlags);
        if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
            argsForZygote.add("--mount-external-default");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
            argsForZygote.add("--mount-external-read");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
            argsForZygote.add("--mount-external-write");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_FULL) {
            argsForZygote.add("--mount-external-full");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_INSTALLER) {
            argsForZygote.add("--mount-external-installer");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_LEGACY) {
            argsForZygote.add("--mount-external-legacy");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_PASS_THROUGH) {
            argsForZygote.add("--mount-external-pass-through");
        } else if (mountExternal == Zygote.MOUNT_EXTERNAL_ANDROID_WRITABLE) {
            argsForZygote.add("--mount-external-android-writable");
        }

        argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

        // --setgroups is a comma-separated list
        if (gids != null && gids.length > 0) {
            final StringBuilder sb = new StringBuilder();
            sb.append("--setgroups=");

            final int sz = gids.length;
            for (int i = 0; i < sz; i++) {
                if (i != 0) {
                    sb.append(',');
                }
                sb.append(gids[i]);
            }

            argsForZygote.add(sb.toString());
        }

        if (niceName != null) {
            argsForZygote.add("--nice-name=" + niceName);
        }

        if (seInfo != null) {
            argsForZygote.add("--seinfo=" + seInfo);
        }

        if (instructionSet != null) {
            argsForZygote.add("--instruction-set=" + instructionSet);
        }

        if (appDataDir != null) {
            argsForZygote.add("--app-data-dir=" + appDataDir);
        }
        // ...

        synchronized (mLock) {
            //先跟Zygote建立socket通信，然后通过socket向Zygote进程发送消息并返回结果
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                    zygotePolicyFlags,
                    argsForZygote);
        }
    }
  ```
调用`startViaZygote`方法将接收的进程的参数，写入到`argsForZygote`数组中。最后分别调用了`openZygoteSocketIfNeeded`方法建立与zygote进程的通信，和调用`zygoteSendArgsAndGetResult`方法给zygote进程发送并返回消息。

### openZygoteSocketIfNeeded与Zygote建立通信
  ```
public class ZygoteProcess {

    // 根据当前的abi,选择对应的zygote通信
    @GuardedBy("mLock")
    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        try {
            //向主zygote发起connect()操作
            attemptConnectionToPrimaryZygote();
            //  跟abi匹配成功
            if (primaryZygoteState.matches(abi)) {
                return primaryZygoteState;
            }

            if (mZygoteSecondarySocketAddress != null) {
                //当主zygote没能匹配成功，则采用第二个zygote，发起connect()操作
                attemptConnectionToSecondaryZygote();

                if (secondaryZygoteState.matches(abi)) {
                    return secondaryZygoteState;
                }
            }
        } catch (IOException ioe) {
            throw new ZygoteStartFailedEx("Error connecting to zygote", ioe);
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }

    // 向主zygote建立通信
    @GuardedBy("mLock")
    private void attemptConnectionToPrimaryZygote() throws IOException {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            //向主zygote发起connect()操作
            primaryZygoteState =
                    ZygoteState.connect(mZygoteSocketAddress, mUsapPoolSocketAddress);

            maybeSetApiBlacklistExemptions(primaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
        }
    }

     // 向次Zygote建立通信
    @GuardedBy("mLock")
    private void attemptConnectionToSecondaryZygote() throws IOException {
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            //向第二zygote发起connect()操作
            secondaryZygoteState =
                    ZygoteState.connect(mZygoteSecondarySocketAddress,
                            mUsapPoolSecondarySocketAddress);

            maybeSetApiBlacklistExemptions(secondaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(secondaryZygoteState);
        }
    }

    private static class ZygoteState implements AutoCloseable {

        // ZygoteState.connect建立通信的方法
        static ZygoteState connect(@NonNull LocalSocketAddress zygoteSocketAddress,
                                   @Nullable LocalSocketAddress usapSocketAddress)
                throws IOException {

            DataInputStream zygoteInputStream;
            BufferedWriter zygoteOutputWriter;
            final LocalSocket zygoteSessionSocket = new LocalSocket();

            if (zygoteSocketAddress == null) {
                throw new IllegalArgumentException("zygoteSocketAddress can't be null");
            }

            try {
                // 连接socket服务器
                zygoteSessionSocket.connect(zygoteSocketAddress);
                zygoteInputStream = new DataInputStream(zygoteSessionSocket.getInputStream());
                // 写入和传递socket数据
                zygoteOutputWriter =
                        new BufferedWriter(
                                new OutputStreamWriter(zygoteSessionSocket.getOutputStream()),
                                Zygote.SOCKET_BUFFER_SIZE);
            } catch (IOException ex) {
                try {
                    zygoteSessionSocket.close();
                } catch (IOException ignore) {
                }

                throw ex;
            }

            // 初始化ZygoteState对象
            return new ZygoteState(zygoteSocketAddress, usapSocketAddress,
                    zygoteSessionSocket, zygoteInputStream, zygoteOutputWriter,
                    getAbiList(zygoteOutputWriter, zygoteInputStream));
        }
    }
  ```
`openZygoteSocketIfNeeded`方法通过abi匹配是和zygote通信还是和zygote64通信，通过`ZygoteState.connect`方法通过socket与Zygote建立通信连接。

### zygoteSendArgsAndGetResult给Zygote发送接收消息
  ```
public class ZygoteProcess {

    @GuardedBy("mLock")
    private Process.ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, int zygotePolicyFlags, @NonNull ArrayList<String> args)
            throws ZygoteStartFailedEx {

        // 数组重定义为String
        String msgStr = args.size() + "\n" + String.join("\n", args) + "\n";
        // ...
        // 将参数
        return attemptZygoteSendArgsAndGetResult(zygoteState, msgStr);
    }

    private Process.ProcessStartResult attemptZygoteSendArgsAndGetResult(
            ZygoteState zygoteState, String msgStr) throws ZygoteStartFailedEx {
        try {
            final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
            final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;
            // 写入对应的参数
            zygoteWriter.write(msgStr);
            zygoteWriter.flush();

            // 获取Zygote进程运行结束对应的返回结果
            Process.ProcessStartResult result = new Process.ProcessStartResult();
            //等待socket服务端（即zygote）返回新创建的进程pid;
            result.pid = zygoteInputStream.readInt();
            result.usingWrapper = zygoteInputStream.readBoolean();

            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }

            return result;
        } catch (IOException ex) {
            zygoteState.close();
            Log.e(LOG_TAG, "IO Exception while communicating with Zygote - "
                    + ex.toString());
            throw new ZygoteStartFailedEx(ex);
        }
    }


}

  ```
`zygoteSendArgsAndGetResult`方法通过调用`attemptZygoteSendArgsAndGetResult`方法去发送消息，attemptZygoteSendArgsAndGetResult通过`ZygoteState`拿到BufferedWriter和DataInputStream，通过`BufferedWriter`去写入数据发送，通过`DataInputStream`去等待Zygote创建进程的pid。

## 2. Zygote进程fork新进程
### runSelectLoop 死循环卵化进程
  ```
public class ZygoteInit {

    public static void main(String argv[]) {
        //...
        preload(bootTimingsTraceLog);
        Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
        //循环等待创建进程监听
        caller = zygoteServer.runSelectLoop(abiList);
        if (caller != null) {
            caller.run();
        }
    }
}

class ZygoteServer {

    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
        ArrayList<ZygoteConnection> peers = new ArrayList<>();
        //sServerSocket是socket通信中的服务端，即zygote进程。保存到fds[0]
        socketFDs.add(mZygoteSocket.getFileDescriptor());
        peers.add(null);

        // 死循环监听消息
        while (true) {
            //...
            StructPollfd[] pollFDs;

            if (mUsapPoolEnabled) {
                usapPipeFDs = Zygote.getUsapPipeFDs();
                pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
            } else {
                pollFDs = new StructPollfd[socketFDs.size()];
            }

            int pollIndex = 0;
            for (FileDescriptor socketFD : socketFDs) {
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = socketFD;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;
            }

            //....
            int pollReturnValue;
            try {
                //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
                pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }

            if (pollReturnValue == 0) {
                // The poll timeout has been exceeded.  This only occurs when we have finished the
                // USAP pool refill delay period.

                mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
                mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

            } else {
                boolean usapPoolFDRead = false;

                while (--pollIndex >= 0) {
                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                        continue;
                    }

                    // 当此时的pollIndex为0的时候，表明ZygoteServer启动后，有一个客户端来连接
                    if (pollIndex == 0) {
                        // 收到客户端连接请求，调用acceptCommandPeer函数初始化一个ZygoteConnection对象
                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                        peers.add(newPeer);
                        socketFDs.add(newPeer.getFileDescriptor());

                    } else if (pollIndex < usapPoolEventFDIndex) {
                        // 当前已经至少有一个ZygoteConnection连接建立完成

                        try {
                            // 获取对应的ZygoteConnection对象，并调用其processOneCommand函数
                            ZygoteConnection connection = peers.get(pollIndex);
                            final Runnable command = connection.processOneCommand(this);
                            //....
                        }
                    }
                }
            }
        }
    }

    // 建立ZygoteConnection链接
    private ZygoteConnection acceptCommandPeer(String abiList) {
        try {
            return createNewConnection(mZygoteSocket.accept(), abiList);
        } catch (IOException ex) {
            throw new RuntimeException(
                    "IOException during accept()", ex);
        }
    }

    protected ZygoteConnection createNewConnection(LocalSocket socket, String abiList)
            throws IOException {
        return new ZygoteConnection(socket, abiList);
    }

}
  ```
在`runSelectLoop`方法中死循环调用`Os.poll`处理轮询状态，如果有新进程需要创建则激活继续执行，没有常见新进程事件则一直阻塞。当收到客户端连接时，会调用`acceptCommandPeer`创建一个`ZygoteConnection`。最后调用`ZygoteConnection.processOneCommand`创建新进程。

### ZygoteConnection.processOneCommand 创建新进程
  ```
class ZygoteConnection {

    Runnable processOneCommand(ZygoteServer zygoteServer) {
        String[] args;

        try {
            // 读取zygote socket传递过来的参数数据
            args = Zygote.readArgumentList(mSocketReader);
        } catch (IOException ex) {
            throw new IllegalStateException("IOException on command socket", ex);
        }
        //...
        //将binder客户端传递过来的参数，解析成Arguments对象格式
        ZygoteArguments parsedArgs = new ZygoteArguments(args);

        //...
        // 调用Zygote的forkAndSpecialize函数，fork创建新进程
        pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
                parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
                parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
                parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mIsTopApp,
                parsedArgs.mPkgDataInfoList, parsedArgs.mWhitelistedDataInfoList,
                parsedArgs.mBindMountAppDataDirs, parsedArgs.mBindMountAppStorageDirs);

        try {
            if (pid == 0) {
                // 子进程
                // 调用并返回handleChildProc方法
                return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote);
            } else {
                // 父进程....
                return null;
            }
        }
        //...
    }
}

  ```
`processOneCommand`方法主要做了两件事：
1. 执行`Zygote.forkAndSpecialize`方法通过fork()方法去创建新进程。
2. 对于新创建的新进程，执行`handleChildProc`去处理子进程的操作

#### Zygote.forkAndSpecialize 创建子进程
  ```
public final class Zygote {

    static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags,
                                 int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
                                 int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir,
                                 boolean isTopApp, String[] pkgDataInfoList, String[] whitelistedDataInfoList,
                                 boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs) {

        // Zygote进程的4个Daemon子线程stop
        ZygoteHooks.preFork();
        // 调用JNI方法fork新进程，调用[com_android_internal_os_Zygote.cpp]com_android_internal_os_Zygote_nativeForkAndSpecialize方法
        int pid = nativeForkAndSpecialize(
                uid, gid, gids, runtimeFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
                fdsToIgnore, startChildZygote, instructionSet, appDataDir, isTopApp,
                pkgDataInfoList, whitelistedDataInfoList, bindMountAppDataDirs,
                bindMountAppStorageDirs);
        // Zygote进程的4个Daemon子线程start
        ZygoteHooks.postForkCommon();
        return pid;
    }

    // JNI方法，调用[com_android_internal_os_Zygote.cpp]com_android_internal_os_Zygote_nativeForkAndSpecialize方法
    private static native int nativeForkAndSpecialize(int uid, int gid, int[] gids,
                                                      int runtimeFlags, int[][] rlimits, int mountExternal, String seInfo, String niceName,
                                                      int[] fdsToClose, int[] fdsToIgnore, boolean startChildZygote, String instructionSet,
                                                      String appDataDir, boolean isTopApp, String[] pkgDataInfoList,
                                                      String[] whitelistedDataInfoList, boolean bindMountAppDataDirs,
                                                      boolean bindMountAppStorageDirs);

}
  ```
在创建子进程前，会把Zygote进程的4个Daemon子线程stop，在创建完子进程后，重新对Zygote进程的4个Daemon子线程start。
通过调用`nativeForkAndSpecialize`JNI方法去创建子进程，对应执行[com_android_internal_os_Zygote.cpp]com_android_internal_os_Zygote_nativeForkAndSpecialize方法
  ```
    // com_android_internal_os_Zygote.cpp
    static jint com_android_internal_os_Zygote_nativeForkAndSpecialize(
            JNIEnv*env, jclass, jint uid, jint gid, jintArray gids,
            jint runtime_flags, jobjectArray rlimits,
            jint mount_external, jstring se_info, jstring nice_name,
            jintArray managed_fds_to_close, jintArray managed_fds_to_ignore, jboolean is_child_zygote,
            jstring instruction_set, jstring app_data_dir, jboolean is_top_app,
            jobjectArray pkg_data_info_list, jobjectArray whitelisted_data_info_list,
            jboolean mount_data_dirs, jboolean mount_storage_dirs) {

        //... 创建新进程
        pid_t pid = ForkCommon(env, false, fds_to_close, fds_to_ignore, true);

        if (pid == 0) {
            SpecializeCommon(env, uid, gid, gids, runtime_flags, rlimits,
                    capabilities, capabilities,
                    mount_external, se_info, nice_name, false,
                    is_child_zygote == JNI_TRUE, instruction_set, app_data_dir,
                    is_top_app == JNI_TRUE, pkg_data_info_list,
                    whitelisted_data_info_list,
                    mount_data_dirs == JNI_TRUE,
                    mount_storage_dirs == JNI_TRUE);
        }
        return pid;
    }


    static pid_t ForkCommon(JNIEnv*env, bool is_system_server,
                  const std::vector<int>&fds_to_close,
                  const std::vector<int>&fds_to_ignore,
                            bool is_priority_fork) {
        //...
        //从Zygote进程中复制一个新进程
        pid_t pid = fork();
        //...
        return pid;
    }
  ```
`fork()`是Linux创建进程的标准方法，采用`copy on write`技术，这里要注意的两点是：
1. 创建的子进程跟父进程共用资源，比如新创建子进程跟Zygote进程共用资源，它们共用同一个物理空间，只有在写时要发生改变的时候才会指向两个物理空间

![image.png](https://upload-images.jianshu.io/upload_images/22650779-81d8f972e163b388.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 调用一次fork()会返回两次：
父进程：返回新创建子进程的pid
子进程：返回0
异常： 返回负数

自此已经完成新子进程的创建，下面开始调用``处理关于子进程的操作。

#### handleChildProc 子进程处理
  ```
class ZygoteConnection {

    private Runnable handleChildProc(ZygoteArguments parsedArgs,
                                     FileDescriptor pipeFd, boolean isZygote) {
        //关闭当前的socket连接
        closeSocket();
        //设置进程名称
        Zygote.setAppProcessName(parsedArgs, TAG);

        if (parsedArgs.mInvokeWith != null) {
            //...
        } else {
            if (!isZygote) {
                // 非Zygote进程，执行此方法
                return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                        parsedArgs.mDisabledCompatChanges,
                        parsedArgs.mRemainingArgs, null /* classLoader */);
            } else {
                // Zygote进程
                return ZygoteInit.childZygoteInit(parsedArgs.mTargetSdkVersion,
                        parsedArgs.mRemainingArgs, null /* classLoader */);
            }
        }
    }
}

public class ZygoteInit {

    public static final Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
                                            String[] argv, ClassLoader classLoader) {
        RuntimeInit.redirectLogStreams(); //重定向log输出

        RuntimeInit.commonInit(); // 通用的一些初始化
        ZygoteInit.nativeZygoteInit(); // zygote初始化
        // 最终执行ActivityThread.main()方法
        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
                classLoader);
    }
}
  ```
对于新创建的子进程会复制Zygote进程的Socket，所以会关闭此socket连接，接着调用`ZygoteInit.zygoteInit`方法返回一个Runnable。ZygoteInit.zygoteInit方法会做三件事：
1.  RuntimeInit.commonInit() 做一些通用的设置如设置时区
2. ZygoteInit.nativeZygoteInit() 创建启动Binder
3. RuntimeInit.applicationInit 调用ActivityThread.main()方法

##### RuntimeInit.commonInit() 通用设置
  ```
public class RuntimeInit {

    @UnsupportedAppUsage
    protected static final void commonInit() {
        // 设置默认的未捕捉异常处理方法
        Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));

        // 设置市区，中国时区为"Asia/Shanghai"
        RuntimeHooks.setTimeZoneIdSupplier(() -> SystemProperties.get("persist.sys.timezone"));

        //重置log配置
        LogManager.getLogManager().reset();
        new AndroidConfig();

        // 设置默认的HTTP User-agent格式,用于 HttpURLConnection。
        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        // 设置socket的tag，用于网络流量统计
        NetworkManagementSocketTagger.install();
    }
}
  ```
做些通用的设置，比如设置默认的未捕捉异常处理方法、设置时区，设置log,设置Hppt等。

##### ZygoteInit.nativeZygoteInit() 开启Binder设置
  ```
// JNI方法，对应调用此方法
class AppRuntime :public AndroidRuntime{
        virtual void onZygoteInit()
        {
        sp<ProcessState> proc=ProcessState::self(); // open打开驱动和mmap映射地址
        proc->startThreadPool();//启动新binder线程
        }
        }
  ```
ProcessState::self()会调用open()方法打开Binder设备，执行mmap()方法进行映射。
startThreadPool()启动新Binder线程，循环监听事件。

##### RuntimeInit.applicationInit  执行startClass.main方法
  ```
public class RuntimeInit {

    protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
                                              String[] argv, ClassLoader classLoader) {
        //true代表应用程序退出时不调用AppRuntime.onExit()，否则会在退出前调用
        nativeSetExitWithoutCleanup(true);

        // 设置虚拟机
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
        VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);

        //解析参数
        final Arguments args = new Arguments(argv);

        // 调用args.startClass类的static main()方法
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }

    protected static Runnable findStaticMain(String className, String[] argv,
                                             ClassLoader classLoader) {
        // ...
        Class<?> cl;
        cl = Class.forName(className, true, classLoader);
        // ...
        Method m;
        m = cl.getMethod("main", new Class[]{String[].class});
        // ...
        int modifiers = m.getModifiers();
        // ...
        // 返回此Runnable,此run()方法就是执行此main()方法
        return new MethodAndArgsCaller(m, argv);
    }
}

static class MethodAndArgsCaller implements Runnable {

    private final Method mMethod;

    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
            // 反射执行方法
            mMethod.invoke(null, new Object[]{mArgs});
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
  ```
`args.startClass`就是ActivityThread类，通过反射调用`ActivityThread.main()`方法，MethodAndArgsCaller是一个Runnable类，最终在ZygoteInit.main执行Runnable.run方法。

## 3. Application 的创建与绑定
![](https://upload-images.jianshu.io/upload_images/22650779-d59a1f5f1f9bbcc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

新进程创建后会执行`ActivityThread.main()`方法：
  ```
public final class ActivityThread extends ClientTransactionHandler {
    
    public static void main(String[] args) {
        //...

        // 创建主线程Looper
        Looper.prepareMainLooper();

        // 创建ActivityThread对象
        ActivityThread thread = new ActivityThread();

        // 建立Binder通道,向AMS(system_server进程)发送消息
        thread.attach(false, startSeq);

        // 主线程的Handler，即mH
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        //消息循环运行
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }


    final ApplicationThread mAppThread = new ApplicationThread();
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            // ....
            final IActivityManager mgr = ActivityManager.getService();
            try {
                //通过Binder通道跨进程调用AMS.attachApplication，mAppThread是ApplicationThread
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            //..
        } else {
            //...
        }
        //...
    }
  ```
创建完主线程的`Looper`，并通过调用`attach()`方法，调用Binder跨进程的方式调用`AMS. attachApplication`方法，并让其绑定ApplicationThread类。

  ```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    // 这里的thread是ApplicationThread
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        if (thread == null) {
            throw new SecurityException("Invalid application interface");
        }
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            // 调用attachApplicationLocked方法
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }

    private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
                                            int pid, int callingUid, long startSeq) {
        // 通过 pid 查询到进程信息，之前 fork 后有缓存
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                // 根据pid获取ProcessRecord
                app = mPidsSelfLocked.get(pid);
            }
            //...
        } else {
            app = null;
        }

        //... 做一些进程信息的检查管理，如果为空或不符合将杀死进程

        final String processName = app.processName;
        try {
            // 绑定死亡通知，AMS就会知道进程的死亡
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            //重新启动进程
            mProcessList.startProcessLocked(app,
                    new HostingRecord("link fail", processName),
                    ZYGOTE_POLICY_FLAG_EMPTY);
            return false;
        }

        //..
        //获取应用信息
        ApplicationInfo appInfo = instr != null ? instr.mTargetInfo : app.info;
        //...
        // 回调 bindApplication 方法
        thread.bindApplication(processName, appInfo, providerList, null, profilerInfo,
                null, null, null, testMode,
                mBinderTransactionTrackingEnabled, enableTrackAllocation,
                isRestrictedBackupMode || !normalMode, app.isPersistent(),
                new Configuration(app.getWindowProcessController().getConfiguration()),
                app.compat, getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked(),
                buildSerial, autofillOptions, contentCaptureOptions,
                app.mDisabledCompatChanges);

        // 绑定 thread
        app.makeActive(thread, mProcessStats);

        //...
        return true;
    }
}
  ```
通过pid去查询进程的信息，此pid在进程创建的时候缓存。对进程信息的一系列检查过后，如果检查进程信息不符合会杀死进程或重启进程。此后回调`ApplicationThread.bindApplication`方法，并将ApplicationThread绑定到ProcessRecord.thread中。
  ```
  public final class ActivityThread extends ClientTransactionHandler {


    Application mInitialApplication;

    private class ApplicationThread extends IApplicationThread.Stub {

        @Override
        public final void bindApplication(String processName, ApplicationInfo appInfo,
                                          ProviderInfoList providerList, ComponentName instrumentationName,
                                          ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                                          IInstrumentationWatcher instrumentationWatcher,
                                          IUiAutomationConnection instrumentationUiConnection, int debugMode,
                                          boolean enableBinderTracking, boolean trackAllocation,
                                          boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                                          CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                                          String buildSerial, AutofillOptions autofillOptions,
                                          ContentCaptureOptions contentCaptureOptions, long[] disabledCompatChanges) {
            if (services != null) {
                //...
                //将services缓存起来, 减少binder检索服务的次数
                ServiceManager.initServiceCache(services);
            }

            //发送消息H.SET_CORE_SETTINGS
            setCoreSettings(coreSettings);

            //初始化AppBindData
            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providerList.getList();
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            data.autofillOptions = autofillOptions;
            data.contentCaptureOptions = contentCaptureOptions;
            data.disabledCompatChanges = disabledCompatChanges;
            //发送消息H.BIND_APPLICATION,对应执行handleBindApplication()方法
            sendMessage(H.BIND_APPLICATION, data);
        }
    }

    @UnsupportedAppUsage
    private void handleBindApplication(AppBindData data) {
        //...
        //获取LoadedApk对象
        data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
        //...
        Application app;
        try {
            // 此处data.info是指LoadedApk, 创建应用Application对象
            app = data.info.makeApplication(data.restrictedBackupMode, null);

            mInitialApplication = app;
        }

    }

    public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,
                                                 CompatibilityInfo compatInfo) {
        return getPackageInfo(ai, compatInfo, null, false, true, false);
    }

    private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
                                     ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
                                     boolean registerPackage) {
        final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
        synchronized (mResourcesManager) {
            WeakReference<LoadedApk> ref;
            if (differentUser) {
                //不支持跨用户
                ref = null;
            } else if (includeCode) {
                ref = mPackages.get(aInfo.packageName);
            } else {
                ref = mResourcePackages.get(aInfo.packageName);
            }

            LoadedApk packageInfo = ref != null ? ref.get() : null;

            if (packageInfo != null) {
                if (!isLoadedApkResourceDirsUpToDate(packageInfo, aInfo)) {
                    List<String> oldPaths = new ArrayList<>();
                    LoadedApk.makePaths(this, aInfo, oldPaths);
                    packageInfo.updateApplicationInfo(aInfo, oldPaths);
                }
                // 返回LoadedApk
                return packageInfo;
            }

            //创建LoadedApk对象
            packageInfo =
                    new LoadedApk(this, aInfo, compatInfo, baseLoader,
                            securityViolation, includeCode
                            && (aInfo.flags & ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

            // 返回 LoadedApk
            return packageInfo;
        }
    }

}
  ```
`getPackageInfoNoCheck`方法会创建一个`LoadedApk`类，这个类存储了App应用信息。此后调用`LoadedApk. makeApplication`创建Application。
  ```
public final class LoadedApk {

    public Application makeApplication(boolean forceDefaultAppClass,
                                       Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;
        // 查找到 application 的 class
        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            // 默认是 android.app.Application
            appClass = "android.app.Application";
        }

        try {
            final java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                initializeJavaContextClassLoader();
            }
            //...

            // 创建一个ContextImpl
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);

            // 创建和绑定 Application ，并绑定ContextImpl
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {

        }
        mActivityThread.mAllApplications.add(app);
        //设置mApplication对象值
        mApplication = app;

        if (instrumentation != null) {
            try {
                // 调用 Application 的 onCreate 方法
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                //...
            }
        }

        return app;
    }

}

public class Instrumentation {

    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        Application app = (Application) clazz.newInstance();
        app.attach(context);
        return app;
    }
}

class ContextImpl extends Context {

    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        return createAppContext(mainThread, packageInfo, null);
    }

    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
                                        String opPackageName) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, null,
                0, null, opPackageName);
        context.setResources(packageInfo.getResources());
        context.mIsSystemOrSystemUiContext = isSystemOrSystemUI(context);
        return context;
    }
}

  ```
`makeApplication`方法先查看应用是否定义了`Application`，如果没有使用默认的`android.app.Application`。之后创建一个`ContextImpl`，它包含了LoadedApk等应用信息。创建万之后调用`Instrumentation.newApplication`方法把ContextImpl设置给新创建的Application。最后调用`Application.onCreate`。

## 总结
1. 四大组件启动前，会检查是否创建新进程
2. 调用AMS. startProcessLocked 通过socket发送进程创建消息
3. Zygote进程收到AMS消息后，通过fork创建新进程
4. 对用子进程，会调用其`ActivityThread.main`方法
5. `ActivityThread.attch`向AMS发起ApplicationThread绑定，之后回调ApplicationThread。
6. ApplicationThread创建Application类，并为其绑定ContextImpl和执行其onCreate方法。

