# HTTP包 - Body 为何设计成ReadCloser ？


为什么提出这个问题，源于自己对于http.Request.Body的一次非正常使用...


---
先提出一个比较常规的理解，TCP连接底层对应操作系统的socket文件句柄，如果不关闭会很快耗尽openfiles的限制，或者浪费系统资源。

可能你会觉得奇怪或者理所应当，对啊，就是因为这样，所以在Go的内置http包里面，将reader接口以及closer接口组合成方便使用的"大接口"，在完成read操作之后就可以close掉了，避免系统资源的占用。

下面是针对http包文档的一个截取:
```bigquery
	// Body is the request's body.
	//
	// For client requests, a nil body means the request has no
	// body, such as a GET request. The HTTP Client's Transport
	// is responsible for calling the Close method.
	//
	// For server requests, the Request Body is always non-nil
	// but will return EOF immediately when no body is present.
	// The Server will close the request body. The ServeHTTP
	// Handler does not need to.
	Body io.ReadCloser


```
看文档描述是比较简洁的，HTTP的Client和Transport保证响应体总是非空的的，即使响应没有响应体或者长度为0的响应体。同时，关闭响应体是调用者的责任。

那么问题就出现了，为什么body需要被关闭呢，不关闭又会如何呢？

解答之前我们要先找出所谓的body是什么（下面的源码从14.9版本搬运）

```bigquery
// 解析IO输入，也就是body相关的解析方法，无关逻辑会是注释的形式存在
func readTransfer(msg interface{}, r *bufio.Reader) (err error) {
    
    // 针对输入源进行区分，比如是request or response

    switch ...

    // 对encoding , content length 等做transfer
    
    transfer ...     

    // 完事就会开始准备 body reader,当然会继续对content length做判定然后处理
    // 这里主要做几件事
    //1. 对chunked进行解析
	t.Body = &body{src: internal.NewChunkedReader(r), hdr: msg, r: r, closing: t.Close}
    //2. 长度进行判定处理
    t.Body = &body{src: io.LimitReader(r, realLength), closing: t.Close}
    ...

}

// body 会在Transport中在进行一个赋值，就拿response的body来说
// 代码来源 transport.go 
body := &bodyEOFSignal{
...
resp.Body = &gzipReader{body: body}

```
从上面来看，其实body是在net中对tcp连接进行多层嵌套：
1. 一开始，作为输入的bufio.Reader，很明显利用了buf，将多次小的read操作替换为一次"大"的读操作，作用很明显，减少系统调用的次数；

2. 如果是chunked，chunkedReader会再解析chunked格式的编码；

3. 代码实现看，tcp连接在读取完body之后(content length判定)不会直接关闭，这样会导致read操作阻塞

4. 这时LimitReader会在读完之后，发出eof信号，在读到eof信号或者提前关闭时会对readLoop 发出回收连接的通知;

readLoop其实涉及到HTTP事务以及Go针对这个事务的实现 ，找时间再单独记录下来，这里你可以简单理解为，一次请求就是一个事务，我特地去IBM的文档找了一个相关的定义：
```bigquery
用户应用程序与提供者应用程序之间的 OSLC 交互使用标准的 HTTP 请求及响应。支持的 HTTP 请求方法包括 GET、POST、PUT 和 PATCH。HTTP 响应在 HTTP 头中提供信息，请求不成功时，将返回 HTTP 错误码。
```

通过伪代码可以看出来，如果body没有被完全读取完/没有被主动关闭，那么其实这次请求的事务就没有完成，此时的资源是无法被回收，除非连接超时。那么跟body是否被close有没有直接联系呢？从实现上看，body的内容被读完，资源就能被回收啦，那么我们只要做好抛弃body时的close就足够了，但是会存在特殊的情况，比如因为意料之外的原因导致body的数据没有被读完，会造成更多不必要的问题，这个时候，close反而成为了更好的处理方式。

