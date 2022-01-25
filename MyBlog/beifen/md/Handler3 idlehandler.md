Handler3 idlehandler



Android IdleHandler 原理浅析

```java
public static interface IdleHandler {
    boolean queueIdle();
}
```



```java
Looper.myQueue().addIdleHandler(new IdleHandler() {  
    @Override  
    public boolean queueIdle() {  
        //你要处理的事情
        return false;    
    }  
});
```

## MessageLogging

Looper中提供了mLogging，可以用来监控卡顿。这个监控卡顿的方法是基于消息队列实现，通过替换Looper的Printer实现。



https://segmentfault.com/a/1190000021135856?utm_source=tag-newest