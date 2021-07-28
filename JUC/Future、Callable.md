Runnable没有返回值，不能抛出异常，只能在run方法中try/catch

但是Callable可以

```java 
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

## 用法

### 线程池submit方法返回Future对象

任务提交后线程池会<font color='red'>返回一个空的Future容器</font>，当线程的任务一旦执行完毕，也就是当我们可以获取结果的时候，线程池把该<font color='red'>结果填入</font>到之前给我们的那个Future中，此时我们便可以从Future中获取任务执行的结果

### 线程方法抛出异常

不管线程抛出的是什么异常，future的get方法抛出的都是ExecutionException异常。只有在get的时候才能感知线程抛出了异常

### 用FutureTask来创建Future



## cancel方法

- 如果这个任务<font color='red'>还没开始执行</font>，任务会被正常取消，未来也不会执行，方法返回true
- 如果任务<font color='red'>已完成</font>，或者<font color='red'>已取消</font>，那么cancel方法执行失败，返回false
- 如果<font color='red'>任务正在执行</font>，那么这个方法不会直接取消该任务，而是根据我们填入的参数mayInterruptIfRunning是true还是false做判断：
  - true：给任务发送中断信号。适用于任务能够处理中断
  - false：任务继续执行。适用于不能处理中断任务和不清楚是否支持，以及仅仅为了给还未开始的任务做中断

## FutureTask

- 用来获取Future和任务的结果

- 是一种<font color='red'>包装器</font>，可以把Callable转换成Future和Runnable，它<font color='red'>同时实现二者的接口</font>

  