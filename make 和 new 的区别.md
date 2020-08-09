# 前言

在 go 中对某种类型进行初始化时会用到 `make` 和 `new`, 因为它们的功能相似，所以初学者可能对它们的感到困惑；本文将由浅入深的介绍其功能和区别

# 结论

长话短说，先放上结论：

| 方法 | 作用     | 作用对象                           | 返回值                             |
| ---- | -------- | ---------------------------------- | ---------------------------------- |
| new  | 分配内存 | 值类型和用户定义的类型             | 初始化为零值，**返回指针**         |
| make | 分配内存 | 内置引用类型（map, slice, channel) | 初始化为零值，**返回引用类型本身** |



以上为 `make` 和 `new` 的区别，如果有人追问能说的更详细点吗？

哦豁，这就很尴尬了

# 正文

短话长说，我们看看 `make` 和 `new` 究竟做了什么

## new

先说要点，**new 用来分配内存，并初始化零值，返回零值指针**



**在编译过程中，使用 new 大致会产生 2 种情况：**

1. **若该对象申请的空间为 0，则返回表示空指针的 `zerobase` 变量**，这类对象比如：`slice`, `map`, `channel` 以及一些结构体等。

   ```go
   // path: src/runtime/malloc.go
   
   // base address for all 0-byte allocations
   var zerobase uintptr
   ```

2. 其他情况则会使用 `runtime.newobject` 函数：

   ```go
   // path: src/runtime/malloc.go
   func newobject(typ *_type) unsafe.Pointer {
   	return mallocgc(typ.size, typ, true)
   }
```
   
   这一块的内容也比较简单， `runtime.newoject` 调用了 `runtime.mallocgc` 函数去开辟一段内存空间，然后返回那块空间的地址

这里可以做个简单的例子去验证一下：

```go
func main() {
   a := new(map[int]int)
    fmt.Println(*a)  // nil， 参考情况 1

   b := new(int)
   fmt.Println(*b)    // 0, 参考情况 2
}
```

说到底 `new` 实现什么功能呢，可以这样去理解

```go
// a := new(int); 其他类型以此类比
var a int
return &a
```

## make

`make`是用来初始化 `map`, `slice`, `channel` 这几种特定类型的

在编译过程中，用 `make` 去初始化不同的类型会调用不同的底层函数：

1. 初始化 `map`, 调用 `runtime.makemap`
2. 初始化 `slice`, 调用 `runtime.makeslice`
3. 初始化 `channel`，调用 `runtime.makechan`

接下来我们看这些函数的源码部分，探究它们与 `new` 的不同，如果了解这几种类型的源码，很容易理解下面的代码；如果不了解这块内容的同学可以跟着注释走，了解流程就可以了

`runtime.makemap`:

```go
// path: src/runtime/map.go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	...
   // 初始化 Hmap
   if h == nil {
      h = new(hmap)
   }
    
   // 生成 hash 种子
   h.hash0 = fastrand()
    
   // 计算 桶 的数量
   B := uint8(0)
   for overLoadFactor(hint, B) {
      B++
   }
   h.B = B
   if h.B != 0 {
      var nextOverflow *bmap
       
      // 创建 桶
      h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
      ...
   }
   return h
}
```

这里为了方便查看，省去了部分代码。我们可以看到这里的步骤很多，` h = new(hmap)`只是其中的一部分

`runtime.makeslice`:

```go
// path: src/runtime/slice.go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    // 计算占用空间和是否溢出
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    
   // 一些边界条件处理 
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
           // panic： len 超出范围 
			panicmakeslicelen()
		}
        // panic: cap 超出范围 
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

这里其实和 `new` 底层的 `runtime.newobject` 很相似了，只是这里多了一些异常处理

`runtime.makechan`:

```go
// path: src/runtime/chan.go
func makechan(t *chantype, size int) *hchan {
   ...
   var c *hchan
    
   // 针对不同情况下对 channel 实行不同的内存分配策略
   switch {
   case mem == 0:
      // 无缓冲区,只给 hchan 分配一段内存
      c = (*hchan)(mallocgc(hchanSize, nil, true))
      c.buf = c.raceaddr()
   case elem.ptrdata == 0:
      // channel 不包含指针，给 hchan 和 缓冲区分配一段连续的内存
      c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
      c.buf = add(unsafe.Pointer(c), hchanSize)
   default:
      // 单独给 hchan 和 缓冲区分配内存
      c = new(hchan)
      c.buf = mallocgc(mem, elem, true)
   }

   // 初始化 hchan 的内部字段 
   c.elemsize = uint16(elem.size)
   c.elemtype = elem
   c.dataqsiz = uint(size)
   ...
}
```

这里省略了部分代码，包括一些异常处理



总之，`make` 相对于 `new` 来说，做的事情更多，`new` 只是开辟了内存空间， `make` 为更加复杂的数据结构开辟内存空间并对一些字段进行初始化

**注意**：有心细的同学可以发现，**`runtime,makemap`, `runtime,makeslice`, `runtime.makechan`返回的是指针类型，但并不意味着用 make 初始化后，返回的是指针类型**，这里上面列出来的是比较核心部分的源码，并不是所有的源码



想了解 make 的更多内容，大家可以尝试看一下 `map`， `slice` 和 `channel` 的源码

## 灵魂拷问

这里有 2 个问题可以帮大家更好的去理解：

1. **可以用 new 去初始化 map, slice 和 channel 吗？**

   首先我们回忆一下 `new` 的功能，简单理解如下：

   ```go
   var i int
   return &int
   ```

   如果我们要去初始化上面几种类型要怎么去做：

   ```go
   	m := *new(map[int]int)  // 先取值，因为 new 返回的是指针
   	s := *new([]int)
   	ch := *new(chan int)
   ```

   在上述代码中，使用 `new` 去初始化这几个类型，是不会 panic  的

   针对上述代码的情况，我们可以分类讨论：

   1. **map**, new 没有对 map 做创建桶等初始操作，所以当我们**添加键值对的时候回 panic, 查询 和 删除不存在的 key 时不会引发 panic**, 因为查询和删除都要查找桶和 key的过程，如果没有对应的桶和key，查询返回零值，删除则不作操作
   2. **channel**，也没有对 channel 的缓冲区开辟内存空间以及更多的内部初始话操作，所创建的 channel 始终是 nil, **往里面发送或从里面接收数据都会引发 panic**
   3. **slice, 使用 new 创建的是 nil 切片，它是可以正常使用的，**因为在切片 append 的过程中调用 mallocgc 来申请到一块内存，返回一个新的切片，然后赋值给 nil 切片

2. **可以用 make 去初始化其他类型吗，如 int, string ?**

   不可以，因为 make 没有对其他类型提供相应的底层方法 

# 参考资料

* make 和 new

   [https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/)

* map 源码分析

  [https://mp.weixin.qq.com/s/2CDpE5wfoiNXm1agMAq4wA](https://mp.weixin.qq.com/s/2CDpE5wfoiNXm1agMAq4wA)

* slice 源码分析

  [https://mp.weixin.qq.com/s/MTZ0C9zYsNrb8wyIm2D8BA](https://mp.weixin.qq.com/s/MTZ0C9zYsNrb8wyIm2D8BA)

* channel  源码分析

  [https://mp.weixin.qq.com/s/MTZ0C9zYsNrb8wyIm2D8BA](https://mp.weixin.qq.com/s/MTZ0C9zYsNrb8wyIm2D8BA)