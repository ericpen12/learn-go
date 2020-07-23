- [前言](#前言)
- [正文](#正文)
  * [什么是 defer](#什么是-defer)
  * [如何使用defer](#如何使用defer)
    + [释放资源](#释放资源)
    + [defer 捕获异常](#defer-捕获异常)
    + [实现代码追踪](#实现代码追踪)
    + [记录函数的参数与返回值](#记录函数的参数与返回值)
  * [defer 的陷阱](#defer-的陷阱)
    + [defer 和 函数返回值](#defer-和-函数返回值)
    + [defer 和 for](#defer-和-for)
    + [defer 关闭文件](#defer-关闭文件)
    + [defer 和 panic](#defer-和-panic)
- [总结](#总结)
- [最后](#最后)
- [参考资料](#参考资料)

# 前言


初学 go 的同学都应该了解 defer, defer 让人又爱又恨；当 defer 和 闭包结合在一起的时候更是一大杀器，  会用的人是伤敌一万，而不会用的人是自损八千

本文希望从一个个问题来带大家重新认识 defer。



先抛出一个简单的问题来引入全文，下面程序正确的输出内容是什么？

```go
package main

import "fmt"

func main() {
	fmt.Println(f1())
	fmt.Println(f2())
}

func f1() (n int) {
	n = 1
	defer func() {
		n++
	}()
	return n
}

func f2() int {
	n := 1
	defer func() {
		n++
	}()
	return n
}
```

输出：

```
2
1
```

如果对上面内容有疑惑，没关系我们开始进入正文


# 正文

## 什么是 defer

defer 是Go语言提供的一种用于注册**延迟调用**的机制，以用来**保证一些资源被回收和释放**。

defer 注册的延迟调用可以在当前函数执行完毕后执行（包括通过return正常结束或者panic导致的异常结束）

当defe注册了的函数或表达式逆序执行，先注册的后执行，类似于**栈** ”先进后出“ 

下面看一个例子：

```go
package main

import "fmt"

func main() {
   f()
}

func f() {
   defer func() {
      fmt.Println(1)
   }()
   defer func() {
      fmt.Println(2)
   }()
   defer func() {
      fmt.Println(3)
   }()
}
```

输出：

```go
3
2
1
```

## 如何使用defer

### 释放资源

使用 defer 可以在一定程度上避免资源泄漏，尤其是有很多 return 语句的场景，很容易忘记或者由于逻辑上的错误导致资源没有关闭。

下面的程序便是因为使用 return 后，关闭资源的语句没有执行，导致资源泄漏：

```go
f, err := os.Open("test.txt")
if err != nil {
   return
}
f.process()
f.Close()
```
此处更好的做法如下：
```go
f, err := os.Open("test.txt")
if err != nil {
   return
}
defer f.Close()

// 对文件进行操作
f,process()
```

此处当程序顺利执行后，defer 会释放资源；**defer 需要先注册后使用**，比如此处，打开文件异常时，程序执行到 return 语句时便会退出当前函数，没有经过 defer，所以此处defer 不会执行

### defer 捕获异常

在 go 中没有 `try` 和 `catch` , 当程序出现异常是，我们需要从异常中恢复。我们这时可以利用 defer + recover 进行异常捕获

```go
func f() {
    defer func() {
       if err := recover(); err != nil {
          fmt.Println(err)
       }
    }()
    // do something
    panic("panic")
}
```

注意，**`recover()` 函数在在defer中用匿名函数调用才有效**，以下程序不能进行异常捕获：

```go
func f() {
	if err := recover(); err != nil {
		fmt.Println(err)
	}
	//	do something
	panic("panic")
}
```

### 实现代码追踪

下面提供一个方法能追踪到程序时进入或离开某个函数的信息，此处可以用来测试特定函数有没有被执行

```go
func trace(msg string) { fmt.Println("entering:", msg) }
func untrace(msg string) { fmt.Println("leaving:", msg) }
```

下面将演示如何使用这两个函数：

```go
func a() {
	defer un(trace("func a()"))
	fmt.Println("in func a()")
}

func trace(msg string) string {
	fmt.Println("entering:", msg)
	return msg
}
func un(msg string) { fmt.Println("leaving:", msg) }
```

### 记录函数的参数与返回值

有时候程序返回结果不符合预期是， 大家可能手动打印 log 调试，此时**使用 defer 记录函数的参数和返回值**，避免手动多处打印调试语句

```go
func func1(s string) (n int, err error) {
	defer func() {
		log.Printf("func1(%q) = %d, %v", s, n, err)
	}()
	return 7, nil
}
```

**实现代码追踪** 和 **记录函数的参数与返回值** 这两个技巧来自无闻翻译的 **《the way to go》，**本文稍有改动，详情可以参阅 [defer 和追踪](https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/06.4.md)

## defer 的陷阱

### defer 和 函数返回值

情况1：

这里列出开头的例子：

```go
func f1() (n int) {
	n = 1
	defer func() {
		n++
	}()
	return n
}

func f2() int {
	n := 1
	defer func() {
		n++
	}()
	return n
}
```

这里 `f1()` 的返回结果是 2， 而 `f2()` 的返回结果是 1， 为什么会有这样的变化呢？

**return xxx 这一条语句并不是一条原子指令**， 它编译后的整体的流程是：

1. 返回值 = xxx
2. 调用defer函数
3. 空的 return

**记住上面几点可以解决绝大多数使用 defer 后的返回值问题**

我们再来看上面 2 个例子：

```go
func f1() (n int) {
	n = 1
	defer func() {
		n++
	}()
	return n
}
```

分解 `f1()`：

```
func f1() (n int) {
	// 1. 返回值 = xxx
	n = 1
	
	// 2. 调用defer函数
	func() {
		n++
	}()
	
	// 3. 空的 return
	return
}
```

所以返回的 n 是被修改过的

继续看 `f2()`:

```go
func f2() int {
	n := 1
	defer func() {
		n++
	}()
	return n
}
```

分解 `f2()`，此处惊现中文编程：

```go
func f2() int {
	n := 1
	// 1. 返回值 = xxx
	匿名返回值 = n

	// 2. 调用defer函数
	func() {
		n++
	}()

	// 3. 空的 return
	return
}
```

此处 defer 注册的匿名函数只能对 n 进行操作，不能影响到所谓的 `匿名返回值`，所以不会对返回值造成影响。

那么问题来了，如果如果 defer 注册的匿名函数传入 n 的指针会对返回值产生影响吗？

```go
func f2() int {
   n := 1
   defer func(n *int) {
      *n++
   }(&n)
   return n
}
```

答案是 **不会产生影响的**，因为在`1. 返回值 = xxx` 的过程是值赋值，返回值并不会因为 n 的改变而改变

但是如果`f2()` 中的 n 不是 int 类型而是 slice 的话，结果就会有所不同，defer 注册的匿名函数可以改变底层数组

```go
func f2() []int {
	n := []int{1}
	defer func() {
		n[0] = 2
	}()
	return n
}
```

此处要留意 **值类型和引用类型**

希望读者结合上述知识和闭包相关知识，尝试思考下面给出几个例子：

```go
func f3() (n int) {
   n = 1
   defer func(n int) {
      n++
   }(n)
   return n          
}

func f4() (n int) {
   n = 1
   defer func(n *int) {
      *n++
   }(&n)
   return n           
}

type N struct {
	x int
}
func f5() *N {
	n := &N{1}
	defer func() {
		n.x++
	}()
	return n 
}
```

答案：

```go
f3: n = 1
f4: n = 2
f5: n.x = 2
```

### defer 和 for

在 for 中使用 defer 产生的问题一般比较隐晦，在特定场景下就很致命，先看下面这个例子：

```go
for _, file := range files {
    if f, err = os.Open(file); err != nil {
        return
    }
    // 这是错误的方式，当循环结束时文件没有关闭
    defer f.Close()
    // 对文件进行操作
    f.Process(data)
}
```

循环结尾处的 **defer 没有执**行，所以**文件一直没有关闭**



此处敲黑板！**defer仅在函数返回时才会执行，在循环的结尾或其他一些有限范围的代码内不会执行**

更好的做法是：

```go
for _, file := range files {
    if f, err = os.Open(file); err != nil {
        return
    }
    // 对文件进行操作
    f.Process(data)
    // 关闭文件
    f.Close()
 }
```

上述问题自无闻翻译的 **《the way to go》**，详情可以参阅 [发生错误时使用defer关闭一个文件](https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/16.3.md)



再看另外一个例子：

```go
func main() {
   for _, file := range files {
      write(file)
   }
}

func write(file string) {
   ...
   if f, err = os.Open(file); err != nil {
      return
   }
   defer f.Close()
   // 对文件进行操作
   f.Process(data)
}
```

此处关于 defer 用法没有明显问题，但是需要注意的是：**defer 会推迟资源的释放，所以尽量不要在 for 中使用；defer 相对于普通函数来说有一定的性能损耗**

### defer 关闭文件

针对于 defer 关闭文件，这里存在一个问题，举个例子：

```go
f, err := os.Open(file)
if err != nil {
    panic(err)
}
defer f.Close()
// 对文件进行操作
f.process()
```

在这个例子中貌似没有什么问题，正常情况下，对文件操作之后使用 defer 释放资源，当打开文件失败是引发 panic。但是此处当 打开文件失败时， f 为 nil, 此时使用 defer 释放资源会引发新的 panic 

此处更好的做法是：

```go
f, err := os.Open(file)
if err != nil {
    panic(err)
}
if f != nil {
    defer f.Close()
}
```

### defer 和 panic

老规矩，先举例子，判断此处的返回值：

```go
func f() {
	defer func() {
		fmt.Println(1)
	}()
	defer func() {
		fmt.Println(2)
	}()
	panic("panic")
}
```

输出：

```
2
1
panic: panic
...
```

此处我本以为 panic 会先打印出来，然而 panic 是最后打印出来的。此处注意**panic会停掉当前正的程序，在这之前，它会有序地执行完当前协程defer列表里的语句**

# 总结

1. defer 是一种用于注册**延迟调用**的机制，以用来**保证一些资源被回收和释放**
2. defer 注册的函数**逆序执行**（**先注册后执行**）
3. **return xxx 语句不是一条原子指令**，使用 defer 时可能会改变返回值
4. defer 会推迟资源的释放，所以尽量不要在 for 中使用
5. defer 相对于普通函数来说**有一定的性能损耗**
6. defer 注册之后才会执行（放在 return 之后的 defer 不会执行）

# 最后

以上为编者学习 defer 时，参考多篇文章后的总结。由于能力有限，疏忽和不足之处难以避免，欢迎读者指正，以便及时修改。

若本文对你有帮助的话，请点 **star**  :star: 一下，感谢你的支持！

# 参考资料

* Golang之轻松化解defer的温柔陷阱

  [https://qcrao.com/2019/02/12/how-to-keep-off-trap-of-defer/](https://qcrao.com/2019/02/12/how-to-keep-off-trap-of-defer/)

* defer  案例

  [https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html)

* defer-panic-and-recover

  [https://blog.golang.org/defer-panic-and-recover](https://blog.golang.org/defer-panic-and-recover)

* defer 使用的例子

   [https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/06.4.md](https://github.com/unknwon/the-way-to-go_ZH_CN/blob/master/eBook/06.4.md)

* 《 go 语言核心编程》