### 1. 定义
java注解是JDK1.5引进的一种技术，注解本身是一种**元数据**，元数据就是描述数据的数据，它对程序本身没有任何的影响。配合反射可以在运行时处理注解，配合apt tool可以在编译使处理注解，apt tool在JDK1.6已经被整合到javac中。

### 2. 作用
注解对程序的起到标识和传值的作用，对本身的程序运用并没有影响。可以配合反射(运行时的注解)和自定义注解处理器(编译时的注解)处理注解。java本身也内置一些注解，比如**@Override**表示重写父类的方法，**@Deprecated**表示改方法或属性已经过时。

### 3. 入门

#### 3.1 定义
在Java中通过**@interface**来表示这是一个注解：
  ```
public @interface AnnotationTest {
}
  ```

#### 3.2 元注解
仅仅像上面定义一个注解是没用的，还需要有元注解的修饰，所谓的元注解，就是注解的注解。java的元注解包括：@Retention、@Target、@Documented、@Inherited、@Repeatable。

**@Retention 保留的意思，即注解的生命周期保留到什么时候，它有三个保留阶段，范围逐渐增大**
  ```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    /**
     * Returns the retention policy.
     * @return the retention policy
     */
    RetentionPolicy value();
}
  ```

类型 | 范围|
-|-
**RetentionPolicy.RUNTIME** | 注解可以保留在运行时、编译、源码期间，可以通过反射、注解处理器获得 | 
**RetentionPolicy.CLASS** | 注解可以保留在编译阶段，但在虚拟机运行阶段时会被丢弃，可以通过注解处理器获得 | 
**RetentionPolicy.SOURCE** | 注解只保留在源码阶段，在编译器编译时就会被丢弃。|
  ```
@Retention(RetentionPolicy.RUNTIME)  // 运行、编译、源码阶段
@Retention(RetentionPolicy.CLASS)  // 编译阶段
@Retention(RetentionPolicy.SOURCE) // 源码阶段
public @interface AnnotationTest {
}

  ```


**@Target 翻译是目标的意思，表示注解能标识于什么类型中**
  ```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    /**
     * Returns an array of the kinds of elements an annotation type
     * can be applied to.
     * @return an array of the kinds of elements an annotation type
     * can be applied to
     */
    ElementType[] value();
}
  ```
类型 | 范围|
-|-
**ElementType.TYPE** | 注解目标在类，接口（包括注解接口）、枚举类 | 
**ElementType.FIELD** | 注解目标在属性变量(包括枚举) | 
**ElementType.METHOD** | 注解目标在方法 |
**ElementType.PARAMETER** | 注解目标在方法的形式参数 | 
**ElementType.CONSTRUCTOR** | 注解目标在构造方法 | 
**ElementType.LOCAL_VARIABLE** | 注解目标在局部变量声明。|
**ElementType.ANNOTATION_TYPE** | 注解目标在注解类中 | 
**ElementType.PACKAGE** | 注解目标在包中 | 
**ElementType.TYPE_PARAMETER** | 1.8后才支持，泛型类型|
**ElementType.TYPE_USE** | 1.8后才支持，除了PACKAGE外的所有类型。|
  ```
@Retention(RetentionPolicy.SOURCE)
@Target({ElementType.CONSTRUCTOR, ElementType.TYPE, ElementType.METHOD})
public @interface AnnotationTest {
}

  ```

**@Documented将注解的元素包含在javadoc中**
  ```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Documented {
}
  ```

**@Inherited有该元注解修饰的注解，注解标识的父类，子类同样有该注解**
  ```
@Retention(RetentionPolicy.SOURCE)
@Target({ElementType.TYPE})
@Inherited()
public @interface AnnotationTest {
}

@AnnotationTest
public abstract class A {
}

public class B extends A {
}

  ```
这里的子类B，由于AnnotationTest注解被@Inherited修饰有继承关系，所以B也继承了AnnotationTest注解。

**@Repeatable是JDK1.8才加入的一个元注解，这个的作用就是允许同一个注解在一个类型上标注多次。**
  ```
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Filters.class)
public @interface Filter {
    String value();
}

@Retention(RetentionPolicy.RUNTIME)
@interface Filters {
    Filter[] value();
}

  ```

#### 3.3 属性
注解只能起到标识的作用，如果想要传递参数，可以通过给注解设置属性来传递参数。
注解的属性设置跟类的属性设置不一样，但跟类的方法设置相似。
  ```
@Retention(RetentionPolicy.SOURCE)
@Target({ElementType.TYPE})
public @interface AnnotationTest {
    int age();

    String name();
}
  ```
注解的属性的赋值，通过名字来设置：
  ```
@AnnotationTest(age = 12, name = "zhangsan")
public abstract class A {
}
  ``` 
也可以给注解的属性设置默认值,那赋值时可以不用写：
  ```
@Retention(RetentionPolicy.SOURCE)
@Target({ElementType.TYPE})
public @interface AnnotationTest {
    int age() default 20;

    String name();
}

@AnnotationTest(name = "zhangsan")
public abstract class A {
}

  ```
如果你的属性只有一个，而且属性的名字是value，那么在赋值时可以不写名字：
  ```
@Retention(RetentionPolicy.SOURCE)
@Target({ElementType.TYPE})
public @interface AnnotationTest {
    int value();
}

@AnnotationTest(23)
public abstract class A {
}
  ```



