## 1. **什么是 Hashtable？**

`Hashtable` 是 Java 中一个 **基于哈希表** 的键值对存储容器，早在 JDK 1.0 就已经引入。它的主要特点是：
- **线程安全**：通过对所有关键方法加锁，保证了多线程下的安全性。
- **键值都不允许为 `null`**。

---

## 2. **Hashtable 的实现原理**

`Hashtable` 的底层和 `HashMap` 类似，都是基于 **数组 + 链表** 的结构实现的。它将键值对根据哈希值存储在数组的某个位置上，发生冲突时会使用链表解决。

简单来说：
1. 通过键的 `hashCode` 计算哈希值，定位到数组中的某个桶。
2. 如果该桶已被占用，则将新元素加入链表中，形成冲突处理。
3. 支持动态扩容，避免哈希冲突过多导致性能下降。

---

## 3. **核心源码分析**

接下来我们详细看下 `Hashtable` 的核心源码，包括初始化、插入、查找、删除等操作。

### **3.1 初始化**

`Hashtable` 的构造函数初始化了底层数据结构，并设置了初始容量和负载因子。

```java
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
    if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
        throw new IllegalArgumentException("Illegal Load: " + loadFactor);
    }
    this.loadFactor = loadFactor; 
    table = new Entry<?,?>[initialCapacity]; // 初始化数组
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1); // 计算扩容阈值
}
```

**关键点**：
- `table` 是存储数据的数组。
- `loadFactor` 表示负载因子，决定了哈希表的密度（默认值为 0.75）。
- `threshold` 是扩容的临界值，等于 `容量 × 负载因子`。

---

### **3.2 插入操作**

插入的核心方法是 `put`，它会将键值对存储到数组中。如果发生哈希冲突，就用链表解决。

```java
public synchronized V put(K key, V value) {
    if (key == null || value == null) {
        throw new NullPointerException(); // 键值都不允许为 null
    }

    int hash = key.hashCode(); // 计算哈希值
    int index = (hash & 0x7FFFFFFF) % table.length; // 计算桶的位置

    for (Entry<K,V> e = table[index]; e != null; e = e.next) { 
        if ((e.hash == hash) && e.key.equals(key)) { // 找到相同键
            V old = e.value;
            e.value = value; // 替换旧值
            return old;
        }
    }

    addEntry(hash, key, value, index); // 插入新节点
    return null;
}
```

**源码分析**：
1. **计算哈希值**：用 `hashCode()` 得到哈希值，再对数组长度取模，定位到对应的桶。
2. **检查冲突**：遍历桶里的链表，看有没有相同的键，如果有就更新值。
3. **添加新节点**：调用 `addEntry` 方法，在链表的头部插入新节点。

---

### **3.3 查找操作**

查找的核心方法是 `get`，根据键找到对应的值。

```java
public synchronized V get(Object key) {
    int hash = key.hashCode(); // 计算哈希值
    int index = (hash & 0x7FFFFFFF) % table.length; // 定位到桶

    for (Entry<K,V> e = table[index]; e != null; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) { // 找到键
            return e.value; // 返回值
        }
    }
    return null; // 没找到
}
```

**逻辑很简单**：
1. 通过哈希值定位到桶。
2. 遍历链表，匹配键，找到就返回值。

---

### **3.4 删除操作**

删除的核心方法是 `remove`，它会把键值对从哈希表中移除。

```java
public synchronized V remove(Object key) {
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % table.length;

    Entry<K,V> prev = null;
    for (Entry<K,V> e = table[index]; e != null; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) { // 找到要删除的节点
            if (prev != null) {
                prev.next = e.next; // 链表中删除节点
            } else {
                table[index] = e.next; // 链表头删除节点
            }
            count--; // 元素个数减 1
            return e.value; // 返回被删除的值
        }
    }
    return null; // 没找到
}
```

**源码分析**：
1. 定位到桶。
2. 遍历链表，找到要删除的节点。
3. 更新前后节点的指针，断开要删除的节点。

---

### **3.5 扩容机制**

当哈希表的元素个数超过 `threshold` 时，会触发扩容。

```java
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    int newCapacity = (oldCapacity << 1) + 1; // 扩容为原来的两倍+1
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1); // 更新扩容阈值
    table = newMap; // 替换为新的数组

    for (int i = oldCapacity; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i]; old != null; ) {
            Entry<K,V> e = old;
            old = old.next;
            
            int index = (e.hash & 0x7FFFFFFF) % newCapacity; // 重新计算哈希值
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```

---

## 4. **Hashtable 的使用场景**

虽然 `Hashtable` 是线程安全的，但在现代开发中，它的使用场景很有限：
1. **早期多线程环境**：在 JDK 1.0 和 1.1 时，没有更好的选择。
2. **简单场景**：当需要线程安全的哈希表，又不想引入外部同步时。

---

## 5. **Hashtable 和其他集合类的性能差异**

| 特性               | `Hashtable`         | `HashMap`           | `ConcurrentHashMap`   |
|--------------------|---------------------|---------------------|-----------------------|
| 线程安全           | 是                  | 否                  | 是（更高效）          |
| 键值是否允许为 null | 否                  | 是                  | 否                   |
| 扩容性能           | 较低                | 较高                | 较高                 |
| 适用场景           | 简单线程安全场景     | 单线程场景          | 高并发场景            |

---

## 6. **常见问题**

1. **Hashtable 和 HashMap 的区别？**
   - `Hashtable` 是线程安全的，`HashMap` 不是。
   - `Hashtable` 不允许 `null` 键和值，`HashMap` 允许。

2. **Hashtable 的线程安全如何实现的？**
   - 通过 `synchronized` 锁住方法，保证多线程安全。

3. **为什么不推荐使用 Hashtable？**
   - 性能较差。
   - 更推荐用 `ConcurrentHashMap`，它支持高并发。

