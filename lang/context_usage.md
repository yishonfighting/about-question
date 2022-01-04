关于context的源码分析里面说到，context包简单方便。对外提供了一个创建根节点的函数:

```
// background是一个空的context，不可取消，没有值，没有超时时间
func Background() Context
```
在根节点的基础上，可以按需创建不同的子节点context，包括：
```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context
```
上面这些方法，可以保证context在传递的时候，可以通过调用cancel函数发送取消信号，也可以通过Value函数取出context中传递的值。

但是，在使用context的时候，也并非无所顾忌：

1. Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx.
2. Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use.
3. Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.
4. The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.
context 可以说是为了服务端而生的，处理一个请求，启动多个goroutine并行去处理，通过context去传递一些共享数据、通知资源回收等。但是，context也不能尽善尽美，比如项目代码里面十分常见的func Deal(ctx context.Context)，是的，几乎所有的函数都加上了这么一个context参数，强迫症的灾难好吧。

从之前的源码分析可以看出，其实，上面提到的withCancel、withDeadline等函数，实际上创建了一个链表，而底层也没有针对链表做了什么优化，效率都将是O(n) 。

但是，正如官方文档所说的，它并不完美，但是在并发控制和超时控制上，它最简洁优雅，对外提供的函数保证了context的并发安全，所以使用的时候我们可以放心传递。context作为函数参数的时候，应该直接放在第一个参数的位置，命名为ctx。虽然底层有兼容，但是，不要把context嵌套在自定义的类型之中。



传递共享的数据
在微服务架构之中，有一个完善的链路追踪是系统发现问题、追踪问题的根本，甚至是细化到一个请求到处理，也非常依赖不同协程之间的变量，而这个时候，可以将Context应用到函数之中进行传递


goroutine的取消
假定现在我们有个功能需要不断发起心跳，并且在收到心跳之后需要进行一个计算之后回包，这个过程中用户可能会退出，这个时候，我们也可以使用context
