# Go管道

- 本质是一种数据结构——队列
- 线程安全，多goroutine访问时，不需要加锁
- channel是有类型的，一个channel只能存放一种类型数据

内置的close关闭channel的时候，不能往里面写，但是还可以读，同时，如果关闭了管道，这个语句ok会返回false

```go
v, ok := -<ch
```

如果编译器发现一个管道只有写而没有协程去读，就会报dead lock错误，但是如果有协程读，即便读的很慢，写的协程不过是阻塞着，不会报错

## 为啥是线程安全的？

channel在底层是一个hchan结构体，位于src/runtime/chan.go里

```go
type hchan struct {
	//channel队列里面总的数据量
	qcount   uint           // total data in the queue
	// 循环队列的容量，如果是非缓冲的channel就是0
	dataqsiz uint           // size of the circular queue
	// 缓冲队列，数组类型。是一个环形队列
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	// 元素占用字节的size
	elemsize uint16
	// 当前队列关闭标志位，非零表示关闭
	closed   uint32
	// 队列里面元素类型
	elemtype *_type // element type
	// 队列send索引
	sendx    uint   // send index
	// 队列索引
	recvx    uint   // receive index
	// 等待channel的G队列。
	recvq    waitq  // list of recv waiters
	// 向channel发送数据的G队列。
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	// 全局锁
	lock mutex
}
```

假如我们创建一个带buffer的channel，底层数据模型是：

![img](https://img-blog.csdnimg.cn/20190307092857857.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTMyNjE=,size_16,color_FFFFFF,t_70)

### 协程写入数据

会直接把数据存入circular queue(send)

![img](https://img-blog.csdnimg.cn/20190105123351626.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTMyNjE=,size_16,color_FFFFFF,t_70)

1. 先获取全局锁；
2. 然后enqueue元素(通过移动拷贝的方式)；
3. 释放锁；



**<font color='blue'>如果这个时候队列写入3个task，要写入第4个task怎么办？</font>**

这时候G1线程会设置为waiting状态，gmp模型中m会去执行另一个处于就绪态的goroutine

**<font color='blue'>那么blocked的G1如何恢复？</font>**

实际上hchan数据结构里面的sendq存储了sender的等待队列，其原型如下：

![img](https://img-blog.csdnimg.cn/20190307093911177.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTMyNjE=,size_16,color_FFFFFF,t_70)

等待队列里面是sudog结构的**单链表**，sudog持有一个G代表goroutine对象引用，elem代表channel里面保存的元素。当G1执行`ch<-task4`的时候，G1会创建一个sudog然后保存进入sendq队列，实际上hchan结构如下图：

![img](https://img-blog.csdnimg.cn/20190307094057854.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTMyNjE=,size_16,color_FFFFFF,t_70)

这个时候如果G2读取一个数据，其过程如下：

1. G2调用 `t:=<-ch` 获取一个元素；
2. 从channel的buffer里面取出一个元素task1；
3. **从sender等待队列里面pop一个sudog；**
4. 将task4复制buffer中task1的位置，然后更新buffer的sendx和recvx索引值；
5. 这时候需要将G1置为Runable状态，表示G1可以恢复运行，G2会唤醒G1，G1进入gmp队列中等待调度；

### 协程读取数据

![img](https://img-blog.csdnimg.cn/20190105123802359.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTMyNjE=,size_16,color_FFFFFF,t_70)

1. 先获取全局锁；
2. 然后dequeue元素(通过移动拷贝的方式)；
3. 释放锁；

**<font color='blue'>读取空channel怎么办？</font>**

![img](https://img-blog.csdnimg.cn/201903070942045.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTMyNjE=,size_16,color_FFFFFF,t_70)

和上面类似，会创建一个代表G2的sudog存入recvq等待队列，然后G2进入waiting。这个时候如果G1写入，就将task存入buffer，然后唤醒G2。

但是有一个更直接的办法，direct send；就是G1直接把数据写入到G2中的elem中，这样就不用走G2中的elem复制到buffer中，再从buffer复制给G1。