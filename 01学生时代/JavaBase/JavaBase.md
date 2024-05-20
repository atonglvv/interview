# 面向对象

## 封装

## 继承

### 优点

使用继承的主要优点是代码的可重用性，因为继承使子类能够重用其父类的代码。

多态性（可扩展性）是另一个好处，允许引入新的功能而不影响现有的派生类。

## 多态

通俗来说，就是多种形态，具体点就是去完成某个行为，当不同的对象去完成时会产生出不同的状态。在编程语言和类型论中，多态指为不同数据类型的实体提供统一的接口。多态类型可以将自身所支持的操作套用到其它类型的值上。

子类对象指向父类引用就是一种多态。

# 抽象类与接口

## 抽象类与接口的相同之处

- 抽象类和接口都不能被实例化，都用于被其他类实现或继承
- 他们都可以包含抽象方法，并且在其他类（非抽象类，抽象类继承抽象类可以不实现抽象方法）继承或实现的时候都必须实现这些抽象方法 

## 抽象类与接口的区别

- 接口只能做方法的声明，而抽象类中既可以包含方法的声明，也可以包含方法的实现。
- 接口里只能定义静态常量，而不能定义成员变量，抽象类中既可以定义静态常量，也可以定义成员变量。
- 接口没有构造函数，而抽象类有构造函数。
- java语法当中只支持类的单继承，而可以存在接口的多实现。
- 接口方法的访问权限必须是公共的，被public修饰

# Java是什么类型的语言？

Java是半编译半解释语言。

# finally与return的顺序？

先执行finally后执行return。

# final、finally、finalize的区别

final：java关键字，修饰类、不能被继承，修饰变量不能被修改，修饰方法不能被重写。
finally：异常处理的一部分，只能在try/catch语句中使用，表示finally中的代码一定会执行。
finalize：Object中的方法，在GC启动且对象将要被回收时由系统调用，不需要程序员手动调用。如下代码：

```java
public class Object {
    /**
     * Called by the garbage collector on an object when garbage collection
     * determines that there are no more references to the object.
     * A subclass overrides the {@code finalize} method to dispose of
     * system resources or to perform other cleanup.
     * <p>
     * The general contract of {@code finalize} is that it is invoked
     * if and when the Java&trade; virtual
     * machine has determined that there is no longer any
     * means by which this object can be accessed by any thread that has
     * not yet died, except as a result of an action taken by the
     * finalization of some other object or class which is ready to be
     * finalized. The {@code finalize} method may take any action, including
     * making this object available again to other threads; the usual purpose
     * of {@code finalize}, however, is to perform cleanup actions before
     * the object is irrevocably discarded. For example, the finalize method
     * for an object that represents an input/output connection might perform
     * explicit I/O transactions to break the connection before the object is
     * permanently discarded.
     * <p>
     * The {@code finalize} method of class {@code Object} performs no
     * special action; it simply returns normally. Subclasses of
     * {@code Object} may override this definition.
     * <p>
     * The Java programming language does not guarantee which thread will
     * invoke the {@code finalize} method for any given object. It is
     * guaranteed, however, that the thread that invokes finalize will not
     * be holding any user-visible synchronization locks when finalize is
     * invoked. If an uncaught exception is thrown by the finalize method,
     * the exception is ignored and finalization of that object terminates.
     * <p>
     * After the {@code finalize} method has been invoked for an object, no
     * further action is taken until the Java virtual machine has again
     * determined that there is no longer any means by which this object can
     * be accessed by any thread that has not yet died, including possible
     * actions by other objects or classes which are ready to be finalized,
     * at which point the object may be discarded.
     * <p>
     * The {@code finalize} method is never invoked more than once by a Java
     * virtual machine for any given object.
     * <p>
     * Any exception thrown by the {@code finalize} method causes
     * the finalization of this object to be halted, but is otherwise
     * ignored.
     *
     * @throws Throwable the {@code Exception} raised by this method
     * @see java.lang.ref.WeakReference
     * @see java.lang.ref.PhantomReference
     * @jls 12.6 Finalization of Class Instances
     */
    protected void finalize() throws Throwable { }
}
```



# 反射中，Class.forName和classloader的区别

# 为什么String类要用final修饰？







# Java集合类有那些

![img](img/001.gif)

# Java虚拟机内存模型的五大区域

程序计数器：指示当前线程所执行的字节码指令的行号指针。

虚拟机栈：存放局部变量表、操作数栈、动态链接和方法出口等信息。

本地方法栈：为虚拟机使用的Native方法服务。

堆：存放对象实例，是垃圾收集器管理的主要区域。

方法区：存放已被加载的类信息、常量、静态变量、编译后的代码等数据。



# Java中异常

Java中的异常都是Throwable类的子类，Throwable类有两个主要的子类：Error和Exception。

## Error

`Error` 类表示严重的错误，通常是虚拟机发生无法恢复的错误。程序员通常不需要直接捕获或处理 `Error`，因为这类错误通常意味着系统出现了不可逆的问题。例如，`OutOfMemoryError` 表示内存不足，`StackOverflowError` 表示堆栈溢出等。

## Exception

`Exception` 类是所有异常的父类。它分为两种：`受检异常（Checked Exception）和非受检异常（Unchecked Exception）`。

- 受检查异常：受检异常是在编译时强制处理的异常，程序必须在代码中显式地处理或者通过 `throws` 关键字声明方法可能抛出的受检异常。典型的受检异常包括 `IOException、SQLException` 等，它们表示程序在运行时可能遇到的外部因素导致的问题。
- 非受检查异常：非受检异常是在运行时可能抛出的异常，也称为`运行时异常（Runtime Exception）`。它们通常是由程序逻辑错误引起的，无法在编译时预测。典型的非受检异常包括 `NullPointerException、ArrayIndexOutOfBoundsException` 等。



# 反射

## 作用

 **Java反射就是在运行状态中，可以获取任意一个类的所有属性和方法（包括私有）并且调用它们；也可以改变它的属性（包括私有）。**这也是Java被视为动态语言（或准动态，为啥要说是准动态，因为一般而言的动态语言定义是程序运行时，允许改变程序结构或变量类型，这种语言称为动态语言。从这个观点看，Perl，Python，Ruby是动态语言，C++，Java，C#不是动态语言）的一个关键特质。动态代理的实现，就依赖于反射！

## 优点

代码更加灵活（毕竟可以获取任一个类的所有属性和方法），为各种框架提供开箱即用的功能提供了便利；

## 缺点

我们在运行时有了分析操作类的能力，这也增加了安全问题，比如可以无视泛型参数的安全检查（**泛型参数的安全检查发生在编译时，而我们在运行时可以分析、修改类**）。同时，反射的性能比较差（但对于框架而言，影响不大）。可以通过这个例程进行验证。









