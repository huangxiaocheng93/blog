# 背景

`ThreadLocal` 是什么，什么场景会出现内存泄露，为什么会导致内存泄漏，我们能应该怎么用。读了挺多文章感觉都解释的不是很清楚，现在尝试自己梳理一次，尝试讲清楚这些问题。

# ThreadLocal是什么

`ThreadLocal`可以用来包装一个变量，它的作用是未每一个线程提供一个变量的副本，避免线程间共享的问题。直白点说就是被`ThreadLocal` 包装的变量是线程安全的，申明方式如下：

```java
ThreadLocal<RequestContext> REQUEST_CONTEXT = new ThreadLocal<>();

```

每个使用`REQUEST_CONTEXT`变量的线程都拥有一个`REQUEST_CONTEXT`的副本，这样就实现了`REQUEST_CONTEXT`对象的线程安全。

# ThreadLocal是怎么实现线程安全的

下面通过源码片段来看一下`ThreadLocal`是怎么实现线程安全的，答案就在`set`方法里面：

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

重点来了，`set`方法中获取了`currentThread`，然后通过`currentThread`来获取`ThreadLocalMap`，通过`getMap`方法，我们可以看出来，`Thread`对象中包含有一个`ThreadLocalMap`对象的实例，到这里`ThreadLocal`是怎么实现线程安全的就清楚了，因为`ThreadLocal`操作的对象就保存在线程里面，妥妥的线程安全。

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

# ThreadLocal为什么会出现内存泄漏

## 什么是内存泄漏

在看`ThreadLocal`什么情况下会导致内存泄漏之前，先复习一下什么叫做内存泄漏。

用比较简单易懂的方式来说明：内存泄漏就是一个对象既无法被获取到引用，又无法正常被GC回收，那么我们就可以说这个对象出现了内存泄漏。

我们分析一下内存泄漏对象的特征：

1. 无法正常获取到它的引用，说明代码已经走过了它的作用域，无法再拿到它的引用了；
2. 无法被GC回收，由于当下GC基本都是用可达性分析算法，一个对象无法被GC回收说明它还关联在根节点上；

## ThreadLocal的弱引用设计

ThreadLocal的弱引用设计用在`ThreadLocalMap`的key上面，`Entry`使用`ThreadLocal`对象作为key，它继承了`WeakReference`，也就是弱引用，这个是我们平时不太常见的。

这里简单解释下弱引用：如果一个对象只被其他对象持有弱引用，没有其他强引用，那么这个对象随时都有可能被GC回收。

接下来用一段代码来举例说明：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalLeakRealDemo {
    private static final ExecutorService executor = Executors.newFixedThreadPool(5);

    public static void main(String[] args) {
        while (true) {

            executor.execute(() -> {
                LeakyContainer container = new LeakyContainer();
                container.doWork();
            });

            try { Thread.sleep(100); }
            catch (InterruptedException e) { e.printStackTrace(); }
        }
    }

    static class LeakyContainer {
        private final ThreadLocal<byte[]> threadLocal = new ThreadLocal<>();

        void doWork() {
            try {
                threadLocal.set(new byte[1024 * 1024]); // 1MB
                System.out.println(Thread.currentThread().getName() + " set value");
            } finally {

            }
        }
    }
}

