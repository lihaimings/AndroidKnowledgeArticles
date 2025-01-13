## **1. LinkedList 介绍**

`LinkedList` 是 Java 集合框架中基于 **双向链表** 实现的 `List`，它既实现了 `List` 接口，又实现了 `Deque`（双端队列）接口。它的特点是：
1. **有序**：插入顺序就是存储顺序。
2. **可重复**：可以存储重复元素。
3. **支持双向链表操作**：可以从头或尾快速插入、删除。
4. **线程不安全**：需要手动同步。

和 `ArrayList` 相比，`LinkedList` 更适合频繁插入、删除的场景。我们会详细对比性能差异。

---

## **2. LinkedList 的内部结构**

### **双向链表的基础**
`LinkedList` 的底层是一个典型的双向链表，每个节点都包含以下信息：
1. `prev`：指向前一个节点。
2. `next`：指向后一个节点。
3. `item`：存储当前节点的值。

看下 `Node` 类的源码：

```java
private static class Node<E> {
    E item;       // 当前节点存储的值
    Node<E> next; // 指向下一个节点
    Node<E> prev; // 指向上一个节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

而 `LinkedList` 本身通过 `first` 和 `last` 两个指针分别指向链表的头和尾。

```java
transient Node<E> first; // 头结点
transient Node<E> last;  // 尾结点
```

---

## **3. 源码解析**

### **3.1 构造方法**

`LinkedList` 的构造方法很简单，基本就是初始化一个空链表。

```java
public LinkedList() {
    first = null; // 初始时链表为空
    last = null;
    size = 0;     // 长度为 0
}
```

---

### **3.2 添加元素**

#### **add(E e) 方法**

把元素加到链表的尾部：

```java
public boolean add(E e) {
    linkLast(e); // 调用 linkLast 方法
    return true;
}
```

#### **linkLast(E e) 方法**

这个方法会把一个新节点挂到链表的尾部：

```java
void linkLast(E e) {
    final Node<E> l = last; // 取到当前尾节点
    final Node<E> newNode = new Node<>(l, e, null); // 新节点的前驱是当前尾节点，后继是 null
    last = newNode; // 更新尾节点为新节点
    if (l == null) { 
        first = newNode; // 如果链表为空，直接把新节点设为头节点
    } else {
        l.next = newNode; // 否则，把旧尾节点的 next 指向新节点
    }
    size++; // 链表长度 +1
    modCount++; // 修改计数器 +1（用于迭代器的快速失败机制）
}
```

---

### **3.3 删除元素**

删除操作分两种：删除头节点和尾节点。

#### **removeFirst()**

删除头节点的源码如下：

```java
public E removeFirst() {
    final Node<E> f = first; // 取到当前头节点
    if (f == null) {
        throw new NoSuchElementException(); // 如果头节点为空，抛异常
    }
    return unlinkFirst(f); // 调用 unlinkFirst 处理
}

private E unlinkFirst(Node<E> f) {
    final E element = f.item; // 取到头节点存储的值
    final Node<E> next = f.next; // 取到下一个节点
    f.item = null; // 断开头节点
    f.next = null;
    first = next; // 更新头节点为原头节点的下一个
    if (next == null) {
        last = null; // 如果链表为空，尾节点也置空
    } else {
        next.prev = null; // 否则把新头节点的 prev 置空
    }
    size--; // 链表长度 -1
    modCount++; // 修改计数器 +1
    return element; // 返回删除的值
}
```

#### **removeLast()**

删除尾节点的逻辑类似，就不详细贴代码了。

---

### **3.4 查找元素**

根据索引查找元素的方法是 `get(int index)`：

```java
public E get(int index) {
    checkElementIndex(index); // 检查索引是否合法
    return node(index).item; // 找到对应的节点并返回值
}
```

查找的核心是 `node(int index)` 方法：

```java
Node<E> node(int index) {
    if (index < (size >> 1)) { // 如果索引在前半部分，从头向后遍历
        Node<E> x = first;
        for (int i = 0; i < index; i++) {
            x = x.next;
        }
        return x;
    } else { // 如果索引在后半部分，从尾向前遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) {
            x = x.prev;
        }
        return x;
    }
}
```

---

## **4. 使用场景**

### **适合的场景**
1. **频繁插入和删除**：尤其是在头部或尾部，时间复杂度是 O(1)。
2. **需要双端队列的功能**：`LinkedList` 支持双端队列，可以快速从两头操作。
3. **数据量较小**：因为链表存储需要额外的空间（`prev` 和 `next` 指针），所以数据量太大会浪费内存。

### **不适合的场景**
1. **频繁随机访问**：链表查找元素需要从头或尾开始遍历，时间复杂度是 O(n)，性能不如 `ArrayList`。
2. **需要频繁排序**：链表的结构不利于排序操作。

---

## **5. 性能分析**

| 操作          | 时间复杂度 | 描述                                      |
|---------------|------------|-------------------------------------------|
| 随机访问       | O(n)       | 需要从头或尾开始遍历链表。                 |
| 插入（头尾）   | O(1)       | 头尾插入不需要移动其他元素。               |
| 插入（中间）   | O(n)       | 找到插入位置需要遍历链表。                 |
| 删除（头尾）   | O(1)       | 头尾删除直接断开节点即可。                 |
| 删除（中间）   | O(n)       | 找到删除位置需要遍历链表。                 |

---

## **6. ArrayList vs LinkedList：怎么选？**

| 特性                | ArrayList                         | LinkedList                         |
|---------------------|-----------------------------------|------------------------------------|
| 底层结构            | 动态数组                         | 双向链表                          |
| 随机访问            | 快，O(1)                         | 慢，O(n)                          |
| 插入/删除（头尾）    | 慢，O(n)                         | 快，O(1)                          |
| 插入/删除（中间）    | 慢，O(n)                         | 慢，O(n)                          |
| 空间占用            | 少（只存数据）                   | 多（存指针）                      |

简单来说：
- 如果需要频繁随机访问，选 `ArrayList`。
- 如果插入、删除特别多，选 `LinkedList`。
- 如果数据量小，性能差异不大，可以随便选。

---

## **7. 总结**

`LinkedList` 是个非常灵活的集合类，特别适合频繁插入和删除的场景。不过它的缺点也很明显，比如随机访问性能较差，内存占用较高。

平时开发中，如果不确定用哪个集合，优先考虑 `ArrayList`，它的使用场景更广。如果明确需要链表的特性，比如双端队列或频繁的插入删除操作，`LinkedList` 会是更好的选择。

希望这篇文章能帮助你更好地理解 `LinkedList` 的实现原理和源码！