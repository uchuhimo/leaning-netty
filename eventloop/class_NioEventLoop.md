
```java
public final class NioEventLoop extends SingleThreadEventLoop {

}
```



```java
@Override
protected Queue<Runnable> newTaskQueue() {
    // This event loop never calls takeTask()
    return PlatformDependent.newMpscQueue();
}
```