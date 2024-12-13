# Android系统架构
android系统架构从上到下分为五层：**应用层、应用框架层、系统运行库层、硬件抽象层、Linux内核层**。如图1所示：

![图1 Android系统框架](https://upload-images.jianshu.io/upload_images/22650779-f403e0c8fbe5ebc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 应用层
应用层就是App应用程序，这里包括了系统级内嵌的App和常规开发人员的非系统级的App，如图二所示。

![图二 应用层](https://upload-images.jianshu.io/upload_images/22650779-6fbe49bb56cc5c82.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

系统级的App主要位于系统源码的**packages**包里面，如图三是packages包的目录：

![图三 packages目录](https://upload-images.jianshu.io/upload_images/22650779-b01668178ad4bd70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 应用框架层（Java API Framework）
应用框架层为开发应用程序提供常规的API，开发人员可以通过应用框架层开发应用，也可以称为Java Framework，除了给上层应用调用API，另一方面也跟C/C++程序库和硬件抽象层交互,如图四所示，

![图四  应用框架层](https://upload-images.jianshu.io/upload_images/22650779-081b7feed4f1a08e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这一层中提供了很多的管理器，如ActivityManger、LocationManager、PackageManager等，如图五是应用框架层提供的组件。

![图五  应用框架层组件](https://upload-images.jianshu.io/upload_images/22650779-e5503554e8432a7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

应用框架层的代码主要位于**framework**包的/base目录里，如图六是base目录的介绍：

![图六  framework/base目录](https://upload-images.jianshu.io/upload_images/22650779-743a7649cd43d094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 系统运行库层（native层）
系统运行库层分为两部分：**C/C++程序库和运行时库，其中运行时库又分为两部分：虚拟机和核心库**，如图七所示。C/C++程序库主要给framework调用给开发人员提供功能。虚拟机有Dalvik和ART，一个是即时编译，一个AOT预编译。其代码主要位于系统源码的/framework，/art, /dalvik 包下。

![图七 native层](https://upload-images.jianshu.io/upload_images/22650779-5dd990790d0525ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 硬件抽象层
硬件抽象层位于操作系统内核和硬件电路的接口层，主要是把硬件抽象分离出来，使操作系统不能直接操作硬件，也使硬件与平台无关性。代码位于系统源码的 hardware包中。

## Linux内核层
Android的核心系统服务用Linux核心，也增加了Android专用的驱动。
最后这五层的架构就很明了，我们在看一下Android的系统架构图：

![ Android系统框架](https://upload-images.jianshu.io/upload_images/22650779-f403e0c8fbe5ebc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)