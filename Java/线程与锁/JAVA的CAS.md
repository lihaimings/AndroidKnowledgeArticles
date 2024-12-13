## CAS的概念
 CAS的全称为：*CompareAndSwap,直译为对比和交换。**

CAS实际是普遍处理器都支持的一条指令，这条指令通过判断当前内存值V、旧的预期值A、即将更新的值B是否相等来对比并设置新值，从而实现变量的**原子性**。

通过前面文章：[线程的三特性](https://www.jianshu.com/p/e5b582d26533)，可以通过**Synchronized**来实现原子性，但是Synchronized在执行为重量级锁时就会进行线程阻塞，如果在很多线程的情况下，就会造成资源成本的增加。而**CAS**的出现就是让线程不阻塞，通过一直自旋的方式进行等待。

Synchronized会线程阻塞称为**悲观锁**，CAS不会使线程阻塞称为**乐观锁**。悲观锁其他没有获取锁的线程是不会执行代码的，而乐观锁是可以使多个线程同时访问代码，可是会判断是否有更新决定是否重新执行。

## CAS的实现原理
CAS的原理：**通过三个参数，当前内存的变量值V、旧的预期值A、即将更新的值B。通过判断是V和A是否相等查看当前变量值是否被其他线程改变，如果相等则变量没有被其他线程更改，就把B值赋予V；如果不相等则做自旋操作。**

举例：假设i的初始值为0，现两线程分别执行i++操作，看看CAS会如何执行：
>1线程：A = 0,B = 1
2线程：A = 0,B = 1

此时两个线程的旧的期望值A、更新的B值都是一样的
 假设1线程先执行，线程1从内存中拿到i的值V,此时V等于0，刚好和旧的期望值A相等，就把更新的值B赋值到内存i值V中。
2线程执行时，此时拿到内存的V值为1，和旧的预期值0不想等，则不做更新B的赋值操作，通过自旋把旧的预期值A=V，并再次确定CAS指令。

![CAS原理流程图](https://upload-images.jianshu.io/upload_images/22650779-4cb840fafba1143b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

JDK提供的原子操作类就是基于CAS来实现原子性，比如：**AtomicInteger、AtomicIntegerArray、AtomicDouble、AtomicBoolean等**

那么就查看`AtomicInteger`的相关代码，看看它是如何实现`i++`这个操作的。
  ```
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    private static final long VALUE;

    static {
        try {
            VALUE = U.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (ReflectiveOperationException e) {
            throw new Error(e);
        }
    }

    private volatile int value;

    // 相当于i++操作
    public final int getAndIncrement() {
        return U.getAndAddInt(this, VALUE, 1);
    }

  ```
这里用到了`sun.misc.Unsafe`这个类，这个类的作用是进行自旋操作和调用本地CAS方法。
`value`用来存储变量；`VALUE`是value变量在内存中的偏移量valueOffset。
在`getAndIncrement()`方法相当于执行保证原子性的执行value++，这个方法调用了`Unsafe.getAndAddInt(this, VALUE, 1)`,说明具体的实现在Unsafe这个类中。

  ```
public final class Unsafe {

    public native int getIntVolatile(Object var1, long var2);
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));  //自旋

        return var5;
    }
}
  ```
在`getAndAddInt`方法中，一共有三个方法参数，var1是AtomicInteger，var2是AtomicInteger的value值在内存偏移量，var4是要更新的增量值。

var5通过`getIntVolatile(var1, var2)`方法获取，是一个native方法，其目的是获取 var1 在 var2 偏移量的值。

通过`while(!CAS())`判断var1的value值和var5相等，则把var5 + var4赋值给value,并且CAS()返回true结束循环，反之通过自旋的方式继续操作。

## CAS带来的问题

### ABA问题
一个线程把变量从A->B再变成A,这时另一个线程执行CAS时会认为这个变量没有被修改过还是原来的A，这就是造成了ABA问题。

针对ABA的问题，JDK也提供了**AtomicStampedReference、AtomicMarkableReference**通过版本号、标记符来解决ABA问题：
AtomicStampedReference通过内置静态类Pair来标记版本号
  ```

    private static class Pair<T> {
        final T reference;
        final int stamp;

        private Pair(T var1, int var2) {
            this.reference = var1;
            this.stamp = var2;
        }

        static <T> AtomicStampedReference.Pair<T> of(T var0, int var1) {
            return new AtomicStampedReference.Pair(var0, var1);
        }
    }

  ```
`reference`为原子操作的变量，`stamp`记录版本号。在更改变量后要随之更改版本号，这样就可以确定ABA的问题是否被更改过，AtomicMarkableReference则是同理用boolean来标记。
AtomicStampedReference的常用方法：
  ```
//构造方法, 传入引用和戳
public AtomicStampedReference(V initialRef, int initialStamp)
//返回引用
public V getReference()
//返回版本戳
public int getStamp()
//如果当前引用 等于 预期值并且 当前版本戳等于预期版本戳, 将更新新的引用和新的版本戳到内存
public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp)
//如果当前引用 等于 预期引用, 将更新新的版本戳到内存
public boolean attemptStamp(V expectedReference, int newStamp)
//设置当前引用的新引用和版本戳
public void set(V newReference, int newStamp) 
  ```


### 自旋问题
就是如果有很多线程，cas()方法的while循环会一直执行，如果长时间的执行就会造成资源的浪费。

## 总结
在解决线程的原子性上，如果你是对代码块进行同步操作，那么可以使用`Synchronized`进行同步操作，如果仅仅是对一个变量的原子操作，那么就可以使用如：`AtomicInteger`等实现过CAS的原子变量。
`CAS`通过内存值、期待值、新值来判断变量是否被其他线程更改。更新失败的线程通过自旋操作，直到更新完成为止。CAS也由此带来了两个问题一个是ABA问题、一个是自旋时间问题。

