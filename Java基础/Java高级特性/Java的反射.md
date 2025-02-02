## 反射的概念
反射包含一个[反]字，那什么是正呢？
一般情况下，使用一个类时，我们通过`类名`直接`new`实例化来使用它，这就叫[正]
  ```
Apple apple = new Apple();
apple.setPrice(5);
  ```
反射则是通过`路径名、类名、对象`通过JDK提供的反射API，来获取和设置这个类的**Class、Constructor、对象、Method、Field**。
反射机制是一种运行时状态，所以类的实例化、获取设置都是在代码运行时生成的，对于像注解这种可以只在源代码、编译器则反射不了。
  ```
Class clz = Class.forName("com.example.javatest.reflection.Apple");
Constructor constructor = clz.getConstructor();
Object object = constructor.newInstance();
Method method = clz.getMethod("setPrice", int.class);
method.invoke(object, 10);
  ```
这段代码就是反射来获取并执行方法，它与正常的[正]不同的是，反射可以通过**Method和DeclaredMethod**去获取包括父类的public方法，和public和privete的自己的方法。不仅在Method这样，在Constructor、Field中也一样。

## Java的编译流程
### Class
了解java类在生命周期，可以帮助我们知道反射的使用流程。
类的生命周期：
> java文件 -> class文件 -> class对象 -> 实例化对象

所以在反射机制中，我们首先获取的是Class的对象。并且可以通过Class对象获取实例化对象。

### Constructor、Method、Field
在Class文件中存储着:构造函数、方法、属性，它们在Class文件中也有对应的对象分别是：**Constructor** 、**Method** 、**Field**它们的路径但是在[java.lang.reflect]中。并且可以通过`Constructor`获取实例化对象，通过`Method`执行相应的方法，通过`Field`获取和设置属性。

## 反射常用的Api
先看看一个简单的反射Demo
  ```
public class Apple {
    private int price;

    public int getPrice() {
        return price;
    }

    public void setPrice(int price) {
        this.price = price;
    }
}

public static void main(String[] args) throws Exception{
    // 正常调用
    Apple apple = new Apple();
    apple.setPrice(5);
    System.out.println("Apple Price =" + apple.getPrice());
    // 使用反射调用
    Class clz = Class.forName("com.example.javatest.reflection.Apple");   
    Constructor constructor = clz.getConstructor();
    Object object = constructor.newInstance();   // 实例对象
    Method method = clz.getDeclaredMethod("setPrice", int.class);
    method.invoke(object, 10);
    Method getMethod = clz.getMethod("getPrice");
    System.out.println("Apple price = " + getMethod.invoke(object));
}

  ```
通过正常调用和反射调用去执行一个方法`setPrice`设置属性。可以看出反射的流程会更加的复杂，但流程大致是：
> 获得Class对象->实例化对象->获取或设置属性、方法（Field.set/get，Method. invoke）

- 获得Class对象实例


  ```
  Class clz = Class.forName("com.example.javatest.reflection.Apple");  
  ```
- 获取Class对象的Constructor


  ```
Constructor constructor = clz.getConstructor();
  ```

- 根据Constructor的newInstance获得实例化对象


  ```
Object object = constructor.newInstance();
  ```

- 获取Class对象的Method


  ```
Method method = clz.getDeclaredMethod("setPrice", int.class);
  ```

- 利用invoke调用方法的执行


  ```
method.invoke(object, 10);
  ```

所以，我们从Class的API开始学起。


### 反射创建Class对象
反射创建Class对象一共有三种方法：
**第一种，通过类的路径名，使用Class.forName()创建Class对象**。
  ```
Class c = Class.forName("java.lang.String");
  ```

**第二种，通过类名，使用.class()创建Class对象**
  ```
Class clz = Apple.class;
  ```
**第三种，通过实例对象，使用.getClass()创建Class对象**
  ```
 Apple apple = new Apple();
Class clz = apple.getClass();
  ```

### 反射创建实例化对象
对于类中Method.Field的执行设置，都需要一个实例化的对象进行执行。反射创建实例化对象有两种方式：一是通过Class对象，二是通过Constructor对象。

第一种，通过Class对象的.newInstance()创建实例对象:
  ```
Class clz = Apple.class;
 Object object = clz.newInstance();
  ```

