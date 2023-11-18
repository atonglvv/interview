# @AfterReturning、@After以及AfterThrowing的区别

```java
/*
 * @Before和@Around的执行顺序 :
 * 若是基于XML 则 其结果 受影响于 配置的先后顺序。
 */

/*
 * @AfterReturning和@After的区别 :
 * (1) @AfterReturning 被代理的方法执行完成之后 要执行的代码。
 * (2) @After 新生成的代理方法执行完成之后 要执行的代码，是放在finally块中的。
 * 注: 若是基于XML 则 两者的执行顺序结果 受影响于 配置的先后顺序。
 */

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object result;
    try {
        //@Before
        result = method.invoke(target, args);
        //@AfterReturning
        return result;
    } catch (InvocationTargetException e) {
        Throwable targetException = e.getTargetException();
        //@AfterThrowing
        throw targetException;
    } finally {
        //@After
    }
}
```

