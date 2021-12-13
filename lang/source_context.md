
context包用短小精悍来形容最适合不过了，可以说，无论是作为源码分析或者是实际项目的使用，它都是十分值得我们去深入学习了解的。

这里着重说源码，后续会开一个小篇章说说context的使用。

context ？
context自Go1.7版本之后被引入标准库，中文翻译是“上下文”，是专门用来简化对于处理单个请求、多个goroutine之间数据共享、信号取消、超时处理等相关操作的，更加准确地说，它其实就是goroutine的上下文，记录了goroutine的运行状态、环境、现场等相关信息。也正是因为context的引入，我们可以在很多的标准库 、 项目中看到接口中设置了context参数(比如database/sql)。

可以从另外一个角度去理解context。从操作系统角度看，线程/进程进行切换的时候，需要先保存当前的寄存器和栈指针，然后载入下一个进程/线程需要的寄存器和栈，此时，寄存器和栈就是进程/线程的context。在最后面补充了一些小的脑洞提问，有兴趣也可以看看。

在由Go编写的服务器中，每个请求都在自己的goroutine中处理。而处理请求的程序通常会启动其他goroutine来访问后端，就好像链接数据库和进行RPC请求。负责处理请求的goroutine集合通常也是需要访问特定请求的值，当请求被取消或者超时的时候，负责处理该请求的所有goroutine都应该快速退出，便于系统回收资源。

