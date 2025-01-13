## 什么是泛型？
**泛型是一种参数化类型，将操作的数据类型(非基本类型)作为一个参数，这种参数类型可以在类、接口、方法中创建，分别叫泛型类、泛型接口、泛型方法。泛型是java层定义的语法，在JVM中并没有泛型这一概念，所以在代码编译阶段通过`类型擦除`把泛型变成基本类型。**

## 为什么要使用泛型？
1. 在泛型之前，只能通过Object进行参数的任意化，在这种情况使用Object参数时需要我们的强制类型转换，这种手动强制转换可能会导致转换成不兼容的其他类型，而此时代码、编译并不会报错，只能在运行时才能发现强转的错误。
2. 泛型则是在代码的编译期间，通过**类型擦除**在编译器中将泛型进行转换，如果发现跟现有的类型不兼容，则会报错。这降低了通过Object的强制转换的带来的错误。
3. 使用泛型，可以提高代码的利用率，使不同的数据类型执行相同的算法。

## 类型参数命名约定
类型参数是单个大写字母，通过字母的规范可以使用大家快速理解类型参数的作用：
T:type(类型)
E:element(元素，集合中就用这个)
K:key(键)
V:value(值)
S、U、V: 第二、三、四个类型

## 泛型接口、泛型类
OrderedPair泛型类实现泛型接口：
  ```
public interface Pair<K, V> {
    K getKey();

    V getValue();
}

public class OrderedPair<K, V> implements Pair<K, V> {

    private K key;
    private V value;

    public OrderedPair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    @Override
    public K getKey() {
        return key;
    }

    @Override
    public V getValue() {
        return value;
    }
}

public class OrderedPair2 implements Pair<String, String> {

    @Override
    public String getKey() {
        return "hello";
    }

    @Override
    public String getValue() {
        return "world";
    }
}

  ```
OrderedPair泛型类的实例类：
  ```
  Pair<String, Integer> p1 = new OrderedPair<String, Integer>("Event", 8);
Pair<String, String> p2 = new OrderedPair<>("hello", "world");
 
  ```

## 原始类型
原始类型简单理解为：在泛型类中，实例化泛型类时并未给T指定特定的数据类型，如下代码：
  ```
public class Box<T>

Box box = new Box();

  ```
这里Box的实例并没有给T指明具体的数据类型，像这种就叫**原始类型**，泛型类型是可以向后兼容原始类型的，比如泛型类型可以赋值给原型类型：
  ```
  Box<String> box = new Box();
  Box box1 = box;
  ```

## 泛型方法
如果在类和接口中没有定义想要的泛型参数，又想在方法中使用泛型参数，则只能如下面中定义泛型方法：
  ```
public class Util {
    public static <K, V> boolean compare(Pair<K, V> p1, Pair<K, V> p2) {
        return p1.getKey().equals(p2.getKey()) &&
                p1.getValue().equals(p2.getValue());
    }
}

 Pair<String, Integer> p1 = new OrderedPair<String, Integer>("Event", 8);
  Pair<String, Integer> p2 = new OrderedPair<>("hello", 9);       
 boolean isEquals = Util.compare(p1, p2);
  ```

## 限定类型和多重限定
如果我们想限定类型参数T，只能是Number类型及其子类，改怎么限定呢？我们可以使用extend,表明数据类型的上限：如下实例
  ```
public class Box<E> {

    private E element;

    public E getValue() {
        return element;
    }

    public <U extends Number> void inject(U u) {
        System.out.println("E: " + element.getClass().getName());
        System.out.println("U: " + u.getClass().getName());
    }
}

 Box<String> box = new Box();
 box.inject(10);
// error   box.inject("10");
  ```
上面的泛型方向限定为Number及其子类(Integer、Float、Double等)。
当然泛型的一个参数类型也可以有**多重限定**，不过这个多重限定只能是一个类和多个接口，而且类限定并且放在第一位：
  ```
public class A { }
public interface B { }
public interface C { }


public class D<T extends A & B & C>
  ```
这就是多重限定，只能是限定一个类且并且放第一位。

## 继承与泛型
如果在已经确定T的数据类型后(如Number)，后面在赋值T时可以是确定类型的子类(如Interger)。如下事例：
  ```
Box<Number> box = new Box<>();
box.setElement(new Integer(10));
box.setElement(new Double(10.1));
  ```
