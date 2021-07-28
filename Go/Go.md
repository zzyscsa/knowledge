# Go协程

一个Go的线程上，可以起多个协程，协程比线程更轻量级，因为线程是直接作用在CPU上的，是操作系统级别的，非常耗费cpu。

特点

- 有独立的栈空间
- 共享程序堆空间
- 调度由用户控制
- 协程是轻量级线程

协程调度模型大多是N:M，意思就是每个线程下可以运行多个协程，而且线程也可以有多个。M远大于N

协程主要是为了提高并发，而且主要是IO并发。协程并不适合并行计算或者并行处理任务，因为同一时刻运行的协程数不可能大于操作系统线程，而且需要协程切换，不如线程池。

## goroutine的调度模型

### MGP模式

- M：操作系统主线程（是物理线程），是真正干活的线程
- P：协程调度执行需要的上下文
- G：代表一个协程

![image-20210124172132766](D:\Typora\picture\image-20210124172132766.png)

阻塞时候的 处理

![image-20210124172157171](D:\Typora\picture\image-20210124172157171.png)



# 垃圾回收

## 发展

v 1.1 ——2013/5 ——STW ——————————百ms-⼏百ms级别
v 1.3 ——2014/6 ——Mark STW, Sweep 并⾏ — 百ms级别
v 1.5 ——2015/8—— 三⾊标记法, 并发标记清除 -10ms级别
v 1.8 ——2017/2—— hybrid write barrier ————sub ms

## 三色标记法

- 有⿊⽩灰三个集合，初始时所有对象都是⽩⾊
- 从Root对象开始标记, 将所有可达对象标记为灰⾊
- 从灰⾊对象集合取出对象, 将其引⽤的对象标记为灰⾊, 放⼊灰⾊集合, 并将⾃⼰标记为⿊⾊
- 重复第三步, 直到灰⾊集合为空, 即所有可达对象都被标记
- 标记结束后, 不可达的⽩⾊对象即为垃圾. 对内存进⾏迭代清扫，回收⽩⾊对象
- 重置GC状态

## 混合写屏障

写屏障：该屏障之前的写操作和之后的写操作相比，先被系统其它组件感知。通俗的讲就是在gc跑的过程中，可以监控对象的内存修改，并对对象进行重新标记。(实际上也是超短暂的stw，然后对对象进行标记)

在上述情况中，`新生成的对象，一律都标位灰色！`

举个例子：

看如下图，一个黑色对象引用了曾经标记的白色对象。

![img](https://b3logfile.com/file/2020/07/solofetchupload839681777523370099-6c568869.webp?imageView2/2/w/1280/format/jpg/interlace/1/q/100)

这时候，写屏障机制被触发，向GC发送信号，GC重新扫描对象并标位灰色。

![img](https://b3logfile.com/file/2020/07/solofetchupload1829703544490445887-b5978f40.webp?imageView2/2/w/1280/format/jpg/interlace/1/q/100)

**`因此，gc一旦开始，无论是创建对象还是对象的引用改变，都会先变为灰色。`**

# Go中常用的并发模型

## 通过channel+select通知实现并发控制

无缓冲的channel需要发送接收**同步**，有缓冲的不需要。

```go
func main() {
	ch := make(chan struct{})
	go func() { 
        fmt.Println("start working")
		time.Sleep(time.Second * 1)
		ch <- struct{}{} 
	}()
	<-ch //会阻塞直到协程完成
}
```

当主 goroutine 运行到 <-ch 接受 channel 的值的时候，如果该 channel 中没有数据，就会一直阻塞等待，直到有值。 这样就可以简单实现并发控制

channel用在比较单纯的goroutine状况下

## 通过sync包中的WaitGroup实现并发控制

在 sync 包中，提供了 WaitGroup ，它会等待它收集的所有 goroutine 任务全部完成。在WaitGroup里主要有三个方法：

- Add, 可以添加或减少 goroutine的数量.
- Done, 相当于Add(-1).
- Wait, 执行后会堵塞主线程，直到WaitGroup 里的值减至0

```go
func main() {
	var wg sync.WaitGroup
	var urls = []string{"http://www.golang.org/", "http://www.google.com/",}
	for _, url := range urls {
		wg.Add(1)
		go func(url string) { 
            defer wg.Done()
			http.Get(url) 
        }(url)
	}
	wg.Wait()
}
```

适合用在把一个job拆成多个子任务，等全部完成后再进行下一步

**内部是使用CAS操作**

## Context上下文

go语言为我们提供的 Context，它就是goroutine 的上下文。 它是包括一个程序的**运行环境、现场和快照等**。每个程序要运行时，都需要知道当前程序的运行状态，通常Go 将这些封装在一个 Context 里，再将它传给要执行的 goroutine 。

context 主要用来在 goroutine 之间传递上下文信息，包括：**取消信号、超时时间、截止时间**、k-v 等。

context 包主要是用来处理多个 goroutine 之间共享数据，及多个 goroutine 的管理。context 包的核心是 struct Context，接口声明如下：

```go
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
	// 当 context 被取消或者到了 deadline，返回一个被关闭的 channel
    Done() <-chan struct{}
	// 在 channel Done 关闭后，返回 context 取消原因
    Err() error
    // 返回 context 是否会被取消以及自动取消时间（即 deadline）
    Deadline() (deadline time.Time, ok bool)
 	// 获取 key 对应的 value
    Value(key interface{}) interface{}
}
```

Done() 返回一个只能接受数据的channel类型，当该context关闭或者超时时间到了的时候，该channel就会有一个**取消信号**

Err() 在Done() 之后，返回context 取消的原因。

Deadline() 设置该context cancel的时间点

Value() 方法允许 Context 对象携带request作用域的数据，该数据必须是线程安全的。

Context 对象是线程安全的，你可以把一个 Context 对象传递给任意个数的 gorotuine，对它执行 取消 操作时，所有 goroutine 都会接收到取消信号。

一个 Context 不能拥有 Cancel 方法，同时我们也只能 Done channel 接收数据。 其中的原因是一致的：接收取消信号的函数和发送信号的函数通常不是一个。 典型的场景是：父操作为子操作操作启动 goroutine，子操作也就不能取消父操作。

适合一次控制全部goroutine

**context底层实现是mutex和channel的结合，前者用于初始部分参数，后者用于通信**

