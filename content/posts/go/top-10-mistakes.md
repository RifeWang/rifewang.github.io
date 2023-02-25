+++
draft = false
date = 2019-07-29T00:11:03+08:00
title = "Go 开发十种常犯错误【译】"
description = "Go 开发十种常犯错误【译】"
slug = ""
authors = []
tags = []
categories = ["Golang"]
externalLink = ""
series = []
disableComments = true
+++

*文本翻译自:
https://itnext.io/the-top-10-most-common-mistakes-ive-seen-in-go-projects-4b79d4f6cd65*

---

本文将会介绍 Go 开发中十种最常犯的错误，内容不算少，请耐心观看。


## 1、未知的枚举值

示例：

```go
type Status uint32

const (
  StatusOpen Status = iota
  StatusClosed
  StatusUnknown
)
```

示例中使用了 iota 创建了枚举值，其结果就是：

```go
StatusOpen = 0
StatusClosed = 1
StatusUnknown = 2
```


现在假设上述 Status 类型将会作为 JSON request 的一部分：

```go
type Request struct {
  ID        int    `json:"Id"`
  Timestamp int    `json:"Timestamp"`
  Status    Status `json:"Status"`
}
```

然后你收到的数据可能是：

```json
{
  "Id": 1234,
  "Timestamp": 1563362390,
  "Status": 0
}
```

这看起来似乎没有任何问题，status 将会被解码为 StatusOpen 。

但是如果另一个请求的数据是这样：

```json
{
  "Id": 1235,
  "Timestamp": 1563362390
}
```

这时 status 即使没有传值（也就是 unknown 未知状态），但由于默认零值，其将会被解码为 StatusOpen ，显然不符合业务语义上的 StatusUnknown 。

最佳实践是将未知的枚举值设置为 0 ：

```go
type Status uint32

const (
  StatusUnknown Status = iota
  StatusOpen
  StatusClosed
)
```


## 2、基准测试

基准测试受到多方面的影响，因此想得到正确的结果比较困难。

最常见的一种错误情况就是被编译器优化了，例如：

```go
func clear(n uint64, i, j uint8) uint64 {
  return (math.MaxUint64<<j | ((1 << i) - 1)) & n
}
```

这个函数的作用是清除指定范围的 bit 位，基准测试可能会这样写：

```go
func BenchmarkWrong(b *testing.B) {
  for i := 0; i < b.N; i++ {
    clear(1221892080809121, 10, 63)
  }
}
```

在这个基准测试中，编译器将会注意到这个 clear 是一个 leaf 函数（没有调用其它函数）因此会将其 inline 。一旦这个函数被 inline 了，编译器也会注意到它没有 side-effects（副作用）。因此 clear 函数的调用将会被简单的移除从而导致不准确的结果。

解决这个问题的一种方式是将函数的返回结果设置给一个全局变量：

```go
var result uint64

func BenchmarkCorrect(b *testing.B) {
  var r uint64
  for i := 0; i < b.N; i++ {
    r = clear(1221892080809121, 10, 63)
  }
  result = r
}
```

此时，编译器不知道这个函数的调用是否会产生 side-effect ，因此基准测试的结果将会是准确的。


## 3、指针

按值传递变量将会创建此变量的副本（简称“值拷贝”），而通过指针传递则只会复制变量的内存地址。

因此，指针传递总是更快吗？显然不是，尤其是对于小数据而言，值拷贝更快性能更好。

原因与 Go 中的内存管理有关。让我们简单的解释一下。

变量可以被分配到 heap 或者 stack 中：

stack 包含了指定 goroutine 中的将会被用到的变量。一旦函数返回，变量将会从 stack 中 pop 移除。
heap 包含了需要共享的变量（例如全局变量等）。

示例：

```go
func getFooValue() foo {
  var result foo
  // Do something
  return result
}
```

这里，result 变量由当前的 goroutine 创建，并且将会被 push 到当前的 stack 中。一旦这个函数返回了，调用者将会收到 result 变量的值拷贝副本，而这个 result 变量本身将会被从 stack 中 pop 移除掉。它仍存在于内存中，直到它被另一个变量擦除，但是它无法被访问到。

现在看下指针的示例：

```go
func getFooPointer() *foo {
  var result foo
  // Do something
  return &result
}
```
result 变量仍然由当前 goroutine 创建，但是函数的调用者将会接受的是一个指针（result 变量内存地址的副本）。如果 result 变量被从 stack 中 pop 移除，那么函数调用者显然无法再访问它。

在这种情况下，为了正常使用 result 变量，Go 编译器将会把 result 变量 escape（转移）到一个可以共享变量的位置，也就是 heap 中。

传递指针也会有另一种情况，例如：

