# Map

Go中的map哈希冲突用的是<font color='red'>链地址法</font>，但是它的实现并不是对冲突的元素采用链表存储，而是采用了数组的形式。

## Map底层结构

```go
type hmap struct {
    // 代表当前哈希表中的元素个数，len(map) 返回的就是该字段值
	count     int 
    // 状态标识，比如正在被写、buckets 和 oldbuckets 在被遍历、等量扩容(Map扩容相关字段)
	flags     uint8
    // buckets（桶）的数量的对数，也就是说该哈希表中桶的数量为 2^B 个
	B         uint8
    // 溢出桶的大致数量
	noverflow uint16
    // 哈希种子，这个值在哈希创建时随机生成，并在计算 key 的哈希的时候会传入哈希函数，以此提高哈希函数的随机性
	hash0     uint32 // hash seed
    // 指向 buckets 数组的指针，数组大小为 2^B，如果元素个数为 0，它为 nil。
	buckets    unsafe.Pointer
    // 如果发生扩容，oldbuckets 是指向老的 buckets 数组的指针，老的 buckets 数组大小是新的buckets 的 1/2。非扩容状态下，它为 nil。
	oldbuckets unsafe.Pointer
    // 表示扩容进度，小于此地址的 buckets 代表已搬迁完成。
	nevacuate  uintptr
    // 这个字段是为了优化 GC 扫描而设计的。当 key 和 value 均不包含指针，并且都可以 <=128 字节时使用。extra 是指向 mapextra 类型的指针。
	extra *mapextra
}
```

## bmap

buckets是一个指针，指向一个类型为bmap的**<font color='red'>结构体数组</font>**（是一个数组！），也就是具体存储 map 键值对的哈希空间

```go
type bmap struct {
	// tophash 包含此桶中每个键的哈希值最高字节（高8位）信息。
    // 如果tophash[0] < minTopHash，tophash[0]则代表桶的搬迁（evacuation）状态。
	tophash [bucketCnt]uint8
}
```

这里的 `tophash` 指的是哈希值的高八位，在 Go 中，Hash 值的分布如下，高八位即 `high-order bits` 部分：

