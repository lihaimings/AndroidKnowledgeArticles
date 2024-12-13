> 本篇文章基于Android6.0源码分析

相关源码文件：
  ```
/system/core/rootdir/init.rc
/system/core/rootdir/init.zygote64.rc

/frameworks/base/cmds/app_process/App_main.cpp
/frameworks/base/core/jni/AndroidRuntime.cpp

/frameworks/base/core/java/com/android/internal/os/
  - ZygoteInit.java
  - Zygote.java
  - ZygoteConnection.java
  
/frameworks/base/core/java/android/net/LocalServerSocket.java
/system/core/libutils/Threads.cpp
  ```

# Zygote进程启动前的概述

通过`init.rc`的文件解析会启动zygote相关的服务从而启动zygote进程。通过`import`导入决定启动哪种类型的zygote服务脚本，这里分为32位和64位架构的zygote服务脚本
  ```
 import /init.${ro.zygote}.rc
  ```
在/system/core/rootdir目录中有四个zygote相关的服务脚本
  ```
init.zygote32.rc // 支持32位的zygote
init.zygote32_64.rc // 即支持32位也支持64位的zygote，其中以32位为主，64位为辅
init.zygote64.rc // 支持64位的zygote
init.zygote64_32.rc // 即支持64位也支持32位的zygote，其中以64位为主，32位为辅

  ```
下面我们分析只分析64位的zygote服务脚本的Android初始化语言：
  ```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
   class main
   socket zygote stream 660 root system
   onrestart write /sys/android_power/request_state wake
   onrestart write /sys/power/state on
   onrestart restart media
   onrestart restart netd
   writepid /dev/cpuset/foreground/tasks

  ```
zygote进程的执行程序为`/system/bin/app_process64`中，其中参数为：`-Xzygote /system/bin --zygote --start-system-server`，classname为`main `。除了在Init进程解析时创建Zygote进程，在`servicemanager、surfaceflinger、systemserver`进程被杀时Zygote进程也会进行重启。

其中/system/bin/app_process64的映射的执行文件为：/frameworks/base/cmds/app_process/app_main.cpp

# Zygote进程启动

