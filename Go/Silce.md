# Slice

切片传递的是引用类型，数组传递的值类型，在函数传参的时候会复制一份数组

## 切片底层结构

```go
type slice struct {
   array unsafe.Pointer
   len   int
   cap   int
}
```

![img](https://img.halfrost.com/Blog/ArticleImage/57_3.png)

## make创建silce

```go
func makeslice(et *_type, len, cap int) slice {
	// 根据切片的数据类型，获取切片的最大容量
	maxElements := maxSliceCap(et.size)
    // 比较切片的长度，长度值域应该在[0,maxElements]之间
	if len < 0 || uintptr(len) > maxElements {
		panic(errorString("makeslice: len out of range"))
	}
    // 比较切片的容量，容量值域应该在[len,maxElements]之间
	if cap < len || uintptr(cap) > maxElements {
		panic(errorString("makeslice: cap out of range"))
	}
    // 根据切片的容量申请内存
	p := mallocgc(et.size*uintptr(cap), et, true)
    // 返回申请好内存的切片的首地址
	return slice{p, len, cap}
}
```

## 扩容

- 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）
- 否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍，即（newcap=doublecap）
- 否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的 1/4，即（newcap=old.cap,for {newcap += newcap/4}）直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）
- 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）

**扩容扩大的容量都是针对原来的容量而言的，而不是针对原来数组的长度而言的。**

**如果原数组还有容量可以扩容，所以执行 append() 操作以后，会在原数组上直接操作，所以这种情况下，扩容以后的数组还是指向原来的数组。**

## nil和空切片

### nil

```go
var slice []int
```

![img](https://img.halfrost.com/Blog/ArticleImage/57_7.png)

nil 切片被用在很多标准库和内置函数中，描述一个不存在的切片的时候，就需要用到 nil 切片。比如函数在发生异常的时候，返回的切片就是 nil 切片。nil 切片的指针指向 nil。

### 空切片

空切片一般会用来表示一个空的集合。比如数据库查询，一条结果也没有查到，那么就可以返回一个空切片。

```go
silce := make( []int , 0 )
slice := []int{ }
```

![img](https://img.halfrost.com/Blog/ArticleImage/57_8.png)

空切片和 nil 切片的区别在于，**空切片指向的地址不是nil，指向的是一个内存地址**，但是它没有分配任何内存空间，即底层元素包含0个元素。

不管是使用 nil 切片还是空切片，对其调用内置函数 append，len 和 cap 的效果都是一样的。