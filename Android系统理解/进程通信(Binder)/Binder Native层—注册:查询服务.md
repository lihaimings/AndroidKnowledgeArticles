Binder系列：
[Binder Kernel层—Binder内核驱动](https://www.jianshu.com/p/fdf12cd7c28d)
[Binder Native层—服务管理(ServiceManager进程)](https://www.jianshu.com/p/fb514e3eed3a)
[Binder Native层—注册/查询服务](https://www.jianshu.com/p/a7e5bbb0ab72)
[Binder Framework层—注册和查询服务](https://www.jianshu.com/p/60be38017422)
[Binder 应用层-AIDL原理](https://www.jianshu.com/p/7ca78da80f95)

> 本篇文章基于Android6.0源码分析

相关源码文件：
  ```
/frameworks/av/media/mediaserver/main_mediaserver.cpp
/frameworks/native/libs/binder/ProcessState.cpp
/frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp
/frameworks/native/libs/binder/IServiceManager.cpp
/frameworks/native/include/binder/IServiceManager.h
/frameworks/native/include/binder/IInterface.h
/frameworks/native/libs/binder/Binder.cpp
/frameworks/native/libs/binder/IPCThreadState.cpp
  ```

## 进程间的通信
 Linux的进程通信(IPC:inter-Process Communication)有：**管道(pipe)、信号(sinal)、信号量(semophore)、消息队列(message)、共享内存(share memory)、套接字(socket)**。Android是基于Linux基础开发的，所以Android也可以使用Linux的IPC通信方式，比如Android的Init进程就是通过Socket套接字进行进程间的通信。除此之外，Android也有自己的进程通信方式。
Android 的特有进程通信有：**序列化(Serializable\Parcelable)、Binder、Bundle、文件共享、ContentProvider、Messenger**。Android的服务进程就是通过**Binder**机制进行进程间的通信。

![IPC通信方式](https://upload-images.jianshu.io/upload_images/22650779-3003f8fa6eeb1238.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么Linux已经提供了那么多的IPC机制，为什么Android还要设置一个Binder的IPC机制呢？原因是Binder在进程间数据的传递性能更佳。

在Linux的IPC通信方式，在进程间的一次数据通信要复制两次数据。数据要经过：用户空间->内核空间->用户空间的两个拷贝。如下图所示：

![Linux的IPC通信](https://upload-images.jianshu.io/upload_images/22650779-2567057341ceabde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Binder的IPC机制中，通过对接收进程的用户空间和内核空间的缓存区进行映射，当发送的用户空间只要进行一次的数据拷贝就可以把数据传递到另一个进程。

![Binder的IPC通信](https://upload-images.jianshu.io/upload_images/22650779-d432970c166e9edb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Binder机制根据安卓的分层，又分为：**Java Binder(Framework层)、Native Binder(Native层)、Kernel Binder(Kernel层)**。而Java和Native层是应用开发者需要掌握的。

`Native层注册/查询服务的概括`
![注册/查询服务的概括](https://upload-images.jianshu.io/upload_images/22650779-f54c757e4ec3b7ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![注册/查询服务的概括](https://upload-images.jianshu.io/upload_images/22650779-6a8cf7793986e9c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/22650779-2185e1fbd73e5ec2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 1. 注册服务
通过分析MediaServer的启动，讲解一个MediaPlayer框架的服务的注册(addService)的案例。
MediaPlayer框架采用的是C/S架构，server进程会通过(addService)注册服务到ServiceManager，如Media服务启动过程就会注册服务到ServiceManager。client端会先通过ServiceManager查询服务，然后跟服务相关的Server进程建立通信，在Client、Server、ServiceManager都是通过Binder通信的。

![MediaPlayer框架](https://upload-images.jianshu.io/upload_images/22650779-f30db8bbf46aeaa9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## MediaServer启动

Media进程的创建也是Init进程解析`Init.rc`文件中创建
  ```
service media /system/bin/mediaserver 
    class main
    user media
    group audio camera inet net_bt net_bt_admin net_bw_acct drmrpc mediadrm     
    ioprio rt 4
  ```
Media进程创建过程中启动多媒体注册服务addService，比如MediaPlayerService注册服务。

![media启动大致流程](https://upload-images.jianshu.io/upload_images/22650779-3185752284712d7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而对应可执行的文件：
**/frameworks/av/media/mediaserver/main_mediaserver.cpp**
  ```
    int main(int argc __unused, char** argv)
    {
        ...
        // 生成ProcessState单例
        sp<ProcessState> proc (ProcessState::self()); // 1
        sp<IServiceManager> sm(defaultServiceManager()); // 2
        ...
        // 注册MediaPlayerService服务
        MediaPlayerService::instantiate();
        ...
        // 启动Binder线程池
        ProcessState::self()->startThreadPool(); // 3
        // 当前线程加入到线程池
        IPCThreadState::self()->joinThreadPool();
    }
  ```
注释1获取ProcessState实例，该过程会打开/dev/binder设备，并使用mmap进行和内核进行虚拟内存映射用于接收数据。
注释2获取得到一个IServicManager，通过这个IServiceManager其他进程可以和ServiceManger进行交互。
注释3是注册MediaPlayerService服务

#### 进程唯一的ProcessState
  ```

    sp<ProcessState> ProcessState::self()
    {
        Mutex::Autolock _l (gProcessMutex);
        if (gProcess != NULL) {
            return gProcess;
        }
        // 实例化ProcessState，跳转到构造函数
        gProcess = new ProcessState;
        return gProcess;
    }

    ProcessState::ProcessState()
    : mDriverFD(open_driver()) // 1. open_driver()函数，打开Binder驱动
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    {
        if (mDriverFD >= 0) {
            // 2. 采用内存映射函数mmap,给binder分配一块虚拟内存空间，大小为BINDER_VM_SIZE= 1M-8K
            mVMStart =
                mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
            //没有足够空间分配给 /dev/binder, 则关闭驱动
            if (mVMStart == MAP_FAILED) {
                close(mDriverFD);
                mDriverFD = -1;
            }
        }
    }

    static int open_driver()
    {
        // 打开 dev/binder 设备，建立与内核Binder驱动的交互通道
        int fd = open("/dev/binder", O_RDWR);
        if (fd >= 0) {
            fcntl(fd, F_SETFD, FD_CLOEXEC);
            int vers = 0;
            status_t result = ioctl(fd, BINDER_VERSION, &vers);
            if (result == -1) {
                ALOGE("Binder ioctl to obtain version failed: %s", strerror(errno));
                close(fd);
                fd = -1;
            }
            if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
                ALOGE("Binder driver protocol does not match user space protocol!");
                close(fd);
                fd = -1;
            }
            size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
            // 通过 ioctl 设置binder驱动和最大的线程数
            result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
            if (result == -1) {
                ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
            }
        } else {
            ALOGW("Opening '/dev/binder' failed: %s\n", strerror(errno));
        }
        return fd;
    }

  ```
**实例化ProcessState**：主要完成两件事：
注释1中通过`open_driver()`方法打开 dev/binder设备，它会返回一个文件操作符fd,这样就可以操作内核Binder驱动。
注释2中通过`mmap`函数，它会在内核虚拟地址空间中申请一块与用户虚拟内存相同大小的内存，然后在映射到物理内存，实现内核虚拟地址空间和用户虚拟内存空间的数据同步。

#### ServiceManager中的Binder机制
  ```
    sp<IServiceManager> defaultServiceManager()
    {
        // 不为null，直接返回
        if (gDefaultServiceManager != NULL) return gDefaultServiceManager;

        {
            // 加一把自动锁
            AutoMutex _l(gDefaultServiceManagerLock);
            while (gDefaultServiceManager == NULL) {
                // gDefaultServiceManager实例，interface_cast<IServiceManager>(ProcessState->getContextObject(NULL)）
                gDefaultServiceManager = interface_cast<IServiceManager>(
                    ProcessState::self()->getContextObject(NULL));
                if (gDefaultServiceManager == NULL)
                    sleep(1);
            }
        }

        return gDefaultServiceManager;
    }

    sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
    {
        return getStrongProxyForHandle(0);
    }

    sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
    {
        //1.  获取handle位置的handle_entry结构体，如果没有则新创建一个handle_entry结构体
        handle_entry * e = lookupHandleLocked(handle);

        if (e != NULL) {
            // 刚开始e->binder 为null
            IBinder * b = e->binder;
            if (b == NULL || !e->refs->attemptIncWeak(this)) {
                // 当hanle 为0 ，则PING_TRANSACTION看ServiceManager进程能不能访问
                if (handle == 0) {
                    Parcel data;
                    status_t status = IPCThreadState ::self()->transact(
                    0, IBinder::PING_TRANSACTION, data, NULL, 0);
                    if (status == DEAD_OBJECT)
                        return NULL;
                }
                //2. 返回 new BpBinder(handle)
                b = new BpBinder (handle);
                e->binder = b;
                if (b) e->refs = b->getWeakRefs();
                result = b;
            } else {
                result.force_set(b);
                e->refs->decWeak(this);
            }
        }

        return result;
    }
  ```
ProcessState的getContextObject函数调用了getStrongProxyForHandle(0)，那么handler就为0，handler是资源标识。
注释1中查询handler对应的资源（handle_entry）是否存在，没有则创建一个空的handle_entry资源并返回。
注释2中创建BpBinder(handler)，并赋值给handle_entry的Binder对象。
另：BpBinder是Client端的代理类，BBinder是Server的代理类,BpBinder通过handler找到对应的BBinder。

`ProcessState::self()->getContextObject(NULL))`返回的是**new BpBinder(0)**。接着看interface_cast(new BpBinder(0)):
  ```
    [IInterface.h]
    // INTERFACE为IServiceManager
    template<typename INTERFACE>
    inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
    {
        // 1 INTERFACE为IServiceManager
        return INTERFACE::asInterface(obj);
    }
  ```
注释1处的代码等价于： IServiceManager::asInterface(obj)

#### IServiceMananger
IServiceMananger用于处理ServiceManager的业务，IServiceMananger是C++代码，所以定义在IServiceMananger.h中
  ```
     class IServiceManager : public IInterface
    {
        public:
        DECLARE_META_INTERFACE(ServiceManager); //1

        // 一些操作SerVice服务
        virtual sp < IBinder > getService (const String16 & name) const = 0;
        virtual sp < IBinder > checkService (const String16 & name) const = 0;
        virtual status_t addService(const String16 & name, const sp < IBinder >& service,
        virtual Vector < String16 > listServices () = 0;

        enum {
            GET_SERVICE_TRANSACTION = IBinder::FIRST_CALL_TRANSACTION,
            CHECK_SERVICE_TRANSACTION,
            ADD_SERVICE_TRANSACTION,
            LIST_SERVICES_TRANSACTION,
        };
    };
  ```
IServiceManager继承IInterface,并定义了一些Service服务的操作，注释1处的宏定义在IInterface.h中：


  ```
    //IInterface.h ， INTERFACE为ServiceManager
    #define DECLARE_META_INTERFACE(INTERFACE)
    static const android::String16 descriptor;
    static android::sp<I##INTERFACE> asInterface(
    const android::sp<android::IBinder>& obj);
    virtual const android::String16& getInterfaceDescriptor() const;
    I##INTERFACE();
    virtual ~I##INTERFACE();

    #define IMPLEMENT_META_INTERFACE(INTERFACE, NAME)
    const android::String16 I##INTERFACE::descriptor(NAME);
    const android::String16&
    I##INTERFACE::getInterfaceDescriptor() const {
        return I##INTERFACE::descriptor;
    }
    // IServiceManager::asInterface返回BpServiceManager(obj)
    android::sp<I##INTERFACE> I##INTERFACE::asInterface(
    const android::sp<android::IBinder>& obj)
    {
        android::sp<I##INTERFACE> intr;
        if (obj != NULL) {
            intr = static_cast<I##INTERFACE*>(
            obj->queryLocalInterface(
            I##INTERFACE::descriptor).get());
            if (intr == NULL) {
                // 返回BpServiceManager对象
                intr = new Bp##INTERFACE(obj);
            }
        }
        return intr;
    }
    I##INTERFACE::I##INTERFACE() { }
    I##INTERFACE::~I##INTERFACE() { }

  ```
interface_cast方法最终返回BPServiceManager对象,新建的BpServiceManager，传入的obj参数为BpBinder。
那么BpServiceManager有什么作用，看看BpServiceManager类：

  ```
    class BpServiceManager : public BpInterface<IServiceManager>
    {
        public:BpServiceManager(const sp < IBinder >& impl): BpInterface<IServiceManager>(impl) {
        }

        virtual sp < IBinder > getService (const String16 & name) const
                {
                    ...
                }

        virtual sp < IBinder > checkService (const String16 & name) const
                {
                    ...
                }

        virtual status_t addService(const String16 & name, const sp < IBinder >& service,
        bool allowIsolated)
        {
            ...
        }

        virtual Vector < String16 > listServices ()
        {
            ...
        }
    };

  ```
BpServiceManager继承BpInterface，并在构造函数中把参数impl传递给BpInterface构造函数，其中impl为BpBinder。BpServiceManager并且实现了Service服务的操作函数，包括有：`getService、checkService、addService、listServices`。继续看BpInterface，看看BpBinder传到哪里？

  ```
 class BpInterface : public INTERFACE, public BpRefBase
    {
       
    };

    BpRefBase::BpRefBase(const sp<IBinder>& o): mRemote(o.get()), mRefs(NULL), mState(0)
    {
        extendObjectLifetime(OBJECT_LIFETIME_WEAK);

        if (mRemote) { mRemote ->
            incStrong(this);           // Removed on first IncStrong().
            mRefs = mRemote->createWeak(this);  // Held for our entire lifetime.
        }
    }
  ```
BpInterface继承BpRefBase，BpRefBase最终把BpBinder赋值给变量mRemote,所以BpServiceManager->mRemote变量就是BpBinder。

defaultServiceManager()的分析就完成了，可以知道生成了一个`BpServiceManage`类，这个类实现了Service的操作函数，并把BpBinder赋值给mRemote变量。如下图是IServiceManager的关系图：
![IServiceManager](https://upload-images.jianshu.io/upload_images/22650779-f8553c54019d49ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### MediaPlayerService注册服务(addService)
 
  ```
   void MediaPlayerService::instantiate() {
        // defaultServiceManager()获取默认SM,= new BpServiceManager(new BpBinder(0))
        defaultServiceManager()->addService(
        String16("media.player"), new MediaPlayerService());
    }
  ```
### Native层注册服务
  ```
   // BpServiceManager
class BpServiceManager : public BpInterface<IServiceManager>
{

    // 获取服务
    virtual status_t addService(const String16& name, const sp<IBinder>& service,
    bool allowIsolated)
    {
        // Parcel 是内存共享读写
        Parcel data, reply;
        // 写入头信息 "android.os.IServiceManager"，ServiceManager收到请求后会判断
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        // name = "media.player"
        data.writeString16(name);
        // service = MediaPlayService对象
        data.writeStrongBinder(service);
        // allowIsolated = false
        data.writeInt32(allowIsolated ? 1 : 0);
         // remote等于mRemote，就是BpBinder
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
}

  ```
addService就是将发送数据包装成data,然后给BpBinder处理，调用BpBinder->transact(ADD_SERVICE_TRANSACTION, data, &reply)函数。

  ```
    [BpBinder.cpp]
    status_t BpBinder::transact(
    uint32_t code, const Parcel& data , Parcel* reply, uint32_t flags)
    {
        if (mAlive) {
            // IPCThreadState ::self()实例化IPCThreadState，然后执行其transact方法
            status_t status = IPCThreadState ::self()->transact(
            mHandle, code, data, reply, flags);
            if (status == DEAD_OBJECT) mAlive = 0;
            return status;
        }

        return DEAD_OBJECT;
    }

    // TLS为线程本地存储空间，是线程私有的，可以获得保存在其的IPCThreadState对象
    IPCThreadState* IPCThreadState::self()
    {
        // 已经有了TLS
        if (gHaveTLS) {
            restart:
            const pthread_key_t k = gTLS;
            // 根据TLS获取IPCThreadState
            IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
            if (st) return st;
            // 返回IPCThreadState对象
            return new IPCThreadState;
        }

        if (gShutdown) return NULL;

        pthread_mutex_lock(&gTLSMutex);

        // 没有TLS
        if (!gHaveTLS) {
            // 创建线程的TLS
            if (pthread_key_create(&gTLS, threadDestructor) != 0) {
                pthread_mutex_unlock(&gTLSMutex);
                return NULL;
            }
            gHaveTLS = true;
        }
        pthread_mutex_unlock(&gTLSMutex);
        goto restart;
    }

    // 向TLS设置IPCThreadState
    IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
    mMyThreadId(gettid()),
    mStrictModePolicy(0),
    mLastTransactionBinderFlags(0)
    {
        // 通过pthread_setspecific/pthread_getspecific来设置和获取IPCThreadState
        pthread_setspecific(gTLS, this);
        clearCaller();
        mIn.setDataCapacity(256); // 存储接收Binder的数据
        mOut.setDataCapacity(256); // 存储发送给Binder的数据
    }

  ```
BpBinder又将数据交给IPCThreadState类来处理，IPCThreadState处理过程中会先通过TLS(Thread Local Storage)线程本地缓存先尝试获取IPCThreadState类，如果没有则设置TLS，并保存IPCThreadState在TLS中。IPCThreadState有两个变量mIn、mOut用来存储接收Binder发来的数据和给Binder发送的数据。看看IPCThreadState的transact函数

  ```
    status_t IPCThreadState::transact(int32_t handle,
    uint32_t code, const Parcel& data ,
    Parcel* reply, uint32_t flags)
    {
        ...
        if (err == NO_ERROR) {
            // 传输数据
            err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
        }

        if ((flags & TF_ONE_WAY) == 0) {

        if (reply) {
            // 等待响应
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(& fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }

        return err;
    }

    status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data , status_t* statusBuffer)
    {
        binder_transaction_data tr;

        tr.target.ptr = 0;
        tr.target.handle = handle; // 目标的handle为0 ，代表是要转发给ServiceManager进程
        tr.code = code;  // code为ADD_SERVICE_TRANSACTION，是添加服务
        tr.flags = binderFlags; // binderFlags为0
        tr.cookie = 0;
        tr.sender_pid = 0;
        tr.sender_euid = 0;

        // data为Parcel对象
        const status_t err = data.errorCheck();
        if (err == NO_ERROR) {
            tr.data_size = data.ipcDataSize(); // datasize
            tr.data.ptr.buffer = data.ipcData(); // data
            tr.offsets_size = data.ipcObjectsCount() * sizeof(binder_size_t);
            tr.data.ptr.offsets = data.ipcObjects();
        } else if (statusBuffer) {
            tr.flags | = TF_STATUS_CODE;
            *statusBuffer = err;
            tr.data_size = sizeof(status_t);
            tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
            tr.offsets_size = 0;
            tr.data.ptr.offsets = 0;
        } else {
            return (mLastError = err);
        }

        //  cmd = BC_TRANSACTION ,  驱动找到 ServiceManager 后会像客户端返回一个 BR_TRANSACTION_COMPLETE 表示通信请求已被接受，然后 Client 进入等待
        mOut.writeInt32(cmd);
        // 写入 binder_transaction_data 数据到mOut
        mOut.write(& tr, sizeof(tr));

        return NO_ERROR;
    }
  ```
writeTransactionData函数主要是通过binder_transaction_data结构体写入目标handler,这里值为0，则代表为ServiceManager，并写入code为ADD_SERVICE_TRANSACTION代表注册服务。
最后把binder_transaction_data结构体写入mOut，并设置命令协议为BC_TRANSACTION，其中以BC_开头代表向Binder写入数据，以BR_开头代表接收Binder数据的命令协议。

### Native层注册服务处理
![](https://upload-images.jianshu.io/upload_images/22650779-308355d07d27a2c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

  ```

    status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
    {
        // 循环执行
        while (1) {
             // 不断执行 talkWithDriver()
            if ((err = talkWithDriver()) < NO_ERROR) break;
            ...
           //  处理Kernel返回的驱动命令
            switch(cmd) {
               // 代表Binder Kernel已经处理完BC_TRANSACTION驱动命令
               case BR_TRANSACTION_COMPLETE:
                   if (!reply && !acquireResult) goto finish;
                   break;
                // 代表Server端已经
                case BR_REPLY:
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        //当reply对象回收时，则会调用freeBuffer来回收内存
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                       ...
                    }
                } else {
                   ...
                }
            }
                default:
                err = executeCommand(cmd);
                if (err != NO_ERROR) goto finish;
                break;
            }
        }
        ...
        return err;
    }

    status_t IPCThreadState::talkWithDriver(bool doReceive)
    {
        ...
        // 与Binder通信的结构体
        binder_write_read bwr;
        const bool needRead = mIn.dataPosition() >= mIn.dataSize();
        const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

        bwr.write_size = outAvail;
        bwr.write_buffer = (uintptr_t) mOut . data ();

        // This is what we'll read.
        if (doReceive && needRead) {
            //接收数据缓冲区信息的填充。如果以后收到数据，就直接填在mIn中了。
            bwr.read_size = mIn.dataCapacity();
            bwr.read_buffer = (uintptr_t) mIn . data ();
        } else {
            bwr.read_size = 0;
            bwr.read_buffer = 0;
        }


        //当读缓冲和写缓冲都为空，则直接返回
        if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

        bwr.write_consumed = 0;
        bwr.read_consumed = 0;
        status_t err;
        do {
            //通过ioctl不停的读写操作，跟Binder Driver进行通信
            if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
            ...
        } while (err == -EINTR);

    if (err >= NO_ERROR)
    {
        // 清空mOut数据
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else
                mOut.setDataSize(0);
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }

        return err;
    }

  ```
talkWithDriver函数通过binder_write_read结构体，先把mOut、mIn里面的数据写入binder_write_read结构体，然后通过ioctl函数和Binder驱动进行通信。

**注册服务总结：**
1. 先获取BpServiceManager和BpBinder,
2. 调用BpServiceManager.addService函数，addService将发送数据包装成data,然后传递ADD_SERVICE_TRANSACTION参数和data参数，给BpBinder处理。
3. BpBinder转交给IPCThreadState类处理，IPCThreadState先从TLS中获取，如果没有在创建。其中有mOut、mIn两个变量存储数据。
4. IPCThreadState先以BC_TRANSACTION命令协议，将发送数据写入binder_transaction_data中，然后把binder_transaction_data写入到mOut变量中。然后将mOut、mIn写入binder_write_read结构体，并通过ioctl与Binder进行数据通信。

##  2. Native层获取查询服务
  ```
class BpServiceManager : public BpInterface<IServiceManager>
{

    virtual sp<IBinder> getService(const String16& name) const
    {
        unsigned n;
        for (n = 0; n < 5; n++){
            // 调用 checkService(name)方法
            sp<IBinder> svc = checkService(name);
            if (svc != NULL) return svc;
            ALOGI("Waiting for service %s...\n", String8(name).string());
            sleep(1);
        }
        return NULL;
    }

    virtual sp<IBinder> checkService( const String16& name) const
    {
        Parcel data, reply;
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeString16(name); // 根据名字向ServiceManager查询服务
        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
        return reply.readStrongBinder(); // 返回要查询服务的 IBinder
    }

};
  ```
`remote()`就是BpBinder,也就是执行`BpBinder.->transact()`,BpBinder又转交给`IPCThreadState.->transact()`处理，IPCThreadState最终调用`waitForResponse()`方法处理。

![](https://upload-images.jianshu.io/upload_images/22650779-308355d07d27a2c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)
  ```
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        
        cmd = (uint32_t)mIn.readInt32();

        switch (cmd) {
        ... 
        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        //当reply对象回收时，则会调用freeBuffer来回收内存
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        ...
                    }
                } else {
                    ...
                }
            }
            goto finish;
        }
    }

    return err;
}

void Parcel::ipcSetDataReference(const uint8_t* data, size_t dataSize,
    const binder_size_t* objects, size_t objectsCount, release_func relFunc, void* relCookie)
{
    binder_size_t minOffset = 0;
    freeDataNoInit();
    mError = NO_ERROR;
    mData = const_cast<uint8_t*>(data);
    mDataSize = mDataCapacity = dataSize;
    //ALOGI("setDataReference Setting data size of %p to %lu (pid=%d)", this, mDataSize, getpid());
    mDataPos = 0;
    ALOGV("setDataReference Setting data pos of %p to %zu", this, mDataPos);
    mObjects = const_cast<binder_size_t*>(objects);
    mObjectsSize = mObjectsCapacity = objectsCount;
    mNextObjectHint = 0;
    mOwner = relFunc;
    mOwnerCookie = relCookie;
    for (size_t i = 0; i < mObjectsSize; i++) {
        binder_size_t offset = mObjects[i];
        if (offset < minOffset) {
            ALOGE("%s: bad object offset %" PRIu64 " < %" PRIu64 "\n",
                  __func__, (uint64_t)offset, (uint64_t)minOffset);
            mObjectsSize = 0;
            break;
        }
        minOffset = offset + sizeof(flat_binder_object);
    }
    scanForFds();
}

  ```
在Kernel返回的驱动命令`BR_REPLY`中，将要查询服务的数据写入到`reply`结构体中。拿到reply数据后，通过Parcel.cpp->readStrongBinder()方法找到要查询服务的BpBinder。

  ```
// 回到最初的 checkService 代码
 virtual sp<IBinder> checkService( const String16& name) const
  {
      Parcel data, reply;
      data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
      data.writeString16(name); // 根据名字向ServiceManager查询服务
      remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
      return reply.readStrongBinder(); // 返回要查询服务的 IBinder
  }

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
                ...
            case BINDER_TYPE_HANDLE:
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
// 会有一个缓存，返回的就是 BpBinder(handle)
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            b = new BpBinder(handle); 
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            // This little bit of nastyness is to allow us to add a primary
            // reference to the remote proxy when this team doesn't have one
            // but another team is sending the handle to us.
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
  ```

## 获取服务总结
- 通过 `CHECK_SERVICE_TRANSACTION`向ServiceManager查询服务的handle信息
- ServiceManager查询好后，向Binder Kernel发送BC_REPLY驱动命令，Binder Kernel向Client端发送BR_REPLY驱动命令
- Client端根据BR_REPLY驱动命令拿到查询服务的handle数据，并返回查询服务的BpBinder(handle)。