当然根据java的多态我们知道，父类也可以拥有子类的实例类，如果类之间有继承的关系，也可以有多态的关系，如
  ```
 ArrayList->List->Collection

List<String> list = new ArrayList<>();

ArrayList<String> arrayList = new ArrayList<>();
List<String> list1 = arrayList;
  ```
可以看到，只要泛型的数据类型一样，类依然可以有多态关系。
但是单单泛型的数据类型有继承关系，而类没有则不能构成多态关系，如下代码会报错：
  ```
Box<Integer> box1 = new Box<>();
Box<Number> box2 = box1;  // 报错
  ```
遇到这种有没有办法解决，是有的，我们把下面的Number的数据类型改为`? extends Number`以下就可以赋值了：
  ```

Box<Integer> box1 = new Box<>();
Box<? extends Number> box2 = box1;
  ```

## ？通配符
**？通配符**并不是泛型，？不能像泛型T一样形成泛型类、泛型接口、泛型方法。**？通配符只是一个实例T的一个参数**，比如`? extends Number`和`Number`是一样的，**不过通配符可以更加灵活规定参数的上下限。**

**上限通配符**
`? extends`即叫上限通配符，可以确定参数是其类型及其子类型。**上限通配符是支持`协变`的，则可以使没有父子关系的类完成多态的赋值**。以下是用List<T>确定T参数是一个Number的上限通配符，完成元素的增加：
  ```
    public static double sumOfList(List<? extends Number> list) {
        double sum = 0.0;
        for (Number number : list) {
            sum += number.doubleValue();
        }
        return sum;
    }


    List<Integer> list2 = Arrays.asList(1, 2, 3);
    System.out.println("sum = " + sumOfList(list2));

  ```
上限通配符是支持协变的即：
  ```
Box<Integer> box1 = new Box<>();
Box<? extends Number> box2 = box1;

  ```

**下限通配符**
`? spuer`即叫下限通配符，可以确定参数的类型及其父类型。下限通配符则支持`逆变`，跟上边的协变是一个道理：
  ```
    public static void addNumbers(List<? super Integer> list) {
        for (int i = 1; i <= 10; i++) {
            list.add(i);
        }
    }

    List<Integer> list2 = Arrays.asList(1, 2, 3);
    addNumbers(list2);

    // 逆变
    Box<Number> box3 = new Box<>();
    Box<? super Integer> box4 = box3;
  ```
## 通配符上下限使用时机
一般生产数据，有变化的数据用：super
用于阅读，使用的数据用：extends
比如实现一个`copy`方法：
  ```
    private void copy(List<? extends Number> list, List<? super Number> list1) {
        list1.addAll(list);
    }
  ```
## 类型擦除
泛型只是java层定义的语法，在JVM中并没有泛型这一概念。所以泛型在编译的时候通过类型擦除变成JVM可以识别的数据类型。一般的类型擦除把泛型变成Object或限定类型，如：
T：Object
T extends Number: Number


## Kotlin泛型

> kotlin的泛型只是给java的泛型的概念换了一种写法，所以对java泛型不熟悉的可以看上面的文章

## 泛型类、泛型接口
通过泛型类OrderedPair去实现泛型接口：
  ```
interface Pair<K, V> {
    var key: K

    var value: V

    fun generate(): K
}

class OrderedPair<K, V>(override var key: K, override var value: V) : Pair<K, V> {

    override fun generate(): K {
        return key
    }

}

  ```
OrderedPair实例类的写法：
  ```
val p1: OrderedPair<String, Int> = OrderedPair("Event", 8)
 val p2 = OrderedPair("hello", "world")

  ```

## 泛型方法
如果在类和接口中没有定义想要的泛型参数，又想在方法中使用泛型参数，则只能如下面中定义泛型方法：
  ```
class Util {
    companion object {
        fun <K, V> compare(p1: Pair<K, V>, p2: Pair<K, V>): Boolean {
            return p1.key == p2.key
                    && p1.value == p2.value
        }
    }

}


val p1: OrderedPair<String, Int> = OrderedPair("Event", 8)
val p2 = OrderedPair("hello", 10)
var isEquals = Util.compare(p1, p2)
  ```

## 限定类型
如果我们想限定类型参数T，只能是Number类型及其子类，改怎么限定呢？在java中我们可以使用extend,在kotlin中使用`：`
  ```
class Box<E> {
    var element: E? = null

    fun <U : Number> inject(u: U) {
        print("u= ${u}")
    }
}

 var box: Box<String> = Box()
box.inject(10)
  ```

