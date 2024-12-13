## 1. 概述
Android的启动优化主要是加速用户打开App到可交互的时间。在这段时间里面经历的Application的启动创建，SplashActivity、MainActivity的启动创建(有些App没有Splash页面)。

- Application的创建过程的调用顺序大致如下：

![](https://upload-images.jianshu.io/upload_images/22650779-3acf8c2fd73d8b50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/940)

- Activity的创建到第一帧显示过程调用顺序如下：

![](https://upload-images.jianshu.io/upload_images/22650779-4b26d005951d5aee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/940)
****
从视觉交互来说，App的启动过程主要显示了3屏：
- 第一屏：在主题中设置`android:windowBackground`的图片，这个过程在**Application.attachBaseContext()——Splash.onWindowFocusChanged()**之间。
- 第二屏：Splash绘制的内容，这个过程在**Splash.onWindowFocusChanged()——Main.onWindowFocusChanged()**之间。
- 第三屏：Main绘制的内容，这个在**Main.onWindowFocusChanged**之后。

所以，启动的优化范围一般在**Application. attachBaseContext ()——Main.onWindowFocusChanged()**之间。其中第一屏和第二屏是的耗时是需要重点关注。这里把第一屏的时间叫做Application的启动耗时，第一屏+第二屏的时间Application启动到可交互页面的启动耗时。

## 2. 启动优化步骤
### 2.1 评估时间数据
在启动优化中，并不是自己觉得哪个地方会耗时，就开始做优化，这样可能带来辛辛苦苦做了一顿优化后，总耗时还是没有好的效果，原因可能是我们直觉耗时的地方并不准确。正确的做法是`先得到启动过程中每个方法的时间`，得到全部时间表后，分析耗时的地方在哪里记录下来。
那怎么获取到每个方法的执行时间呢？，大致如下：
**1. 自带的Profiler**
配置应用启动时开始profiler监控如下：

![](https://upload-images.jianshu.io/upload_images/22650779-3a09e878d5e3517b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/940)

2. **Looper**
在之前的文章[Android卡顿监控](https://www.jianshu.com/p/4f25e150952c)中有对其的具体实现。

3. **Systrace+函数插桩**


### 2.2 梳理业务
在第一步得到时间数据后，下一步就是整理启动过程的业务梳理，避免因优化而影响业务。

## 2.3 优化业务
优化的思想大致如下：
1. 通过上面的两个步骤，看主线程究竟慢哪里
2. 懒加载，包括业务与布局的懒加载
3. 抛到子线程让其自己加载
4. 提前加载，通过多线程提高效率
5. 检查主线程的IO操作
6. 控制线程的数量和GC的次数

**1. Application.onCreate()**
在这个方法中主要是做SDK的初始化和App状态判断，
sdk优化主要有三点：
一：sdk的懒加载，再使用到的时候才去初始化，不要全部放到在Application初始化。
二：对于sdk有依赖的关系的，比如sdk2需要sdk1完成加载后开始初始化，可以利用拓扑算法优化实现sdk加载:[android-startup](https://github.com/idisfkj/android-startup)
三：把sdk放到子线程中自己初始化，不要阻塞主线程的时间，一些必须初始化的sdk，可以通过多线程加载然后通过CountDownLatch进行阻塞和唤醒。

业务代码的优化：
一：尽量不要滥用ContentProvider,因为其是在Application.attachBaseContext就会初始化其ContentProvider.onCreate()方法，会加长启动时间。
二：查看Application主线程耗时的地方，尽量去优化。

**2. Activity.onCreate()**
要弄清楚哪里耗时了，才着手优化。常见优化有：
一： setContentVIew()的耗时，这时要优化布局，减少布局背景的重复渲染、减少层级、对于不一定显示的布局用ViewStub按需加载。子线程加载布局，或提前子线程加载。
二：initView()初始化View时不要做耗时操作，比如一些IO操作、播放器等做到按需懒加载，也可以通过让其在多线程中加载不阻塞主线程，也可以通过多线程提前加载。
三：主页面的ViewPager+Fragment可以通过懒加载按需加载Fragment

**3. Activity.onResume()**
不要在onResume()方法中不要做主线程耗时操作，因为这时页面还没渲染出来，其还要通过Vsync信号来临做绘制三大流程，最终交给屏幕渲染出来，回调执行onWindowFocusChanged()方法。

**4.检查主线程IO操作**
可以在线下通过`StrictMode`类来检查IO操作，如果在主线程有IO操作会输入日志。
```
   private fun startStrictMode() {
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskWrites()
                    .detectDiskReads()
                    .detectNetwork()
                    .penaltyLog()
                    .build()
            )

            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedClosableObjects()
                    .penaltyLog()
                    .build()
            )

        }
    }
```

## 总结
Android启动优化更多的是一种优化思路，因为每个项目都不一样，导致优化的点也不一样，但掌握了优化思路可以以不变应万变。大致分为三个过程：
-  评估时间数据（知道主线程哪里耗时）
- 梳理业务 （不要改出bug了）
- 优化业务 
又大致分为两步：
[1] SDK的加载优化、耗时操作优化(IO操作等)。
[2] 布局优化：常用的有：减少嵌套(ConstraintLayout、merge)、背景减少重复渲染、按需加载(ViewStub)，还有：**异步加载(AsyncLayoutInflater)和编译时创建View(X2C)**
























