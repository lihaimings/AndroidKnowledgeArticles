Binder系列：
[Binder Kernel层—Binder内核驱动](https://www.jianshu.com/p/fdf12cd7c28d)
[Binder Native层—服务管理(ServiceManager进程)](https://www.jianshu.com/p/fb514e3eed3a)
[Binder Native层—注册/查询服务](https://www.jianshu.com/p/a7e5bbb0ab72)
[Binder Framework层—注册和查询服务](https://www.jianshu.com/p/60be38017422)
[Binder 应用层-AIDL原理](https://www.jianshu.com/p/7ca78da80f95)


> 本篇文章基于Android6.0源码分析

相关源码文件：
  ```
/system/core/rootdir/init.rc
/framework/native/cmds/servicemanager/
  - service_manager.c
  - binder.c
  ```
上篇文章我们分析了服务是如何注册的，其中ServiceManager进程作为Server端，负责处理的服务的注册(addService)和获取(getService),那么本篇将讲解ServiceManager进程启动过程，它是如何处理的服务的操作的。
## ServiceManager入口函数
ServiceManager进程由Init进程通过init.rc配置文件中启动：
/system/core/rootdir/init.rc
  ```
    service servicemanager /system/bin/servicemanager
    class core
    user system //1
    group system
    critical //2
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm
  ```
通过配置文件可知，创建一个名为servicemanager的进程，执行程序路径为/system/bin/servicemanager，注释1说明servicemanager是以用户system的身份运行，注释2说明servicemanager是系统的关键服务，关键服务是不会退出，如果退出，系统就会重启，也会重启onrestart修饰的进程服务。

servicemanager的入口函数在service_manager.c中
**/frameworks/native/cmds/servicemanager/service_manager.c**
  ```
      int main(int argc, char **argv)
    {
        // 1
        struct binder_state * bs;
        ...
        // 2
        bs = binder_open(128 * 1024);
        ...
        // 3
        if (binder_become_context_manager(bs)) {
            ALOGE("cannot become context manager (%s)\n", strerror(errno));
            return -1;
        }
        ...
        // 4
        binder_loop(bs, svcmgr_handler);

        return 0;
    }

// binder.c
 struct binder_state
{
    int fd; //binder设备的文件描述符
    void *mapped; //binder设备文件映射到进程的地址空间
    size_t mapsize; //内存映射后，系统分配的地址空间的大小，默认为128KB
};
  ```
注释1中binder_state结构保存Binder的三个信息，文件描述符fd、mmap映射的地址、内存映射的内存大小。
main函数主要做了三件事：
注释2调用binder_open函数打开binder设备
注释3调用binder_become_context_manager让serviceManager成为binder机制的上下文管理者
注释4调用binder_loop，循环等待和处理client端发来的请求。

![ServiceManager启动流程图](https://upload-images.jianshu.io/upload_images/22650779-fc454eb86106fab6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 打开Binder设备
  ```
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

    bs->fd = open(driver, O_RDWR | O_CLOEXEC);//1
    if (bs->fd < 0) {
        fprintf(stderr,"binder: cannot open %s (%s)\n",
                driver, strerror(errno));
        goto fail_open;
    }
    //获取Binder的version
    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {//2
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }

    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);//3
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }
    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
  ```
注释1中调用open打开/dev/binder设置，open对应执行内核的binder_open并返回对应的文件描述符fd,并把它设置给binder_state(bs)的结构体中。如果获取失败则跳转fail_open清理Binder。
注释2获取内核Binder的版本号，如果获取失败或和用户空间的版本号不一致则跳转到fail_open
注释3进行mmap进行内存映射，并申请128k的内存大小，并赋值给bs结构体中保存。

### 注册成为Binder机制的上下文管理者
  ```
int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
  ```
ioctl对应调用内核的binder_ioctl函数，通过传递BINDER_SET_CONTEXT_MGR标识，让serviceManager成为Binder机制的上下文管理者，这个上个文管理者是唯一。


 ### 循环等待和处理client端发来的请求
servicemanager成为Binder机制上下文管理者后，就会开始循环等待client端的请求操作。
  ```
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr; 
    uint32_t readbuf[32];

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;
    binder_write(bs, readbuf, sizeof(uint32_t));//1

    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);//2

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }

        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);//3
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
  ```
注释1处先通过binder_write函数，给Binder传递BC_ENTER_LOOPER参数，告诉Binder开始进入循环状态。
  ```
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                strerror(errno));
    }
    return res;
}
  ```
注释2处开始无限循环的通过向Binder读写数据，来等待client端的请求
注释3和Binder有数据的传递，则调用binder_parse函数开始解析Client发送的请求。

### 查询服务在ServiceManager中处理
查询服务中code为`SVC_MGR_GET_SERVICE`或 `SVC_MGR_CHECK_SERVICE`

  ```
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    ...
    while (ptr < end) {
       ...
        switch(cmd) {
        ...
        case BR_TRANSACTION: {
            ...
            if (func) {
                 ...
                //1. 执行回调函数
                res = func(bs, txn, &msg, &reply);
                // 2. 把执行结果写入 binder 驱动
                binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
            }
            ptr += sizeof(*txn);
            break;
        }
       case BR_REPLY: {}
       case BR_DEAD_BINDER: {}
       case BR_FAILED_REPLY: {}
        ...
    }

    return r;
}

int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        // 找到要查询服务的handle
        handle = do_find_service(bs, s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        //  将handle写入到reply数据中
        bio_put_ref(reply, handle);
        return 0;
        ...
    }

    bio_put_uint32(reply, 0);
    return 0;
}

void bio_put_ref(struct binder_io *bio, uint32_t handle)
{
    struct flat_binder_object *obj;

    if (handle)
        obj = bio_alloc_obj(bio);
    else
        obj = bio_alloc(bio, sizeof(*obj));

    if (!obj)
        return;
    // 向 replay 中写入一个 flat_binder_object
    obj->flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    obj->type = BINDER_TYPE_HANDLE;
    obj->handle = handle;
    obj->cookie = 0;
}

void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;
    
    data.cmd_free = BC_FREE_BUFFER;
    data.buffer = buffer_to_free;
    // 回复命令
    data.cmd_reply = BC_REPLY;
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        ...
    } else {
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    binder_write(bs, &data, sizeof(data));
}

  ```
在查询服务中，ServiceManager会`BR_TRANSACTION`命令中先执行回调函数`svcmgr_handler()`函数，根据名字找到对应服务的handle值，并把值保存在reply的flat_binder_object的结构体中，最后执行`binder_send_reply()`方法通过`BC_REPLY`命令把目标服务的handle值等数据返回给Binder Kernel驱动中。

### 注册服务中ServiceManager中处理
注册服务中code为`SVC_MGR_ADD_SERVICE`

  ```
int binder_parse(struct binder_state *bs, struct binder_io *bio,
               uintptr_t ptr, size_t size, binder_handler func)
{
  ...
  while (ptr < end) {
     ...
      switch(cmd) {
      ...
      case BR_TRANSACTION: {
          ...
          if (func) {
               ...
              //1. 执行回调函数
              res = func(bs, txn, &msg, &reply);
              // 2. 把执行结果写入 binder 驱动
              binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
          }
          ptr += sizeof(*txn);
          break;
      }
     case BR_REPLY: {}
     case BR_DEAD_BINDER: {}
     case BR_FAILED_REPLY: {}
      ...
  }

  return r;
}

int svcmgr_handler(struct binder_state *bs,
                 struct binder_transaction_data *txn,
                 struct binder_io *msg,
                 struct binder_io *reply)
{
...
  // 判断 code 是什么命令
  switch(txn->code) {
   // 添加服务到列表
  case SVC_MGR_ADD_SERVICE:
      // 获取服务的名称
      s = bio_get_string16(msg, &len);
      if (s == NULL) {
          return -1;
      }
      // 获取服务的 handle 的值
      handle = bio_get_ref(msg);
      // 执行添加服务到列表的逻辑
      if (do_add_service(bs, s, len, handle, txn->sender_euid,
          allow_isolated, txn->sender_pid))
          return -1;
      break;
  default:
      ALOGE("unknown code %d\n", txn->code);
      return -1;
  }

  bio_put_uint32(reply, 0);
  return 0;
}

int do_add_service(struct binder_state *bs,
                 const uint16_t *s, size_t len,
                 uint32_t handle, uid_t uid, int allow_isolated,
                 pid_t spid)
{
  struct svcinfo *si;

  if (!handle || (len == 0) || (len > 127))
      return -1;

  //权限检查 
  if (!svc_can_register(s, len, spid)) {
      return -1;
  }

  //服务检索 
  si = find_svc(s, len);
  if (si) {
      if (si->handle) {
          // 服务已注册时，释放之前添加的相应服务
          svcinfo_death(bs, si); 
      }
      si->handle = handle;
  } else {
      si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
      // 内存不足，无法分配足够内存
      if (!si) {  
          return -1;
      }
      // 指定 handle 值
      si->handle = handle;
      si->len = len;
      // 指定当前添加服务的名称
      memcpy(si->name, s, (len + 1) * sizeof(uint16_t)); 
      si->name[len] = '\0';
      si->death.func = (void*) svcinfo_death;
      si->death.ptr = si;
      si->allow_isolated = allow_isolated;
      // 新增保存svclist链表中
      si->next = svclist; 
      svclist = si;
  }

  // 以 BC_ACQUIRE 命令，handle 为目标的信息，通过 ioctl 发送给 binder 驱动
  binder_acquire(bs, handle);
  // 以 BC_REQUEST_DEATH_NOTIFICATION 命令的信息，通过 ioctl 发送给 binder 驱动，主要用于清理内存等收尾工作
  binder_link_to_death(bs, handle, &si->death);
  return 0;
}

struct svcinfo *find_svc(const uint16_t *s16, size_t len)
{
  struct svcinfo *si;

  for (si = svclist; si; si = si->next) {
      //在svclist中查找名字完全一致，则返回查询到的结果
      if ((len == si->len) &&
          !memcmp(s16, si->name, len * sizeof(uint16_t))) {
          return si;
      }
  }
  return NULL;
}

void binder_send_reply(struct binder_state *bs,
                     struct binder_io *reply,
                     binder_uintptr_t buffer_to_free,
                     int status)
{
  struct {
      uint32_t cmd_free;
      binder_uintptr_t buffer;
      uint32_t cmd_reply;
      struct binder_transaction_data txn;
  } __attribute__((packed)) data;
  
  data.cmd_free = BC_FREE_BUFFER;
  data.buffer = buffer_to_free;
  // 回复命令
  data.cmd_reply = BC_REPLY;
  data.txn.target.ptr = 0;
  data.txn.cookie = 0;
  data.txn.code = 0;
  if (status) {
      ...
  } else {
      data.txn.flags = 0;
      data.txn.data_size = reply->data - reply->data0;
      data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
      data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
      data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
  }
  binder_write(bs, &data, sizeof(data));
}

  ```
在注册服务中，先通过`BR_TRANSACTION`命令执行回调函数`svcmgr_handler`,然后增加一个`svcinfo`结构体记录这新建目标的名字和handle值等到链表中保存，然后向Binder Kernel中发送`BC_ACQUIRE`、`BC_REQUEST_DEATH_NOTIFICATION `、`BC_REPLY `命令。

### ServiceManager存储服务的结构体
  ```
struct svcinfo
{
    struct svcinfo *next;
    uint32_t handle; // 服务的handle
    struct binder_death death;
    int allow_isolated;
    size_t len;
    uint16_t name[0]; //服务的名字
};
  ```



### 总结
ServiceManager进程首先打开binder设备和mmap内存映射并保存相关信息，然后ServiceManager申请成为binder机制的上下文管理者。最后先告诉Binder进入循环模式，再循环等待client请求和解析client请求，服务被保存在一个链表中，靠名字和长度来识别服务，如果查询服务则返回其handler值，如果增加service则成功时返回0。