```

当这段代码执行到`System.out.println(Thread.currentThread().getName() + " set value");`时，我们来分析下对象的引用关系：

![Image](https://github.com/user-attachments/assets/d507ac67-e43b-4864-8d4b-028237c92407)

图上使用实线代表强引用，虚线代表弱引用，箭头指向被引用的方向。通过这个图可以看出来：此时threadLocalMap持有了threadLocal对象的弱引用，其他的引用关系都是强引用。

当代码继续执行，从doWork方法退出，走到`Thread.sleep(100); `是，可以思考一下引用关系会是什么样子，如下图：

![Image](https://github.com/user-attachments/assets/45fd8258-5016-4dd7-b7b7-e9046994294d)

此时，线程中的代码已经执行完成了，所以线程栈已经被回收了，container对象也离开了作用域，所以可以正常被GC回收，所以也忽略它。但是Thread对象的引用由于被线程池持有，所以线程对象不会被回收。

现在我们发现了一个问题，byte[]对象已经没有可能再被使用到了，但是仍然存在一条强引用链导致它无法被回收。

这就是`ThreadLocalMap` 将key设计为弱引用的答案：**为了减小内存泄漏带来的影响**。

## 弱引用在怎样解决问题

上图可以看出来，`ThreadLocal`仅剩了一个弱引用指向它，那么它随时都会被回收。当它被回收以后，`ThreadLocalMap` 里面就出现了一个key为null的键值对，那么`ThreadLocalMap` 就能很清楚的知道哪些`value`是不会再被访问的，在后续的操作用就可以做一些清除的动作。

在这种情况下，即使用户没有主动清理，在后续的流程中`ThreadLocalMap` 也可以主动的做出一些清理的动作来避免内存泄露。

我们再假设这里是强引用，看看有什么不同。那么`ThreadLocal`对象即使走出了它的作用域，仍然存在一条从`Thread`对象开始的强引用链，那么它的生命周期将会和`Thread`对象一样长，即使它已经用不上了，那么这个时候如果没有被主动清理，`ThreadLocalMap` 也无法知道哪些对象是无效的，所以就无法做清理。
那么在哪些动作中，`ThreadLocal` 回去做清除动作呢？

在`set`和`get` 方法中都会调用到清理的逻辑。

## ThreadLocal的内存泄漏

当搞懂了上面的内容以后就很容易理解`ThreadLocal`内存泄漏产生的场景了，举个例子来看：


上面的例子有如下几个特征：

1、使用了线程池，线程具有很长的生命周期；

2、线程每次执行`doWork`方法时都创建了新的`LeakyContainer`对象，也就意味每次都是新的`threadLocal`对象；

3、doWork方法在执行完毕后并没有清理`threadLocal` ；

接下来我们分析下OOM的过程：

线程每执行一次都往内存塞进去1MB大小的数据；

等到内存占用到了临界值就会触发GC，但是由于线程都是存活状态，所以这次GC只能回收`threadLocal` 对象，也就是key被回收了，可以想象回收效果并不好；

然后`threadLocal` 继续调用`set`方法去塞数据，此时由于产生了一些key为null的value，所以会触发`threadLocal` 的清理策略，但是需要注意的是`threadLocal` 的策略并不是触发一次就清掉所有异常的value，只会清理操作相关的，所以清理速度肯定是跟不上写入的速度；

程序继续执行，GC无法完成数据清理，`threadLocal` 本身的清理速度也跟不上写入，最终结果只能是OOM。

# 总结

# ThreadLocal常用来做什么

因为ThreadLocal的**线程安全**属性，所以我们常常用他来记录线程相关的信息，例如记录线程的上下文信息。

## 应用场景举例

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

上面实现了一个简单的用户信息的上下文信息，等处理完用户登录信息以后，我们可以使用`setUserInfo`方法将用户信息设置进去，然后在需要使用时，直接使用`getUserInfo`拿出来用户信息，非常的方便。

## ThreadLocal的弱引用设计到底有没有用

个人认为是有用，但是作用并不大。

## ThreadLocal应该怎么使用才正确

事实上一开始举的UserInfo的例子就是非常正确的使用方式，需要保证如下几点：

1、`USER_INFO` 被`static` 修饰，那么它一定有一个强应用，就永远都不会被回收；

2、我们每次set的时候使用的都是同一个threadLocal对象，这样每次都会覆盖原来的数据，同一个线程内不会出现多份UserInfo数据；

3、在使用完后调用remove方法，主动将UserInfo移除，这样做有两个好处，一是避免可能出现的内存泄漏，二是避免线程复用时读取到错误的上下文信息；