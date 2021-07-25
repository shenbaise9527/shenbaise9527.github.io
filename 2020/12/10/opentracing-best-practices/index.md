# OpenTracing最佳实践


## 回顾: OpenTracing的目标是什么?
OpenTracing是一个位于应用程序/库代码和追踪系统之间的一个标准中间层.结构如下:
```
   +-------------+  +---------+  +----------+  +------------+
   | Application |  | Library |  |   OSS    |  |  RPC/IPC   |
   |    Code     |  |  Code   |  | Services |  | Frameworks |
   +-------------+  +---------+  +----------+  +------------+
          |              |             |             |
          |              |             |             |
          v              v             v             v
     +-----------------------------------------------------+
     | · · · · · · · · · · OpenTracing · · · · · · · · · · |
     +-----------------------------------------------------+
       |               |                |               |
       |               |                |               |
       v               v                v               v
 +-----------+  +-------------+  +-------------+  +-----------+
 |  Tracing  |  |   Logging   |  |   Metrics   |  |  Tracing  |
 | System A  |  | Framework B |  | Framework C |  | System D  |
 +-----------+  +-------------+  +-------------+  +-----------+
```

## 用例
下面列出一些OpenTracing的用例,并对其进行详细描述:
| Use case	|Description|
|----|----|
|应用程序代码|编写应用程序代码的开发人员可以使用OpenTracing来描述因果关系,划分控制流,并添加细粒度的日志记录信息.|
|库代码|对于请求进行中间控制的库也可以与OpenTracing集成.例如:Web中间件可以使用OpenTracing为每个请求创建spans,或者ORM库可以使用OpenTracing来描述更高级别的ORM语义并衡量特定SQL查询的执行.|
|OSS服务|除了嵌入式库之后,整个OSS服务都可以采用OpenTracing来与分布式跟踪系统集成,在较大的分布式系统中启动或传播到其它进程.例如:HTTP负载均衡器可以使用OpenTracing包装所有请求,或者在分布式KV存储系统中使用OpenTracing来跟踪读写性能.|
|RPC/IPC框架|任何跨进程边界的任务子系统都可以使用OpenTracing来标准化trace状态,OpenTracing提供了统一的Inject和Extract格式.|

所有上述都可以使用OpenTracing来描述和传播分布式跟踪信息,而不用了解分布式追踪系统的底层实现.

OpenTracing优先考虑易用性的问题,主要是站在调用者的角度,而不是分布式追踪系统的实现者上.

## 举例

### 追踪函数
``` go
func TopLevelFunction(ctx context.Context) {
	tracer := opentracing.GlobalTracer()
	span1 := tracer.StartSpan("toplevelfunction")
	defer span1.Finish()
	subctx := opentracing.ContextWithSpan(ctx, span)

	// 业务逻辑.
	Function2(subctx)
}
```
作为业务逻辑的一部分,调用了`Function2`方法,也想被追踪.为了让`Function2`里的追踪和`TopLevelFunction`里的追踪形成因果关系,必须在`Function2`里要能访问到`span1`,通过`span1`来创建一个子span.
``` go
func Function2(ctx context.Context) {
	span1, ok := opentracing.SpanFromContext(ctx)
	if ok {
		tracer := opentracing.GlobalTracer()
		span2 := tracer.StartSpan("function2", opentracing.ChildOf(span1.Context()))
		defer span2.Finish()
	}
}
```
通过`context.Context`来传递`span`,可以让整个函数调用过程形成一个完整的调用链.

### 服务端追踪
当服务端想要去跟踪每个请求的执行过程,一般需要以下几个步骤.
* 试图中从请求中获取`SpanContext`(客户端已经开启了trace),如果无法获取就开启一个新的trace.
* 在上下文中存储最新创建的`Span`,上下文会通过应用程序代码或RPC框架传播.
* 最后当处理完请求时需要调用`span.Finish()`来关闭`Span`.

**从请求中获取`SpanContext`**

假设有个HTTP服务器,`SpanContext`通过http头从客户端传播到服务端,可通过`request.headers`来访问.
``` go
tracer := opentracing.GlobalTracer()
carrier := opentracing.HTTPHeadersCarrier(r.Header)
spanContext, err := tracer.Extract(opentracing.HTTPHeaders, carrier)
```
把`headers`转换成`carrier`,`tracer`对象知道需要`headers`中的哪些字段,用来重建tracer的状态及Baggage.

**从请求中获取一个已经存在的trace,或者开启一个新的trace**

