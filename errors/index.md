# Go Errors 错误处理


Golang 中的 error 是一个内置的特殊的接口类型：

```go
type error interface {
    Error()  string
}
```

在 Go 1.13 版本之前，有关 error 的方法只有两个：

- `errors.New` :

```go
func New(text string) error
```

- `fmt.Errorf` :

```go
func Errorf(format string, a ...interface{}) error
```

这两个方法都是用来生成一个新的 error 类型的数据。

---

## 1.13 版本之前的错误处理

最常见的，判断是否为 nil ：

```go
if err != nil {
    // something went wrong
}
```

判断是否为某个特定的错误：

```go
var ErrNotFound = errors.New("not found")

if err == ErrNotFound {
    // something wasn't found
}
```

error 是一个带有 Error 方法的接口类型，这意味着你可以自己去实现这个接口：

```go
type NotFoundError struct {
    Name string
}

func (e *NotFoundError) Error() string {
    return e.Name + ": not found"
}
if e, ok := err.(*NotFoundError); ok {
    // e.Name wasn't found
}
```

处理错误的时候我们通常会添加一些额外的信息，记录错误的上下文以便于后续排查：

```go
if err != nil {
    return fmt.Errorf("错误上下文 %v: %v", name, err)
}
```

`fmt.Errorf` 方法会创建一个包含有原始错误文本信息的新的 error ，但是与原始错误之间是没有任何关联的。


然而我们有时候是需要保留这种关联性的，这时候就需要我们自己去定义一个包含有原始错误的新的错误类型，比如自定义一个 QueryError ：

```go
type QueryError struct {
    Query string
    Err   error  // 与原始错误关联
}
```

然后可以判断这个原始错误是否为某个特定的错误，比如 ErrPermission ：

```go
if e, ok := err.(*QueryError); ok && e.Err == ErrPermission {
    // query failed because of a permission problem
}
```

写到这里，你可以发现对于错误的关联嵌套情况处理起来是比较麻烦的，而 Go 1.13 版本对此做了改进。

---
## 1.13 版本之后的错误处理

首先需要说明的是，Go 是向下兼容的，上文中的 1.13 版本之前的用法完全可以继续使用。

1.13 版本的改进是：

- 新增方法 `errors.Unwrap` :

```go
func Unwrap(err error) error
```

- 新增方法 `errors.Is` :

```go
func Is(err, target error) bool
```

- 新增方法 `errors.As` :

```go
func As(err error, target interface{}) bool
```

- `fmt.Errorf` 方法新增了 `%w` 格式化动词，返回的 error 自动实现了 `Unwrap` 方法。


下面进行详细说明。

对于错误嵌套的情况，`Unwrap` 方法可以用来返回某个错误所包含的底层错误，例如 e1 包含了 e2 ，这里 Unwrap e1 就可以得到 e2 。Unwrap 支持链式调用（处理错误的多层嵌套）。


使用 errors.Is 和 errors.As 方法检查错误：

- `errors.Is` 方法检查值：

```go
if errors.Is(err, ErrNotFound) {
    // something wasn't found
}
```

- `errors.As` 方法检查特定错误类型：

```go
var e *QueryError
if errors.As(err, &e) {
    // err is a *QueryError, and e is set to the error's value
}
```

`errors.Is` 方法会对嵌套的情况展开判断，这意味着：

```go
if e, ok := err.(*QueryError); ok && e.Err == ErrPermission {
    // query failed because of a permission problem
}
```

可以直接简写为：

```go
if errors.Is(err, ErrPermission) {
    // err, or some error that it wraps, is a permission problem
}
```

`fmt.Errorf` 方法通过 `%w` 包装错误：

```go
if err != nil {
    return fmt.Errorf("错误上下文 %v: %v", name, err)
}
```

上面通过 `%v` 是直接返回一个与原始错误无法关联的新的错误。
我们使用 `%w` 就可以进行关联了：

```go
if err != nil {
    // Return an error which unwraps to err.
    return fmt.Errorf("错误上下文 %v: %w", name, err)
}
```

一旦使用 `%w` 进行了关联，就可以使用 `errors.Is` 和 `errors.As` 方法了：

```go
err := fmt.Errorf("access denied: %w”, ErrPermission)
...
if errors.Is(err, ErrPermission) ...
```

对于是否包装错误以及如何包装错误并没有统一的答案。


---

*本文参考资料:
https://blog.golang.org/go1.13-errors*