第二种，通过Constructor的.newInstance()创建实例对象:
  ```
Class clz = Apple.class;
Constructor constructor = clz.getConstructor(int.class);
Object object = constructor.newInstance(10);
  ```

### 反射获取类Field
如果我们想知道类中有哪些属性，可以通过Class对象的.getField()系列方法进行获取。
**通过属性的名字获取的Field**


  ```
 Class clz = Apple.class;
Field field = clz.getField("price");
  ```
另外，如果我们需要**获取私有的属性**，需要调用`getDeclaredField()`系列方法，需要一次**获取类的全部属性**，需要调用`getFields()`系列方法。
  ```
// 获取私有/公有属性
Class clz = Apple.class;
Field field = clz.getDeclaredField("price");

// 获取全部属性
Field[] field2 = clz.getDeclaredFields();
for (Field field3 : field2) {
    System.out.println("Apple field = " + field3.getName());
}
  ```

**属性Field的值的获取和设置:**
如果要对Field的值进行获取和设置，那么就必须有实例对象，因为有对象之后才有值。


  ```
Class clz = Apple.class;
Object object = clz.newInstance();
Field field = clz.getDeclaredField("price");
field.setAccessible(true);
field.set(object, 25);
System.out.println("price field = " + field.get(object));
  ```

### 反射获取类Method
**获取类Method方法**
通过Class对象，和方法的名字获得所属方法的对象。
  ```
Class clz = Apple.class;
Method method = clz.getDeclaredMethod("setPrice", int.class);
  ```

**执行Method方法**
  ```
 method.invoke(object, 10);
  ```

**获得方法参数类型**
  ```
Class[] parameters = method.getParameterTypes();
  ```

### 反射获取Constructor
**通过Class对象获取Constructor**
  ```
Class clz = Apple.class;
Constructor constructor = clz.getConstructor();
Object object = constructor.newInstance();
  ```
- 通过Class获取有参数的构造方法


  ```
Class clz = Apple.class;
Constructor constructor = clz.getConstructor(int.class);
Object object = constructor.newInstance(10);
  ```

## 反射工具类
```
public class ReflectUtils {
    private final static String TAG = "ReflectUtils";

    public static <T> T get(Class<?> clazz, String fieldName) throws Exception {
        return new ReflectFiled<T>(clazz, fieldName).get();
    }

    public static <T> T get(Class<?> clazz, String fieldName, Object instance) throws Exception {
        return new ReflectFiled<T>(clazz, fieldName).get(instance);
    }

    public static boolean set(Class<?> clazz, String fieldName, Object object) throws Exception {
        return new ReflectFiled(clazz, fieldName).set(object);
    }

    public static boolean set(Class<?> clazz, String fieldName, Object instance, Object value) throws Exception {
        return new ReflectFiled(clazz, fieldName).set(instance, value);
    }

    public static <T> T invoke(Class<?> clazz, String methodName, Object instance, Object... args) throws Exception {
        return new ReflectMethod(clazz, methodName).invoke(instance, args);
    }


    public static <T> T reflectObject(Object instance, String name, T defaultValue, boolean isHard) {
        if (null == instance) return defaultValue;
        if (isHard) {
            try {
                Method getDeclaredField = Class.class.getDeclaredMethod("getDeclaredField", String.class);
                Field field = (Field) getDeclaredField.invoke(instance.getClass(), name);
                field.setAccessible(true);
                return (T) field.get(instance);
            } catch (Exception e) {
                Log.e(TAG, e.toString());
            }
        } else {
            try {
                Field field = instance.getClass().getDeclaredField(name);
                field.setAccessible(true);
                return (T) field.get(instance);
            } catch (Exception e) {
                Log.e(TAG, e.toString());
            }
        }
        return defaultValue;
    }

    public static <T> T reflectObject(Object instance, String name, T defaultValue) {
        return reflectObject(instance, name, defaultValue, true);
    }

    public static Method reflectMethod(Object instance, boolean isHard, String name, Class<?>... argTypes) {
        if (isHard) {
            try {
                Method getDeclaredMethod = Class.class.getDeclaredMethod("getDeclaredMethod", String.class, Class[].class);
                Method method = (Method) getDeclaredMethod.invoke(instance.getClass(), name, argTypes);
                method.setAccessible(true);
                return method;
            } catch (Exception e) {
                Log.e(TAG, e.toString());
            }
        } else {
            try {
                Method method = instance.getClass().getDeclaredMethod(name, argTypes);
                method.setAccessible(true);
                return method;
            } catch (Exception e) {
                Log.e(TAG, e.toString());
            }

        }
        return null;
    }

    public static Method reflectMethod(Object instance, String name, Class<?>... argTypes) {
        boolean isHard = Build.VERSION.SDK_INT <= 29;
        return reflectMethod(instance, isHard, name, argTypes);
    }

}

```