![alt desc](https://pic2.zhimg.com/80/v2-0c955ffd61cc6f4e07546bfa81311bf9_1440w.jpg)

鉴于上面的模型，在很多场景下，一个请求会衍生出很多协程，而这些协程都有有关联关系的：他们之间共享全局变量、有共同的deadline、可以同时被关闭。在Go里面，协程的关闭一般会用channel + select的方式来控制，但是在这个场景下就会比较麻烦，这时，context的出现就显得十分必要跟优雅了。



源码分析（1.16.3）
既然了解了context产生的背景以及场景，接下我们就要了解下context的原理了。（本地brew安装的Go，更新的时候升级到了比较高的版本，但是不妨碍我们分析）

Go的context包其实就是context.go这个文件，总共500来行代码，其中大篇幅都是注释。

结构体 、 接口 的类图如下：

![alt total](https://pic2.zhimg.com/80/v2-1082d96b45add2d46404a7c43f3fb689_1440w.jpg)

一个个看其实现。

Context 接口实现
```
type Context interface {
    // returns the time when work done on behalf of this context should be canceled.  
    // 返回 context 是否被取消以及其deadline时间
    Deadline() (deadline time.Time , ok bool)

    // 当 context 被取消 或者到了deadline ，会返回一个关闭的channel
    // 返回的 channel 是一个只读的channel 
    Done <-chan struct{}

    // 当channel Done关闭之后，可以以此获取context取消的原因
    Err() error

    // 获取context中 key对应的value
    Value(key interface{}) interface{}
}
```
源码的注释一直提到simultaneously 、 asynchronously等字眼，也就是说，Context接口的方法其实都是幂等的。方法大致的功能其实上面的源码我有简化，可以补充说明下：

Done() 返回一个只读的channel；读一个关闭的channel可以读出对应类型的零值，而且也没有其他地方会向这个channel推入值，相当于就是实现了一个receive-only的channel。

canceler接口实现
```
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```
从上面的实现关系其实可以看出，context包里面有两个类型实现了canceler接口：*cancelCtx 、*timerCtx。（可看最后的问题2思考）

emptyCtx类型
emptyCtx是对Context接口的一个很有趣的实现，源码如下:
```
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}
```
源码备注是这样的，emptyCtx是一个永远不会被cancel，没有值，没有deadline的空context 。 这里派生一个比较有趣的问题，emptyCtx是整个Context的灵魂，因为我们对context的所有操作都是基于emptyCtx去做再次封装：（具体的灵魂点，可以看看问题3的讨论）

```
var (
        // background 作为所有context的根节点
        // 通常用在main 函数之中
	background = new(emptyCtx)
        // todo 作用如其名，经常用于你不确定要传递什么类型的context的场景, “占位符“了属于是
	todo       = new(emptyCtx)
)
```

内置三大结构体（一）： cancelCtx
正如后面提到的问题2 ，Context不提供cancel，只做自己的事情，context包内置了一个可以取消的Context，实现了canceler接口。
```
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
有一个值得关注的点，Done()方法是懒创建的，只有在第一次被调用的时候才会被创建，而且返回的是一个只读的channel。如果我们这个时候去读这个channel，整个协程就会被锁住，如果配合上select，在被关闭的第一时间就可以直接读出零值。
```
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```
cancel() 方法的功能就是关闭channel，递归取消所有子节点。可以轻松达到关闭channel，并将取消信号传递给所有子节点：
```
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
但是这个内置可取消的context是私有的，源码提供了一个创建方法：
```
// WithCancel returns a copy of parent with a new Done channel. The returned
// context's Done channel is closed when the returned cancel function is called
// or when the parent context's Done channel is closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.

// 传入的parent Context通常是一个background，作为根节点
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```
其实上面的源码有一个问题，创建parent的cancelCtx的时候removeFromParent是false，而后续的CancelFunc则是true，有兴趣可以看看文章最末的问题3.

cancel操作有一个最为核心的处理propagateCancel方法：
```
// 用于传播cancel行为
func propagateCancel(parent Context, child canceler) {
   done := parent.Done()
   if done == nil {
      return // parent is never canceled
   }

   select {
   case <-done:
      // parent is already canceled
      child.cancel(false, parent.Err())
      return
   default:
   }

   if p, ok := parentCancelCtx(parent); ok {
      // 如果parent已经取消，那么将取消信号传递到parent这颗子树
      // 否则将child加入到根节点方向最近的cancelCtx的children中
      p.mu.Lock()
      if p.err != nil {
         // parent has already been canceled
         child.cancel(false, p.err)
      } else {
         if p.children == nil {
            p.children = make(map[canceler]struct{})
         }
         p.children[child] = struct{}{}
      }
      p.mu.Unlock()
   } else {
      // 其实这里的else咋一看有点没必要，如果找不到可以取消的父节点，那么监听parent.Done()其实就是多此一举
      // 而 child.Done()又没有实际的处理逻辑

      // 这里可以看看上面parentCancelCtx的方法，可以看下一个代码块
      // 如果外部封装了context这个时候取消parent是无法通过传递的方式告知子节点的
      // 所以这里会启动一个goroutine来监听parent的取消信号，用于顺利取消child context
      // 再说明下，第一个case说明了当父节点取消时候，就取消子节点，可以确保父子节点的信号传递
      // 第二个case则是当子节点自己取消了的时候，就退出当前select，父节点之后的信号就可以忽略了
      // 如果移除第二个case，这个goroutine就泄漏了，不过这么做的话，如果父节点取消了，就会重复子节点取消，但是问题不大
      atomic.AddInt32(&goroutines, +1)
      go func() {
         select {
         case <-parent.Done():
            child.cancel(false, parent.Err())
         case <-child.Done():
         }
      }()
   }
}
```
里面包含的一个方法
```
// parentCancelCtx returns the underlying *cancelCtx for parent.
// It does this by looking up parent.Value(&cancelCtxKey) to find
// the innermost enclosing *cancelCtx and then checking whether
// parent.Done() matches that *cancelCtx. (If not, the *cancelCtx
// has been wrapped in a custom implementation providing a
// different done channel, in which case we should not bypass it.)

// 该方法会返回底层的cancelCtx指针
// 可以用于查找封装了的*cancelCtx，然后检查是否parent.Done()匹配
// 如果没有，*cancelCtx在自定义的实现中提供了一个不同的done 的channel，这种情况下，我们不应该绕过它
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
   done := parent.Done()
   if done == closedchan || done == nil {
      return nil, false
   }
   p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
   if !ok {
      return nil, false
   }
   p.mu.Lock()
   ok = p.done == done
   p.mu.Unlock()
   if !ok {
      return nil, false
   }
   return p, true
```

内置三大结构体（二）：timerCtx
```
// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
   cancelCtx
   timer *time.Timer // Under cancelCtx.mu.

   deadline time.Time
}
```
timerCtx基于cancelCtx，内部使用了一个定时器定时执行cancel方法。跟cancelCtx类似，timerCtx也提供了一个创建方法:(官方注释连用法都有，赞)
```
// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete:
//
// 	func slowOperationWithTimeout(ctx context.Context) (Result, error) {
// 		ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
// 		defer cancel()  // releases resources if slowOperation completes before timeout elapses
// 		return slowOperation(ctx)
// 	}
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    // 传入deadline的时间是绝对时间
    return WithDeadline(parent, time.Now().Add(timeout))
}
```
这里不太多展开去描述，看下下面的中文注释即可：
```
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
   if parent == nil {
      panic("cannot create context from nil parent")
   }
   if cur, ok := parent.Deadline(); ok && cur.Before(d) {
      // The current deadline is already sooner than the new one.
      // 这么处理可以不用单独处理子节点的计时器时间到了之后，自动调用cancel函数
      return WithCancel(parent)
   }
   c := &timerCtx{
      cancelCtx: newCancelCtx(parent),
      deadline:  d,
   }
   propagateCancel(parent, c)
   dur := time.Until(d)
   if dur <= 0 {
      c.cancel(true, DeadlineExceeded) // deadline has already passed
      return c, func() { c.cancel(false, Canceled) }
   }
   c.mu.Lock()
   // 这里加锁是因为无论是progagate还是后续的cancel内部都是有做互斥处理的
   // 如果ctx已经被加入到某个cancelCtx的children中，而且在未初始化Timer之前就调用了cancel方法
   // 那么就会存在context err的并发访问
   defer c.mu.Unlock()
   if c.err == nil {
      c.timer = time.AfterFunc(dur, func() {
         c.cancel(true, DeadlineExceeded)
      })
   }
   return c, func() { c.cancel(true, Canceled) }
}
```
内置三大结构（三）：valueCtx

```
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
   Context
   key, val interface{}
}
```
对外的创建方法：
```
// WithValue returns a copy of parent in which the value associated with key is
// val.
//
// Use context Values only for request-scoped data that transits processes and
// APIs, not for passing optional parameters to functions.
//
// The provided key must be comparable and should not be of type
// string or any other built-in type to avoid collisions between
// packages using context. Users of WithValue should define their own
// types for keys. To avoid allocating when assigning to an
// interface{}, context keys often have concrete type
// struct{}. Alternatively, exported context key variables' static
// type should be a pointer or interface.
func WithValue(parent Context, key, val interface{}) Context {
   if parent == nil {
      panic("cannot create context from nil parent")
   }
   if key == nil {
      panic("nil key")
   }
   if !reflectlite.TypeOf(key).Comparable() {
      panic("key is not comparable")
   }
   return &valueCtx{parent, key, val}
}
```
官方注释说的比较明白，对于key而言，要求是可比较的，因为之后要通过key来取值。

通过层层传递的context，最终就会形成类似的结构

![alt chain](https://pic3.zhimg.com/80/v2-f30a2c03da55973b1b532c0c012429fa_1440w.jpg)

类似链表
通过withValue函数，可以创建出一个一层嵌套一层的valueCtx，存储了goroutine之间可以共享的变量。

取值过程自然就是沿着链表一直往上找了，父节点无法知道子节点存储的value。

这里就引入了一个问题，也是context一直被诟病的点：

因为WithValue创建的context节点其实就是创建一个链表的过程，无论两个节点的值是什么，在查找的时候，会往上一直查找到最后一个context节点，其实就是一个低效率低链表查询，在实际项目中，体现的效果就是，一个context中，很多地方、子函数会向context里面塞不用的k-v，然后在某个地方去使用，这个时候这些值就存在很多不可控的因素了，比如值覆盖，传递的内容。所以，尽量不要使用context去传值！

源码分析到此结束，有兴趣可以看看后面的几个问题思考，后续会开篇幅说说context的使用以及一些总结

问题1 ： context可以理解为是一个“全局变量”吗？

在软件设计的工程中，对全局变量基本持否定态度。
1、代码变得耦合；
2、暴露了多余的信息；
3、 全局变量在多线程环境下使用锁，浪费CPU资源；
但是它也有好的方面：提升了某些变量的作用域，保证了这些数据的生命周期。为了解决负面影响，很多语言出现了“不那么全局的全局变量”，就比如 针对“线程局部”、“包局部”的全局变量而出现的this ，匿名形式的 “闭包”。有一个说法就是：每一段程序都有很多外部变量（极度简单的函数略过），一旦有了外部变量，这段程序就不完整，不能独立运行，而为了让他们能运行，就要给所有的外部变量设置值，而这些值的集合就叫做Context。
问题2 ： Context 接口为什么还要提供一个canceler，而不是自己提供cancel()方法呢？

“取消“这一操作是非强制性的。调用方不应该关心、干涉被调用方的情况。被调用方的责任只会是决定怎么return，也就是说，调用方只需要发送“cancel”，被调用方收到信息之后进一步处理即可，所以Context接口并没有定义cancel方法。
cancel 操作应该可以传递。cancel某个函数的时候，与之关联的其他函数应该也被cancel，也正是如此，Done()方法返回了一个只读的channel，所有的函数只需要监听此channel即可。
问题3 ： 可取消的Context什么时候应该将context从当前节点中删除，也即是removeFromParent 设置为true？

当removeFromParent 为true时，cancelCtx 会将当前节点中的context 从父节点 context中删除,而源码中，只有在调用WithCancel()的时候，也就是新创建一个可取消的context的时候，其返回的cancelFunc函数会传入true，当cancelFunc被调用到的时候，会该context从父节点中删除，不会影响其他子节点。当然，整个节点回收就很简单了，直接将c.children = nil就可以了。其实这个问题的答案也就很显而易见了，删除局部子节点，避免并发遍历、删除同一个map。
