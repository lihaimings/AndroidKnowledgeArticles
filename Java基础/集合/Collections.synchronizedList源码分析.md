## **什么是 Collections.synchronizedList？**

`Collections.synchronizedList` 是 Java 提供的一种线程安全的 `List`。它通过对一个普通的 `List` 加锁，来确保在多线程环境下的安全性。

**使用方法非常简单**：

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
list.add("Hello");
list.add("World");
```

这段代码的核心就在于 `synchronizedList`，它会返回一个线程安全的 `List`。

---

## **实现原理**

`Collections.synchronizedList` 的本质是通过对一个普通 `List` 的方法加锁，来实现线程安全。它使用内部类 `SynchronizedList` 来包装原始的 `List`。

我们来看看源码中的核心实现部分。

### **1. 构造方法**

`Collections.synchronizedList` 的实现非常简单，它通过一个装饰模式，直接返回了 `SynchronizedList` 对象：

```java
public static <T> List<T> synchronizedList(List<T> list) {
    return new SynchronizedList<>(list);
}
```

`SynchronizedList` 是一个静态内部类：

```java
static class SynchronizedList<E> implements List<E>, Serializable {
    final List<E> list;  // 被包装的原始 List
    final Object mutex;  // 锁对象

    SynchronizedList(List<E> list) {
        this.list = Objects.requireNonNull(list);
        this.mutex = this; // 默认以自身作为锁
    }

    SynchronizedList(List<E> list, Object mutex) {
        this.list = Objects.requireNonNull(list);
        this.mutex = mutex; // 支持自定义锁
    }
}
```

**可以看到几点：**
1. 原始 `List` 被传入后，直接被 `list` 字段引用。
2. 使用了一个 `mutex` 锁（默认是 `this`），用来同步所有操作。
3. 支持自定义锁对象，这在某些复杂场景下非常有用。

---

### **2. 加锁操作**

`SynchronizedList` 的每个方法都通过 `synchronized` 关键字对 `mutex` 加锁。以 `add` 方法为例：

```java
public boolean add(E e) {
    synchronized (mutex) { // 加锁
        return list.add(e); // 调用原始 List 的方法
    }
}
```

其他方法也都是类似的，比如 `get`：

```java
public E get(int index) {
    synchronized (mutex) {
        return list.get(index);
    }
}
```

这种方式简单粗暴，但非常有效：通过一个全局锁对象，保证了线程间操作的互斥性。

---

### **3. 遍历操作**

对于遍历操作（比如 `iterator`），`SynchronizedList` 提供了一些额外的同步建议：

```java
public Iterator<E> iterator() {
    return list.iterator(); // 直接返回原始 List 的迭代器
}
```

**注意**：  
这里并没有对 `iterator` 方法加锁，返回的迭代器本身并不是线程安全的。因此，官方文档明确建议在使用迭代器时，手动对整个集合加锁：

```java
synchronized (list) {
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        System.out.println(iterator.next());
    }
}
```

---

### **4. 其他方法**

类似的，`SynchronizedList` 对 `remove`、`clear` 等写操作也都加了同步锁。比如：

```java
public boolean remove(Object o) {
    synchronized (mutex) {
        return list.remove(o);
    }
}

public void clear() {
    synchronized (mutex) {
        list.clear();
    }
}
```

所有这些操作的本质都很简单：用 `synchronized` 包裹原始 `List` 的方法调用。

---

## **使用场景**

`Collections.synchronizedList` 适合以下场景：

1. **多线程环境下的简单集合操作**：  
   如果你的应用中存在多线程访问同一个 `List`，但读写操作相对简单，可以使用 `synchronizedList` 来快速实现线程安全。

2. **轻量级线程安全需求**：  
 

3. **无需复杂并发优化**：  
   对于读写频率不高的场景，`synchronizedList` 的性能是可以接受的。

---

## **性能差异**

### **优点**

1. **简单易用**：  
   `Collections.synchronizedList` 通过装饰器模式，对 `List` 的所有操作都加锁，开发者几乎不需要额外关注线程安全问题。

2. **线程安全**：  
   所有方法都通过一个全局锁来保证线程安全性，适用于多线程读写混合的场景。

3. **支持自定义锁**：  
   如果应用中有更复杂的同步需求，可以自定义 `mutex` 锁。

---

### **缺点**

1. **性能较低**：  
   所有操作都使用全局锁，同一时刻只有一个线程可以访问集合，可能导致性能瓶颈。

2. **遍历操作不安全**：  
   返回的迭代器本身不是线程安全的，需要额外加锁。

3. **不适合读多写少场景**：  
   如果读操作远多于写操作，可以考虑使用 `CopyOnWriteArrayList`，它的读性能会更高。

---

## **与其他线程安全集合的对比**

| 特性                    | `Collections.synchronizedList` | `CopyOnWriteArrayList` | `ConcurrentHashMap` |
|-------------------------|--------------------------------|------------------------|----------------------|
| **实现原理**            | 全局锁                        | 写时复制               | 分段锁或 CAS         |
| **读性能**              | 一般                          | 高                     | 高                   |
| **写性能**              | 一般                          | 较低                   | 高                   |
| **适用场景**            | 读写均衡                      | 读多写少               | 高并发读写           |
| **遍历安全性**          | 手动加锁                      | 安全                   | 安全                 |

---

## **总结**

`Collections.synchronizedList` 是一个经典的线程安全工具，它通过加锁实现了简单有效的线程安全性。在多线程环境下，如果你的需求比较简单，`synchronizedList` 是一个不错的选择。

然而，由于它使用全局锁，性能在高并发场景下可能会成为瓶颈。因此，在读多写少的场景中，可以选择 `CopyOnWriteArrayList`；而在高并发读写的场景中，`ConcurrentHashMap` 或其他高级并发集合会更适合。
