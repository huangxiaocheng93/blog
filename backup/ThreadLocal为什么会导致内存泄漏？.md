一直在看到各种文章说ThreadLocal会导致内存泄漏，但是具体是什么场景下会泄漏，具体怎么导致的泄漏实际上都讲的不是很清楚，这次来研究清楚。

# ThreadLocal是个啥？

`ThreadLocal`是一个包装类，被`ThreadLocal`包装的变量会和线程挂钩，实现线程安全。
```java
ThreadLocal<RequestContext> REQUEST_CONTEXT = new ThreadLocal<>();
```
每个使用`REQUEST_CONTEXT`变量的线程都拥有一个`REQUEST_CONTEXT`的副本，这样就实现了`REQUEST_CONTEXT`的线程安全。

# ThreadLocal常用来做什么？
因为ThreadLocal的**线程安全**属性，所以我们常常用他来记录线程相关的信息，例如记录线程的上下文信息。

应用场景举例
在日常的开发过程中，我们经常会遇到需要登录态的接口，在接口处理逻辑中经常会使用到当前登录用户的信息，此时就用ThreadLocal来创建一个线程的上下文信息来记录登录用户信息就很合适，可以在任意地方获取上下文信息，比带着用户信息一直往下传递方便很多：
```java
public class UserInfoContext {
    private static final ThreadLocal<UserInfo> USER_INFO = new ThreadLocal<>();

    public static void setUserInfo(UserInfo userInfo) {
        USER_INFO.set(userInfo);
    }

    public static UserInfo getUserInfo() {
        return USER_INFO.get();
    }


    public static void clean() {
        USER_INFO.remove();
    }

    @Data
    public static class UserInfo {

        private String userId;
    }
}
```
上面面我们实现了一个简单的用户信息的上下文信息，等处理完用户登录信息以后，我们可以使用`setUserInfo`方法将用户信息设置进去，然后在需要使用时，直接使用`getUserInfo`拿出来用户信息，非常的方便。

# ThreadLocal的实现

接下来我们用代码片段加上图解的方式来讲解ThreadLocal的实现，参考我们上面new了一个`ThreadLocal`包装的`UserInfo`对象出来，那它是怎么实现线程安全的呢，答案就在`set`方法里面：
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocal.ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

ThreadLocal.ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocal.ThreadLocalMap(this, firstValue);
}
```
我们可以看到这里面出现了一个`ThreadLocalMap`类型的对象，`ThreadLocalMap`是`ThreadLocal`中的一个内部类。

重点来了，`set`方法中获取了`currentThread`，然后通过`currentThread`来获取`ThreadLocalMap`，通过`getMap`方法，我们可以看出来，`Thread`对象中包含有一个`ThreadLocalMap`对象的实例，到这里`ThreadLocal`是怎么实现线程安全的就清楚了，引文`ThreadLocal`操作的对象就保存在线程里面，妥妥的线程安全。

我们再来看看`ThreadLocalMap`的结构：
```java
static class ThreadLocalMap {
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /**
         * The value associated with this ThreadLocal.
         */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private ThreadLocal.ThreadLocalMap.Entry[] table;
    
}
```

`ThreadLocalMap`是一个类似`HashMap`的结构，如果不知道`HashMap`长啥样，那另外去学习`HashMap`。
这里我们主要关注里面`Entry`的结构，`Entry`使用`ThreadLocal`对象本身作为key，我们传进来的`UserInfo`对象作为value，最最最关键的是它继承了`WeakReference`，也就是弱引用，这个是我们平时不太常见的。


这里简单解释下弱引用：如果一个对象只被其他对象持有弱引用，没有其他强引用，那么这个对象随时都有可能被GC回收。

当我们创建了一个`ThreadLocal`对象，并且调用了`set`方法以后，我们来看看当前对象的引用关系是什么情况（不考虑其他存活对象对`ThreadLocal`和`UserInfo`的关系，主要看他们和线程对象之间的引用关系）：

<img width="514" alt="Image" src="https://github.com/user-attachments/assets/be1a5802-c197-4ea2-a2a7-aec3959ea3fe" />

上图我们可以看出来，当`ThreadLocal`一旦走出了它的作用域，那么它就仅剩了一个弱引用指向它，那么它就岌岌可危，随时都会被回收。

为什么会这么设计呢？我们先来看看，假如这里是强引用，那么`ThreadLocal`对象即使走出了它的作用域，仍然存在一条从`Thread`对象开始的强引用链，那么它的生命周期将会和`Thread`对象一样长，即使它已经用不上了，而且会导致UserInfo对象也无法回收；
那么弱引用会有什么区别呢？弱引用的场景下，当`ThreadLocal`一旦走出了它的作用域，那么它就仅剩了一个弱引用指向它，然后他就会被回收，当`ThreadLocal`被回收以后，`Entry`中的key会变成null，当下次线程的`ThreadLocalMap`对象被操作到的时候，key为null的value也会同步被清空掉，看到这里，应该就清楚这里为什么会设计为弱引用了。

# 再看内存泄露

## 什么是内存泄露

在看`ThreadLocal`什么情况下会导致内存泄露之前，先复习一下什么叫做内存泄露。用简单易懂的方式来描述就是：如果一个对象无法再被使用到

当我们理解了`ThreadLocal`的原理以后就很容易理解内存泄露产生的场景了，我总结有以下几个要素：
1- 使用线程池，线程具有超长的生命周期，例如tomcat线程池等等；
2- `ThreadLocal`在离开作用域之前没有调用`remove`方法；
