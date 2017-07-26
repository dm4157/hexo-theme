id: java1
title: 类型信息
categories: core java
tags: [class,java]
---

> 运行时类型信息使得你可以在程序运行时发现和使用类型信息

java运行时识别对象和类的信息主要有两种方式：
- `RTTI`
- `反射`

## RTTI
> 接口与父类都是一种**窗口**，透过他们只能看到具体实现的一部分

`RTTI`(Run-Time Type Identification), 在运行时识别一个对象的类型。
有了`RTTI`才有`多态`， 而`多态`是面向对象编程的基本目标。
**举个栗子**
```java
List<String> lizi = new ArrayList<>();
```
这种写法的好处是方便替换具体实现，可能开始的时候觉得随机读取比较多就用`ArrayList`, 之后发现增删改比较多，可以简单的将实现替换为`LinkedList`即可。除非要用到具体实现的具体特性，否则一直建议使用更宽泛的类型引用。

## Class
### 获得Class对象的两种方式
- Class.forName(name) 获得Class对象的同时会加载类
- XXXX.class 只获得Class对象，不会加载类。学名是类字面常量。
```java
// Class.forName 方法， 找到合适的类加载器调用本地方法forName0
@CallerSensitive
public static Class<?> forName(String name, boolean initialize,
                               ClassLoader loader)
    throws ClassNotFoundException
{
    Class<?> caller = null;
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        // Reflective call to get caller class is only needed if a security manager
        // is present.  Avoid the overhead of making this call otherwise.
        caller = Reflection.getCallerClass();
        if (sun.misc.VM.isSystemDomainLoader(loader)) {
            ClassLoader ccl = ClassLoader.getClassLoader(caller);
            if (!sun.misc.VM.isSystemDomainLoader(ccl)) {
                sm.checkPermission(
                    SecurityConstants.GET_CLASSLOADER_PERMISSION);
            }
        }
    }
    return forName0(name, initialize, loader, caller);
}
// 从参数上就能看出来这货是要加载类了
private static native Class<?> forName0(String name, boolean initialize, ClassLoader loader, Class<?> caller) throws ClassNotFoundException;
```
### 值得注意的点
> 编译器已知的static final修饰的常量，这个值不需要进行类初始化就能访问到。**为啥呢？**

```java
class Initable {
  static final int staticFinal = 57;
  static final int staticFinal2 = ClassInitialization.rand.nextInt(1000);
  static {
    System.out.println("类初始化啦");
  }
}
class ClassInitialization {
  public static Random rand = new Random(57);
  public static void main(String[] args) {
    // 没有触发类的初始化
    Class initable = Initable.class;
    // 没有触发类的初始化
    System.out.println(Initable.staticFinal);
    // 触发类的初始化
    System.out.println(Initable.staticFinal2)
  }
}

// 输出：
// 57
// 类初始化啦
// 241
```


### 判断对象与类型的关系
判断一个对象是否是一个类型或者其类型是这个类型的子类，有两种方式：
```java
Class type = ...;
Object obj = ...;
// 方法1
if (obj instanceOf type) {
  ...
}
// 方法2
if(type.isInstance(obj)) {
  ...
}
```

## 反射
> 世界是平衡的

正如英文单词reflection的含义一样，使用反射API的时候就好像在看一个Java类在水中的倒影一样。知道了Java类的内部 结构之后，就可以与它进行交互，包括创建新的对象和调用对象中的方法等。这种交互方式与直接在源代码中使用的效果是相同的，但是又额外提供了运行时刻的灵活性。使用反射的一个最大的弊端是**性能比较差**。相同的操作，用反射API所需的时间大概比直接的使用要慢一两个数量级。不过现在的JVM实现中，反射操作的性能已经有了很大的提升。在灵活性与性能之间，总是需要进行权衡的。应用可以在适当的时机来使用反射API。

### 基本用法
Java反射API的第一个主要作用是获取程序在运行时刻的内部结构。只要有了java.lang.Class类的对象，就可以通过其中的方法来获取到该类中的构造方法、域和方法。对应的方法分别是getConstructor、getField和getMethod。这三个方法还有相应的getDeclaredXXX版本，区别在于getXXX只会获得“允许”访问的内容(即public修饰的)， 而getDeclaredXXX会获得所有修饰的内容；
> 使用Java反射API的时候可以绕过Java默认的访问控制检查，只需要在获取到Constructor、Field和Method类的对象之后，调用setAccessible方法并设为true即可。有了这种机制，就可以很方便的在运行时刻获取到程序的内部状态。

反射API的另外一个作用是在运行时刻对一个Java对象进行操作。 这些操作包括动态创建一个Java类的对象，获取某个域的值以及调用某个方法。在Java源代码中编写的对类和对象的操作，都可以在运行时刻通过反射API来实现。
**举个栗子**
```Java
// 简单类
class MyClass {
    public int count;
    public MyClass(int start) {
        count = start;
    }
    public void increase(int step) {
        count = count + step;
    }
}

// 一般操作
MyClass myClass = new MyClass(0);
myClass.increase(2);
System.out.println("Normal -> " + myClass.count);

// 反射操作
try {
    // 获取构造方法
    Constructor constructor = MyClass.class.getConstructor(int.class);
    // 创建对象
    MyClass myClassReflect = constructor.newInstance(10);
    // 获取方法
    Method method = MyClass.class.getMethod("increase", int.class);  
    // 调用方法
    method.invoke(myClassReflect, 5);
    // 获取域
    Field field = MyClass.class.getField("count");
    // 获取域的值
    System.out.println("Reflect -> " + field.getInt(myClassReflect));
} catch (Exception e) {
    e.printStackTrace();
}
```

