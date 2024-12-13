## JVM的整体架构
![JVM架构](https://upload-images.jianshu.io/upload_images/22650779-12bae22e3c0efc26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**ClassLoader**:——负责加载已被编译的java文件(.class),验证连接。分配和初始化静态变量和静态代码。

**运行时数据区**：——负责所有的程序数据：堆、方法区、栈。往期关于运行时数据区的介绍文章：[[JVM入门指南01]内存区域与溢出异常](https://www.jianshu.com/p/14c1895af3cb)

**执行引擎**：——执行我们编译和加载的代码并清理产生垃圾的区域。关于垃圾回收清理，可以看往期的垃圾回收介绍文章：[[JVM入门指南02]GC垃圾回收机制
](https://www.jianshu.com/p/3e49acd0d875)

关于`运行时数据区`和`垃圾回收机制`在前面两篇文章中已经讲过，所以这篇主要讲`类加载`和 `执行引擎的执行`。

## 类加载ClassLoader
### Java类加载机制
java的类加载主要是把对.Class文件字节码的检查和生成Class对象。过程有：**加载、验证、准备、解析、初始化**。其中验证、准备、解析称为连接，则过程有：**加载、连接、初始化**。
**加载**：根据类的全限定名读取字节码，并生成相应的Class对象。
**连接**：
1. 验证： 对字节码的验证
2. 准备：给静态变量分配内存，并给一个默认值(还没有赋值)
3. 解析：将常量池中的符号引用替换为直接引用。

**初始化**：初始化阶段就是执行类构造器<clinit>()方法的过程

![java类加载](https://upload-images.jianshu.io/upload_images/22650779-09112a34e1f0d0b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

java类加载中有两个特性：**双亲委托、缓存机制**
**双亲委托**：在加载一个类时，会先递归让父类加载器去尝试加载类，如果父类可以加载则下面的子类加载器则不用加在，如果父类加载器也不可以加载，则递归给子类加载器尝试加载类。
**缓存机制**：加载过的Class都会被缓存起来，当需要使用到某个Class时，会先从缓存中查找该Class，没有才从类加载器中加载。

### Android类加载机制
Android的加载器类的通用父类为`ClassLoader`，其本身有一个内部类`BootClassLoader`用于加载FrameWork层的类，ClassLoader主要实现了`loadClass()`方法用于实现缓存机制和双亲委托机制。`BaseDexClassLoader`同样继承`ClassLoader`，是`PathClassLoader`和`DexClassLoader`的父类，其中BaseDexClassLoader主要是实现了`findClass()`方法用于自己加载dex。PathClassLoader加载App安装目录内的dex，DexClassLoader加载任意位置的dex，
![Android类加载机制关系图](https://upload-images.jianshu.io/upload_images/22650779-bb1dd02677929e8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 执行引擎
### JVM的执行引擎
**Interpreter & JIT**：
这两种解释器是并肩工作的。
`Interpreter`是`解释执行解释器`，解释执行解释器在程序运行时，会把字节码翻译成机器码。解释执行的缺点是当一个方法被重复执行，每一个都需要重新解释执行一遍。
`JIT(Just In Time)`是`即时编译器`，它会把一些经常执行的、大量重复执行的热区代码进行即时编译成机器码并将其更改为本机代码，下次执行热区代码时就可以直接调用本机代码不用再次解释。
![JVM执行引擎解释器](https://upload-images.jianshu.io/upload_images/22650779-3d481ea6fe92848e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Android的执行引擎
Android手机发展以来经历了两个虚拟机：`Dalvik`、`ART`。JVM是按照有无限电量和几乎无限的存储的设备而设计的，但是Android设备则是电量和存储资源都很有限，所以Android设备并没有直接采用JVM来作为虚拟机使用，而是通过规范改造优化后的`Dalvik`，在Android5.0之后更是再次更换改造后的`ART`。

Android的主要改造优化有三点：
1. 运行数据区的栈更改为`寄存器`，减少操作数栈的出入栈操作。
2. JVM虚拟机接收的.class文件更改为`dex文件`。java/kotlin经过java/kotlin编译器后编译成.class文件，这些.class文件通过dex编译器打包编译成dex文件。dex文件的执行效率更高，需要的空间更少。
![Android的打包](https://upload-images.jianshu.io/upload_images/22650779-75e8f9a2a8cfac0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. 执行引擎除了有Interpreter和JIT编译器，在ART中还有`AOT`编译器。



### Dalvik的执行引擎
Dalvik虚拟机的执行引擎和JVM的执行引擎一样，都是一般代码在运行时通过`解释执行解释器`编译，热区代码进行`即时编译器`编译。但是Dalvik在Android5.0之后就不再使用了。
![Dalvik的执行引擎](https://upload-images.jianshu.io/upload_images/22650779-2d25de65534ad77f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ART的执行引擎
在Android5.0以上，安卓改用了ART作为虚拟机，ART虚拟机中新增了一个**AOT编译器**，在应用安装的时候，AOT编译器将  `dex`文件编译为一个`.oat`二进制文件，App运行时直接执行`.oat`文件，不用再编译文件。这样做，使App的运行速度更快，但也带来了两个问题：1. 安装应用的时间久，2. 内存占用较大。
![ART](https://upload-images.jianshu.io/upload_images/22650779-4dbad5d7c2de1d62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ART的执行引擎后优化为：**Interpreter、JIT、AOT一起执行**：
1. 如果没有`.ort`文件，程序第一次执行使用Interpreter解释执行解释器执行。
2. 遇到热区代码，则用JIT解释器执行，并把其存储在一个Profils文件缓存中
3. 设置在空闲的时刻，启动AOT解释器把Profils文件缓存的代码执行为.ort文件，下次再执行的时候，则执行.ort文件。

![ART优化](https://upload-images.jianshu.io/upload_images/22650779-9289fd9140054629.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Android的.dex编译
这块不属于Android的执行引擎部分，但本人是学Android的，所以就顺带讲讲。

![dex文件编译过程](https://upload-images.jianshu.io/upload_images/22650779-c34c4050f0abd9d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过java/kotlin编译器生成的.class文件，还需要经过`desugar`、`proguard`两个步骤：
**desugar**: 俗称脱糖，因为在Dalivk/ART虚拟机中并不会支持那么多的java字节码,对于一个高版本的java的语法字节码，要通过`desugar`将其转换成安卓虚拟机能识别的代码。
**proguard**: 主要做一些混淆操作等。

但这也会带来更长时间的编译，开发人员要等待程序运行起来的时间就越久。然后Google又把`dusugar`和`dex编译器`作为合并优化合为一个**D8**
![D8优化](https://upload-images.jianshu.io/upload_images/22650779-bb903e89a73f4f50.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

后面Google又进一步做了优化，又用**R8**代替了proguard和D8，并对字节码做了优化。
![R8](https://upload-images.jianshu.io/upload_images/22650779-93be3e6d6e14bb4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
proguard和R8的优化有：
1. 去掉无用的类、方法、变量
2. 代码优化，如指令重排
3. 混淆，将类、方法名进行混淆。










