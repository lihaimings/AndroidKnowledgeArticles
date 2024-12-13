Binder系列：
[Binder Kernel层—Binder内核驱动](https://www.jianshu.com/p/fdf12cd7c28d)
[Binder Native层—服务管理(ServiceManager进程)](https://www.jianshu.com/p/fb514e3eed3a)
[Binder Native层—注册/查询服务](https://www.jianshu.com/p/a7e5bbb0ab72)
[Binder Framework层—注册和查询服务](https://www.jianshu.com/p/60be38017422)
[Binder 应用层-AIDL原理](https://www.jianshu.com/p/7ca78da80f95)

> 本篇文章基于Android6.0源码分析

相关源码文件：
  ```
framework/base/core/java/android/os/
   - IBinder.java
   - Binder.java(内含BinderProxy类)
   - Parcel.java
   - IInterface.java
   - IServiceManager.java
   - ServiceManager.java
   - ServiceManagerNative.java(内含ServiceManagerProxy类)

framework/base/core/java/com/android/internal/os/
   - BinderInternal.java

framework/base/core/jni/
   - AndroidRuntime.cpp
   - android_os_Parcel.cpp
   - android_util_Binder.cpp

frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
  ```
## 概述
framework层的查询和注册服务流程图：

![framework层](https://upload-images.jianshu.io/upload_images/22650779-b24f9badeff15a3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在android中Binder机制分为三层，分别对应中Android的分层：
1. Java Binder  (对应Framework层)
2. Native Binder (对应Native层)
3. Kernel Binder (对应Kernel层)

在此前的几遍文章中介绍了native层和Kernel层的Binder机制，其架构可以总结为：

![](https://upload-images.jianshu.io/upload_images/22650779-220036c16f7b057d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中在native层的分析过程中，BpBinder代表了client端与server的交互代理类，BBinder则代表server。那么上图可以改成

![](https://upload-images.jianshu.io/upload_images/22650779-31ced4db17cd6ece.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Native Binder的类使用的是c/c++，Android的应用开发语言是Java,所以需要Java Binder(framework层)对上层进行支持。
Java Binder也是基于C/S架构，Java Binder可以说是Native Binder的映射，通过JNI注册映射Native层的方法：

![](https://upload-images.jianshu.io/upload_images/22650779-d781f3b03d11d85b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/22650779-68916364fb31f6f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. **ServiceManager.java:**通过getIServiceManager()方法获取到ServiceManagerProxy对象，ServiceManager.java的getService、addService的具体操作交给ServiceManagerProxy对象的getService、addService。
2. **ServiceManagerNative:**其asInterface()方法返回ServiceManagerProxy对象。ServiceManager.java通过调用其asInterface()方法，传入一个带BpBinder的BpBinderProxy的参数获得一个ServiceManagerProxy。ServiceManagerNative的mRemote变量中。
3. **ServiceManagerProxy:**是Java层具体执行addService、getService的实现类，作用相当于Native层的BpServiceManager。
4. **BinderProxy:**其mObject有BpBinder
5. **BinderInternal:**内部有一个GcWatcher类，用于处理和调试与Binder相关的垃圾回收。其getContextObject方法通过JNI注册返回一个带BpBinder的BinderProxy
6. **Binder:**其成员变量mObject和方法execTransact()用于native方法

## JNI注册
Java Binder通过JNI与Native Binder通信，关于JNI的注册是在Zygote进程启动的时候进行注册的。在此前文章[Android系统启动-Zygote进程
](https://www.jianshu.com/p/b5fdb50a8dd3)有介绍通过`startReg`方法对`gRegJNI`宏定义的数组进行JNI的注册。
关于Java Binder的JNI注册是在`gRegJNI`对应`register_android_os_Binder`，其对应的执行文件是**/frameworks/base/core/jni/android_util_Binder.cpp**
  ```
int register_android_os_Binder(JNIEnv* env)
{
    // 1. 注册Binder类
    if (int_register_android_os_Binder(env) < 0)
        return -1;
    // 2. 注册BinderInternal类
    if (int_register_android_os_BinderInternal(env) < 0)
        return -1;
    // 3. 注册BinderProxy类
    if (int_register_android_os_BinderProxy(env) < 0)
        return -1;
}

  ```
register_android_os_Binder函数做了三件事，分别是：
1.注册Binder类
2.注册BinderInternal类
3.注册BinderProxy类

上面的Java Binder的架构图标志了它们间的关系，其中Binder和BinderProxy继承IBinder,BinderProxy代表客户端，Binder代表服务端。
- IBinder接口定义了很多整型变量，其中有一个`FLAG_ONEWAY `整型变量设置了则客户端发送了消息就返回，并不会阻塞等待服务端返回信息。
- Binder、BinderProxy都继承IBinder，其中BinderProxy代表客户端，Binder代表服务端
- Parcel 它可以在进程间进行数据传递，Parcel既可以传递基本数据类型也可以传递Binder对象。

### Binder类的注册
调用int_register_android_os_Binder方法进行Binder类的注册
  ```
// Binder类函数对应的执行方法
static const JNINativeMethod gBinderMethods[] = {
     /* name, signature, funcPtr */
    { "getCallingPid", "()I", (void*)android_os_Binder_getCallingPid },
    { "getCallingUid", "()I", (void*)android_os_Binder_getCallingUid },
    { "clearCallingIdentity", "()J", (void*)android_os_Binder_clearCallingIdentity },
    { "restoreCallingIdentity", "(J)V", (void*)android_os_Binder_restoreCallingIdentity },
    { "setThreadStrictModePolicy", "(I)V", (void*)android_os_Binder_setThreadStrictModePolicy },
    { "getThreadStrictModePolicy", "()I", (void*)android_os_Binder_getThreadStrictModePolicy },
    { "flushPendingCommands", "()V", (void*)android_os_Binder_flushPendingCommands },
    { "init", "()V", (void*)android_os_Binder_init },
    { "destroy", "()V", (void*)android_os_Binder_destroy },
    { "blockUntilThreadAvailable", "()V", (void*)android_os_Binder_blockUntilThreadAvailable }
};

const char* const kBinderPathName = "android/os/Binder";

static int int_register_android_os_Binder(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kBinderPathName);

    // 利用gBinderOffsets存放Binder.java相当信息
    gBinderOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z"); // 存放Binder的execTransact()方法
    gBinderOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");
    // 对Binder类的方法JNI注册
    return RegisterMethodsOrDie(
        env, kBinderPathName,
        gBinderMethods, NELEM(gBinderMethods));
}

static struct bindernative_offsets_t
{
    // Class state.
    jclass mClass;
    jmethodID mExecTransact;

    // Object state.
    jfieldID mObject;

} gBinderOffsets;

  ```

### BinderInternal类的注册
通过调用int_register_android_os_BinderInternal方法进行BinderInternal类的注册
  ```
// BinderInternal类函数对应的执行方法
static const JNINativeMethod gBinderInternalMethods[] = {
     /* name, signature, funcPtr */
    { "getContextObject", "()Landroid/os/IBinder;", (void*)android_os_BinderInternal_getContextObject },
    { "joinThreadPool", "()V", (void*)android_os_BinderInternal_joinThreadPool },
    { "disableBackgroundScheduling", "(Z)V", (void*)android_os_BinderInternal_disableBackgroundScheduling },
    { "handleGc", "()V", (void*)android_os_BinderInternal_handleGc }
};

const char* const kBinderInternalPathName = "com/android/internal/os/BinderInternal";

static int int_register_android_os_BinderInternal(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kBinderInternalPathName);

    // 通过gBinderInternalOffsets保存BinderInternal类的相关信息
    gBinderInternalOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderInternalOffsets.mForceGc = GetStaticMethodIDOrDie(env, clazz, "forceBinderGc", "()V");

    // 对BinderInternal类的方法JNI注册
    return RegisterMethodsOrDie(
        env, kBinderInternalPathName,
        gBinderInternalMethods, NELEM(gBinderInternalMethods));
}


static struct binderinternal_offsets_t
{
    // Class state.
    jclass mClass;
    jmethodID mForceGc;

} gBinderInternalOffsets;
  ```

### BinderProxy类的注册
通过调用int_register_android_os_BinderProxy方法进行BinderInternal类的注册

  ```
// BinderProxy类函数对应的执行方法
static const JNINativeMethod gBinderProxyMethods[] = {
     /* name, signature, funcPtr */
    {"pingBinder",          "()Z", (void*)android_os_BinderProxy_pingBinder},
    {"isBinderAlive",       "()Z", (void*)android_os_BinderProxy_isBinderAlive},
    {"getInterfaceDescriptor", "()Ljava/lang/String;", (void*)android_os_BinderProxy_getInterfaceDescriptor},
    {"transactNative",      "(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z", (void*)android_os_BinderProxy_transact},
    {"linkToDeath",         "(Landroid/os/IBinder$DeathRecipient;I)V", (void*)android_os_BinderProxy_linkToDeath},
    {"unlinkToDeath",       "(Landroid/os/IBinder$DeathRecipient;I)Z", (void*)android_os_BinderProxy_unlinkToDeath},
    {"destroy",             "()V", (void*)android_os_BinderProxy_destroy},
};

const char* const kBinderProxyPathName = "android/os/BinderProxy";

static int int_register_android_os_BinderProxy(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, "java/lang/Error");
    gErrorOffsets.mClass = MakeGlobalRefOrDie(env, clazz);

     // 通过gBinderProxyOffsets保存BinderProx类的相关信息
    clazz = FindClassOrDie(env, kBinderProxyPathName);
    gBinderProxyOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    gBinderProxyOffsets.mConstructor = GetMethodIDOrDie(env, clazz, "<init>", "()V");
    gBinderProxyOffsets.mSendDeathNotice = GetStaticMethodIDOrDie(env, clazz, "sendDeathNotice",
            "(Landroid/os/IBinder$DeathRecipient;)V");

    gBinderProxyOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");
    gBinderProxyOffsets.mSelf = GetFieldIDOrDie(env, clazz, "mSelf",
                                                "Ljava/lang/ref/WeakReference;");
    gBinderProxyOffsets.mOrgue = GetFieldIDOrDie(env, clazz, "mOrgue", "J");

    clazz = FindClassOrDie(env, "java/lang/Class");
    gClassOffsets.mGetName = GetMethodIDOrDie(env, clazz, "getName", "()Ljava/lang/String;");

    // 对BinderProx类的方法JNI注册
    return RegisterMethodsOrDie(
        env, kBinderProxyPathName,
        gBinderProxyMethods, NELEM(gBinderProxyMethods));
}

  ```


## Java Binder 注册服务
在Native Binder层又一个ServiceManager.cpp管理着服务，在Java Bidner层中也有一个ServiceManager.java可以通过它 getService、addService获取和注册服务。
下面以ActivityManagerService的注册为例，查看Java Binder的ServiceManager的注册服务过程：
  ```
// ActivityManagerNative继承Binder，所以ActivityManagerService就是Binder
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    public void setSystemProcess() {
        try {
            //1. 注册ServiceManager
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
             ...
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }
    }
}
  ```
ActivityManagerService本身是一个Binder,在注释1处`Context.ACTIVITY_SERVICE`为"activity"，并传入了this的AMS对象作为参数，调用ServiceManager.addServie()进行注册。

### ServiceManager.addService
Java Binder的Service要注册，就是通过 ServiceManager.addService来注册服务。
  ```
    public static void addService(String name, IBinder service, boolean allowIsolated) {
        try {
            // 1. getIServiceManager()返回的是BinderProxy
            getIServiceManager().addService(name, service, allowIsolated);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }

private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

         // 2. 获取返回sServiceManager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }

  ```
根据1和2的注释可以把代码分为三段：
- BinderInternal.getContextObject()
- ServiceManagerNative.asInterface()
- getIServiceManager().addService()

### 1.1 BinderInternal.getContextObject()
BinderInternal.getContextObject()方法返回一个Binderproxy对象，其对象的mObject参数是BpBinder。这是一个Native方法，找到它的对应函数：
**frameworks/base/core/jni/android_util_Binder.cpp**
  ```
static const JNINativeMethod gBinderInternalMethods[] = {
     /* name, signature, funcPtr */
    { "getContextObject", "()Landroid/os/IBinder;", (void*)android_os_BinderInternal_getContextObject },
    ...
};

// 对应函数
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    // 1. 创建一个BpBinder(0)
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    // 2. 创建一个BinderProxy对象，并把BpBinder设置到其mObject变量
    return javaObjectForIBinder(env, b);
}

// 
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
{
    // 创建java的BinderProxy对象
    object = env->NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
    if (object != NULL) {
         //  把 BpBinder(0) 的地址设置给 BinderProxy 的 mObject 属性
        env->SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
    }
    // 返回BinderProxy对象
    return object;
}

  ```
BinderInternal.getContextObject()函数通过JNI注册返回一个BinderProxy对象，并把 BpBinder(0) 的地址设置给 BinderProxy 的 mObject 属性。

### 1.2 ServiceManagerNative.asInterface()
ServiceManagerNative.asInterface()函数的作用是用BinderProxy作为参数创建一个ServiceManagerProxy对象。
  ```
    static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ServiceManagerProxy(obj);
    }


class ServiceManagerProxy implements IServiceManager {
   // 通过改造函数将BinderProxy赋值到mRemote中
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    private IBinder mRemote;
}

  ```
ServiceManagerNative.asInterface()生成ServiceManagerProxy对象，Java层ServiceManagerProxy的对象对标Native层的BpServiceManager对象，都是对getService、addService方法进行实现，通过Parcel进行数据封装，后交给BinderProxy或BpBinder进行处理。

所以：
  ```
 sServiceManager = ServiceManagerNative
             .asInterface(BinderInternal.getContextObject());
  ```
等价于：
  ```
sServiceManager = new ServiceManagerProxy（BinderProxy);
  ```

### 1.3 getIServiceManager().addService()
根据1.1、1.2讲解知道getIServiceManager()返回ServiceManagerProxy对象，所以 getIServiceManager().addService()等于`ServiceManagerProxy.addService()`：
  ```
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    public void addService(String name, IBinder service, boolean allowIsolated)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        data.writeStrongBinder(service); // 1. 将service写入
        data.writeInt(allowIsolated ? 1 : 0);
        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0); // 2.调用BinderProxy.transact()
        reply.recycle();
        data.recycle();
    }
}
  ```
`addService`函数就是将请求数据赋值在Parcel结构体中，其中注释1中`data.writeStrongBinder(service)`就是将service对象写入data的Parcel结构体中。
注释2中，`mRemote`就是构造函数传进来的`BinderProxy`，所以执行的是BinderProxy.transact()方法。

#### BinderProxy.transact()
BinderProxy.transact()通过JNI注册方法，将java的Parcel转换成C++的Parcel,之后调用BinderProxy.mObject的BpBinder，执行transact()方法。
  ```
final class BinderProxy implements IBinder {
 
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }

    public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;
     ...
}
  ```
native  transactNative方法对应的执行函数为android_os_BinderProxy_transact：
**/frameworks/base/core/jni/android_util_Binder.cpp**
  ```
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    // 1. 转成native层的Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);
    // 2. 获取到BpBinder
    IBinder* target = (IBinder*)
        env->GetLongField(obj, gBinderProxyOffsets.mObject);
    // 3. 执行BpBinder.transact()
    status_t err = target->transact(code, *data, reply, flags);
 
    return JNI_FALSE;
}
  ```
BinderProxy.transact()方法通过mObject拿到BpBinder,然后调用BpBinder的transact()方法。

#### Parcel.writeStrongBinder (Java)
  ```
public final class Parcel {
    private static native void nativeWriteStrongBinder(long nativePtr, IBinder val);

     public final void writeStrongBinder(IBinder val) {
        nativeWriteStrongBinder(mNativePtr, val);
    }
}
  ```
`native nativeWriteStrongBinder`JNI注册后对应的执行函数为android_os_Parcel_writeStrongBinder：
**/frameworks/base/core/jni/android_os_Parcel.cpp**

  ```
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        // 1. 存储ibinderForJavaObject(env, object)后的对象
        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}

sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;
    // 1. 如果是java层的Binder对象，返回JavaBBinderHolder.get()对象
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        return jbh != NULL ? jbh->get(env, obj) : NULL;
    }
    //2. 如果是java层的BinderProxy对象，
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return (IBinder*)
            env->GetLongField(obj, gBinderProxyOffsets.mObject);
    }

    return NULL;
}

class JavaBBinderHolder : public RefBase
{
public:
    sp<JavaBBinder> get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);
        sp<JavaBBinder> b = mBinder.promote();
        if (b == NULL) {
            b = new JavaBBinder(env, obj);
            mBinder = b;
            ALOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%" PRId32 "\n",
                 b.get(), b->getWeakRefs(), obj, b->getWeakRefs()->getWeakCount());
        }

        return b;
    }
}
  ```
ibinderForJavaObject()根据传进来的对象，判断是java层的Binder类还是BinderProxy类。AMS继承java层的Binder类。所以会执行注释1,Binder类在初始化的时候会通过JNI生成一个JavaBBinderHolder对象保存在mObject变量中，先通过gBinderOffsets.mObject拿到JavaBBinderHolder，最后通过调用JavaBBinderHolder.get()方法返回一个JavaBBinder。
所以Parcel存储的不是ASM对象，而是JavaBBinder对象。

**解析JavaBBinder:**
JavaBBinder继承BBinder，Client找到Server后，会调用Service的JavaBBinder的onTransact方法
  ```
class JavaBBinder : public BBinder
{
public:
    JavaBBinder(JNIEnv* env, jobject object)
        : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
    {
        ALOGV("Creating JavaBBinder %p\n", this);
        android_atomic_inc(&gNumLocalRefs);
        incRefsCreated(env);
    }

    virtual status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
    {
         // ...
         // 执行Binder类的execTransact()方法
        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
        //  ...
    }
}

  ```
JavaBBinder的onTransact方法，调用了Binder的execTransact()方法。

  ```
public class Binder implements IBinder {

    private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
  
        try {
            res = onTransact(code, data, reply, flags);
        } catch (RemoteException e) {
         
        } catch (RuntimeException e) {
        return res;
    }

    protected boolean onTransact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
       ...
    }

}
  ```
Binder类的execTransact方法又调用其onTransact()方法，Binder的子类可以通过重写`onTransact()`的方法来实现client的交互。比如AMS和它的父类ActivityManagerNative都重写了其`onTransact`方法。

#### 注册服务总结
1. ServiceManager方法通过`getIServiceManager()`得到ServiceMangerProxy(BinderProxy)对象。
2. addService()方法中写入Parcel的是JavaBBinder对象，通信时通过它调用Binder类的onTransact()方法。

## Java Binder 查询服务
通过查询ActivityManagerService服务为例，来学习查询服务的过程：
  ```
ServiceManager.getService("activity")
  ```
查看ServiceManager.getService方法
  ```
public final class ServiceManager {

    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

}
  ```
通过上面的注册服务知道，getIServiceManager() 返回ServiceManagerProxy。
  ```
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        // 1. 写入查询服务的name
        data.writeString(name);
        // 2. 通过BinderProxy最终调用BpBinder. transact()方法
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        // 3. 根据回复的数据查到查询服务的BBinder
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }
}
  ```
注释1，向data写入查询服务的name。
注释2，上面已经分析过，通过BinderProxy最终调用BpBinder. transact()方法。
注释3，通过`reply.readStrongBinder()`拿到查询服务的BBinder，接下来分析如何获取。

#### reply.readStrongBinder()
  ```
public final class Parcel {
 private static native IBinder nativeReadStrongBinder(long nativePtr);

    public final IBinder readStrongBinder() {
        return nativeReadStrongBinder(mNativePtr);
    }
}
  ```
nativeReadStrongBinder对应的执行函数为android_os_Parcel_readStrongBinder
**/frameworks/base/core/jni/android_os_Parcel.cpp**
  ```
static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
{
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        // 1. javaObjectForIBinder()是创建一个BinderProxy对象，并把IBinder设置到其mObject变量
        return javaObjectForIBinder(env, parcel->readStrongBinder());
    }
    return NULL;
}

  ```
注释1中，javaObjectForIBinder上面已经分析过，是创建一个BinderProxy对象，并把IBinder设置到其mObject变量。
那这里 `parcel->readStrongBinder()`方法就是在reply中通过handle寻找IBinder：
  ```
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}

status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);

    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                //  根据handle找到对应的BpBidner(handle)
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}

  ```

#### 查询服务总结：
1. 根据查询服务的handle找到对应的BpBinder。