假设客户端没有发送相应字段的值,在服务端就无法从`Header`中获取到,上文中的`spanContext`可能为`nil`.在这种情况下,服务端需要开启一个新的trace.
``` go
tracer := opentracing.GlobalTracer()
carrier := opentracing.HTTPHeadersCarrier(r.Header)
spanContext, err := tracer.Extract(opentracing.HTTPHeaders, carrier)
var span opentracing.Span
if spanContext == nil || err != nil {
	span = tracer.StartSpan(r.RequestURI)
} else {
	span = tracer.StartSpan(r.RequestURI, opentracing.ChildOf(spanContext))
}

ext.HTTPMethod.Set(span, r.Method)
```
`ext.HTTPMethod.Set`是给`span`设置一个附加信息,等同于`span.SetTag("http.method", r.Method)`.

`StartSpan`的第一个参数是operationName,用来指定新创建的`Span`的名字.举例,若HTTP请求是`POST`类型且URI为`/save_user/123`,这此时`Span`的名字会被设置为`/save_user/123`.OpenTracing规范不会强制要求应用程序如何给`Span`命名.

**进程内请求上下文传播**

请求上下文是指:对于一个请求,所有处理这个请求的层都可以访问到同一个`context(上下文)`.可以通过特定值,如用户id、token、请求截止时间等来获取这个`context`,也可以用这个方式来获取当前正在追踪的`Span`.