```go
func main()  {
  p := &foo{}
  f(p)
}
```
由于我们在相同的 goroutine（main 函数）中调用 f 函数，这里的 p 变量无需被 escape 到 heap 中，它只会被推送到 stack 中，并且 sub-function 也就是这里的 f 函数是可以直接访问到 p 变量的。

stack 为什么更快？主要有两个原因：

- stack 几乎没有垃圾回收。正如上文所述，一个变量创建后 push 到 stack 中，其函数返回后则从 stack 中 pop 掉。对于未使用的变量无需复杂的过程来回收它们。
- stack 从属于一个 goroutine ，与 heap 相比，stack 中的变量不需要同步，这也导致了 stack 性能上的优势。

总之，当我们创建一个函数时，我们的默认行为应该是使用值而不是指针，只有当我们想用共享变量时才应该使用指针。


如果我们遇到性能问题，一种可能的优化就是检查指针在某些特定情况下是否有帮助。如果你想要知道编译器何时将变量 escape 到 heap ，可以使用以下命令：

```
go build -gcflags "-m -m"
```


## 4、从 for/switch 或 for/select 中 break

例如：

```go
for {
  switch f() {
  case true:
    break
  case false:
    // Do something
  }
}

for {
  select {
  case <-ch:
  // Do something
  case <-ctx.Done():
    break
  }
}
```

注意，break 将会跳出 switch 或 select ，但不会跳出 for 循环。

为了跳出 for 循环，一种解决方式是使用带标签的 break ：

```go
loop:
  for {
    select {
    case <-ch:
    // Do something
    case <-ctx.Done():
      break loop
    }
  }
```


## 5、errors 管理

Go 中的错误处理一直以来颇具争议。

推荐使用 https://github.com/pkg/errors 库，这个库遵循如下规则：

```
An error should be handled only once. Logging an error is handling an error. So an error should either be logged or propagated.
```

而当前的标准库（只有一个 New 函数）却很难去遵循这一点，因为我们可能希望为错误添加一些上下文并具有某种形式的层次结构。

假设我们在调用某个 REST 请求操作数据库时会碰到以下问题：

```
unable to server HTTP POST request for customer 1234
 |_ unable to insert customer contract abcd
     |_ unable to commit transaction
```

通过上述 pkg/errors 库，我们可以处理如下：

```go
func postHandler(customer Customer) Status {
  err := insert(customer.Contract)
  if err != nil {
    log.WithError(err).Errorf("unable to server HTTP POST request for customer %s", customer.ID)
    return Status{ok: false}
  }
  return Status{ok: true}
}

func insert(contract Contract) error {
  err := dbQuery(contract)
  if err != nil {
    return errors.Wrapf(err, "unable to insert customer contract %s", contract.ID)
  }
  return nil
}

func dbQuery(contract Contract) error {
  // Do something then fail
  return errors.New("unable to commit transaction")
}
```
最底层通过 errors.New 初始化一个 error ，中间层 insert 函数向其添加更多上下文信息来包装此 error ，然后父级调用者通过记录日志来处理错误，每一层都对错误进行了返回或者处理。


我们可能还想检查错误原因以进行重试。例如我们有一个外部库 db 处理数据库访问，其可能会返回一个 db.DBError 的错误，为了实现重试，我们必须检查具体的错误原因：

```go
func postHandler(customer Customer) Status {
  err := insert(customer.Contract)
  if err != nil {
    switch errors.Cause(err).(type) {
    default:
      log.WithError(err).Errorf("unable to server HTTP POST request for customer %s", customer.ID)
      return Status{ok: false}
    case *db.DBError:
      return retry(customer)
    }

  }
  return Status{ok: true}
}

func insert(contract Contract) error {
  err := db.dbQuery(contract)
  if err != nil {
    return errors.Wrapf(err, "unable to insert customer contract %s", contract.ID)
  }
  return nil
}
```
如上所示，通过  pkg/errors 库的 errors.Cause 即可轻松实现。

一种经常会犯的错误是只部分使用  pkg/errors 库，例如：

```go
switch err.(type) {
default:
  log.WithError(err).Errorf("unable to server HTTP POST request for customer %s", customer.ID)
  return Status{ok: false}
case *db.DBError:
  return retry(customer)
}
```
这里直接使用 err.(type) 是无法捕获到 db.DBError 然后进行重试的。


## 6、slice 初始化

有时候我们明确的知道一个 slice 切片的最终长度。例如我们想要将一个 Foo 切片 convert 为 Bar 切片，这意味着两个 slice 切片的长度是相同的。

然而有些人却经常初始化 slice 切片如：

```go
var bars []Bar
bars := make([]Bar, 0)
```

slice 切片并不是一个神奇的结构，当没有更多可用空间时，它会进行扩容，也就是其将会自动创建一个具有更大容量的新数组并复制所有的元素。

现在，让我们想象一下如果切片需要多次扩容，即使时间复杂度保持为 O(1) ，但在实践中，它也会对性能造成影响。尽可能避免这种情况。


