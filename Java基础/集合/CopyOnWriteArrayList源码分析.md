
## **什么是 CopyOnWriteArrayList？**

`CopyOnWriteArrayList` 是 Java 并发包（`java.util.concurrent`）中的一个线程安全的集合类。它通过**写时复制**（Copy-On-Write）的方式，实现了在多线程环境下的安全读写。

**核心思想**：  
- **读操作**：不加锁，直接读取数据，性能很高。  
- **写操作**：每次修改（增删改）都会复制出一个新数组，在新数组上进行修改，最后用新数组替换旧数组。

因此，`CopyOnWriteArrayList` 更适合**读多写少**的场景，比如缓存数据、配置项、订阅者列表等。

---

## **实现原理**

### **1. 内部数据结构**

`CopyOnWriteArrayList` 的数据存储使用一个 `volatile` 修饰的数组 `array`：

```java
private transient volatile Object[] array;
```

- 使用 `volatile` 保证了多线程间的可见性。
- 读操作直接访问 `array`，写操作在新数组上完成，最后将新数组赋值给 `array`。

---

### **2. 核心源码解析**

#### **（1）读取操作**

读取操作直接返回底层数组的内容，不加锁，效率非常高。比如 `get(int index)` 方法：

```java
public E get(int index) {
    return getArray()[index];
}

final Object[] getArray() {
    return array; // 返回当前的数组引用
}
```

因为底层数组是 `volatile` 修饰的，所以每次读操作都能拿到最新的数据。

---

#### **（2）写入操作**

写入操作（增删改）通过复制原数组来实现。在修改时会创建一个新数组，然后用新数组替换旧数组。以 `add(E e)` 方法为例：

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 加锁，确保只有一个线程能修改
    try {
        Object[] elements = getArray(); // 获取当前数组
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1); // 复制原数组并扩容
        newElements[len] = e; // 在新数组末尾添加元素
        setArray(newElements); // 用新数组替换旧数组
        return true;
    } finally {
        lock.unlock(); // 释放锁
    }
}
```

这里用到了一个可重入锁 `ReentrantLock` 来保证写操作的互斥性。

---

#### **（3）删除操作**

删除操作也需要先复制数组，然后在新数组上完成删除操作。比如 `remove(int index)`：

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = (E) elements[index]; // 获取要删除的元素
        int numMoved = len - index - 1;
        Object[] newElements = new Object[len - 1]; // 创建新数组
        System.arraycopy(elements, 0, newElements, 0, index); // 复制前半部分
        if (numMoved > 0) {
            System.arraycopy(elements, index + 1, newElements, index, numMoved); // 复制后半部分
        }
        setArray(newElements); // 替换旧数组
        return oldValue; // 返回被删除的元素
    } finally {
        lock.unlock(); // 释放锁
    }
}
```

删除操作的核心也是**复制数组后替换引用**。

---

#### **（4）写时复制的核心方法**

核心方法是 `setArray(Object[] newArray)`，它将新数组赋值给 `array`：

```java
final void setArray(Object[] a) {
    array = a; // 用新数组替换旧数组
}
```

因为 `array` 是 `volatile` 修饰的，替换后其他线程可以立刻看到最新的数据。

---

## **使用场景**

1. **读多写少的场景**：  
   比如缓存系统、订阅者列表、配置信息等，读操作非常频繁，但写操作较少。

2. **多线程读写**：  
   在多线程环境下，`CopyOnWriteArrayList` 无需对读操作加锁，可以提供非常高的读性能。

3. **不可变场景**：  
   写时复制的特性也非常适合需要创建不可变快照的场景，比如遍历的同时修改。

---

## **性能差异**

### **优点**
1. **读性能高**：读操作不需要加锁，直接访问底层数组，非常快。
2. **线程安全**：写操作通过锁机制和写时复制，保证了线程安全性。
3. **无锁读取**：适合读多写少的场景，写操作不影响读操作。

### **缺点**
1. **写性能低**：每次写操作都需要复制整个数组，代价很高，尤其是当数组很大时。
2. **内存消耗大**：每次写操作都会创建一个新数组，旧数组需要等垃圾回收，内存占用较多。
3. **不适合写多场景**：频繁写入会导致性能瓶颈。

---

## **总结**

`CopyOnWriteArrayList` 是一个为**读多写少场景**而优化的线程安全集合。通过写时复制和 `volatile` 机制，完美解决了读操作与写操作的线程安全问题。尽管写操作性能较低，但它在特定场景（如缓存、订阅者列表等）中非常有用。

如果你的业务场景读操作居多，而写操作较少，可以大胆选择 `CopyOnWriteArrayList`；但如果写操作频繁，建议选择其他线程安全的集合（如 `ConcurrentHashMap` 或使用显式同步的 `ArrayList`）。