```
public class ReflectFiled<Type> {
    private static final String TAG = "ReflectFiled";
    private Class<?> mClazz;
    private String mFieldName;

    private boolean mInit;
    private Field mField;

    public ReflectFiled(Class<?> clazz, String fieldName) {
        if (clazz == null || fieldName == null || fieldName.length() == 0) {
            throw new IllegalArgumentException("Both of invoker and fieldName can not be null or nil.");
        }
        this.mClazz = clazz;
        this.mFieldName = fieldName;
    }

    private synchronized void prepare() {
        if (mInit) {
            return;
        }
        Class<?> clazz = mClazz;
        while (clazz != null) {
            try {
                Field f = clazz.getDeclaredField(mFieldName);
                f.setAccessible(true);
                mField = f;
                break;
            } catch (Exception e) {
            }
            clazz = clazz.getSuperclass();
        }
        mInit = true;
    }

    public synchronized Type get() throws NoSuchFieldException, IllegalAccessException,
            IllegalArgumentException {
        return get(false);
    }

    @SuppressWarnings("unchecked")
    public synchronized Type get(boolean ignoreFieldNoExist)
            throws NoSuchFieldException, IllegalAccessException, IllegalArgumentException {
        prepare();
        if (mField == null) {
            if (!ignoreFieldNoExist) {
                throw new NoSuchFieldException();
            }
            Log.w(TAG, String.format("Field %s is no exists.", mFieldName));
            return null;
        }
        Type fieldVal = null;
        try {
            fieldVal = (Type) mField.get(null);
        } catch (ClassCastException e) {
            throw new IllegalArgumentException("unable to cast object");
        }
        return fieldVal;
    }

    public synchronized Type get(boolean ignoreFieldNoExist, Object instance)
            throws NoSuchFieldException, IllegalAccessException, IllegalArgumentException {
        prepare();
        if (mField == null) {
            if (!ignoreFieldNoExist) {
                throw new NoSuchFieldException();
            }
            Log.w(TAG, String.format("Field %s is no exists.", mFieldName));
            return null;
        }
        Type fieldVal = null;
        try {
            fieldVal = (Type) mField.get(instance);
        } catch (ClassCastException e) {
            throw new IllegalArgumentException("unable to cast object");
        }
        return fieldVal;
    }

    public synchronized Type get(Object instance) throws NoSuchFieldException, IllegalAccessException {
        return get(false, instance);
    }

    public synchronized Type getWithoutThrow(Object instance) {
        Type fieldVal = null;
        try {
            fieldVal = get(true, instance);
        } catch (NoSuchFieldException e) {
            Log.i(TAG, "getWithoutThrow, exception occur :%s", e);
        } catch (IllegalAccessException e) {
            Log.i(TAG, "getWithoutThrow, exception occur :%s", e);
        } catch (IllegalArgumentException e) {
            Log.i(TAG, "getWithoutThrow, exception occur :%s", e);
        }
        return fieldVal;
    }

    public synchronized Type getWithoutThrow() {
        Type fieldVal = null;
        try {
            fieldVal = get(true);
        } catch (NoSuchFieldException e) {
            Log.i(TAG, "getWithoutThrow, exception occur :%s", e);
        } catch (IllegalAccessException e) {
            Log.i(TAG, "getWithoutThrow, exception occur :%s", e);
        } catch (IllegalArgumentException e) {
            Log.i(TAG, "getWithoutThrow, exception occur :%s", e);
        }
        return fieldVal;
    }

    public synchronized boolean set(Object instance, Type val) throws NoSuchFieldException, IllegalAccessException,
            IllegalArgumentException {
        return set(instance, val, false);
    }

    public synchronized boolean set(Object instance, Type val, boolean ignoreFieldNoExist)
            throws NoSuchFieldException, IllegalAccessException, IllegalArgumentException {
        prepare();
        if (mField == null) {
            if (!ignoreFieldNoExist) {
                throw new NoSuchFieldException("Method " + mFieldName + " is not exists.");
            }
            Log.w(TAG, String.format("Field %s is no exists.", mFieldName));
            return false;
        }
        mField.set(instance, val);
        return true;
    }


    public synchronized boolean setWithoutThrow(Object instance, Type val) {
        boolean result = false;
        try {
            result = set(instance, val, true);
        } catch (NoSuchFieldException e) {
            Log.i(TAG, "setWithoutThrow, exception occur :%s", e);
        } catch (IllegalAccessException e) {
            Log.i(TAG, "setWithoutThrow, exception occur :%s", e);
        } catch (IllegalArgumentException e) {
            Log.i(TAG, "setWithoutThrow, exception occur :%s", e);
        }
        return result;
    }

    public synchronized boolean set(Type val) throws NoSuchFieldException, IllegalAccessException {
        return set(null, val, false);
    }

    public synchronized boolean setWithoutThrow(Type val) {
        boolean result = false;
        try {
            result = set(null, val, true);
        } catch (NoSuchFieldException e) {
            Log.i(TAG, "setWithoutThrow, exception occur :%s", e);
        } catch (IllegalAccessException e) {
            Log.i(TAG, "setWithoutThrow, exception occur :%s", e);
        } catch (IllegalArgumentException e) {
            Log.i(TAG, "setWithoutThrow, exception occur :%s", e);
        }
        return result;
    }

}

```

