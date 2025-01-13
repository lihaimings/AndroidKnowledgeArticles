
## 1. **LinkedHashMap 的实现原理**

1. **`LinkedHashMap` 的继承关系**  
2. **核心字段与构造方法**  
3. **插入操作的实现**  
4. **访问顺序的更新逻辑**  
5. **删除节点的实现**  
6. **`removeEldestEntry` 实现 LRU 缓存**  
7. **遍历的实现逻辑**

---

### 1. **继承关系**
`LinkedHashMap` 继承自 `HashMap`，它实际上是对 `HashMap` 的增强，增加了维护顺序的功能：

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```

- **`HashMap` 提供了键值对的存储逻辑（基于哈希表的结构）。**
- **`LinkedHashMap` 在 `HashMap` 的基础上维护了一个双向链表来实现顺序。**

---

### 2. **核心字段与构造方法**

我们先看看 `LinkedHashMap` 额外添加的字段：

### **核心字段**

```java
transient LinkedHashMap.Entry<K,V> head; // 双向链表的头节点
transient LinkedHashMap.Entry<K,V> tail; // 双向链表的尾节点

final boolean accessOrder; // true 表示按访问顺序排序，false 表示按插入顺序
```

- `head` 和 `tail` 是双向链表的头尾，维护整个链表结构。
- `accessOrder` 用于决定顺序是 **插入顺序** 还是 **访问顺序**。

---

### **构造方法**

看一下构造方法的定义，理解如何初始化：

```java
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor); // 调用父类 HashMap 的构造方法
    this.accessOrder = accessOrder;    // 初始化顺序模式
}
```

- `initialCapacity`：初始容量。
- `loadFactor`：负载因子，决定扩容时机。
- `accessOrder`：`true` 表示按访问顺序排序，`false` 表示按插入顺序排序。

---

### 3. **插入操作的实现**

当我们调用 `put` 方法时，`LinkedHashMap` 会在 `HashMap` 的基础上更新双向链表。

关键代码在 `HashMap` 的 `putVal` 方法中调用了 `afterNodeInsertion`：

```java
void afterNodeInsertion(boolean evict) { 
    // 在插入新节点后，更新双向链表
    LinkedHashMap.Entry<K, V> last = tail;
    tail = (LinkedHashMap.Entry<K, V>) newNode;
    if (last == null) {
        head = tail; // 如果链表为空，头尾都是新节点
    } else {
        last.after = tail; // 维护双向链表指针
        tail.before = last;
    }
}
```

**详细解析**：

- 如果是第一次插入数据（`last == null`），说明链表为空，头尾都指向新节点。
- 如果链表已有数据，则将新节点加到尾部，并更新双向链表指针（`last.after` 和 `tail.before`）。

---

### 4. **访问顺序的更新逻辑**

如果 `LinkedHashMap` 的排序模式是 **访问顺序**（`accessOrder = true`），每次访问节点（通过 `get` 方法或迭代器），会把该节点移到链表尾部。

关键代码在 `afterNodeAccess` 方法：

```java
void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) { 
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
        LinkedHashMap.Entry<K,V> b = p.before;
        LinkedHashMap.Entry<K,V> a = p.after;

        p.after = null; // 将节点从链表中移除
        if (b == null) {
            head = a; // 如果是头节点，更新头节点
        } else {
            b.after = a; // 前节点的后指针更新
        }
        if (a != null) {
            a.before = b; // 后节点的前指针更新
        } else {
            last = b; // 如果是尾节点，更新尾节点
        }

        if (last == null) {
            head = p; // 如果链表为空
        } else {
            p.before = last; // 移到尾部
            last.after = p;
        }
        tail = p; // 更新尾节点
    }
}
```

**详细解析**：

1. 检查是否启用了访问顺序（`accessOrder = true`）。
2. 如果访问的节点不是尾节点，将节点从链表中移除。
3. 调整前后节点的指针，移除该节点。
4. 将节点插入到链表的尾部。

---

### 5. **删除节点的实现**

当我们调用 `remove` 方法时，`LinkedHashMap` 会从链表中移除对应的节点。

删除节点的逻辑在 `afterNodeRemoval`：

```java
void afterNodeRemoval(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
    LinkedHashMap.Entry<K,V> b = p.before;
    LinkedHashMap.Entry<K,V> a = p.after;

    p.before = p.after = null; // 清除当前节点的前后指针
    if (b == null) {
        head = a; // 如果是头节点，更新头节点
    } else {
        b.after = a; // 更新前节点的后指针
    }
    if (a == null) {
        tail = b; // 如果是尾节点，更新尾节点
    } else {
        a.before = b; // 更新后节点的前指针
    }
}
```

**详细解析**：

1. 如果节点是头节点，更新链表的头指针。
2. 如果节点是尾节点，更新链表的尾指针。
3. 断开当前节点与前后节点的链接。

---

### 6. **`removeEldestEntry` 实现 LRU 缓存**

`LinkedHashMap` 提供了一个钩子方法 `removeEldestEntry`，可以用来移除最老的节点。默认实现返回 `false`，我们可以通过子类重写它：

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false; // 默认不移除
}
```