### 处理泛型
比如在代码中声明了一个域是List<String>类型的，虽然在运行时刻其类型会变成原始类型List，但是仍然可以通过反射来获取到所用的实际的类型参数。
```java
Field field = Pair.class.getDeclaredField("myList"); //myList的类型是List
Type type = field.getGenericType();
if (type instanceof ParameterizedType) {     
    ParameterizedType paramType = (ParameterizedType) type;     
    Type[] actualTypes = paramType.getActualTypeArguments();     
    for (Type aType : actualTypes) {         
        if (aType instanceof Class) {         
            Class clz = (Class) aType;             
            System.out.println(clz.getName()); //输出java.lang.String         
        }     
    }
}
```

### 动态代理
下面的代码用来代理一个实现了List接口的对象。所实现的功能也非常简单，那就是禁止使用List接口中的add方法。如果在getList中传入一个实现List接口的对象，那么返回的实际就是一个代理对象，尝试在该对象上调用add方法就会抛出来异常。
```java
public List getList(final List list) {
    return (List) Proxy.newProxyInstance(DummyProxy.class.getClassLoader(), new Class[] { List.class },
        new InvocationHandler() {
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if ("add".equals(method.getName())) {
                    throw new UnsupportedOperationException();
                }
                else {
                    return method.invoke(list, args);
                }
            }
        });
 }
```

## 类加载器
### 概念
顾名思义，类加载器(Class Loader)用来加载类到Java虚拟机(JVM)中。一般来说，JVM使用Java类的方式如下：Java源程序(.java文件)在经过编译器编译之后就被转换成Java字节代码(.class文件)。类加载器负责读取Java字节代码，并转换成 java.lang.Class类的一个实例。每个这样的实例用来表示一个Java类。通过此实例的 newInstance()方法就可以创建出该类的一个对象。实际的情况可能更加复杂，比如Java字节代码可能是通过工具动态生成的，也可能是通过网络下载的。
基本上所有的类加载器都是 java.lang.ClassLoader类的一个实例。

### java.lang.ClassLoader
java.lang.ClassLoader类的基本职责就是根据一个指定的类的名称，找到或者生成其对应的字节代码，然后从这些字节代码中定义出一个Java类，即java.lang.Class类的一个实例。除此之外，ClassLoader还负责加载Java应用所需的资源，如图像文件和配置文件等。不过本文只讨论其加载类的功能。
为了完成加载类的这个职责，ClassLoader提供了一系列的方法。

|方法|说明|
|:-|:-|
|getParent()|返回该类加载器的父类加载器。|
|loadClass(String name)|加载名称为name的类，返回的结果是 java.lang.Class类的实例。|
|findClass(String name)|查找名称为name的类，返回的结果是 java.lang.Class类的实例。|
|findLoadedClass(String name)|查找名称为 name的已经被加载过的类，返回的结果是 java.lang.Class类的实例。|
|defineClass(String name, byte[] b, int off, int len)|把字节数组b中的内容转换成 Java 类，返回的结果是 java.lang.Class类的实例。这个方法被声明为 final的。|
|resolveClass(Class<?> c)|链接指定的Java类。|

### 组织结构
Java中的类加载器大致可以分成两类，一类是系统提供的，另外一类则是由Java应用开发人员编写的。系统提供的类加载器主要有下面三个：
- 引导类加载器(bootstrap class loader)：它用来加载Java的核心库，是用原生代码来实现的，并不继承自java.lang.ClassLoader。
- 扩展类加载器(extensions class loader)：它用来加载 Java 的扩展库。Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。
- 系统类加载器(system class loader)：它根据 Java 应用的类路径(CLASSPATH)来加载Java类。一般来说，Java应用的类都是由它来完成加载的。可以通过ClassLoader.getSystemClassLoader()来获取它。
除了系统提供的类加载器以外，开发人员可以通过继承 java.lang.ClassLoader类的方式实现自己的类加载器，以满足一些特殊的需求。
```java
// 演示类加载组织结构
public class ClassLoaderTree {

   public static void main(String[] args) {
       ClassLoader loader = ClassLoaderTree.class.getClassLoader();
       while (loader != null) {
           System.out.println(loader.toString());
           loader = loader.getParent();
       }
   }
}
// 输出
// sun.misc.Launcher$AppClassLoader@7adf9f5f
// sun.misc.Launcher$ExtClassLoader@5b2133b1
```
第一个输出的是 ClassLoaderTree类的类加载器，即系统类加载器。它是 sun.misc.Launcher$AppClassLoader类的实例；第二个输出的是扩展类加载器，是 sun.misc.Launcher$ExtClassLoader类的实例。需要注意的是这里并没有输出引导类加载器，这是由于有些JDK的实现对于父类加载器是引导类加载器的情况，getParent()方法返回null。

### 加载类的过程
类加载器首先检查这个类的Class是否已经加载，如果未加载，默认的类加载器会找到.class字节码，加载、链接、初始化；
- **加载** 由类加载器执行， 查找字节码， 并根据字节码创建Class对象
- **链接** 验证类中的字节码，为静态域分配存储空间，并且如果必要将解析这个类对其他类的引用并创建
- **初始化** 如果该类具有超类，则对其初始化，执行静态初始化器和静态初始化块
