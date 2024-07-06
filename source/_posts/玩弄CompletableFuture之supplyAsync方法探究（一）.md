# supplyAsync方法探究（一）

[TOC]



## 要点

1. **异步执行任务**：`supplyAsync` 方法将任务提交到线程池中异步执行，并返回一个 `CompletableFuture` 来表示任务的结果。
2. **任务包装和提交**：`AsyncSupply` 类包装了 `Supplier` 任务，并实现了 `Runnable` 接口，能够在线程池中执行。
3. **任务完成处理**：`AsyncSupply` 在任务完成后会调用 `postComplete` 方法，确保所有依赖任务都能正确地执行。
4. **依赖任务管理**：`Completion` 类负责管理和处理依赖任务，通过链表结构避免无限递归，确保所有回调任务都能按顺序执行。

通过 `CompletableFuture.supplyAsync` 方法，Java 提供了一种强大的方式来处理异步任务及其依赖关系，使得编写异步代码更加简洁和高效。



## 方法介绍

supplyAsync方法是CompletableFuture类里面的一个静态方法，参数是Suppiler（函数式接口，只有一个get方法，实现必须有返回值）和Executor（可选，如果不填写的话，默认是ForkJoinPool）。



代码示例：

```java
public static void main(String[] args) {
    // 创建一个CompletableFuture
    CompletableFuture<Integer> initialFuture = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(1000); // 模拟耗时操作
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
        return 42; // 返回结果
    });

    // 使用thenApply创建依赖
    CompletableFuture<String> dependentFuture = initialFuture.thenApply(result -> {
        System.out.println("Received result: " + result);
        return "Processed result: " + result;
    });
    String res = dependentFuture.get();
    System.out.println(res);
}
```



**通过代码示例，可以看出来，这个方法的作用是将任务进行异步执行，然后将任务的返回值封装到CompletableFuture中。**



## 方法调用链

**supplyAsync内部代码：**

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(asyncPool, supplier);
}

static <U> CompletableFuture<U> asyncSupplyStage(Executor e,
                                                 Supplier<U> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<U> d = new CompletableFuture<U>();
    e.execute(new AsyncSupply<U>(d, f));//核心代码在这里，AsyncSupply是一个内部类，f是我们需要异步执行的任务
    return d;
}


```

**这个内部类实现了Runable接口，所以可以作为一个任务交到线程池里面去执行！**



**AsyncSupply内部类的核心代码，就是线程池执行的代码，**

```java
//这个方法是线程池里面的线程启动后，执行的run方法
public void run() {
    CompletableFuture<T> dependentFuture; 
    Supplier<T> resultSupplier;
    if ((dependentFuture = dep) != null && (resultSupplier = fn) != null) {
        dep = null; fn = null;
        if (dependentFuture.result == null) {
            try {
                // 如果CompletableFuture没有设置结果，则将开头使用的函数式编程的结果设置到里面
                dependentFuture.completeValue(resultSupplier.get());
            } catch (Throwable ex) {
                // 如果在获取结果过程中发生异常，将异常封装到CF里面。
                dependentFuture.completeThrowable(ex);
            }
        }
        //进行后续处理，CompletableFuture编排的核心所在。
        dependentFuture.postComplete();
    }
}

```



**postCompleta代码：**

简单概括就是：
     	1、这段代码通过一个 while 循环处理 CompletableFuture 实例的所有回调任务，确保所有依赖任务都能在 CompletableFuture 完成后正确地执行。
		2、使用 CAS 操作确保线程安全，并通过链表结构避免无限递归。

```java
final void postComplete() {
   
    CompletableFuture<?> f = this; // 初始化 f 为当前 CompletableFuture 实例
    Completion h; // 声明 Completion 类型的变量 h，表示当前栈顶的 Completion 任务
    while ((h = f.stack) != null || // 当 f 的栈顶不为空时进入循环
           (f != this && (h = (f = this).stack) != null)) { // 或者当 f 不是当前实例且其栈顶不为空时进入循环
        CompletableFuture<?> d; // 用于存储新产生的 CompletableFuture 实例
        Completion t; // 当前 Completion 任务的下一个任务
        if (f.casStack(h, t = h.next)) { // 尝试通过 CAS 操作将栈顶设置为其下一个任务
            if (t != null) { // 如果存在下一个任务
                if (f != this) { // 如果 f 不是当前实例
                    pushStack(h); // 将当前任务重新推回栈顶
                    continue; // 继续下一次循环
                }
                h.next = null; // 如果 f 是当前实例，解除当前任务与下一个任务的链接
            }
            // 尝试执行当前任务，如果成功，返回新产生的 CompletableFuture 实例（此处并不会发生覆盖，CF实例互相不干扰）
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}

```

**注：**Completion是一个内部类，类似于维护了一个回调任务的列表，next是Completion的属性，类型是Completion；tryFire是Completion里面的方法，返回CompletableFuture实例。

## 总结

本文从CompletableFuture的静态方法supplyAsync出发，层层递进，探究了该方法是如何进行异步执行任务的，以及探究了CompletableFutur是如何实现异步编排。