### **实现 LRU 缓存**
如下代码实现一个固定大小为 5 的 LRU 缓存：

```java
LinkedHashMap<Integer, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, String> eldest) {
        return size() > 5; // 超过大小时移除最老的节点
    }
};
```

---

### 7. **遍历逻辑的实现**

`LinkedHashMap` 的遍历逻辑基于双向链表，它保证了有序性。以下是 `entrySet` 的迭代器实现：

```java
final class LinkedEntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
    public Map.Entry<K,V> next() {
        LinkedHashMap.Entry<K,V> e = next;
        next = e.after; // 迭代器移动到链表的下一个节点
        return e;
    }
}
```

每次迭代时，都会根据双向链表的 `after` 指针获取下一个节点。

---


## 2. **使用场景和性能差异**

### **2.1 使用场景**

1. **需要有序的 Map**：
   - 如果你需要按插入顺序存储键值对，比如日志记录。
2. **实现 LRU 缓存**：
   - 使用 `accessOrder = true` 和 `removeEldestEntry`，轻松搞定 LRU 缓存。
3. **需要一致的遍历顺序**：
   - 和 `HashMap` 不同，`LinkedHashMap` 的遍历顺序是固定的，适合需要稳定顺序的场景。

---

### **2.2 性能差异**

| 操作         | `HashMap`     | `LinkedHashMap` |
|--------------|---------------|-----------------|
| 插入性能     | O(1)          | O(1)（稍慢，因为维护了链表） |
| 查询性能     | O(1)          | O(1)           |
| 移除性能     | O(1)          | O(1)（稍慢，因为需要更新链表指针） |
| 内存开销     | 较低          | 较高（因为多了两个指针） |



### **Q1: LinkedHashMap 和 HashMap 的区别？**

- **有序性**：`HashMap` 无序，`LinkedHashMap` 有序（插入顺序或访问顺序）。
- **数据结构**：`HashMap` 是数组 + 单向链表/红黑树，`LinkedHashMap` 是数组 + 双向链表。
- **内存开销**：`LinkedHashMap` 比 `HashMap` 多两个指针。

---

### **Q2: LinkedHashMap 是线程安全吗？**

不是线程安全的。可以用 `Collections.synchronizedMap()` 包装成线程安全的版本。

---

### **Q3: 如何用 LinkedHashMap 实现 LRU 缓存？**

通过设置 `accessOrder = true` 和重写 `removeEldestEntry` 方法：

```java
LinkedHashMap<Integer, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, String> eldest) {
        return size() > 10; // 超过 10 个元素时移除最老的
    }
};
```

---

### **Q4: 为什么 LinkedHashMap 的遍历是有序的？**

因为它在每个节点上维护了一个双向链表，然后通过头尾指针，保证了插入顺序或访问顺序。
