一直在看到各种文章说ThreadLocal会导致内存泄漏，但是具体是什么场景下会泄漏，具体怎么导致的泄漏实际上都讲的不是很清楚，这次来研究清楚。

# ThreadLocal是个啥？

`ThreadLocal`是一个包装类，被`ThreadLocal`包装的变量会和线程挂钩，实现线程安全。例如我们常用的使用场景：在Http请求中记录请求的上下文信息。
```
private static final ThreadLocal<RequestContext> REQUEST_CONTEXT = new ThreadLocal<>();
```
每个使用`REQUEST_CONTEXT`变量的线程都拥有一个`REQUEST_CONTEXT`的副本，这样就实现了`REQUEST_CONTEXT`的线程安全。

# ThreadLocal的实现



