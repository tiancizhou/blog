---
title: JDK代理和CGLib代理的区别
tag:
  - 查缺补漏
category: 
  - 走马观花
---
# JDK动态代理和CGLib动态代理的区别

JDK和CGLib代理都是Java默认动态代理实现机制；

JDK代理的实现原理是：

1、创建一个拦截器，实现InvocationHandler接口，重写invoke方法，在这个方法里面实现代理的逻辑。

2、生成代理对象实例，通过Proxy.newProxyInstance(被代理对象的类加载器，被代理对象的实现接口，实现invovationHandler接口的拦截器类)，返回的就是代理类的实例，实际上就是对接口的实现类。

```java
//1、创建短信发送接口
public interface SmsService {
    String send(String message);
}

//2.实现发送短信的接口public class SmsServiceImpl implements SmsService {
public String send(String message) {
    System.out.println("send message:" + message);
    return message;
}

//3、定义一个JDK动态代理类

public class DebugInvocationHandler implements InvocationHandler {
    /**
     * 代理类中的真实对象
     */
    private final Object target;

    public DebugInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method " + method.getName());
        Object result = method.invoke(target, args);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method " + method.getName());
        return result;
    }
}

//4、获取代理对象的工厂类
public class JdkProxyFactory {
    public static Object getProxy(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(), // 目标类的类加载器
                target.getClass().getInterfaces(),  // 代理需要实现的接口，可指定多个
                new DebugInvocationHandler(target)   // 代理对象对应的自定义 InvocationHandler
        );
    }
}

//5、实际使用
SmsService smsService = (SmsService) JdkProxyFactory.getProxy(new SmsServiceImpl());
smsService.send("java");
```



由于JDK动态代理，被代理的类必须实现了接口，所以对于那些没有实现接口的类，就无法使用JDK动态代理来进行代理。转而使用CGLib动态代理。



CGLib的实现原理是：

1、创建一个拦截器，实现MethodInterceptor接口，重写interceptor方法，在这个方法里面实现代理的逻辑。

2、通过Enhancer类，获取代理对象。



```java
//1、创建一个拦截器
public class DebugMethodInterceptor implements MethodInterceptor {


    /**
     * @param o           被代理的对象（需要增强的对象）
     * @param method      被拦截的方法（需要增强的方法）
     * @param args        方法入参
     * @param methodProxy 用于调用原始方法
     */
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method " + method.getName());
        Object object = methodProxy.invokeSuper(o, args);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method " + method.getName());
        return object;
    }

}

//2、通过Enhancer创建代理类
import net.sf.cglib.proxy.Enhancer;
public class CglibProxyFactory {

    public static Object getProxy(Class<?> clazz) {
        // 创建动态代理增强类
        Enhancer enhancer = new Enhancer();
        // 设置类加载器
        enhancer.setClassLoader(clazz.getClassLoader());
        // 设置被代理类
        enhancer.setSuperclass(clazz);
        // 设置方法拦截器
        enhancer.setCallback(new DebugMethodInterceptor());
        // 创建代理类
        return enhancer.create();
    }
}

AliSmsService aliSmsService = (AliSmsService) CglibProxyFactory.getProxy(AliSmsService.class);
aliSmsService.send("java");
```



由此可见JDK动态代理和CGLib动态代理区别：

1. 前者只能代理实现了接口的类，而后者可以代理没有实现任何接口的类。
2. 前者是通过实现接口来进行增强代理，后者是通过生成子类进行增强代理。