![图1 zygote执行流程](https://upload-images.jianshu.io/upload_images/22650779-daafcb93db8a4450.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如图1所示，zygote进程启动时会先启动`app_main`类的`main()`方法：
  ```
    // 参数argv为 ：  -Xzygote /system/bin --zygote --start-system-server
    int main(int argc, char* const argv[])
    {
        // 创建一个AppRuntime实例，AppRuntime 继承 AndoirdRuntime
        AppRuntime runtime (argv[0], computeArgBlockSize(argc, argv));
        //忽略第一个参数
        argc--;
        argv++;

    
        // 解析参数并对变量赋值
        bool zygote = false;
        bool startSystemServer = false;
        bool application = false;
        String8 niceName;
        String8 className;

        ++i;  // Skip unused "parent dir" argument.
        while (i < argc) {
            const char * arg = argv [i++];
            if (strcmp(arg, "--zygote") == 0) {
                 // 参数中有--zygote
                zygote = true;
                niceName = ZYGOTE_NICE_NAME;
            } else if (strcmp(arg, "--start-system-server") == 0) {
                // 参数中有--start-system-server
                startSystemServer = true;
            } else if (strcmp(arg, "--application") == 0) {
                application = true;
            } else if (strncmp(arg, "--nice-name=", 12) == 0) {
                niceName.setTo(arg + 12);
            } else if (strncmp(arg, "--", 2) != 0) {
                className.setTo(arg);
                break;
            } else {
                --i;
                break;
            }
        }

  
        if (zygote) {
           //如果zygote为true,则调用AndroidRuntime的start方法，并传入了"com.android.internal.os.ZygoteInit"参数
           runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
        } else if (className) {
            runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
        } else {
            fprintf(stderr, "Error: no class name or --zygote supplied.\n");
            app_usage();
            LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
            return 10;
        }
    }
  ```
在app_main的mian()方法中，主要是根据zygote的脚本的参数进行解析，在解析到有`--zygote`字符后，则确定执行`AndroidRuntime.start`方法，并且第一个参数传为`com.android.internal.os.ZygoteInit`。

## AndroidRuntime.start()
在此方法中，主要做了三件事：
· 创建虚拟机实例
· JNI方法的注册
· 调用参数的main()方法
  ```
    // 这里的className为：com.android.internal.os.ZygoteInit
    void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
    {
        
        /* start the virtual machine */
        JniInvocation jni_invocation;
        jni_invocation.Init(NULL);
        JNIEnv * env;
         // 1. 创建虚拟机
        if (startVm(& mJavaVM, &env, zygote) != 0) {
        return;
         }
        onVmCreated(env);
        // 2. JNI方法注册
        if (startReg(env) < 0) {
            ALOGE("Unable to register all android natives\n");
            return;
        }

        // 解析classname参数
       //将"com.android.internal.os.ZygoteInit"转换为"com/android/internal/os/ZygoteInit"
        char * slashClassName = toSlashClassName(className);
        jclass startClass = env->FindClass(slashClassName);

        if (startClass == NULL) {
        } else {
            //  得到ZygoteInit的main方法
            jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
            if (startMeth == NULL) {
            } else { env ->
                  // 3. 执行ZygoteInit的main方法
                CallStaticVoidMethod(startClass, startMeth, strArray);
            }
        }
        free(slashClassName);

    }
  ```
对start方法进行了一些删减后，主要是通过`startVm `创建虚拟机，通过`startReg(env)`进行JNI方法注册，最后解析className参数，去执行`ZygoteInit.main方法`。下面将逐一分析这三种状态。

### 1. 创建虚拟机实例
**startVm**：
  ```
 int AndroidRuntime::startVm(JavaVM** pJavaVM, JNIEnv** pEnv, bool zygote)
    {
        // ... 
        // JNI检测功能
        bool checkJni = false;
        property_get("dalvik.vm.checkjni", propBuf, "");
        if (strcmp(propBuf, "true") == 0) {
            checkJni = true;
        } else if (strcmp(propBuf, "false") != 0) {
            /* property is neither true nor false; fall back on kernel parameter */
            property_get("ro.kernel.android.checkjni", propBuf, "");
            if (propBuf[0] == '1') {
                checkJni = true;
            }
        }
        ALOGD("CheckJNI is %s\n", checkJni ? "ON" : "OFF");
        if (checkJni) {
            addOption("-Xcheck:jni");
        }

         // /虚拟机产生的trace文件，主要用于分析系统问题，路径默认为/data/anr/traces.txt
        parseRuntimeOption("dalvik.vm.stack-trace-file", stackTraceFileBuf, "-Xstacktracefile:");

        //对于不同的软硬件环境，这些参数往往需要调整、优化，从而使系统达到最佳性能
        parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
        parseRuntimeOption("dalvik.vm.heapsize", heapsizeOptsBuf, "-Xmx", "16m");

        parseRuntimeOption(
            "dalvik.vm.heapgrowthlimit",
            heapgrowthlimitOptsBuf,
            "-XX:HeapGrowthLimit="
        );
        parseRuntimeOption("dalvik.vm.heapminfree", heapminfreeOptsBuf, "-XX:HeapMinFree=");
        parseRuntimeOption("dalvik.vm.heapmaxfree", heapmaxfreeOptsBuf, "-XX:HeapMaxFree=");
        parseRuntimeOption(
            "dalvik.vm.heaptargetutilization",
            heaptargetutilizationOptsBuf,
            "-XX:HeapTargetUtilization="
        );
        // ...
        // 初始化虚拟机
        if (JNI_CreateJavaVM(pJavaVM, pEnv, & initArgs) < 0) {
        ALOGE("JNI_CreateJavaVM failed\n");
        return -1;
    }

        return 0;
    }

  ```
startVm方法里面有很多代码，但主要分为三步，第一步是检测，第二步是软硬件参数的设置，第三步是初始化虚拟机。

### 2. JNI方法的注册
**startReg**
  ```
    int AndroidRuntime::startReg(JNIEnv* env)
    {
        androidSetCreateThreadFunc((android_create_thread_fn) javaCreateThreadEtc);

        ALOGV("--- registering native functions ---\n");
        env->PushLocalFrame(200);
        //进程JNI方法的注册
        if (register_jni_procs(gRegJNI, NELEM(gRegJNI), env) < 0) { env ->
            PopLocalFrame(NULL);
            return -1;
        }
        env->PopLocalFrame(NULL);
        return 0;
    }

    // 这里的array[]是gRegJNI,它是一个映射了很多方法的数组
    static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env)
    {
        // 执行很多映射的方法
        for (size_t i = 0; i < count; i++) {
        if (array[i].mProc(env) < 0) {
            return -1;
        }
    }
        return 0;
    }
  ```
startReg方法是对JNI方法的注册，它通过一个有很多宏定义的数组，并执行数组定义的方法，进行对JNI和Java层方法一一映射。

### 3. 调用ZygoteInit.main方法
  在AndroidRuntime.start()方法的最后，通过反射执行了其`ZygoteInit.main()`方法。
  ```
      if (startClass == NULL) {
      } else {
          //  得到ZygoteInit的main方法
          jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
          "([Ljava/lang/String;)V");
          if (startMeth == NULL) {
          } else { env ->
                // 3. 执行ZygoteInit的main方法
              CallStaticVoidMethod(startClass, startMeth, strArray);
          }
      }
  ```
通过反射去执行`ZygoteInit.main`方法，也是第一次进入java语言的世界。所以AndroidRuntime的start方法做了三件事，一是初始化虚拟机，二是JNI方法的注册，三是通过反射执行ZygoteInit.main方法。

## ZygoteInit.main
Zygote进程用于创建管理`framework`层的`SystemServer`进程，还用于创建App进程，就是应用App启动创建进程时，是由Zygote进程创建的，并且Zygote创建子进程将使用copy on write的技术，就是子进程直接继承父进程的现有的资源，在子进程对于共有的资源是`读时共享，写时复制`。
ZygoteInit.main方法中主要做了四件事：
· 注册服务端的socket，用于接收创建子进程的信息
· 提前预加载类和资源，用于子进程共享
· 创建SystemServer进程，其管理着framework层
· 循环监听服务socket,创建子进程

  ```
    public static void main(String argv[])
    {
        try {
            // 创建服务端Soctet，用于接收创建子进程信息
            registerZygoteSocket(socketName);
           // 提前预加载类和资源
            preload();
            // gc操作
            gcAndFinalize();
            // 创建SystemServer进程
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }
           // 用服务socket监听创建进程信息，并创建子进程
            runSelectLoop(abiList);

            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }

  ```
通过`registerZygoteSocket`方法去创建服务端的socket，`preload()`方法去提前加载类和资源，`startSystemServer`方法去创建SystemServer进程去管理framework层，`runSelectLoop`方法循环监听创建子进程。

### 1. 注册服务端Socket
**registerZygoteSocket**
  ```
    private static void registerZygoteSocket(String socketName)
    {
        if (sServerSocket == null) {
            int fileDesc;
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System . getenv (fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException (fullSocketName + " unset or invalid", ex);
            }

            try {
                 // 创建服务端的socket
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                sServerSocket = new LocalServerSocket (fd);
            } catch (IOException ex) {
                throw new RuntimeException (
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
  ```
创建一个服务端的socket用于接口多个客户端的信息接收，在后面的`runSelectLoop`方法用于监听服务端的socket信息，以便创建子进程。

### 2. 预加载资源
**preload**
  ```
    static void preload()
    {
        preloadClasses();  //预加载位于/system/etc/preloaded-classes文件中的类
        preloadResources();  //预加载资源，包含drawable和color资源
        preloadOpenGL(); //预加载OpenGL 
        preloadSharedLibraries(); //预加载"android","compiler_rt","jnigraphics"这3个共享库
        preloadTextResources(); //预加载 文本连接符资源
        WebViewFactory.prepareWebViewInZygote(); //仅用于zygote进程，用于内存共享的进程
    }
  ```
`preloadClasses()`方法通过Class.forName()反射的方法去加载类，`preloadResources`方法主要是加载位于com.android.internal.R.array.preloaded_drawables和com.android.internal.R.array.preloaded_color_state_lists的资源。
提前加载资源的好处是，在复制创建子进程时，提前加载好的资源可以给子进程直接使用，不用第二次创建，但不好的地方是每个创建的子进程都有拥有很多资源，而不管是否需要。

### 3. 启动SystemServer进程
**startSystemServer**
  ```
    private static boolean startSystemServer(String abiList, String socketName)
    throws MethodAndArgsCaller, RuntimeException
    {
        // 通过数组保存创建systemserver进程的信息
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
            parsedArgs = new ZygoteConnection . Arguments (args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            // 创建systemserver进程
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

        /* pid==0 则是子进程，就是systemserver */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            // 完成system_server进程剩余的工作
            handleSystemServerProcess(parsedArgs);
        }

        return true;
    }

  ```
通过`Zygote.forkSystemServer`去创建SystemServer进程，其进程是管理着framework的，我们将在下一篇分析SystemServer进程进程的启动。

### 4. 循环等待孵化进程
**runSelectLoop**

  ```
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller
    {
        // FileDescriptor数组
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
         // ZygoteConnection数组
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
        //sServerSocket是socket通信中的服务端，即zygote进程。保存到fds[0]
        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            // StructPollfd数组，并将相应位置fds的值赋值
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd ();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                 //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException ("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {

                //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；
               // 否则进入continue，跳出本次循环。
                if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
               }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer (abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                     //i>0，则代表通过socket接收来自对端的数据，并执行相应操作
                    boolean done = peers.get (i).runOnce();
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
  ```
在runSelectLoop方法中有一个轮询的状态，如果有事件接收则会去执行`runOnce()`的方法操作：
  ```
    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller
    {

        String args [];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            //读取socket客户端发送过来的参数列表
            args = readArgumentList();
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            Log.w(TAG, "IOException on command socket " + ex.getMessage());
            closeSocket();
            return true;
        }

        if (args == null) {
            // EOF reached.
            closeSocket();
            return true;
        }

     
        try {
             //将binder客户端传递过来的参数，解析成Arguments对象格式
            parsedArgs = new Arguments (args);
            ...
           // fork创建一个新的进程
            pid = Zygote.forkAndSpecialize(
                parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                parsedArgs.appDataDir
            );
        } catch (ErrnoException ex) {
            logAndPrintError(newStderr, "Exception creating pipe", ex);
        } catch (IllegalArgumentException ex) {
            logAndPrintError(newStderr, "Invalid zygote arguments", ex);
        } catch (ZygoteSecurityException ex) {
            logAndPrintError(
                newStderr,
                "Zygote security policy prevents request: ", ex
            );
        }

        try {
            if (pid == 0) {
               // 处理子进程
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

                // should never get here, the child is expected to either
                // throw ZygoteInit.MethodAndArgsCaller or exec().
                return true;
            } else {
                // 父进程
                IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }

  ```
所以在`runSelectLoop`方法中，通过客户端的socket不断的和服务端的socket通信的监听，通过调用起runOnce方法去不断的创建新的进程。

# 总结
Zygote进程的启动过程主要有:
1. 创建虚拟机和JNI方法的注册
2. 注册服务Socket和提前加载系统类和资源
3. 创建SystemServer进程
4. 循环等待孵化进程


