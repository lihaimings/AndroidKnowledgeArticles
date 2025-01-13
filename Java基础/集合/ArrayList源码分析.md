
## **1. ArrayList 简单介绍**

`ArrayList` 是 Java 集合框架中一个非常经典的集合类，它实现了 `List` 接口，特点是：
1. **有序**：存进去的顺序不会乱。
2. **可重复**：允许存储重复的元素。
3. **基于数组**：底层是一个动态数组。
4. **线程不安全**：如果多线程并发修改，需要额外同步。

---

## **2. ArrayList 的内部结构**

我们先来看看 `ArrayList` 的几个核心成员变量：

```java
// 默认初始容量
private static final int DEFAULT_CAPACITY = 10;

// 空数组（在无参数构造函数中用到）
private static final Object[] EMPTY_ELEMENTDATA = {};

// 用来存储元素的数组
transient Object[] elementData;

// 当前 ArrayList 中存储的元素数量
private int size;
```

这里面最重要的就是 `elementData`，它是一个动态数组，所有的元素都存储在这个数组里。`size` 是当前数组的实际大小，用来记录存了多少元素。

---

## **3. 源码分析**

### **3.1 构造方法**

`ArrayList` 提供了三种构造方法，分别是：
1. **无参数构造函数**：
   ```java
   public ArrayList() {
       this.elementData = EMPTY_ELEMENTDATA; // 初始化为空数组
   }
   ```
   初始时 `elementData` 是空的，等第一次插入元素的时候才会扩容到默认的 10。

2. **指定初始容量的构造函数**：
   ```java
   public ArrayList(int initialCapacity) {
       if (initialCapacity > 0) {
           this.elementData = new Object[initialCapacity];
       } else if (initialCapacity == 0) {
           this.elementData = EMPTY_ELEMENTDATA;
       } else {
           throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
       }
   }
   ```
   这个构造方法可以让你自己决定数组的初始大小。

3. **通过一个集合来初始化**：
   ```java
   public ArrayList(Collection<? extends E> c) {
       elementData = c.toArray();
       size = elementData.length;
   }
   ```

---

### **3.2 添加元素**

#### **add(E e) 方法**
这是最常用的方法，用来往数组里添加一个元素。它的源码如下：

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 确保数组容量够用，不够就扩容
    elementData[size++] = e;          // 把元素放到数组里，并更新 size
    return true;
}
```

#### **ensureCapacityInternal(int minCapacity) 方法**
确保容量够用，不够的话就扩容。具体代码：
```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) { 
        // 如果是空数组，扩容到默认容量 10 或最小需要的容量
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity); // 具体检查容量
}
private void ensureExplicitCapacity(int minCapacity) {
    if (minCapacity - elementData.length > 0) {
        grow(minCapacity); // 扩容
    }
}
```

#### **grow(int minCapacity) 方法**
扩容的方法，默认扩容为原来的 1.5 倍：
```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 新容量 = 旧容量 + 旧容量的一半
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity; // 如果 1.5 倍还不够，就直接按最小需求扩容
    }
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        newCapacity = hugeCapacity(minCapacity); // 超出最大容量限制时处理
    }
    elementData = Arrays.copyOf(elementData, newCapacity); // 拷贝到新数组
}
```

#### 总结：
扩容是 `ArrayList` 的核心机制，默认扩容为 1.5 倍，保证动态数组的灵活性和性能。

---

### **3.3 删除元素**

删除元素主要有两种情况：
1. **按索引删除**：`remove(int index)`。
2. **按对象删除**：`remove(Object o)`。

#### **按索引删除**
```java
public E remove(int index) {
    rangeCheck(index); // 检查索引是否合法
    E oldValue = elementData(index); // 取出旧值
    int numMoved = size - index - 1;
    if (numMoved > 0) {
        // 把后面的元素往前移动
        System.arraycopy(elementData, index + 1, elementData, index, numMoved);
    }
    elementData[--size] = null; // 把最后一个元素设为 null，帮助垃圾回收
    return oldValue;
}
```

#### **按对象删除**
```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++) {
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
        }
    } else {
        for (int index = 0; index < size; index++) {
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
        }
    }
    return false;
}
```

删除操作会移动数组中的元素，性能不是特别高。

---

### **3.4 获取元素**

`get(int index)` 方法用于根据索引获取元素：
```java
public E get(int index) {
    rangeCheck(index); // 检查索引合法性
    return elementData(index); // 返回对应的数组值
}
```

---

## **4. 使用场景**

### **适合的场景**
1. **频繁随机访问**：比如获取某个位置的元素，时间复杂度是 O(1)。
2. **数据量不特别大且增长速度适中**：扩容开销相对较小。
3. **读多写少**：读操作性能非常高，但写操作（尤其是插入和删除）会有性能损失。

### **不适合的场景**
1. **频繁插入或删除**：插入和删除会移动大量元素，性能不如 `LinkedList`。
2. **线程不安全的场景**：需要手动同步或者用线程安全的集合，比如 `CopyOnWriteArrayList`。

---

## **5. 性能分析**

| 操作          | 时间复杂度 | 描述                                      |
|---------------|------------|-------------------------------------------|
| 随机访问       | O(1)       | 数组可以通过索引快速定位到元素。            |
| 插入（尾部）   | O(1)       | 添加到末尾无需移动元素（不考虑扩容）。       |
| 插入（中间）   | O(n)       | 插入点之后的元素都需要移动。                |
| 删除（尾部）   | O(1)       | 删除末尾元素无需移动其他元素。              |
| 删除（中间）   | O(n)       | 删除点之后的元素都需要往前移动。            |
| 扩容操作       | O(n)       | 需要将旧数组中的元素拷贝到新数组。          |

---

## **6. 总结**

`ArrayList` 是一个强大的集合类，适用于需要随机访问和动态增长的场景。它的底层实现基于动态数组，通过扩容机制保证了使用的灵活性。

使用的时候，记住以下几点：
1. 如果写操作很多，可以考虑用 `LinkedList` 或其他集合。
2. 在多线程环境下，要注意 `ArrayList` 的线程安全问题。
3. 提前合理设置初始容量，避免频繁扩容带来的性能开销。