![img](https://img-blog.csdnimg.cn/img_convert/bb3eaa1b6a042ab8bdb57c6cc1e3f864.webp?x-oss-process=image/format,png)

在运行期间，bmap的结构不只只有tophash字段，**因为哈希表中可能存储不同类型的键值对（例如声明了接口类型），而且 Go 语言也不支持泛型，所以键值对占据的内存空间大小只能在编译时进行推导**。`bmap` 中的其他字段在运行时也都是通过计算内存地址的方式访问的，所以它的定义中就不包含这些字段。所以在编译期间通过 `cmd/compile/internal/gc.bmap` 函数重建了它的结构，动态地创建一个新的结构：

```go
type bmap struct {
    //hash值的高八位
    topbits  [8]uint8
    // key 的数组
    keys     [8]keytype
    // value 的数组
    values   [8]valuetype
    // 对齐内存使用的，不是每个 bmap 都有会这个字段，需要满足一定条件
    pad      uintptr
    // 溢出桶，也是指向一个 bmap，上面的字段 topbits、keys、elems 长度为 8，最多存8组键值对，存满了就往指向的这个 bmap 里存
    overflow uintptr
}
```

一个bmap的模型如下:

![img](https://img-blog.csdnimg.cn/img_convert/5c56ef7c4106760234fcaf4ba55c31aa.webp?x-oss-process=image/format,png)

可以看出几点：

- **key和value是各自存起来的**，不是key/value/key/value的形式，好处是能消除对齐填充所需要的字段（padding）
- 一个桶中**最多存放8个键值对**，如果还有多余的键落入该桶中，就需要再构建一个桶，称为<font color='red'>溢出桶</font>，用overflow指针连起来

## mapextra

当 map 的 `key` 和 `value` 都不是指针，并且 `size` 都小于 128 字节的情况下（**键和值超过 128 个字节后，会被转换成指针**），会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。但实际上，bmap有一个指针，就是overflow字段，那么这个时候就会把overflow移动到extra字段来，这个字段里的指针指向溢出桶。

```go
type mapextra struct {
   
   // 如果 key 和 value 都不包含指针，并且可以被 inline(<=128 字节)
   // 就使用 hmap 的 extr a字段来存储 overflow buckets，这样可以避免 GC 扫描整个 map
   // 然而 bmap.overflow 也是个指针。所以其实 bmap.overflow 的指针也是指向了
   // hmap.extra.overflow 和 hmap.extra.oldoverflow 中
   // overflow 包含的是 hmap.buckets 的 overflow 的 buckets
   // oldoverflow 包含扩容时的 hmap.oldbuckets 的 overflow 的 bucket
   overflow    *[]*bmap
   oldoverflow *[]*bmap

   // 指向空闲的 overflow bucket 的指针
   nextOverflow *bmap
}
```

**所以实际上 `bmap.overflow` 和 `hmap.extra.overflow` 所指向的地址是一样的，都是溢出桶的内存地址，只是在某些特殊情况下用 `hmap.extra.overflow` 代替 `bmap.overflow` ，从而优化了 GC 过程。**

## map整体结构

![img](https://img-blog.csdnimg.cn/img_convert/a3647de3340e08b6700ce3e15cb9d30a.webp?x-oss-process=image/format,png)

## 一些常量

```go
const (
	// 一个桶中最多容纳的键值对的对数，也就是一个桶最多容纳 2^3=8 个
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits

    // 触发扩容的装载因子为 13/2=6.5
	loadFactorNum = 13
	loadFactorDen = 2

	// 键和值超过 128 个字节后，会被转换成指针
	maxKeySize  = 128
	maxElemSize = 128

	// 数据偏移量，大小为 bmap 结构体的大小，它需要正确的对齐，
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)
	// 每个桶（如果有溢出，则包含它的 overflow 的链桶）在搬迁完成状态（evacuated* states）下，
    // 要么会包含它所有的键值对，要么一个都不包含（但不包括调用 evacuate() 方法阶段，
    // 该方法调用只会在对 map 发起 write 时发生，在该阶段其他 goroutine 是无法查看该map的（map 非并发安全））。
    // 简单的说，在非写过程的状态中，桶里的数据要么一起搬走，要么一个都还未搬。
    // tophash 除了放置正常的高 8 位 hash 值，还会存储一些特殊状态值（标志该 cell 的搬迁状态）。
    // 正常的tophash值，最小应该是5，以下列出的就是一些特殊状态值：
    // 表示 cell 为空，并且比它高索引位的 cell 或者 overflows 中的 cell 都是空的。（初始化 bucket 时，就是该状态）
    emptyRest      = 0
    // 空的cell，cell已经被搬迁到新的bucket
	emptyOne       = 1
    // 键值对已经搬迁完毕，key 在新 buckets 数组的前半部分
	evacuatedX     = 2
    // 键值对已经搬迁完毕，key 在新 buckets 数组的后半部分
	evacuatedY     = 3
    // cell 为空，整个 bucket 已经搬迁完毕
	evacuatedEmpty = 4
    // tophash的最小正常值
	minTopHash     = 5
	// flags
    // 可能有迭代器在使用 buckets
	iterator     = 1
    // 可能有迭代器在使用 oldbuckets
	oldIterator  = 2
    // 有协程正在向 map 写入 key
	hashWriting  = 4
    // 等量扩容
	sameSizeGrow = 8
	// 用于迭代器检查的 bucket ID
	noCheck = 1<<(8*sys.PtrSize) - 1
)
```

## key定位

假定 key 经过哈希计算后得到 **64bit** 位的哈希值。如果 B=5，buckets 数组的长度，即桶的数量是 32（2 的 5 次方）。

![img](https://img-blog.csdnimg.cn/img_convert/3fb27bfcdbd4b275a083d2189d1b712d.png)

哈希值低位（`low-order bits`）用于**<font color='red'>选择桶</font>**，哈希值高位（`high-order bits`）用于**<font color='red'>在一个独立的桶中区别出键</font>**。当 B 等于 5 时，那么选择的哈希值低位也是 5 位，即 01010，它的十进制值为10，代表 10 号桶。再用哈希值的高 8 位，找到此 key 在桶中的位置。最开始桶中还没有 key，那么新加入的 key 和 value 就会被放入第一个 key 空位和 value 空位。

当两个不同的 key 落在了同一个桶中，这时就发生了哈希冲突。go 的解决方式是**链地址法**（这里只描述非扩容且该 key 是第一次添加的情况）：在桶中按照顺序寻到第一个空位并记录下来，后续在该桶和它的溢出桶中均未发现存在的该 key，将 key 置于第一个空位；否则，去该桶的溢出桶中寻找空位，如果没有溢出桶，则添加溢出桶，并将其置溢出桶的第一个空位。

例如，下图中的 B 值为 5，所以桶的数量为 32。通过哈希函数计算出待插入 key 的哈希值，低 5 位哈希00110，对应于 6 号桶；高 8 位10010111，十进制为 151，由于桶中前 6 个 cell 已经有正常哈希值填充了(遍历)，所以将 151 对应的高位哈希值放置于第 7 位cell（第8个 cell 为empty Rest，表明它还未使用），对应将 key 和 value 分别置于相应的第七个空位。

![img](https://img-blog.csdnimg.cn/img_convert/91a9ad4e811e0dbd5303147f05680be6.webp?x-oss-process=image/format,png)

## 查找key

通过 `key` 查找 `value` 的方式有以下两种：

```go
v     := hash[key] // => v     := *mapaccess1(maptype, hash, &key)
v, ok := hash[key] // => v, ok := mapaccess2(maptype, hash, &key)
```

- 当接受一个参数时，会使用 `mapaccess1`，该函数仅会返回一个指向目标值的指针；
- 当接受两个参数时，会使用 `mapaccess2`，除了返回目标值之外，它还会返回一个用于表示当前键对应的值是否存在的 `bool` 值。

mapaccess2和mapaccess1差不多，不过是多返回了一个bool类型

`mapaccess1` 查找 `key` 的代码实现如下：

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 如果开启了竞态检测 -race
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
    // 如果开启了 memory sanitizer -msan
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
    // 如果 map 为空或者元素个数为 0，返回零值
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.hasher(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
    // 注意，这里是按位与操作
    // 当 h.flags 对应的值为 hashWriting（代表有其他 goroutine 正在往 map 中写 key）时，
    // 那么位计算的结果不为 0，因此抛出以下错误。
	// 这也表明，go 的 map 是非并发安全的
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
    // 不同类型的 key，会使用不同的 hash 算法，这里获取 hash 值
	hash := t.hasher(key, uintptr(h.hash0))
    // 返回 1 << b-1，即 low-order bits，用于下面与操作筛选出对应的 bucket
	m := bucketMask(h.B)
    // 按位与操作，找到对应的 bucket
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	// 如果 oldbuckets 不为空，那么证明 map 发生了扩容
  	// 如果有扩容发生，老的 buckets 中的数据可能还未搬迁至新的 buckets 里
  	// 所以需要先在老的 buckets 中找
    if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// 增量扩容情况下，老 buckets 数组的大小是原来的一半
			m >>= 1
		}
        // 找到 oldbucket 地址
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 如果在 oldbuckets 中 tophash[0] 的值，为 evacuatedX、evacuatedY，evacuatedEmpty 其中之一
		// 则 evacuated() 返回为true，说明搬迁完成。
    	// 因此，只有当搬迁未完成时，才会从此 oldbucket 中遍历
		if !evacuated(oldb) {
			b = oldb
		}
	}
    // 取出当前 key 值的 tophash 值，即高八位的值
	top := tophash(hash)
bucketloop:
    // 以下是查找的核心逻辑
  	// 双重循环遍历：外层循环是从桶到溢出桶遍历；内层是桶中的 cell 遍历
  	// 跳出循环的条件有三种：
    // 1. 第一种是已经找到 key 值；
    // 2. 第二种是当前桶再无溢出桶；
  	// 3. 第三种是当前桶中有 cell 位的 tophash 值是 emptyRest，这个值它代表此时的桶后面的 cell 还未利用，所以无需再继续遍历。
    // 初识时 b 为 key 所在的桶，此时是在正常桶中
	for ; b != nil; b = b.overflow(t) {
        // 该桶存放最多 8 个键值对，依次对比 tophash 值是否相等
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
                // 不相等且满足第三种条件，跳出循环
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
                // 不相等继续遍历
				continue
			}
            // 因为在 bucket 中 key 是用连续的存储空间存储的，因此可以通过 bucket 地址 + 数据偏移量（bmap 结构体的大小，也就是 tophash 占用的大小，因为 dataOffset 定义的是声明的 bmap）+ keysize 的大小，得到 k 的地址
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // 如果 key 是指针（当键超过 maxKeySize（128个字节）时会转换为指针），则需要进行解引用
			if t.() {
				k = *((*unsafe.Pointer)(k))
			}
            // 判断 key 是否相等
			if t.key.equal(key, k) {
                // 同理，value 的地址也是相似的计算方法，只是再要加上8个 keysize 的内存地址
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                // 如果 value 是指针，解引用
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
                // 返回找到的值
				return e
			}
		}
	}
    // 所有的bucket都未找到，则返回零值
	return unsafe.Pointer(&zeroVal[0])
}
```

1. 判断 map 是否为空，为空的话返回零值
2. 通过按位与的操作检测 map 是否有其他线程在进行写入，有的话则抛出错误。这也表明 map **不是并发安全的**
3. 对 key 取 hash 获得哈希值，通过**哈希值的低位确定在哪一个 bucket**
4. 判断是否发生了扩容，如果发生了扩容并且键值搬迁未完成，则需要先到 oldbuckets 中查找
5. 开始内外层循环遍历，依次对比 tophash 是否相等，直到找到对应的 key 或者找不到退出循环，查找过程结束。外层循环是从桶到溢出桶遍历；内层是桶中的 cell 遍历

![img](https://img-blog.csdnimg.cn/img_convert/c3f9e377aa4a0f6503b3657994b58f6e.webp?x-oss-process=image/format,png)

## 写入key

向 map 中插入或者修改 `key`，最终调用的是 `mapassign` 函数。

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 如果 h 是空指针，赋值会引起panic
    // 例如以下语句
    // var m map[string]int
    // m["k"] = 1
    if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
    // 如果开启了竞态检测 -race
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
    // 如果开启了memory sanitizer -msan
	if msanenabled {
		msanread(key, t.key.size)
	}
    // 同样检查是否有其他 goroutine 正在对 map 进行 key 写入，有的话抛出错误
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
    // 获取哈希值
	hash := t.hasher(key, uintptr(h.hash0))

	// 将 flags 的值与 hashWriting 做按位"异或"运算并赋值到 flags
   	// 因为在当前 goroutine 可能还未完成 key 的写入，再次调用 t.hasher 会发生 panic。
	h.flags ^= hashWriting

    // 这种情况在初始化 map 并且 hint<=8(bucketCnt)时，由于直接在堆上进行分配，所以会出现
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    // bucketMask 返回值是 2 的 B 次方减 1，即 low-order bits
   	// 因此，通过 hash 值与 bucketMask 返回值做按位与操作获取 low-order bits，
    // 返回在 buckets 数组中的第几号桶
	bucket := hash & bucketMask(h.B)
    // 如果 map 正在搬迁（即h.oldbuckets != nil）中,则先进行搬迁工作。
	if h.growing() {
		growWork(t, h, bucket)
	}
    // 计算出上面求出的 bucket 的内存位置
   	// post = start + bucketNumber * bucketsize
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	// 获取高八位哈希值
    top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
bucketloop:
	for {
        // 遍历桶中的 8 个 cell
		for i := uintptr(0); i < bucketCnt; i++ {
        
            // 这里分两种情况：
            
        	// 第一种情况是 cell 位的 tophash 值和当前 tophash 值不相等
       		// 在 b.tophash[i] != top 的情况下，这个位置有可能会是一个空槽位
       		// 一般情况下 map 的槽位分布是这样的，e 表示 empty:
       		// [h0][h1][h2][h3][h4][e][e][e]
       		// 但在执行过 delete 操作时，可能会变成这样:
       		// [h0][h1][e][e][h5][e][e][e]
       		// 所以如果再插入的话，会尽量往前面的位置插
       		// [h0][h1][e][e][h5][e][e][e]
       		//          ^
       		//          ^
       		//       这个位置
       		// 所以在循环的时候还要顺便把前面的空位置先记下来
       		// 因为有可能在后面会找到相等的 key，也可能找不到相等的 key
			if b.tophash[i] != top {
                // 当前是一个空槽位并且这是第一个空位
				if isEmpty(b.tophash[i]) && inserti == nil {
                    // 记录这个空闲的位置tophash的地址，用于后面赋值，同时获取将要插入 k，v 位置的地址
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
                // 查找到末尾了，break 跳出循环
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
            // 第二种情况是 cell 位的 tophash 值和当前的 tophash 值相等
            // 查找当前 cell 位的 key 值
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
            // 注意，即使当前 cell 位的 tophash 值相等，不一定它对应的 key 也是相等的
            // 所以还要做一个 key 值判断，判断如果 key 值不相等，记录下一轮遍历
			if !t.key.equal(key, k) {
				continue
			}
			// 如果已经有该 key 了，就更新它
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
            // 这里获取到了要插入 key 对应的 value 的内存地址
       		// pos = start + dataOffset + 8*keysize + i*elemsize
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			// 如果顺利到这，就直接跳到done的结束逻辑中去
            // 这种情况下不会触发下面的情况，直接跳到 done
            goto done
		}
        // 如果桶中的 8 个 cell 遍历完，还未找到对应的空 cell 或覆盖 cell，
        // 那么就进入它的溢出桶中去遍历
		ovf := b.overflow(t)
        // 如果连溢出桶中都没有找到合适的 cell，跳出循环。
		if ovf == nil {
			break
		}
		b = ovf
	}

	
    // 在已有的桶和溢出桶中都未找到合适的 cell 供 key 写入，那么有可能会触发以下两种情况
  	
    // 情况一：
  	// 判断当前 map 的装载因子是否达到设定的 6.5 阈值，或者当前 map 的溢出桶数量是否过多。如果存在这两种情况之一，则进行扩容操作。
  	// 注意，这里 hashGrow() 实际并未完成扩容，对哈希表数据的搬迁（复制）操作是通过 growWork() 来完成的。
  	// 重新跳入 again 逻辑，在进行完 growWork() 操作后，再次遍历新的桶。
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        // 此处涉及的扩容机制后面讲
		hashGrow(t, h)
		goto again
	}

    // 情况二：
	// 在不满足情况一的条件下，会为当前桶再新建溢出桶，
    // 并将 tophash，key 插入到新建溢出桶的对应内存的 0 号位置
	if inserti == nil {
		// 获取新建溢出桶的位置（后面有分析这个函数的实现）
		newb := h.newoverflow(t, b)
        // 记录这个空闲的位置tophash的地址，用于后面赋值，同时获取将要插入 k、v 的位置的指针
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// 如果 key 是指针（当键超过 maxKeySize（128个字节）时会转换为指针），则需要进行解引用获取值
	if t.indirectkey() {
		kmem := newobject(t.key)
        // 赋值
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
    // value 同理
	if t.indirectelem() {
		vmem := newobject(t.elem)
        // 获取 value 应该插入的地址
		*(*unsafe.Pointer)(elem) = vmem
	}
    // 更新 key 值
	typedmemmove(t.key, insertk, key)
    // 将 tophash 赋值到bmap tophash数组的[i]位置
	*inserti = top
	h.count++

done:
    // 再判断一次当前 map 是否有其他 goroutine 在写
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
        // 获取 value 应该插入的地址
		elem = *((*unsafe.Pointer)(elem))
	}
    // 返回 value 的底层内存位置
	return elem
}
```

在上面的写入流程中，在已有的桶和溢出桶中都未找到合适的 `cell` 供 `key` 写入，并且尚不满足扩容阈值的情况下，会通过 `newoverflow` 获得新的溢出桶。

 `mapassign` 函数并没有将插入 `key` 对应的 `value` 写入对应的内存，而是返回了 `value` 应该插入的内存地址。也就是说 **map 并不会在 `mapassign` 这个运行时函数中将值拷贝到桶中，该函数只会返回内存地址**。真正的赋值操作是**在编译期间插入**的

做个总结：

- 检测 map 中 `hmap` 是否为空指针，是否有其他 `groutine` 在进行写入，如果有抛出错误；标记 `flags` 表示当前 `groutine` 在对 map 进行写操作
- 根据哈希值的低位获得对应的 bucket
- 判断是否正在进行扩容后的搬迁，如果是则先进行搬迁
- 计算高八位的哈希值，首先循环遍历桶中的 8 个 cell，如果 8 个 cell 遍历完还没有，则到溢出桶中进行遍历。这里分两种情况：
  - 第一种情况是 cell 位的 `tophash` 值和当前 `tophash` 值不相等，因为有可能删除导致中间个别 `cell` 没有值，但是在后面又还有可能找到相等的 `tophash`，所以这时会**先记录第一个位置为空的地址**，然后继续遍历，**<font color='red'>目的是尽可能往前插入键值</font>**
  - 第二种情况是 `cell` 位的 `tophash` 值和当前的 `tophash` 值相等，此时会查看当前的 key 值，因为即使当前 `cell` 位的 `tophash` 值相等，不一定它对应的 `key` 也是相等的。不相等则继续遍历，相等则获取当前 `value` 的地址直接返回(相当于更新 value）。

- 如果遍历所有之后没有找到合适的位置，则需要新建溢出桶来承载键值对。这里也分为两种情况：
  - 第一种是需要判断当前 map 的装载因子是否达到 6.5 的阈值或者溢出桶过多，这两种情况都需要对 map 进行扩容处理后再重新遍历
  - 第二种情况则是在在不满足情况一的条件下，为当前桶新建溢出桶，并将 `tophash`，`key` 插入到新建溢出桶的对应内存的 0 号位置，然后返回 `value` 的底层内存位置

## 扩容map

map 在扩容的时候有两个指标：**装载因子**和**溢出桶的数量**。

当 map 将要添加、修改或删除 `key` 时，都会检查是否需要扩容

扩容主要有两个条件：

1. **判断已经达到装载因子的临界点**，即`元素个数 > 桶（bucket）总数 * 6.5`，这时候说明大部分的桶可能都快满了（即平均每个桶存储的键值对达到 6.5 个），如果插入新元素，有大概率需要挂在溢出桶（overflow bucket）上。
2. **判断溢出桶是否太多**，当`桶总数 < 2 ^ 15` 时，如果`溢出桶总数 >= 桶总数`，则认为溢出桶过多。当`桶总数 >= 2 ^ 15` 时，直接与 `2 ^ 15` 比较，当`溢出桶总数 >= 2 ^ 15` 时，即认为溢出桶太多了。

在某些场景下，比如不断的增删，这样会造成 overflow 的 bucket 数量增多，但负载因子又不高，未达不到第 1 点的临界值，就不能触发扩容来缓解这种情况。这样会造成**桶的使用率不高，值存储得比较稀疏**，查找插入效率会变得非常低，因此有了第 2 点判断指标。

针对这两种条件，有两种扩容方法：

- 针对 1，将 B + 1，新建一个 buckets 数组，也就是说新的 buckets 大小是原来的 2 倍，然后旧 buckets 数据搬迁到新的 buckets。该方法称之为**<font color='red'>增量扩容</font>**。
- 针对 2，并不扩大容量，buckets 数量维持不变，重新做一遍类似增量扩容的搬迁动作，把松散的键值对重新排列一次，以使 bucket 的使用率更高，进而保证更快的存取。该方法称之为**<font color='red'>等量扩容</font>**

扩容主要是hashGrow和growWork函数：

```go
func hashGrow(t *maptype, h *hmap) {
	// 如果达到条件 1，那么将 B 值加 1，相当于是原来的 2 倍
   	// 否则对应条件 2，进行等量扩容，所以 B 不变
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
    // 记录老的buckets
	oldbuckets := h.buckets
    // 申请新的 buckets 空间
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

    // 注意 &^ 运算符，这块代码的逻辑是转移标志位
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// 提交 grow
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
    // 搬迁进度为0
	h.nevacuate = 0
    // overflow buckets 数为0
	h.noverflow = 0

    // 如果发现 hmap 是通过 extra 字段来存储 overflow buckets 时
	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}
}
```

`hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。

真正搬迁 buckets 的动作在 `growWork()` 函数中

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 为了确认搬迁的 bucket 是我们正在使用的 bucket
  	// 即如果当前 key 映射到老的 bucket1，那么就搬迁该 bucket1。
	evacuate(t, h, bucket&h.oldbucketmask())

	// 如果还未完成扩容工作，则再搬迁一个bucket。
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

核心是evacuate()函数，而且**<font color='red'>扩容是渐进式的</font>**，每次最多搬迁两个桶。

![img](https://img-blog.csdnimg.cn/img_convert/0b38207941d7615ba09e18090b655208.webp?x-oss-process=image/format,png)

如下图中，当原始的 B = 3 时，旧 buckets 数组长度为 8，在编号为 2 的 bucket 中，其 2 号 cell 和 5 号 cell，它们的低 3 位哈希值相同（不相同的话，也就不会落在同一个桶中了），但是它们的低 4 位分别是 0010、1010。当发生了增量扩容，2 号就会被搬迁到新 buckets 数组的 2 号 bucket 中去，5 号被搬迁到新 buckets 数组的 10 号 bucket 中去，它们的桶号差距是 2 的 3 次方。

![img](https://img-blog.csdnimg.cn/img_convert/5f06051480f023dbfc8ce490c465d4bd.webp?x-oss-process=image/format,png)

在源码中，有 `bucket x` 和 `bucket y` 的概念，其实就是增量扩容到原来的 2 倍，桶的数量是原来的 2 倍，前一半桶被称为 `bucket x`，后一半桶被称为 `bucket y`。一个 bucket 中的 key 可能会分裂到两个桶中去，分别位于 `bucket x` 的桶，或 `bucket y` 中的桶。所以在搬迁一个 cell 之前，需要知道这个 cell 中的 key 是落到哪个区间（而对于同一个桶而言，搬迁到 `bucket x` 和 `bucket y` 桶序号的差别是老的 buckets 大小，即 `2^old_B`）。

![img](https://img-blog.csdnimg.cn/img_convert/bd8ab23aac27bcfd7caf6cd8f8f1a42e.webp?x-oss-process=image/format,png)

说下整体流程吧：

1. 首先判断是需要等量扩容还是增量扩容，增量扩容的条件是`元素个数 > 桶（bucket）总数 * 6.5`。而等量扩容的条件是当`桶总数 < 2 ^ 15` 时，如果`溢出桶总数 >= 桶总数`，则认为溢出桶过多;当`桶总数 >= 2 ^ 15` 时，直接与 `2 ^ 15` 比较，当`溢出桶总数 >= 2 ^ 15` 时，即认为溢出桶太多了
2. 记录当前 `buckets` 数据，申请新的 `buckets` 空间
3. 将 `oldbuckets` 指向原有的 `buckets`，设置 `nextOverflow` 指针
4. 通过 `growWork` 函数执行旧键值对的搬迁，搬迁采用渐进式的方式，次至多搬迁 2 个bucket。
5. 搬迁过程会进行 rehash，对于等量扩容，则只需要将旧的键值对取出来依次放入即可；对于增量扩容，从哈希值的倒数 B 位（原来的 B 值）多取 1 位，变成 B+1位（这里其实是因为扩容后容量为 2 倍，所以在最高位多取一位相当于乘以 2）。然后根据多取出来的一位是 0 还是 1 进行分配，0 则分配到前一般桶 `bucket x` 加上偏移的位置；1 则分配到后一般桶 `backet y` 对应加上偏移的位置。