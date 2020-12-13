# bytes

# errors

**error 是一个接口类型，errors 包实现了其中的接口**，实现了该接口就是 error 类型

```go
// path: builtin/builtin.go
type error interface {
	Error() string
}
```

**errors.go**

```go
func New(text string) error {
	return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

当我们去定义一个结构体去是实现那个接口时，这个结构体也可以是看做 error 类型

**wrap.go**

**Unwrap**

可以去掉 error 链最外一层的包装, 仅限最外层

example:

```go
err1 := func()error {
  return fmt.Errorf("new %w", os.ErrNotExist)
}()

t.Log("test == :", err1 == os.ErrNotExist)  // false
t.Log("test Unwrap:", errors.Unwrap(err1) == os.ErrNotExist) // true
```

如果需要去掉外层所有的封装，可以用 `Is`

实现：

```go
func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	})
	if !ok {
		return nil
	}
	return u.Unwrap()
}
```

**Is**

报告err链中的任何错误是否与目标匹配。

该链由err本身组成，其后是通过重复调用Unwrap获得的错误序列

```go
err2 := func() error {
    return fmt.Errorf("aa%w",
        func() error {
            return fmt.Errorf("aa%w", os.ErrNotExist)
        }(),
    )
}()

t.Log("test == :", err2 == os.ErrNotExist)
t.Log("test Is: ", errors.Is(err2, os.ErrNotExist))  // true
t.Log("test Unwrap:", errors.Unwrap(errors.Unwrap(err2)) == os.ErrNotExist) // false

```

**As**

功能几乎和Is 一样，但是用来比较当前err 和特定的指针error,但不能为nil

```go
err := file()
if err != os.ErrNotExist {
    t.Error("not match")
}
var pathErr *os.PathError
t.Log(errors.As(err, &pathErr))
t.Log(errors.As(err, &os.ErrNotExist))
t.Log(errors.Is(err, os.ErrNotExist))
```





# io.util

```
sort.Slice(list, func(i, j int) bool { return list[i].Name() < list[j].Name() })
```