OpenTracing规范中并没有规定请求上下文的传输实现方式,但这点是非常重要的,便于我们理解后面的章节.一般有两种常见的基数:
* 隐式传输,`context`需要被储存在平台特定的位置,允许应用程序在任何地方都能访问到.常用的RPC框架会利用`thread-local`或`continuation-local`来存储,或者是全局变量(在单线程程序中).这种方式的缺点是性能低下,并且有些平台如Go是不支持线程本地存储的,隐式传输就几乎不可能实现了.
* 显示传输,要求应用程序代码包装和传递一个`context`对象.这种方式的缺点在于向应用程序暴露了底层的实现,[Go blog post](https://blog.golang.org/context)这篇文章提供了这种方式的深层次解析.
``` go
func HandleHttp(w http.ResponseWriter, req *http.Request) {
    ctx := context.Background()
    ...
    BusinessFunction1(ctx, arg1, ...)
}

func BusinessFunction1(ctx context.Context, arg1...) {
    ...
    BusinessFunction2(ctx, arg1, ...)
}

func BusinessFunction2(ctx context.Context, arg1...) {
    parentSpan := opentracing.SpanFromContext(ctx)
    childSpan := opentracing.StartSpan(
        "...", opentracing.ChildOf(parentSpan.Context()), ...)
    ...
}
```

### 客户端追踪
当一个应用程序扮演RPC客户端的角色时,在调用外部接口前可以开启一个新的`Span`,在请求期间传播该`Span`.下面通过一个HTTP请求来展示如何处理.
``` go
func tracedPost(ctx context.Context, operation, url string, body []byte) error {
    parent_span := opentracing.SpanFromContext(ctx)
    tracer := opentracing.GlobalTracer()
    span := tracer.StartSpan(
        operation,
        opentracing.ChildOf(parent_span.Context()),
        opentracing.Tag{"http.url", url},
        opentracing.Tag{"http.method", http.MethodPost})
    defer span.Finish()
    reader := bytes.NewBuffer(body)
    req, err := http.NewRequest(http.MethodPost, url, reader)
    if err != nil {
        ext.LogError(span, err)

        return err
    }

    req.Header.Set("Content-Type", "application/json")
    err = tracer.Inject(span.Context(), opentracing.HTTPHeaders, req.Header)
    if err != nil {
        ext.LogError(span, err)

        return err
    }

    cli := http.Client{Timeout: time.Second * 5}
    rsp, err := cli.Do(req)
    if err != nil {
        ext.LogError(span, err)

        return err
    }

    ext.HTTPStatusCode.Set(span, uint16(rsp.StatusCode))

    return nil
}
```
* 首先从`context`中获取`parent_span`,可以跟上游`Span`构成一个链条.
* 针对http请求创建一个新的`Span`,设置相应的Tag,然后调用`Inject`把需要传播的信息注入到`req.Header`中,在服务端就可以利用`Header`来重组`Span`.
* 当有错误发生的时候,调用`ext.LogError`把错误信息关联到`Span`上.
* 最后把相应的状态码作为Tag设置到`Span`上.

### 使用Baggage/分布式上下文传输
上面的例子都是通过网络在客户端和服务端之间传递`Span/Tracer`,包含任意的`Baggage`.客户端可以利用`Baggage`来传播一些附加信息到服务端及任何其下游服务.
``` go
// 客户端.
span = span.SetBaggageItem("auto_token", "token")

// 服务端.
token := span.BaggageItem("auto_token")
```

### Logging事件
在上面[客户端追踪](#客户端追踪)的例子中已经使用过Log了.可以记录事件而不需要有`payload`,而不仅在`Span`被创建和完成时.举个例子,应用程序在执行过程中可能会需要记录`cache miss`事件,可以通过在请求上下文中来获取当前`Span`然后把该事件附加到`Span`中.
``` go
span := opentracing.SpanFromContext(ctx)
// 不带payload的.
span.LogEvent("cache_miss")
// 带payload的.
span.LogEventWithPayload("cache_miss", 1)
```

`Tracer`会自动记录该事件的时间戳,与应用于整个`Span`的tags相反.也可以将外部提供的时间戳与事件想关联,可以查看[opentracing-go](https://github.com/opentracing/opentracing-go)中`Span`接口的`LogFields`方法.

### 使用外部时间戳记录Span
因为各种各样的原因,在有些场景下会将OpenTracing兼容的tracer集成到服务中.比如一个用户有一个日志文件,其中包含大量来自黑盒系统(如HAProxy)产生的Span数据,为了把这些数据导入到OpenTracing兼容的系统中,API必须提供一种方法通过外部自定义时间戳来记录`Span`.
``` go
span := tracer.StartSpan("operationname", opentracing.StartTime(time.Now()))
span.FinishWithOptions(opentracing.FinishOptions{FinishTime: time.Now()})
```

### 开启tracer之前设置好采样策略
大多数分布式追踪系统都会通过应用不同的采样策略来减少需要记录和处理的追踪数据的总量.有时开发人员希望有一种方式来确保一个tracer数据会被系统记录(采样),如在HTTP请求中包含一个特殊的参数(`debug=true`).OpenTracing API标准化了一些有用的tags,其中一个叫`sampling.priority`(采样优先级):精确的实现是由追踪系统的实现者决定的,但任何大于0(默认)代表一条tracer的高优先级.为了传递这个属性到追踪系统中,需要在追踪前进行预处理,如下:
``` go
b, err := strconv.ParseBool(req.Header.Get("debug"))
if b && err == nil {
	span := tracer.StartSpan(
		"operationname", 
		opentracing.Tag{string(ext.SamplingPriority), 1}, 
	)
}
```

### 追踪消息总线的方案
有两种类别的消息总线需要处理,包括消息队列和发布/订阅(主题).

从追踪的视角来看,消息总线的类型并不重要,只是要将生产者关联的`SpanContext`传播到零个或多个消费者中.然后消费者就有责任创建`Span`来封装都消息的处理,并建立对传播来的`SpanContext`的`FollowsFrom`引用.

以RPC客户端为例,生产者在发送消息之前开启了一个新`Span`,并跟随消息传播该`Span`的`Context`.在消息成功发布到消息总线上后这个`Span`就完成了.下面展示代码是如何实现的:
``` python
def traced_send(message, operation):
    # retrieve current span from propagated message context
    parent_span = get_current_span()

    # start a new span to represent the message producer
    span = tracer.start_span(
        operation_name=operation,
        child_of=parent_span.context,
        tags={'message.destination': message.destination}
    )

    # propagate the Span via message headers
    tracer.inject(
        span.context,
        format=opentracing.TEXT_MAP_FORMAT,
        carrier=message.headers)

    with span:
        messaging_client.send(message)
    except Exception e:
        ...
        raise
```

接下来消费者会判断消息中是否包含了`SpanContext`,如果有,就会用它来与生产者的`Span`建立联系.
``` python
extracted_context = tracer.extract(
    format=opentracing.TEXT_MAP_FORMAT,
    carrier=message.headers
)
span = tracer.start_span(operation_name=operation, references=follows_from(extracted_context))
span.set_tag('message.destination', message.destination)
```

### 基于队列的同步请求-响应
尽管使用不多,但有些消息平台/标准(如JMS)支持在消息头中提供ReplyTo目标的功能.消费者收到消息后,它会将结果返回到指定的目的地.

这种模式常用来模拟同步请求/响应,这种情况下消费者和生产者之间是`ClildOf`的关系.

但此模式也可以用于委托来指示将结果告知第三方.在这种情况下,它将被视为两个单独的消息交换并具有链接每个阶段的`Follows From`关系类型(A->B->C).

由于很难区分这两种情况,不建议将面向消息中间件用于同步的请求/响应模式,因此从建议跟踪角度忽略请求/响应方案.

## 参考
* [语义约定](https://opentracing.io/specification/conventions/)
* [Best Practices](https://opentracing.io/docs/best-practices/)
* [最佳实践](https://wu-sheng.gitbooks.io/opentracing-io/content/pages/instrumentation/)
* [opentracing-tutorial](https://github.com/yurishkuro/opentracing-tutorial)

