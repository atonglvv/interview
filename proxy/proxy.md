# JDK-Proxy

代理类并不是在 Java 代码中定义，而是在运行时根据在 Java 代码中的**指示**动态生成(字节码由`JVM`在运行时动态生成而非预存在任何一个 `.class` 文件中)， **这种在程序运行时创建代理类的代理方式被称为动态代理**，它的优势在于可以方便地对代理类的函数进行统一处理。这是因为所有被代理执行的方法，都是通过`InvocationHandler#invoke()`方法调用，相当于给被代理类所有方法套了一层壳，所以只要在这个方法中统一处理，就可以对所有被代理的方法进行相同的操作了。

## JDK动态代理实现

以下代码展示了动态代理的简单使用，其基本步骤如下：

1. 定义一个公共接口，本例中为 IHello，接口中有一个抽象方法
2. 定义一个实现了公共接口的实体类作为被代理类，本例中被代理类 Hello实现了 IHello接口，重写了接口中的抽象方法
3. 定义一个实现了 InvocationHandler 接口的方法拦截类，重写 invoke() 方法实现拦截到被代理类方法执行时候的处理逻辑
4. 通过 Proxy.newProxyInstance() 方法生成代理对象，持有代理对象之后执行接口方法即可

```java
public class ServiceProxy {

    public interface IHello {
        String sayHi();
    }

    public static class Hello implements IHello {
        @Override
        public String sayHi() {
            return "Hello";
        }
    }

    // 动态代理类
    public static class ProxyHandler<T> implements InvocationHandler {
        private T origin;

        public ProxyHandler(T origin) {
            this.origin = origin;
        }

       /**
         * @param o 代理对象引用
         * @param method 正在执行目标的方法
         * @param objects 目标方法执行时的入参
        */
        @Override
        public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
            String s = "proxy";
            s += method.invoke(origin, objects);
            return s;
        }
    }

    public static void main(String[] args) {
        IHello IHello = (IHello) getInstance(IHello.class, new ProxyHandler<>(new Hello()));

        System.out.println(IHello.toString());

        generateProxyClass();
    }

    // 创建代理对象
    public static <T> Object getInstance(Class<T> clazz, ProxyHandler<T> handler) {
        return Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, handler);
    }

    // 通过以下代码可以将 JVM 中加载的代理类输出成 class 文件，之后就可以使用反编译工具查看代理类的源码。
    private static void generateProxyClass() {
        byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", Hello.class.getInterfaces());
        String path = "/Users/nathan.yang/workspace/algorithm_Java/out/StuProxy.class";
        try (FileOutputStream fos = new FileOutputStream(path)) {
            fos.write(classFile);
            fos.flush();
            System.out.println("代理类文件写入成功");
        } catch (Exception e) {
            System.out.println("写文件错误");
        }
    }
}
```

##  动态代理原理

![](img\jdk-proxy01.png)

### **Proxy#newProxyInstance()**

**Proxy#newProxyInstance()** 方法是动态代理的入口，其生成动态代理对象主要有以下几个步骤：

1. getProxyClass0() 方法生成代理类

2. 获取到代理类后将 InvocationHandler 对象入参，反射调用构造方法生成动态代理对象

```java
public static Object newProxyInstance(ClassLoader loader,
                                       Class<?>[] interfaces,
                                       InvocationHandler h)
     throws IllegalArgumentException
 {
     Objects.requireNonNull(h);

     final Class<?>[] intfs = interfaces.clone();
     final SecurityManager sm = System.getSecurityManager();
     if (sm != null) {
         checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
     }

     /*
      * Look up or generate the designated proxy class.
      */
     Class<?> cl = getProxyClass0(loader, intfs);

     /*
      * Invoke its constructor with the designated invocation handler.
      */
     try {
         if (sm != null) {
             checkNewProxyPermission(Reflection.getCallerClass(), cl);
         }

         final Constructor<?> cons = cl.getConstructor(constructorParams);
         final InvocationHandler ih = h;
         if (!Modifier.isPublic(cl.getModifiers())) {
             AccessController.doPrivileged(new PrivilegedAction<Void>() {
                 public Void run() {
                     cons.setAccessible(true);
                     return null;
                 }
             });
         }
         return cons.newInstance(new Object[]{h});
     } catch (IllegalAccessException|InstantiationException e) {
         throw new InternalError(e.toString(), e);
     } catch (InvocationTargetException e) {
         Throwable t = e.getCause();
         if (t instanceof RuntimeException) {
             throw (RuntimeException) t;
         } else {
             throw new InternalError(t.toString(), t);
         }
     } catch (NoSuchMethodException e) {
         throw new InternalError(e.toString(), e);
     }
 }
```

### Proxy#getProxyClass0()

`Proxy#getProxyClass0()` 方法其实是从一个 `WeakCache` 中去获取代理类，其获取逻辑是如果缓存类中没有代理类的话就调用`ProxyClassFactory#apply()`，通过代理类工厂去即时生成一个代理类，其步骤如下：

首先通过指定的类加载器去验证目标接口是否可被其加载，通过接口所在包等条件决定代理类所在包及代理类的全限定名称，代理类名称是`包名+$Proxy+id`。

