# Android & Java 知识点汇总

欢迎来到我的 **Android & Java 开发知识库文章**！这里是我总结的各个技术点，涵盖了 **Java基础**、**Android基础**、**Android框架**、**性能优化** 和 **Android系统理解** 等内容。每个分类下面都有相关的文章。

![Android Knowledge](https://img.shields.io/badge/Android%20Knowledge-Articles-blue)

---

## 📚 文章目录

| 分类                     | 文章链接                           |
|------------------------|----------------------------------|
| **Java基础**             | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Java)                   |
| - 集合                   | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Java/%E9%9B%86%E5%90%88)                      |
| - 线程与锁               | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Java/%E7%BA%BF%E7%A8%8B%E4%B8%8E%E9%94%81)                   |
| - JVM                    | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Java/JVM)                       |
| - 泛型注解反射            | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Java/%E6%B3%9B%E5%9E%8B%E6%B3%A8%E8%A7%A3%E5%8F%8D%E5%B0%84)               |
| **Android基础**          | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E5%9F%BA%E7%A1%80)                |
| **Android框架**          | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E6%A1%86%E6%9E%B6)                |
| **性能优化**             | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)                   |
| **Android系统理解**      | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3)            |
| - 系统启动进程           | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B)               |
| - 系统服务               | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1)                   |
| - 四大组件               | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6)                   |
| - UI体系                 | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/UI%E4%BD%93%E7%B3%BB)                     |
| - 进程通信 (Binder)      | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1(Binder))            |
| - 线程通信 (Handler)     | [查看](https://github.com/lihaimings/AndroidKnowledgeArticles/tree/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%BA%BF%E7%A8%8B%E9%80%9A%E4%BF%A1(Handler))           |

---

## 💻 Java基础
这一部分包含 Java 开发中的基础知识，包括**集合、线程与锁、JVM、泛型注解反射**。

### 🔹 集合
- 📄 [HashMap源码](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E9%9B%86%E5%90%88/HashMap%E6%BA%90%E7%A0%81.md)

### 🔹 线程与锁
- 📄 [线程和进程的区别](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E7%BA%BF%E7%A8%8B%E4%B8%8E%E9%94%81/%E7%BA%BF%E7%A8%8B%E5%92%8C%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%8C%BA%E5%88%AB.md)
- 📄 [线程的原子可见可序性](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E7%BA%BF%E7%A8%8B%E4%B8%8E%E9%94%81/%E7%BA%BF%E7%A8%8B%E7%9A%84%E5%8E%9F%E5%AD%90%E5%8F%AF%E8%A7%81%E5%8F%AF%E5%BA%8F%E6%80%A7.md)
- 📄 [JAVA的线程池ThreadPoolExecutor](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E7%BA%BF%E7%A8%8B%E4%B8%8E%E9%94%81/JAVA%E7%9A%84%E7%BA%BF%E7%A8%8B%E6%B1%A0ThreadPoolExecutor.md)
- 📄 [JAVA的CAS](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E7%BA%BF%E7%A8%8B%E4%B8%8E%E9%94%81/JAVA%E7%9A%84CAS.md)


### 🔹 JVM
- 📄 [[JVM入门指南01]内存区域与溢出异常](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/JVM/%5BJVM%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%9701%5D%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8.md)
- 📄 [[JVM入门指南02]GC垃圾回收机制](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/JVM/%5BJVM%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%9702%5DGC%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E6%9C%BA%E5%88%B6.md)
- 📄 [[JVM入门指南03]类加载和Android虚拟机](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/JVM/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%92%8CAndroid%E8%99%9A%E6%8B%9F%E6%9C%BA.md)

### 🔹 泛型注解反射
- 📄 [Java和Kotlin泛型](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E6%B3%9B%E5%9E%8B%E6%B3%A8%E8%A7%A3%E5%8F%8D%E5%B0%84/Java%E5%92%8CKotlin%E6%B3%9B%E5%9E%8B.md)
- 📄 [Java的注解](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E6%B3%9B%E5%9E%8B%E6%B3%A8%E8%A7%A3%E5%8F%8D%E5%B0%84/Java%E7%9A%84%E6%B3%A8%E8%A7%A3.md)
- 📄 [Java的反射](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Java%E5%9F%BA%E7%A1%80/%E6%B3%9B%E5%9E%8B%E6%B3%A8%E8%A7%A3%E5%8F%8D%E5%B0%84/Java%E7%9A%84%E5%8F%8D%E5%B0%84.md)

---

## 📱 Android基础
这一部分包含了 Android 开发的基础知识。

- 📄 [Android四大组件和启动模式](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E5%9F%BA%E7%A1%80/Android%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E5%92%8C%E5%90%AF%E5%8A%A8%E6%A8%A1%E5%BC%8F.md)

---
## ⚙️ Android框架
这部分讲解 Android 框架的核心组件、架构设计，第三方开源库。

- 📄 [MVVM + Jetpack框架的使用](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E6%A1%86%E6%9E%B6/MVVM%20%2B%20Jetpack%E6%A1%86%E6%9E%B6%E7%9A%84%E4%BD%BF%E7%94%A8.md)
- 📄 [Glide源码分析](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E6%A1%86%E6%9E%B6/Glide%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
- 📄 [OkHttp源码分析](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E6%A1%86%E6%9E%B6/OkHttp%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)
- 📄 [Rxjava源码分析](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E6%A1%86%E6%9E%B6/Rxjava%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

--- 
## 🚀 性能优化
讲解 Android 应用性能优化的技巧，包括启动、布局、内存优化、流畅度优化等。

- 📄 [Android的ANR原理分析](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/Android%E7%9A%84ANR%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.md)
- 📄 [Android卡顿监控](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/Android%E5%8D%A1%E9%A1%BF%E7%9B%91%E6%8E%A7.md)
- 📄 [Android启动和布局优化](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/Android%E5%90%AF%E5%8A%A8%E5%92%8C%E5%B8%83%E5%B1%80%E4%BC%98%E5%8C%96.md)
- 📄 [Android内存泄漏监控](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/Android%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E7%9B%91%E6%8E%A7.md)
- 📄 [Android包体积优化](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/Android%E5%8C%85%E4%BD%93%E7%A7%AF%E4%BC%98%E5%8C%96.md)

--- 

## 🔧 Android系统理解
这部分深入探讨 Android 系统的机制，包括：**系统启动进程、系统服务、四大组件、UI体系、进程通信(Binder)、线程通信(Handler)**。

### 🔹 系统启动进程
- 📄 [Android系统框架](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B/Android%E7%B3%BB%E7%BB%9F%E6%A1%86%E6%9E%B6.md)
- 📄 [Android系统启动-Init进程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B/Android%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8-Init%E8%BF%9B%E7%A8%8B.md)
- 📄 [Android系统启动-Zygote进程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B/Android%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8-Zygote%E8%BF%9B%E7%A8%8B.md)
- 📄 [Android系统启动-SystemServer进程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B/Android%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8-SystemServer%E8%BF%9B%E7%A8%8B.md)
- 📄 [Android系统启动-Launcher进程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B/Android%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8-Launcher%E8%BF%9B%E7%A8%8B.md)

### 🔹 系统服务
- 📄 [ActivityManagerService的启动流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1/AMS/ActivityManagerService%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)
- 📄 [PackageManagerService启动流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1/PMS/PackageManagerService%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)
- 📄 [WindowManagerService的启动](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1/WMS/WindowManagerService%E7%9A%84%E5%90%AF%E5%8A%A8.md)
- 📄 [WindowManagerService之窗口的创建](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1/WMS/WindowManagerService%E4%B9%8B%E7%AA%97%E5%8F%A3%E7%9A%84%E5%88%9B%E5%BB%BA.md)

### 🔹 四大组件
- 📄 [Activity的启动流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6/Activity%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)
- 📄 [Service的启动流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6/Service%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)
- 📄 [BroadcastReceiver的启动流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6/BroadcastReceiver%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)
- 📄 [ontentProvider的启动流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6/ContentProvider%E7%9A%84%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)
- 📄 [App进程的启动过程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E8%BF%9B%E7%A8%8B/App%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.md)

### 🔹 UI体系
- 📄 [view的绘制流程](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/UI%E4%BD%93%E7%B3%BB/view%E7%9A%84%E7%BB%98%E5%88%B6%E6%B5%81%E7%A8%8B.md)
- 📄 [事件拦截分发原理分析](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/UI%E4%BD%93%E7%B3%BB/%E4%BA%8B%E4%BB%B6%E6%8B%A6%E6%88%AA%E5%88%86%E5%8F%91%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.md)

### 🔹 进程通信 (Binder)
- 📄 [Binder 应用层-AIDL原理](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1(Binder)/Binder%20%E5%BA%94%E7%94%A8%E5%B1%82-AIDL%E5%8E%9F%E7%90%86.md)
- 📄 [Binder Framework层—注册和查询服务](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1(Binder)/Binder%20Framework%E5%B1%82%E2%80%94%E6%B3%A8%E5%86%8C%E5%92%8C%E6%9F%A5%E8%AF%A2%E6%9C%8D%E5%8A%A1.md)
- 📄 [Binder Native层—注册:查询服务](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1(Binder)/Binder%20Native%E5%B1%82%E2%80%94%E6%B3%A8%E5%86%8C%3A%E6%9F%A5%E8%AF%A2%E6%9C%8D%E5%8A%A1.md)
- 📄 [Binder Native层—服务管理(ServiceManager进程)](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1(Binder)/Binder%20Native%E5%B1%82%E2%80%94%E6%9C%8D%E5%8A%A1%E7%AE%A1%E7%90%86(ServiceManager%E8%BF%9B%E7%A8%8B).md)
- 📄 [Binder Kernel层—Binder内核驱动](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1(Binder)/Binder%20Kernel%E5%B1%82%E2%80%94Binder%E5%86%85%E6%A0%B8%E9%A9%B1%E5%8A%A8.md)

### 🔹 线程通信 (Handler)
- 📄 [Handler消息机制解析](https://github.com/lihaimings/AndroidKnowledgeArticles/blob/main/Android%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3/%E7%BA%BF%E7%A8%8B%E9%80%9A%E4%BF%A1(Handler)/Handler%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6%E8%A7%A3%E6%9E%90.md)

---

## 💬 联系我

- **简书博客**：[简书博客链接](https://www.jianshu.com/u/8367a9f79122)
- **GitHub**：[GitHub 项目链接](https://github.com/lihaimings/AndroidKnowledgeArticles)

---

### 🖋️ 小贴士
- 📅 本库会持续更新，欢迎 Star 或 Fork，让我们一起进步！



