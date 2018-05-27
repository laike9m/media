很长一段时间，我都搞不懂 Go 的 [context](https://golang.org/pkg/context/) package。最近我读了两篇文章：

- [Context API explained](https://siadat.github.io/post/context)
- [Context isn’t for cancellation](https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation)

终于理解了 context 是在干什么，以及为何它让人费解。

从设计的目的上说，**context 是用来控制 goroutine 的，确切地说，是为了当你需要的时候，能干掉一个 goroutine**。比如你用 `go someFunc()` 创建了成千上万个 goroutine，一切都很美好。突然你不想让它们继续执行了，这时你发现，糟了，你根本不知道这些 goroutine 放在哪，更别说控制它们了。不像其它语言用多线程时往往有个 Thread object，goroutine 并没有保存在某个变量里。怎么办？

## 一. 如何让 goroutine 从内部退出？

办法是每次创建 goroutine，都传进去一个 context，把函数定义成这样

```
func someFunc(ctx context.Context) {
  select {
  case <-ctx.Done():
      return
  }
}
```

这个模板包含了实现 goroutine 控制的必要元素。首先有一个 `context.Context` 类型的参数 `ctx`，它有一个最重要的方法 `Done()`。 `case <-ctx.Done():` 本质上就是在做这样一个判断 `if ctx.goroutine_should_be_cancelled()`，如果是，那它就进入这个 case 并且 return，结束当前 goroutine 的执行，否则进别的 case 继续执行。`Done()` 的实现机理在这里并不重要，我们只需要知道它在做判断即可。（ 其原理是返回一个 read-only 的 channel，这个 channel 在 `ctx`被 cancel 的时候关闭，而读被关闭的 channel 会直接返回一个 zero value 而不会阻塞。）

这里用两个例子展示 `case <-ctx.Done():` 和别的控制逻辑（比如循环，别的 channel）是怎么结合使用的，比较简单所以就不解释了。

Example 1

```
func someFunc(ctx context.Context) {
    for {
        innerFunc()
        select {
            case <-ctx.Done():
                // ctx is canceled
                return
            default:
                // ctx is not canceled, continue immediately
        }
    }
}
```

Example 2

```
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
    // Run the HTTP request in a goroutine and pass the response to f.
    tr := &http.Transport{}
    client := &http.Client{Transport: tr}
    c := make(chan error, 1)
    go func() { c <- f(client.Do(req)) }()
    select {
    case <-ctx.Done():
        tr.CancelRequest(req)
        <-c // Wait for f to return.
        return ctx.Err()
    case err := <-c:
        return err
    }
}
```

理论上说，不传 context 而是传 channel 来控制也做得到，但为每个 goroutine 都创建一个 channel 并不符合 channel 的用法。

## 二. 上层如何告诉 goroutine 该退出了？

现在我们已经知道 `<-ctx.Done()` 用来在 goroutine 内部用来判断当前 goroutine 是否应该退出。那么外界该如何控制？有两种方式，要么是手动 cancel，要么是设定条件自动 cancel。

先看手动 cancel：

```
ctx, cancel = context.WithCancel(context.Background())
```

`WithCancel` 返回一个被包装过的 context，以及一个 cancel 函数。只要把 ctx 传进 goroutine，在合适的时候调用 `cancel()` 就可以告诉 ctx，这个 goroutine 应该退出了，从而进入 `case <-ctx.Done():` 。

自动 cancel 可以设置两种条件：1. 设置一个 deadline，时间达到或超过 deadline 则 cancel；2. 设置一个 timeout，超时则 cancel。函数是 `WithDeadline` 和 `WithTimeout`，用法和 `WithCancel` 一样。

```
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

## 三. Context For Pythonista

自然我们要看一下其它语言对与并发是如何控制的。比如，在 Python 里人们怎么让一个线程停止的？

[Is there any way to kill a Thread in Python?](https://stackoverflow.com/questions/323972/is-there-any-way-to-kill-a-thread-in-python)

首先必须要说明，一般情况下不应该去试图干掉一个线程。当你需要这么做时，最简单的方法是在线程类里加入一个 `threading.Event` object 成员变量，之后在线程里去定期检查 `event.is_set()`。甚至于你都可以直接用一个 Boolean。对于 Java 也是类似的方法（当然实际上 Java 有更暴力的 `Thread.interrupt()`，不过我们假定用户想自己控制线程的退出）。

观察一下会发现，不管是 Python 里继承了 `threading.Thread` 的类，还是 Java 里实现了 `Runnable ` 接口的类，它们都是个类，从而你可以在其实例中添加任何需要的变量。而且，由于每个线程都是一个实例，你可以去调用上面定义的方法，比如 `terminate()`，来设置之前提到的 Boolean 的值。

Go 就不行了，由于没有“goroutine object”，必须要以参数的形式把一个实现控制功能的载体也就是 context 传进去。再仔细观察一下，`case <-ctx.Done():` 实现了 wait，和 Python 的 `threading.Event.wait()` 功能完全相同——毕竟这种等待的机制非常有用。

## 四. 为什么 context 看起来很奇怪？

context 奇怪的根源，在于它混合了两种不同的目的，更确切地说，是因为加入了 `WithValue`。 `WithValue` 非常好理解，就是外界在 context 上 set 一个 (key, value) pair，然后在 goroutine 内部读取。比如

```
ctx := context.WithValue(context.Background(), "key", "value")
go func(ctx) {
   value := ctx.Value("key")
}()
```

KV 存储嘛，谁都知道在干嘛。但是我们不要忘了 context 的主要用途是控制 goroutine 的退出。于是就很奇怪了，明明是控制 goroutine 退出的 indicator，为什么又能当成 KV 存储来用呢？这个问题困扰了许多人，以至于 [Dave Cheney](https://dave.cheney.net/) 专门写文章吐槽：《[Context isn’t for cancellation](https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation)》，他的观点是，context 顾名思义应该就是做存储用的，用来 cancel 才奇怪，所以应当：

> I propose context.Context becomes just that; a request scoped association list of copy on write values.

那篇文章还有个有意思的点，就是当前的 context 其实并不能做到完美的控制，不过这里暂且不提。总之，这个混合了两种目的的设计，不论是谁都觉得奇怪。我猜想，设计者的初衷应该是不想让人们传两个变量，索性都塞进 context。

`WithValue` 相当于 Python 里的 `threading.local`，但显然，没人会把  `threading.local` 和 `threading.Event` 合成一个东西来用。