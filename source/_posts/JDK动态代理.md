---
title: JDK动态代理

date: 2019-1-14 20:45:00

tags: 动态代理
---

###  一、代理模式

代理模式（Proxy）：给某一个对象提供一个代理，并由代理对象控制对原对象的引用。

![](https://api.superbed.cn/pic/5c3c36429dc6d6264c4a7ca3)

代理类Proxy与被代理类RealSubject均实现了Subject接口，并且Proxy持有RealSubject对象的引用。当Client调用Request()方法时，将调用Proxy.Request()，并由Proxy.Request()代理调用realSubject.Request()



### 二、JDK动态代理

JDK中基于反射实现了动态代理，可以在程序运行时生成代理类。其核心在于java.lang.reflect.InvocationHandler与 java.lang.reflect.Proxy，InvocationHandler是一个接口只包含invoke方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```

首先需要实现该接口，并且持有被代理类target的引用，当代理类的任何方法被调用时会转发到调用invoke方法。因此，可以在invoke方法中添加代理逻辑比如：打印日志、安全审计等，最后再通过反射调用被代理类的方法。

```java
public class InvocationHandlerImpl<T> implements InvocationHandler {

    private T target;

    public InvocationHandlerImpl(T target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before hello");
        // 调用被代理的方法
        Object result = method.invoke(target, args);
        System.out.println("after hello");
        return result;
    }

}
```

而在程序运行时动态生成代理类则是依靠Proxy.newProxyInstance方法，该方法的参数分别为：

1. ClassLoader loader，类加载器
2. Class<?>[] interfaces，代理类需要实现的接口列表
3. InvocationHandler h，上文实现的代理执行类

``` java
public static Object getProxy(Object target) {
    InvocationHandlerImpl invocationHandler = new InvocationHandlerImpl(target);
    return Proxy.newProxyInstance(target.getClass().getClassLoader(), 
                                  target.getClass().getInterfaces(), 
                                  invocationHandler);
}
```

生成的代理类已经实现了指定的接口，因此使用起来与正常的实现没有任何区别
```java
public static void main(String[] args) {
    HelloService proxyHelloService = (HelloService) getProxy(new HelloServiceImpl());
    proxyHelloService.hello("drowyue");
}
```

输出如下：

```
before hello
Hello drowyue
after hello
```



###  三、原理

Proxy.newProxyInstance的核心代码如下，首先，getProxyClass0方法生成代理类，然后通过反射创建该代理类的对象并返回

``` java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    // 生成代理类
    Class<?> cl = getProxyClass0(loader, intfs);
    // 反射生成代理类对象，注意构造函数参数为InvocationHandler
    final Constructor<?> cons = cl.getConstructor(constructorParams);
    final InvocationHandler ih = h;
    return cons.newInstance(new Object[]{h});
}
```

进一步跟踪getProxyClass0方法，并将实际生成代理类保存并反编译如下。代理类继承Proxy并实现了HelloService，m0~m3四个变量分别为equals、toString、hashCode、hello，四个方法。每一个方法的实现都是调用super.h.invoke方法，而父类中的h变量正是前面定义的InvocationHandler，从而实现了将接口中每个方法均代理到InvocationHandler.invoke中。

```java
public final class HelloServiceImpl extends Proxy implements HelloService {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    // 生成参数为InvocationHandler的构造函数
    public HelloServiceImpl(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String hello(String var1) throws  {
        try {
            return (String)super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.drowyue.java.reflect.proxy.HelloService").getMethod("hello", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```




