## Chapter 31《Combining Scala and Java》

###从Java角度看Scala
- `scala`一般要和`Java`在大型程序中使用，使用`Java`中的框架。`Scala`的实现方式是将代码翻译成为标准的`Java`字节码，`Scala`的特性尽可能地直接映射为`Java`中的特性。可以使用`javap`来查看`.class`中`Scala`进行的翻译。
####1. 值类型
 对于值类型，在能够确定的情况下，优先翻译成为`Java`中的值类型用以获得更好的性能，如果不能确定，则翻译为包装类型。
####2. 单例对象
- 采用静态和实例方法相结合的方式来翻译单例对象，如果是单个的对象，则生成类`Test$`和`Test`，
```java
public final class Test$ {
  public static final Test$ MODULE$;
  public static {};
  public void main(java.lang.String[]);
  public java.lang.String select(java.lang.String[]);
}
```
其中，编译器插入了代码，保证在运行的时候私有化了`Test$`的构造方法，并对`MODULE$`进行了赋值。在`Test`类中
```java
public final class Test {
  public static java.lang.String select(java.lang.String[]);
  public static void main(java.lang.String[]);
}
```
对`Test$`中的所有方法在`Test`中都存在有`static`的转发版本。在书中说如果`object`有了对应的`class`，则在`java`的`class`中不会添加额外的转发方法，但是在实际过程中发现即使有了对应的`class`，`java`中的`class`也添加了额外的转发方法。
####3.接口
- 特质被翻译成为接口。如果在特质中有实现的部分，使用`abstract class`进行实现。
###注解
- 主要讨论的是`Java`中的注解。如果是`@deprecated`，则在`Java`代码上也加入该注解，这样，当`Java`代码试图访问`Scala`中带有`@deprecated`的代码时，会抛出警告。`@volatile`也是同样的操作。对与序列化的注解，`Scala`中的`@serializable`在`Java`中会翻译为实现了`Serializable`接口，`@SerialVersionUID(1234L)`被翻译成为
```java
// Java serial version marker
private final static long SerialVersionUID = 1234L
```
 `@transient`也是在`Java`代码上加上同样的注解。
####1.异常抛出
- `Scala`并不检查是由否有异常抛出，所以转换得到的所有`Java`代码都是不带`throw`的。因为在`Java`中，大程序可能就是吞下并隐藏了许多异常，所以在`Scala`中并不采用这种`throw`，`catch`的方法进行处理。如果生成的`Java`代码中必须要有`throws`异常的语法，则在`scala`中使用`@throws`的注解，
```scala
class Reader(fname: String) {
    private val in =
        new BufferedReader(new FileReader(fname))
    @throws(classOf[IOException])
    def read() = in.read()
}
```
####2.Java注解
现在的`Java`框架注解都可以直接应用在`Scala`代码中。
####3.定制注解
如果需要注解在`Java`反射的时候可以看到，则必须使用`Java`中的注解，并使用`javac`来编译它。因为`Scala`不能完全的支持`Java`中的注解，使用的反射机制也是`Java`的。可以这么使用：
```java
import java.lang.annotation.*; // This is Java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Ignore { }
```
  ```scala
  
object Tests {
    @Ignore
    def testData = List(0, 1, -1, 5, -5)
    def test1 = {
        assert(testData == (testData.head :: testData.tail))
    }
    def test2 = {
        assert(testData.contains(testData.head))
    }
}
for {
    method <- Tests.getClass.getMethods
    if method.getName.startsWith("test")
    if method.getAnnotation(classOf[Ignore]) == null // 使用的是Java中的反射API，所以比较使用的是null
} {
    println("found a test method: " + method)
}
```
使用的顺序如下所示：
```scala
$ javac Ignore.java
$ scalac Tests.scala
$ scalac FindTests.scala
$ scala FindTests
found a test method: public void Tests$.test2()
found a test method: public void Tests$.test1()
```
注意到这里是`Tests$`而不是`Tests`，因为使用的是`Java`中的反射`API`，所以在`Tests.getClass.getMethods`的时候类就是`Tests$`，而不是`Test`，同时注意在`Java`中的注解参数只能是常量，不能是`x*2`，`x`是常量这种形式。
###通配符类型
- `Java`和`Scala`是相通的，一般的转换都是非常简单的，`Java`中的`Iterator<Component>` 就是`Scala`中的`Iterator[Component]`，但是如果在`Java`中存在有通配符的泛型，比如`Iterator<?>`或者`Iterator<? extends Father>`这样的类型，在`Scala`中有同样的成为通配符的进行匹配转换。使用的是占位符的思想，使用下划线作为通配符进行转换，`Iterator<?>`就是`Iterator[_]`，表示一个`Iterator`，但是其中的类型是不明的。同样地，可以插入上限符号和下限符号来限制范围。`Iterator[_ <: Father]`，表示必须是`Father`的子类，`Iterator[_ >: Child]`，表示必须是`Child`的父类。
###同时编译Scala代码和Java代码
- 通常如果`Scala`代码依赖`Java`代码的话，首先编译`Java`代码生成对应的`class`文件，再编译`Scala`文件，将`Java class`文件放入到`classpath`中，如果在`Java`中同样引用了`Scala`代码，那么这个方法就失效了，`Scala`提供了一种解决这个问题的方法。
###在Scala2.12中集成Java8
- 在`Java8`中，任意一个需要一个类或者接口对象的地方都可以使用`lamda`表达式替代，这个类或者接口只能含有一个抽象的方法，叫`Single Abstract Method`对象，在`Scala2.12`中对`SAM`进行子类实现的时候，可以直接使用一个函数代替匿名类的实例。
####在Scala2.12中使用Java8中的流
- `Java`中的`Stream`是一个函数数据结构，提供了一个`map`方法。
