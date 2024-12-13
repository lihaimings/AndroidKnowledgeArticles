# Android开机启动流程
如图1所示，是Android开机启动大致流程，其中流程大致为`加载BootLoader -> 启动Linux系统内核-> 创建Init进程(native层-> framework -> app)`。
![图1 来自Gityuan.com](https://upload-images.jianshu.io/upload_images/22650779-3922069c4d4849d0.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

本文章将重点讲解Init进程的启动流程，其中Init进程的终点则是**创建解析文件的子进程，并且守护这个子进程(进行重启)**。先从整体在布局分析

![图2 开机启动流程](https://upload-images.jianshu.io/upload_images/22650779-75017fa448315846.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## Android开机启动Linux内核
如上面的图1所思，在Android开机时，先ROM加载程序预定的程序BootLoader,然后BootLoader会去检查RAM、初始化硬件参数等就会启动Linux内核。
当Linux Kernel内核启动后，就会初始化各种硬件环境、加载驱动程序等。当Linux内核加载完成后，就会启动用户空间的第一个进程Init进程。这里的Linux内核层称为内核空间，native层以上则称为用户空间。所以`Init进程是所有用户进程的父进程`。

## Init进程的启动过程
本篇代码基于Android6.0源码分析(只要会一个版本，其他版本的基本也能看懂)，线上源码阅读地址：[http://androidxref.com/](http://androidxref.com/), Init进程代码目录主要在:
  ```
/system/core/init/Init.cpp
/system/core/rootdir/init.rc
/system/core/init/init_parser.cpp
/system/core/init/builtins.cpp
/system/core/init/signal_handler.cpp
  ```
Init进程是用户进程的第一个进程，其pid = 1，其主要用来初始化和启动属性服务，创建子进程如Zygote进程等。在Linux内核加载完成后，第一件事就是创建Init进程，我们先看Init进程的创建后执行的第一个main()函数
`/system/core/init/init.cpp`
  ```
int main(int argc, char**argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }
    //清理umask
    umask(0);
    if (is_first_stage) {
         // 创建和挂载启动所需的文件目录
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        mount("proc", "/proc", "proc", 0, NULL);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
    }
    // 初始化Kernel的Log,可以获取Kernel的Log
    klog_init();
    if (!is_first_stage) {
        // 1.对属性服务进行初始化，分配空间
        property_init();
    }
    // 2. 处理异常退出的子进程，会把子进程的信号全部清除，并重启该子进程定义的重启服务
    signal_handler_init();
    // 3. 启动属性服务
    start_property_service();
    // 4. 解析init.rc配置文件
    init_parse_config_file("/init.rc");
 
    while (true) {
        if (!waiting_for_exec) {
            execute_one_command();
            // 5. 重启死去的子进程
            restart_processes();
        }
    }
    return 0;
}

  ```
在main()函数通过精简代码中，首先创建和挂载所需要的文件目录，这些都是系统运行时的目录。然后则是我们重点分析的场景：
1. 注释1中调用property_init进行**属性服务的初始化**，在注释3中调用start_property_service来**启动属性服务**。
2. 在注释2中通过signal_handler_init进行**子进程的信号处理**；
3. 在注释4中**解析init.rc的配置脚本文件**，在这个脚本文件中说明了要创建启动什么进程；
4. 在注释5中，restart_processes则循环查看**重启守护子进程**。

### 1. 子进程异常处理
调用`signal_handler_init`进行子进程信号量的处理，子进程在暂停或退出时会发出一个终止的SIGCHLD信号，而signal_handler_init函数则对这个信号进行监听，并对这个子进程的信息进行删除重置，并执行子进程在脚本文件的配置的onrestart的服务。
signal_handler_init函数的实现主要在：`/system/core/init/signal_handler.cpp`
  ```
void signal_handler_init() {
    int s[ 2];
    // 创建 socket pair
    if (socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0, s) == -
        ERROR("socketpair failed: %s\n", strerror(errno));
        exit(1);
    }
    signal_write_fd = s[0];
    signal_read_fd = s[1];
    //如果捕获到SIGCHLD信号则写入signal_write_fd
    struct sigaction act;
    memset( & act, 0, sizeof(act));
    act.sa_handler = SIGCHLD_handler;
    // SA_NOCLDSTOP事Init进程在子进程终止时才捕获SIGCHLD
    act.sa_flags = SA_NOCLDSTOP;
    sigaction(SIGCHLD, & act, 0);
     // 判断并对终止的进程做处理
    reap_any_outstanding_children();
    // 注册signal_read_fd状态进行回调执行handle_signal函数，handle_signal本质还是执行上面的reap_any_outstanding_children()方法
    register_epoll_handler(signal_read_fd, handle_signal);
}

/system/core/init/init.cpp
  void register_epoll_handler(int fd, void (*fn)())
    {
        epoll_event ev;
        ev.events = EPOLLIN;
        ev.data.ptr = reinterpret_cast < void*>(fn);
         // 将fd的可读事件加入到epoll_fd的监听队列中，并回调相应函数
        if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, & ev) ==-1){
        ERROR("epoll_ctl failed: %s\n", strerror(errno));
    }
    }

  ```
接下来，我们看看`handle_signal`回调函数，对于发出终止信号的子进程怎么处理？
  ```
static void handle_signal() {
    // Clear outstanding requests.
    // 把signal_read_fd写入buf
    char buf[ 32];
    read(signal_read_fd, buf, sizeof(buf));
    // 重点执行此函数
    reap_any_outstanding_children();
}

static void reap_any_outstanding_children() {
   // 通过wait_for_one_process()函数，遍历寻找终止的子进程
    while (wait_for_one_process()) {
    }
}

// 查看是否为退出进程，如果是则移除属性，并启动配置文件的onrestart的服务。
static bool wait_for_one_process() {
    int status;
     // 根据pid判断是否为退出进程，没退出则pid=0
    pid_t pid = TEMP_FAILURE_RETRY(waitpid(-1, & status, WNOHANG));
    if (pid == 0) {
        return false;
    } else if (pid == -1) {
        ERROR("waitpid failed: %s\n", strerror(errno));
        return false;
    }
     // 执行到这里说明是退出进程
    // 根据pid找到相应的service配置脚本文件
    service * svc = service_find_by_pid(pid);
    std::string name;
    // 如果没有配置文件则直接退出
    if (!svc) {
        return true;
    }
   
    //当flags为RESTART，且不是ONESHOT时，先kill进程组内所有的子进程或子线程
    if (!(svc -> flags & SVC_ONESHOT) || (svc -> flags & SVC_RESTART)) {
        NOTICE("Service '%s' (pid %d) killing any children in process group\n", svc -> name, pi
        kill(-pid, SIGKILL);
    }

     //移除当前服务svc中的所有创建过的socket
    for (socketinfo * si = svc -> sockets; si; si = si -> next) {
        char tmp[ 128];
        snprintf(tmp, sizeof(tmp), ANDROID_SOCKET_DIR"/%s", si -> name);
        unlink(tmp);
    }
     //当flags为EXEC时，释放相应的服务
    if (svc -> flags & SVC_EXEC) {
        INFO("SVC_EXEC pid %d finished...\n", svc -> pid);
        waiting_for_exec = false;
        list_remove( & svc -> slist);
        free(svc -> name);
        free(svc);
        return true;
    }
    
    //执行serive配置文件的onrestart启动相应的服务
    struct listnode*node;
    list_for_each(node, & svc -> onrestart.commands){
        command * cmd = node_to_item(node, struct command, clist);
        cmd -> func(cmd -> nargs, cmd -> args);
    }
    svc -> NotifyStateChange("restarting");
    return true;
}

  ```
signal_handler_init对子进程的信号处理就完成了，可以就是对终止子进程的进行注册回调，并移除相关的属性，并启动revice文件的onrestart的配置服务。

### 2. 解析Init.rc
init.rc是非常重要的配置文件，位于`/system/core/rootdir/`目录中。它是由Android初始化语言编写的脚本，[Android初始化语言的学习可以参考这篇文章](https://www.kancloud.cn/androidguy/android-depth/109807)，Android的初始化语言大致分为：： `Action、Command、Service、Option、Import`五种类型，但Init.rc主要分为import的导入、service启动服务、on命令。
  ```
// import导入
import /init.environ.rc
import /init.usb.rc
...

on early-init
    write /proc/1/oom_score_adj -1000
    restorecon /adb_keys
    start ueventd
...
on nonencrypted
    class_start main // 启动classname为mian的服务
    class_start late_start

...

service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm

service surfaceflinger /system/bin/surfaceflinger
service media /system/bin/mediaserver
service installd /system/bin/installd

  ```
在`on`中有`class_start main `其意思是启动classname为mian的服务，在Init进程中会启动一个很重要的Zygote子进程，接下来分析一些Zygote服务脚本的定义，Zygote的service服务脚本并不在Init.rc中，而是自己定义了独自的Zygote服务脚本文件，位于`/system/core/rootdir/init.zygote64.rc`
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
这里的脚本是启动一个名叫zygote的进程，并且执行的路径是/system/bin/app_process64，后面则是参数。`class main`：是这个进程的classname叫做main，`onrestart`是在上面的子进程信号处理时会进行重启的进程。所以在Init.rc文件中会启动zygote进程。

在Init.rc文件中，除了启动`zygote`进程之外，还会启动一些如`servicemanager、surfaceflinger 、media、installd `等重要的进程。

那么这些进程创建的过程又是如何的呢？，请看下一节的启动解析的服务。

### 3. 启动解析的服务
在配置的脚本文件中通过`class_start main`去启动一个ClassName为main的服务，其中class_start中对应的启动函数为`do_class_start`,位于/system/core/init/builtins.cpp目录中。
  ```
    int do_class_start(int nargs, char **args)
    {
        service_for_each_class(args[1], service_start_if_not_disabled);
        return 0;
    }

     // 遍历寻找相同名字的classname，然后执行参数函数，也就是执行上面的service_start_if_not_disabled
    void service_for_each_class(const char *classname,
    void (*func)(struct service *svc))
    {
        struct listnode * node;
        struct service * svc;
        list_for_each(node, & service_list) {
        svc = node_to_item(node, struct service, slist);
        if (!strcmp(svc->classname, classname)) {
        func(svc);
    }
    }
    }


    static void service_start_if_not_disabled(struct service *svc)
    {
        if (!(svc->flags & SVC_DISABLED)) {
        // 只要没有设置 disabled 选项 就执行此函数创建进程
        service_start(svc, NULL);
    } else { svc ->
        flags | = SVC_DISABLED_START;
    }
    }

void service_start(struct service *svc, const char *dynamic_args)
{
    ...
    // fork 创建了一个子进程
    pid_t pid = fork();
    if (pid == 0) {
        // 代表子进程，子进程会继承父进程的所有资源，可简单理解为读时共享写事复制
        ...
        // 启动子进程
        execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
    }
}
  ```
所以通过配置的service的文件，通过`fork()`去创建一个子进程，并且子进程的pid=0，如果不是则是父进程。并调用execve()函数启动子进程。

### 4.  初始化和启动属性服务
属性服务类似一个注册表，填入到这个注册表的属性，在系统或软件重启时，还会对这个注册表的属性进行初始化的加载。
在init.cpp文件中main()函数，就通过这两个函数分别进行初始化和启动
> property_init()
start_property_service()

**property_init**

  ```
    void property_init()
    {
        if (property_area_initialized) {
            return;
        }

        property_area_initialized = true;

        if (__system_property_area_init()) {
            return;
        }

        pa_workspace.size = 0;
        pa_workspace.fd = open(PROP_FILENAME, O_RDONLY | O_NOFOLLOW | O_CLOEXEC);
        if (pa_workspace.fd == -1) {
            ERROR("Failed to open %s: %s\n", PROP_FILENAME, strerror(errno));
            return;
        }
    }
  ```
property_init()函数主要做的事情就是通过`__system_property_area_init`方法进行内存的创建。

**start_property_service**
  ```
    void start_property_service()
    {
        // 创建socket
        property_set_fd =
            create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
        0666, 0, 0, NULL);
        if (property_set_fd == -1) {
            ERROR("start_property_service socket creation failed: %s\n", strerror(errno));
            exit(1);
        }

        listen(property_set_fd, 8);
         // 对property_set_fd进行监听，并调用handle_property_set_fd回调函数
        register_epoll_handler(property_set_fd, handle_property_set_fd);
    }
  ```


### 5. 守护解析的服务
最后，通过while循环，当有子进程终止时，让让可以重新启动的子进程重新启动
  ```
    while (true)
    {
        if (!waiting_for_exec) {
            execute_one_command();
            restart_processes();

        }
    }

    static void restart_processes()
    {
        process_needs_restart = 0;
        service_for_each_flags(
            SVC_RESTARTING,
            restart_service_if_needed
        );
    }

    // 遍历寻找，并回调函数参数
    void service_for_each_flags(unsigned matchflags,
    void (*func)(struct service *svc))
    {
        struct listnode * node;
        struct service * svc;
        list_for_each(node, & service_list) {
        svc = node_to_item(node, struct service, slist);
        if (svc->flags & matchflags) {
        func(svc);
    }
    }
    }


    static void restart_service_if_needed(struct service *svc)
    {
        time_t next_start_time = svc->time_started+5;

        if (next_start_time <= gettime()) { svc ->
            flags & = (~SVC_RESTARTING);
            // 重新启动进程
            service_start(svc, NULL);
            return;
        }

        if ((next_start_time < process_needs_restart) ||
            (process_needs_restart == 0)
        ) {
            process_needs_restart = next_start_time;
        }
    }

  ```

## 总结
Init进程的创建过程，主要就是通过解析`Init.rc`文件去创建一些重要的子进程比如Zagote、surfaceflinger进程，它的启动是用`fork、exec`去启动子进程，然后就是对子进程的终止进行监听回调，对于一些可以重启的服务进行重启和创建进程。



