---
title: Bean的生命周期
tag:
  - 查缺补漏
category: 
  - 走马观花
---
# Bean的生命周期

![img](../img/postimg/bean的生命周期.jpeg)



我觉得Spring的生命周期可以分为四个阶段，实例化、属性赋值、初始化、销毁。



初始化阶段可以细分为多个阶段，

1. 检查aware接口的实现情况：beanNameAware接口、classLoaderAware接口等
2. 执行BeanPostProcessor前置方法；
3. 检查initializingBean接口的实现；
4. 检查是否实现了自定义init方法；
5. 执行BeanPostProcessor后置方法；



销毁阶段可以分为两个阶段，

1. 检查是否实现DisposalBean接口
2. 检查是否实现了自定义的destory方法；





```java
@PostConstruct
publicvoidinit() { 
    System.out.println("MyBean is going through init."); 
} 

@PreDestroy
publicvoiddestroy() {
    System.out.println("MyBean will be destroyed now."); 
}  
```





```java
package com.example.demo;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("Before Initialization : " + beanName);
        if (bean instanceof MyBean) {
            // 对 MyBean 做一些操作
            ((MyBean) bean).setName("Modified by BeanPostProcessor before init");
        }
        return bean; // you can return any other object as well
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("After Initialization : " + beanName);
        if (bean instanceof MyBean) {
            // 对 MyBean 做一些操作
            System.out.println("MyBean after initialization: " + bean);
        }
        return bean; // you can return any other object as well
    }
}
```