## out上限通配符
在java中`? extends`叫上限通配符，而在kotlin中`out`叫上限通配符，可以确定参数是其类型及其子类型。**上限通配符是支持`协变`的，则可以使没有父子关系的类完成多态的赋值。**以下是用List<T>确定T参数是一个Number的上限通配符，完成元素的增加：
  ```
fun sumOfList(list: List<out Number>): Double {
        var sum: Double = 0.0
        for (number in list) {
            sum += number.toDouble()
        }
        return sum
    }


val list = Arrays.asList(1, 2, 3)
val sum = sumOfList(list)
  ```
out上限通配符是支持协变的即：
  ```
val box3: Box<Int> = Box()
val box4: Box<out Number> = box3 
 ```
kolint在out上的扩展，在java中通配符?:extends只能在确定类型参数的时候使用，并不可以在泛型类、泛型接口中使用，但out就可以直接在类、接口中直接声明该泛型为上下限：
  ```
interface List<out E> : Collection<E> {
}
  ```

## 下限通配符
在java`? spuer`即叫下限通配符，在kotlin`in`即叫下限通配符,可以确定参数的类型及其父类型。下限通配符则支持`逆变`，跟上边的协变是一个道理：
  ```
fun printBox(box: Box<in Int>) {
        println("number= ${box.element}")
    }

val box5: Box<Number> = Box()
box5.element = 10
printBox(box5)

  ```



## 泛型的实现原理


### **1. 泛型的核心机制：类型擦除（Type Erasure）**

#### **1.1 编译器插入类型检查和强制类型转换**
虽然运行时泛型信息被擦除，但编译器在编译阶段会插入**强制类型转换和类型检查**代码，以确保类型安全。

**示例：**
```java
List<String> list = new ArrayList<>();
list.add("Hello");  // 编译器检查，类型安全
String str = list.get(0); // 编译器自动插入强制类型转换
```
编译后等价于：
```java
List list = new ArrayList();
list.add("Hello");
String str = (String) list.get(0); // 强制类型转换
```

---
  
  **示例：**
  ```java
  List<String> list = new ArrayList<>();
  list.add("Hello");  // 编译通过
  list.add(1);        // 编译报错：不允许添加 Integer 类型
  ```

#### **1.2 类型擦除**
- 泛型类型信息在**运行时被擦除**，编译器会将泛型参数替换为**原始类型（Raw Type）**。
  - 如果没有定义泛型的上界，类型会被擦除为 `Object`。
  - 如果定义了泛型上界（如 `<T extends Number>`），则被替换为上界类型（如 `Number`）。
  
  **示例：**
  ```java
  public class GenericClass<T> {
      private T value;

      public T getValue() {
          return value;
      }

      public void setValue(T value) {
          this.value = value;
      }
  }
  ```
  **编译后：**
  ```java
  public class GenericClass {
      private Object value;

      public Object getValue() {
          return value;
      }

      public void setValue(Object value) {
          this.value = value;
      }
  }
  ```

#### **1.3 类型擦除的影响**
- **运行时无法获取泛型的具体类型**：由于类型信息在编译后被擦除，运行时不能直接通过 `instanceof` 或反射获取泛型参数的具体类型。
  ```java
  List<String> list = new ArrayList<>();
  if (list instanceof List<String>) {  // 编译错误
      System.out.println("This is a List<String>");
  }
  ```

- **桥接方法（Bridge Method）**：为了解决类型擦除带来的方法签名冲突，Java 编译器会生成桥接方法。
  ```java
  class Parent<T> {
      public T getValue() {
          return null;
      }
  }

  class Child extends Parent<String> {
      @Override
      public String getValue() {
          return "Child";
      }
  }
  ```
  **编译后：**
  ```java
  public class Child extends Parent {
      @Override
      public String getValue() {
          return "Child";
      }

      // 桥接方法
      public Object getValue() {
          return this.getValue();
      }
  }
  ```



- **泛型数组的限制**
由于类型擦除的存在，**Java 不允许直接创建泛型数组**。这是因为泛型的具体类型在运行时不可见，数组需要运行时维护具体的类型信息。

```java
List<String>[] array = new ArrayList<String>[10]; // 编译错误
```

解决方法：
```java
List<?>[] array = new ArrayList<?>[10];
```

### **总结**
Java 泛型的底层实现通过**编译时类型检查**和**运行时类型擦除**平衡了**类型安全性**和**向下兼容性**。尽管带来了运行时类型信息丢失等限制，但泛型在类型安全、代码复用性和可读性上提供了显著的优势。