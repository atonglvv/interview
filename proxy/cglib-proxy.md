# cglib-proxy

## CGLIB介绍

CGLIB(Code Generation Library)是一个开源、高性能、高质量的Code生成类库（代码生成包）。

它可以在运行期扩展Java类与实现Java接口。Hibernate用它实现PO(Persistent Object 持久化对象)字节码的动态生成，Spring AOP用它提供方法的interception（拦截）。

CGLIB的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。但不鼓励大家直接使用ASM框架，因为对底层技术要求比较高。

## 如何使用

首先，引入CGLIB的依赖：

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.5</version>
</dependency>
```

这里我们以操作用户数据的UserDao为例，通过动态代理来对其功能进行增强（执行前后添加日志）。UserDao定义如下：

```java
public class UserDao {

    public void findAllUsers(){
        System.out.println("UserDao 查询所有用户");
    }

    public String findUsernameById(int id){
        System.out.println("UserDao 根据ID查询用户");
        return "公众号：程序新视界";
    }
}
```

创建一个拦截器，实现接口net.sf.cglib.proxy.MethodInterceptor，用于方法的拦截回调。

```java
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * @author sec
 * @version 1.0
 * @date 2020/3/24 8:14 上午
 **/
public class LogInterceptor implements MethodInterceptor {

    /**
     *
     * @param obj 表示要进行增强的对象
     * @param method 表示拦截的方法
     * @param objects 数组表示参数列表，基本数据类型需要传入其包装类型，如int-->Integer、long-Long、double-->Double
     * @param methodProxy 表示对方法的代理，invokeSuper方法表示对被代理对象方法的调用
     * @return 执行结果
     * @throws Throwable 异常
     */
    @Override
    public Object intercept(Object obj, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        before(method.getName());
        // 注意这里是调用invokeSuper而不是invoke，否则死循环;
        // methodProxy.invokeSuper执行的是原始类的方法;
        // method.invoke执行的是子类的方法;
        Object result = methodProxy.invokeSuper(obj, objects);
        after(method.getName());
        return result;
    }

    /**
     * 调用invoke方法之前执行
     */
    private void before(String methodName) {
        System.out.println("调用方法" + methodName +"之【前】的日志处理");
    }

    /**
     * 调用invoke方法之后执行
     */
    private void after(String methodName) {
        System.out.println("调用方法" + methodName +"之【后】的日志处理");
    }
}
```

实现MethodInterceptor接口的intercept方法。该方法中参数：

- obj：表示要进行增强的对象；
- method：表示要被拦截的方法；
- objects：表示要被拦截方法的参数；
- methodProxy：表示要触发父类的方法对象。

在方法的内部主要调用的methodProxy.invokeSuper，执行的原始类的方法。如果调用invoke方法否会出现死循环。

客户端使用示例如下：

```java
import net.sf.cglib.proxy.Enhancer;

public class CglibTest {

    public static void main(String[] args) {

        // 通过CGLIB动态代理获取代理对象的过程
        // 创建Enhancer对象，类似于JDK动态代理的Proxy类
        Enhancer enhancer = new Enhancer();
        // 设置目标类的字节码文件
        enhancer.setSuperclass(UserDao.class);
        // 设置回调函数
        enhancer.setCallback(new LogInterceptor());
        // create方法正式创建代理类
        UserDao userDao = (UserDao) enhancer.create();
        // 调用代理类的具体业务方法
        userDao.findAllUsers();
        userDao.findUsernameById(1);
    }
}
```

执行客户端的main方法打印结果如下：

```txt
调用方法findAllUsers之【前】的日志处理
UserDao 查询所有用户
调用方法findAllUsers之【后】的日志处理
调用方法findUsernameById之【前】的日志处理
UserDao 根据ID查询用户
调用方法findUsernameById之【后】的日志处理
```

可以看到，我们方法前后已经被添加上对应的“增强处理”。

## cglib生成的class

在main方法内的第一行我们也可以添加如下设置，来存储代理类的class文件。

```java
// 代理类class文件存入本地磁盘，可反编译查看源码
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/atong/temp");
```

再次执行程序，我们可以看到在对应目录下生成三个class文件：

```txt
UserDao?EnhancerByCGLIB?1169c462.class
UserDao?EnhancerByCGLIB?1169c462?FastClassByCGLIB?22cae79c.class
UserDao?FastClassByCGLIB?197ae7fa.class
```

部分反编译代码如下：

```java
import java.lang.reflect.Method;
import net.sf.cglib.core.ReflectUtils;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class UserDao?EnhancerByCGLIB?1169c462 extends UserDao implements Factory {
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$findAllUsers$0$Method;
    private static final MethodProxy CGLIB$findAllUsers$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$findUsernameById$1$Method;
    private static final MethodProxy CGLIB$findUsernameById$1$Proxy;
    private static final Method CGLIB$equals$2$Method;
    private static final MethodProxy CGLIB$equals$2$Proxy;
    private static final Method CGLIB$toString$3$Method;
    private static final MethodProxy CGLIB$toString$3$Proxy;
    private static final Method CGLIB$hashCode$4$Method;
    private static final MethodProxy CGLIB$hashCode$4$Proxy;
    private static final Method CGLIB$clone$5$Method;
    private static final MethodProxy CGLIB$clone$5$Proxy;

    public final void findAllUsers() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$findAllUsers$0$Method, CGLIB$emptyArgs, CGLIB$findAllUsers$0$Proxy);
        } else {
            super.findAllUsers();
        }
    }

    final String CGLIB$findUsernameById$1(int var1) {
        return super.findUsernameById(var1);
    }

    public final String findUsernameById(int var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        return var10000 != null ? (String)var10000.intercept(this, CGLIB$findUsernameById$1$Method, new Object[]{new Integer(var1)}, CGLIB$findUsernameById$1$Proxy) : super.findUsernameById(var1);
    }

    // ...
}
```

从反编译的源码可以看出，代理对象继承UserDao，拦截器调用intercept()方法，intercept()方法由自定义的LogInterceptor实现，所以最后调用LogInterceptor中的intercept()方法，从而完成了由代理对象访问到目标对象的动态代理实现。

## CGLIB创建动态代理类过程

（1）查找目标类上的所有非final的public类型的方法定义；

（2）将符合条件的方法定义转换成字节码；

（3）将组成的字节码转换成相应的代理的class对象；

（4）实现MethodInterceptor接口，用来处理对代理类上所有方法的请求。

## JDK动态代理与CGLIB对比

JDK动态代理：基于Java反射机制实现，必须要实现了接口的业务类才生成代理对象。

CGLIB动态代理：基于ASM机制实现，通过生成业务类的子类作为代理类。

JDK Proxy的优势：

最小化依赖关系、代码实现简单、简化开发和维护、JDK原生支持，比CGLIB更加可靠，随JDK版本平滑升级。而字节码类库通常需要进行更新以保证在新版Java上能够使用。

基于CGLIB的优势：

无需实现接口，达到代理类无侵入，只操作关心的类，而不必为其他相关类增加工作量。高性能。