## 7、context 管理

context.Context 经常被开发者误解。官方文档的描述是：

```
A Context carries a deadline, a cancelation signal, and other values across API boundaries.
```

这个描述很宽泛，以致于一些人对为什么以及如何使用它感到困惑。


让我们试着详细说明下，一个 context 可以包含：

- 一个 deadline 。其可以是持续时间（例如 250 毫秒）或者具体某个时间点（例如 2019-01-08 01:00:00），一旦达到 deadline 则所有正在进行的活动都会取消（比如 I/O 请求，等待某个 channel 输入等等）。
- 一个取消 signal 信号（基本上是 <-chan struct{} ）。这里的行为是类似的，一旦收到取消信号则必须停止正在进行中的活动。
- 一组 key/value（基于 interface{} 类型）。

需要说明的是，一个 context 是可组合的，例如既包含一个 deadline 又包含一组 key/value 。此外，多个 goroutine 可以共享同一个 context ，因此取消信号可能会导致多个 goroutine 中的活动被停止。

例如由同一个 context 引发的连环取消，我们要注意使用父子形式的 context ，以此来区分管理，避免相互影响。


## 8、未使用 -race

测试时未使用 -race 选项也是常见的，它是有价值的工具，我们应该在测试时始终启动它。


## 9、使用文件名作为输入

假设我们要实现一个函数去统计文件中的空行数，我们可能这样做：

```go
func count(filename string) (int, error) {
  file, err := os.Open(filename)
  if err != nil {
    return 0, errors.Wrapf(err, "unable to open %s", filename)
  }
  defer file.Close()

  scanner := bufio.NewScanner(file)
  count := 0
  for scanner.Scan() {
    if scanner.Text() == "" {
      count++
    }
  }
  return count, nil
}
```

这看起来很自然，filename 文件名作为输入，在函数内部打开文件。

然而，如果我们想要对此函数进行单元测试，输入可能是普通文件，或者空文件，或者其它不同编码类型的文件等等，此时则很容易变得难以管理。另外，如果我们想对某个 HTTP body 实现相同的逻辑，那么我们不得不创建一个另外的函数。

Go 提供了两个很棒的抽象：io.Reader 和 io.Writer 。我们可以传递 io.Reader 抽象数据源而不是 filename 。这样不管是文件也好，HTTP body 也好，byte buffer 也好，我们都只需要使用 Read 方法即可。

上述例子中，我们甚至可以缓冲输入以逐行读取，因此我们可以使用 bufio.Reader 和它的 ReadLine 方法：

```go
func count(reader *bufio.Reader) (int, error) {
  count := 0
  for {
    line, _, err := reader.ReadLine()
    if err != nil {
      switch err {
      default:
        return 0, errors.Wrapf(err, "unable to read")
      case io.EOF:
        return count, nil
      }
    }
    if len(line) == 0 {
      count++
    }
  }
}
```


而打开文件的操作则交由 count 的调用者去完成：

```go
file, err := os.Open(filename)
if err != nil {
  return errors.Wrapf(err, "unable to open %s", filename)
}
defer file.Close()
count, err := count(bufio.NewReader(file))
```


这样，无论数据源如何我们都可以调用 count 函数，同时这有有利于我们进行单元测试，因为我们可以简单的从字符串中创建一个 bufio.Reader :

```go
count, err := count(bufio.NewReader(strings.NewReader("input")))
```


## 10、Goroutines 和 Loop 循环变量

示例：

```go
ints := []int{1, 2, 3}
for _, i := range ints {
  go func() {
    fmt.Printf("%v\n", i)
  }()
}
```

输出将会是什么？1 2 3 吗？当然不是。

上例中，每个 goroutine 共享同一个变量实例，因此将会输出 3 3 3（最有可能）。

这个问题有两种解决方式。第一种是将 i 变量传递给 closure 闭包（ inner function ）：

```go
ints := []int{1, 2, 3}
for _, i := range ints {
  go func(i int) {
    fmt.Printf("%v\n", i)
  }(i)
}
```


第二种方式是在 for 循环范围内创建另一个变量：

```go
ints := []int{1, 2, 3}
for _, i := range ints {
  i := i
  go func() {
    fmt.Printf("%v\n", i)
  }()
}
```

---

以上就是全部内容，相关问题更多深入内容可参考：

- *https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html?#watch_out_for_compiler_optimisations*
- *https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html*
- *https://www.youtube.com/watch?v=ZMZpH4yT7M0&feature=youtu.be*
- *https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully*
- *http://p.agnihotry.com/post/understanding_the_context_package_in_golang/index.html*
- *https://medium.com/@val_deleplace/does-the-race-detector-catch-all-data-races-1afed51d57fb*
- *https://github.com/golang/go/wiki/CommonMistakes*