通过 `ProxyGenerator.generateProxyClass()` 生成字节码数组，然后调用 `native`方法`defineClass0()` 将其动态生成的代理类字节码加载到内存中。

```java
private static Class < ? > getProxyClass0(ClassLoader loader,
    Class < ? > ...interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}

public Class < ? > apply(ClassLoader loader, Class < ? > [] interfaces) {

    Map < Class < ? > , Boolean > interfaceSet = new IdentityHashMap < > (interfaces.length);
    for (Class < ? > intf : interfaces) {
        /*
         * Verify that the class loader resolves the name of this
         * interface to the same Class object.
         */
        Class < ? > interfaceClass = null;
        try {
            interfaceClass = Class.forName(intf.getName(), false, loader);
        } catch (ClassNotFoundException e) {}
        if (interfaceClass != intf) {
            throw new IllegalArgumentException(
                intf + " is not visible from class loader");
        }
        /*
         * Verify that the Class object actually represents an
         * interface.
         */
        if (!interfaceClass.isInterface()) {
            throw new IllegalArgumentException(
                interfaceClass.getName() + " is not an interface");
        }
        /*
         * Verify that this interface is not a duplicate.
         */
        if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
            throw new IllegalArgumentException(
                "repeated interface: " + interfaceClass.getName());
        }
    }

    String proxyPkg = null; // package to define proxy class in
    int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

    /*
     * Record the package of a non-public proxy interface so that the
     * proxy class will be defined in the same package.  Verify that
     * all non-public proxy interfaces are in the same package.
     */
    for (Class < ? > intf : interfaces) {
        int flags = intf.getModifiers();
        if (!Modifier.isPublic(flags)) {
            accessFlags = Modifier.FINAL;
            String name = intf.getName();
            int n = name.lastIndexOf('.');
            String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
            if (proxyPkg == null) {
                proxyPkg = pkg;
            } else if (!pkg.equals(proxyPkg)) {
                throw new IllegalArgumentException(
                    "non-public interfaces from different packages");
            }
        }
    }

    if (proxyPkg == null) {
        // if no non-public proxy interfaces, use com.sun.proxy package
        proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
    }

    /*
     * Choose a name for the proxy class to generate.
     */
    long num = nextUniqueNumber.getAndIncrement();
    String proxyName = proxyPkg + proxyClassNamePrefix + num;

    /*
     * Generate the specified proxy class.
     */
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
        proxyName, interfaces, accessFlags);
    try {
        return defineClass0(loader, proxyName,
            proxyClassFile, 0, proxyClassFile.length);
    } catch (ClassFormatError e) {
        /*
         * A ClassFormatError here means that (barring bugs in the
         * proxy class generation code) there was some other
         * invalid aspect of the arguments supplied to the proxy
         * class creation (such as virtual machine limitations
         * exceeded).
         */
        throw new IllegalArgumentException(e.toString());
    }
}
```

反射获取到代理类参数为 `InvocationHandler.class` 的构造器，其实也就是 `Proxy` 的带参构造器，调用构造器`cons.newInstance(new Object[]{h})`生成代理对象。

```
protected Proxy(InvocationHandler h) {
    Objects.requireNonNull(h);
    this.h = h;
}
```

通过以下代码可以将 `JVM` 中加载的代理类输出成 `class` 文件，之后就可以使用反编译工具查看代理类的源码。

```java
private static void generateProxyClass() {
     byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", Hello.class.getInterfaces());
     String path = "/Users/nathan/workspace/algorithm_Java/out/StuProxy.class";
     try (FileOutputStream fos = new FileOutputStream(path)) {
         fos.write(classFile);
         fos.flush();
         System.out.println("代理类文件写入成功");
     } catch (Exception e) {
         System.out.println("写文件错误");
     }
 }
```

生成的代理类源码如下，很明显可以看到该类实现动态代理的原理：

- 通过 `static` 代码块将被代理类中每一个方法封装为 `Method` 对象，生成方法表

- 代理类对象执行被代理类同名方法时，通过其父类`Proxy`保留的指向`InvocationHandler`对象的引用调用 `InvocationHandler#invoke()` 方法，完成动态代理

```java
public final class $Proxy0 extends Proxy implements IHello {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws {
        super(var1);
    }

    public final String sayHi() throws {
        try {
            // 父类 Proxy 保留的指向 InvocationHandler 对象的引用调用 invoke() 方法
            return (String) super.h.invoke(this, m3, (Object[]) null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws {
            try {
                return (String) super.h.invoke(this, m2, (Object[]) null);
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        ......

        // 方法表
        static {
            try {
                m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
                m3 = Class.forName("ServiceProxy$IHello").getMethod("sayHi");
                m2 = Class.forName("java.lang.Object").getMethod("toString");
                m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            } catch (NoSuchMethodException var2) {
                throw new NoSuchMethodError(var2.getMessage());
            } catch (ClassNotFoundException var3) {
                throw new NoClassDefFoundError(var3.getMessage());
            }
        }
}
```