```
public class ReflectMethod {
    private static final String TAG = "ReflectFiled";
    private Class<?> mClazz;
    private String mMethodName;

    private boolean mInit;
    private Method mMethod;
    private Class[] mParameterTypes;

    public ReflectMethod(Class<?> clazz, String methodName, Class<?>... parameterTypes) {
        if (clazz == null || methodName == null || methodName.length() == 0) {
            throw new IllegalArgumentException("Both of invoker and fieldName can not be null or nil.");
        }
        this.mClazz = clazz;
        this.mMethodName = methodName;
        this.mParameterTypes = parameterTypes;
    }

    private synchronized void prepare() {
        if (mInit) {
            return;
        }
        Class<?> clazz = mClazz;
        while (clazz != null) {
            try {
                Method method = clazz.getDeclaredMethod(mMethodName, mParameterTypes);
                method.setAccessible(true);
                mMethod = method;
                break;
            } catch (Exception e) {
            }
            clazz = clazz.getSuperclass();
        }
        mInit = true;
    }

    public synchronized <T> T invoke(Object instance, Object... args) throws NoSuchFieldException, IllegalAccessException,
            IllegalArgumentException, InvocationTargetException {
        return invoke(instance, false, args);
    }

    public synchronized <T> T invoke(Object instance, boolean ignoreFieldNoExist, Object... args)
            throws NoSuchFieldException, IllegalAccessException, IllegalArgumentException, InvocationTargetException {
        prepare();
        if (mMethod == null) {
            if (!ignoreFieldNoExist) {
                throw new NoSuchFieldException("Method " + mMethodName + " is not exists.");
            }
            Log.w(TAG, "Field %s is no exists" + mMethodName);
            return null;
        }
        return (T) mMethod.invoke(instance, args);
    }


    public synchronized <T> T invokeWithoutThrow(Object instance, Object... args) {
        try {
            return invoke(instance, true, args);
        } catch (NoSuchFieldException e) {
            Log.e(TAG, "invokeWithoutThrow, exception occur :%s", e);
        } catch (IllegalAccessException e) {
            Log.e(TAG, "invokeWithoutThrow, exception occur :%s", e);
        } catch (IllegalArgumentException e) {
            Log.e(TAG, "invokeWithoutThrow, exception occur :%s", e);
        } catch (InvocationTargetException e) {
            Log.e(TAG, "invokeWithoutThrow, exception occur :%s", e);
        }
        return null;
    }
}

```


