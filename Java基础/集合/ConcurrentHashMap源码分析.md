
## 1. **ConcurrentHashMap 是啥？**

`ConcurrentHashMap` 是 Java 提供的一种线程安全的哈希表实现，属于 **JUC**（Java 并发工具包）的一部分。从名字就能看出，它是为了解决并发环境下哈希表线程安全的问题。

特点可以总结为：
1. **线程安全**：支持多线程并发读写，避免数据竞争。
2. **高性能**：通过分段锁（JDK 1.7）或 CAS+同步机制（JDK 1.8）实现锁粒度的优化，性能远超传统的 `Hashtable`。
3. **不允许 `null` 键和值**：和 `Hashtable` 一样，不允许存储 `null` 键或 `null` 值。

---

## 2. **实现原理（JDK 1.7 和 JDK 1.8 的区别）**

### **2.1 JDK 1.7：分段锁实现**

JDK 1.7 的 `ConcurrentHashMap` 采用的是 **分段锁**（`Segment`）的思想。简单来说，把整个哈希表分成若干段，每段独立加锁，减少锁的竞争。

1. **核心结构**：
   - `Segment[]`：分段锁的数组，每个 `Segment` 维护一个独立的小哈希表，内部用链表解决哈希冲突。
   - 每次操作，只锁定相关的 `Segment`，而不是整个哈希表。

2. **流程**：
   - **读操作**：直接访问，不加锁（乐观读）。
   - **写操作**：只锁定写入的那一段，保证线程安全。

虽然分段锁提高了并发性能，但问题也很明显，比如内存占用高、维护复杂等。

---

### **2.2 JDK 1.8：CAS+同步机制**

到了 JDK 1.8，`ConcurrentHashMap` 改用 **CAS（Compare-And-Swap）+ Synchronized** 的机制，彻底丢掉了 `Segment`，结构变得更简单，性能更高。

1. **核心结构**：
   - **Node[] table**：底层是一个 `Node` 数组，类似于 `HashMap` 的结构。
   - **链表+红黑树**：用链表解决冲突，当链表长度超过 8 时，转为红黑树，提高效率。

2. **并发控制**：
   - **CAS**：通过 CAS 操作实现无锁更新，比如初始化桶或插入新值。
   - **Synchronized**：当链表/树操作复杂时，退化为 `synchronized` 锁定整个桶。

3. **为什么比 1.7 更快？**
   - 锁粒度更细，只锁定冲突的桶，而不是整个段。
   - 减少内存占用，结构更简洁。

---

## 3. **源码分析**

接下来我们详细分析下 JDK 1.8 的 `ConcurrentHashMap` 核心源码，包括初始化、插入、扩容等。

### **3.1 初始化**

`ConcurrentHashMap` 的初始化是懒加载的，只有第一次插入时才会真正分配空间。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; 
    int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0) // 说明其他线程正在初始化
            Thread.yield(); 
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 用 CAS 抢占初始化权限
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sizeCtl = n - (n >>> 2); // 设置扩容阈值（默认 0.75）
                }
            } finally {
                sizeCtl = sc; // 恢复状态
            }
            break;
        }
    }
    return tab;
}
```

**源码解析**：
1. 初始化是线程安全的，用 CAS 保证只有一个线程能成功。
2. `sizeCtl` 表示容量控制标志，初始化时为负数，表明正在初始化。

---

### **3.2 插入操作**

插入的核心方法是 `putVal`。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException(); // 禁止 null 键或值
    int hash = spread(key.hashCode()); // 计算哈希值
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0) 
            tab = initTable(); // 懒初始化
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) 
                break; // 用 CAS 插入新节点
        }
        else if ((fh = f.hash) == MOVED) 
            tab = helpTransfer(tab, f); // 说明正在扩容
        else { 
            V oldVal = null;
            synchronized (f) { // 锁定桶
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { 
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value; // 更新值
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null); // 链表插入新节点
                                break;
                            }
                        }
                    } else if (f instanceof TreeBin) 
                        oldVal = ((TreeBin<K,V>)f).putTreeVal(hash, key, value);
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) 
                    treeifyBin(tab, i); // 链表转为红黑树
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount); // 更新元素计数，判断是否需要扩容
    return null;
}
```

**流程解析**：
1. **懒初始化**：如果哈希表还没初始化，先初始化。
2. **CAS 插入**：如果目标桶是空的，用 CAS 插入新节点。
3. **链表操作**：如果发生哈希冲突，用 `synchronized` 锁定桶，链表尾插。
4. **红黑树转换**：当链表长度超过 8 时，转为红黑树。
5. **扩容检查**：插入完成后，更新元素计数，判断是否需要扩容。

---

### **3.3 扩容**

扩容的逻辑和 `HashMap` 类似，但引入了并发控制。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length;
    for (int i = 0; i < n; i++) {
        Node<K,V> f = tabAt(tab, i);
        if (f == null) 
            continue;
        synchronized (f) { // 锁定桶
            // 数据迁移到新表
        }
    }
}
```

**并发扩容特点**：
1. 分段迁移，多个线程可以同时进行扩容，减少阻塞。
2. 每次只迁移部分数据，避免扩容卡住整个系统。

---

## 4. **使用场景**

`ConcurrentHashMap` 在以下场景中非常适用：
- 高并发的键值对存储，比如缓存系统、共享配置等。
- 需要频繁读写的场景。

---

## 5. **性能差异**

| 特性                  | `Hashtable`      | `HashMap`         | `ConcurrentHashMap`  |
|-----------------------|------------------|-------------------|----------------------|
| 线程安全              | 是               | 否                | 是                  |
| 锁机制               | 方法级锁         | 无                | CAS+部分锁          |
| 并发性能             | 低               | 不支持            | 高                  |
| 是否允许 `null` 键值 | 否               | 是                | 否                  |

---

## 6. **常见问题**

1. **ConcurrentHashMap 和 Hashtable 的区别？**
   - `ConcurrentHashMap` 使用更细粒度的锁，性能更高。
   - `ConcurrentHashMap` 不锁整个表，只锁冲突的桶。

2. **JDK 1.7 和 JDK 1.8 的实现区别？**
   - 1.7 用分段锁，1.8 改用 CAS+同步机制。

3. **为什么 ConcurrentHashMap 不允许 `null` 键值？**
   - 避免 `null` 和不存在的值混淆，简化并发处理。