### 反射的底层实现过程
1. **类加载过程与反射基础**
   - **类加载机制回顾**
     - 在Java中，类的生命周期包括加载、验证、准备、解析和初始化等阶段。当程序首次使用某个类时，类加载器（ClassLoader）负责将类的字节码文件加载到内存中，并将其转换为一个`Class`对象。这个`Class`对象是反射操作的核心基础，它包含了类的所有元信息，如类名、父类、接口、字段、方法和构造函数等。
     - 例如，有一个简单的`Person`类：
```java
class Person {
    private String name;
    public Person(String name) {
        this.name = name;
    }
    public String getName() {
        return name;
    }
}
```
 - 当程序中第一次出现`Person`类相关的操作（如`new Person("John")`或者`Class.forName("Person")`）时，类加载器会查找并加载`Person`类的字节码文件，然后创建对应的`Class`对象。
   - **获取`Class`对象的方式与反射启动**
     - 有三种常见方式获取`Class`对象。一是使用`类名.class`（如`Person.class`），这种方式在编译时就可以确定类的情况下使用；二是通过`对象.getClass()`（如`new Person("John").getClass()`），用于已经有类的实例的情况；三是使用`Class.forName("全限定类名")`（如`Class.forName("com.example.Person")`），常用于在运行时根据类名动态加载类，这是反射操作中比较常用的方式，特别是在加载外部配置的类或者插件类时。
2. **反射操作的底层实现过程（以访问字段为例）**
   - **字段信息获取**
     - 当通过`Class`对象获取字段（`Field`）信息时，例如`Class<? extends Person> personClass = Person.class; Field nameField = personClass.getDeclaredField("name");`，`getDeclaredField`方法会在`Class`对象内部维护的字段列表中查找指定名称的字段。这个字段列表是在类加载阶段解析类字节码文件时填充的。
     - 字节码文件中包含了类的所有字段的信息，包括字段名、类型、修饰符（如`private`、`public`等）。`getDeclaredField`方法会遍历这些信息来找到对应的字段。
   - **访问权限检查与`setAccessible`方法**
     - 正常情况下，对于`private`等受限制的字段，直接访问是不允许的。但是通过`nameField.setAccessible(true)`可以绕过这种访问限制。
     - 从底层实现角度，Java虚拟机（JVM）在执行普通的字段访问操作时，会检查访问权限。而`setAccessible`方法实际上是告诉JVM跳过这种常规的访问检查。当这个方法被设置为`true`后，反射机制在访问字段时，会直接通过底层的字节码操作来获取或设置字段的值，而不考虑语言层面的访问权限限制。
   - **字段值的读取和设置（字节码操作层面）**
     - 当执行`nameField.get(personObject)`来读取字段值时，反射机制会根据字段在对象内存布局中的偏移量来获取值。每个对象在内存中的布局是由其类的结构决定的，字段在对象中的位置是固定的。
     - 对于设置字段值（如`nameField.set(personObject, "New Name")`），反射同样根据偏移量找到字段的位置，然后将新的值存储到对应的内存位置。这些操作涉及到比较底层的字节码指令，JVM会根据反射调用的指令来操作内存中的数据。
3. **反射操作的底层实现过程（以调用方法为例）**
   - **方法信息获取**
     - 当通过`Class`对象获取方法（`Method`）信息时，例如`Method getNameMethod = personClass.getMethod("getName");`，`getMethod`方法（或`getDeclaredMethod`用于获取包括私有方法在内的所有方法）会在`Class`对象的方法列表中查找指定名称和参数类型匹配的方法。
     - 字节码文件中包含了类的所有方法的签名（方法名、参数类型列表、返回类型）和字节码指令。`getMethod`操作会根据这些信息进行查找和匹配。
   - **方法调用过程（动态方法分发）**
     - 当执行`getNameMethod.invoke(personObject)`来调用方法时，反射机制首先会检查方法的访问权限（如果没有设置`setAccessible(true)`且方法是私有的等情况会有访问限制）。
     - 然后，通过方法在对象内存中的入口点（类似于C++中的函数指针）来执行方法的字节码指令。这个入口点是在类加载和解析阶段确定的，JVM会根据方法在类的方法表中的位置来找到对应的字节码指令序列。在执行方法调用时，反射还需要处理参数的传递，将调用`invoke`方法时传入的参数按照方法签名中的参数类型进行类型转换和传递，然后执行方法的字节码指令，最后返回方法的结果。