# OpenTracing分布式链路追踪


## 概述

### 简介
虽然微服务是一种强大的系统架构,但也伴随着新的问题,就是当微服务数量众多且调用链条过长时,在复杂的网络环境下是很难调试和观察分布式事务,无法直接在内存或堆栈中来调试或观察.

在这种情况下,分布式追踪系统进入到视野之中,分布式追踪系统对于描述和分析跨进程事务的问题提供了解决方案.大部分的分布式追踪系统的思想都来源于[Google's Dapper paper](https://ai.google/research/pubs/pub36356)

### 分布式追踪系统的模型
大多数分布式追踪系统的模型来自于Google's Dapper paper.OpenTracing也是一样的,采用了相同的名词和动词.
![模型](/images/tracing1_0.png "tracing模型")
1. Trace: 用来描述在分布式系统中一个完整的事务(这里的事务不是指数据库中的事务,而是指一个完整的业务流).
2. Span: 可命名的、记录耗时的一个工作流片段,Span上可设置多个key:value的tags,也可以记录某个时间点的结构化的log.
3. SpanContext: 追踪信息会伴随着整个分布式事务,会通过网络或者消息总线来传递到下游服务中.包含了trace id、span id和其它需要传播(分布式追踪系统需要传播到下游的)的数据.

### 四个主要的问题
从应用程序层分布式跟踪系统的角度来看，现代软件系统如下图所示：
![系统](/images/tracing2_0.png "系统结构")

现代软件系统中的组件可分为三大类:
* 应用程序和业务逻辑: 自己的代码.
* 广泛使用的共享库: 别人的代码.
* 广泛使用的服务: 别人的基础设施.

这三类组件有不同的需求,并驱动着负责监控应用程序的分布式追踪系统的设计.最终有四个非常重要的设计要点:
* 追踪系统的API: 应用程序如何使用?
* 传播协议: 在RPC请求中与应用程序一起发送的内容(传递到下游服务中).
* 数据协议: 异步(带外)发送到分析系统中的内容.
* 分析系统: 用于处理追踪数据的数据库和交互式UI.

### OpenTracing是如何解决的?
OpenTracing API提供了标准的、与厂商无关的工具框架.当开发人员想尝试不同的分布式追踪系统时,只需要简单的更改Tracer的配置,而不用为了适配新的分布式追踪系统而重复开发整个追踪过程.

## 什么是分布式追踪?
分布式追踪是一种用来分析和监控应用程序的方法,特别是使用微服务架构的系统.分布式追踪有助于查明发生故障的位置以及导致性能下降的原因.

### 分布式追踪的使用场景
* IT和DevOps团队可以用分布式追踪来监控整个应用程序.分布式追踪特别适合用来调试和监控现代分布式软件体系结构,如微服务.
* 开发人员可以利用分布式追踪来帮助调试和优化代码.

### 什么是OpenTracing?
首先从什么不是OpenTracing开始可能更容易.
* OpenTracing不是一个下载文件或程序.分布式追踪系统要求软件开发人员将追踪代码添加到应用程序的代码中,或者应用程序所使用的框架中.
* OpenTracing不是一个标准,[CNCF](https://www.cncf.io/)不是一个标准化组织.OpenTracing API项目正在努力为分布式追踪系统创建更加标准的API和工具.

OpenTracing是由API规范,已实现该规范的框架和库以及该项目的文档组成.OpenTracing允许开发人员使用不会将其受限于任何一种特定的产品或供应商的API来将追踪代码添加到应用程序中.

关于更多已实现OpenTracing规范的信息,可以查看[已支持的语言列表](https://opentracing.io/docs/supported-languages)和[已支持的分布式追踪系统](https://opentracing.io/docs/supported-tracers/)

## Spans
Span是分布式追踪的主要构建对象,代表分布式系统中已完成的单个工作单元.

分布式系统中的每个组件都会构建一个Span(命名的、记录耗时的一个工作流片段).

Spans可以包含对其它Spans的引用,这样就允许多个Span关联到一个已完成的Trace(把一个请求在分布式系统中的生命周期可视化).

根据OpenTracing规范,每个Span会封装以下内容:
* Operation Name(操作名称).
* 开始时间和结束时间.
* Tags,key:value的集合,伴随整个Span.
* Logs,key:value的集合,记录某个时间点的日志.
* SpanContext.

### Tags
key:value的集合,对Span的自定义标记,可以用来查询、过滤和理解追踪数据.

tags是伴随Span的整个生命周期,在文件[semantic_conventions.md](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)定义了常见场景中Span的常规tags.如`db.instance`表示数据库主机地址,`http.status_code`表示HTTP的响应码,`error`可以设置为`True`表示Span所代表的操作失败了.

### Logs
key:value的集合,可用于抓取Span的特定的日志信息以及应用程序本身的其它调试信息或输出信息.也常用于记录Span某个特定时刻或事件(和tags应用与Span的整个生命周期不同).

### SpanContext
SpanContext用于跨进程边界时携带数据,主要包含两个方面的数据:
1. 依赖于实现的状态来引用trace中不同的span.
    - Tracer定义的spanID和traceID.
2. 任何Baggage Items.
    - 需要跨进程边界传播的key:value数据对.
    - 其它对整个追踪访问有用的数据.

### 举例
```
    t=0            operation name: db_query               t=x

     +-----------------------------------------------------+
     | · · · · · · · · · ·    Span     · · · · · · · · · · |
     +-----------------------------------------------------+

Tags:
- db.instance:"customers"
- db.statement:"SELECT * FROM mytable WHERE foo='bar'"
- peer.address:"mysql://127.0.0.1:3306/customers"

Logs:
- message:"Can't connect to mysql server on '127.0.0.1'(10061)"

SpanContext:
- trace_id:"abc123"
- span_id:"xyz789"
- Baggage Items:
  - special_id:"vsid1738"
```

## Tracers

### 简介
OpenTracing提供了一个开放的、与厂商无关的标准API,用来描述分布式事务,尤其是因果关系、语义和时间.它提供了一个通用的分布式上下文传播框架,该框架由以下API原语组成:
* 在进程间传播元数据上下文.
* 编码和解码元数据上下文之后,通过网络传输它用来进行进程间通信.
* 因果关系追踪: 父子关系、分叉和连接.

OpenTracing消除了众多分布式追踪系统之间的差异.这意味着无论开发人员使用哪个分布式追踪系统,追踪代码都将保持不变.为了在应用程序中使用OpenTracing规范的追踪代码,必须部署兼容OpenTracing的追踪系统,[已支持OpenTracing规范的追踪系统](https://opentracing.io/docs/supported-tracers/).

### Tracer接口
Tracer接口能创建`Spans`,还知道如何跨进程边界注入(序列化)和提取(反序列化)元数据,主要包含三个方面的能力:
* 开启一个新的`Span`.
* 将`SpanContext`注入到`carrier`中.
* 从`carrier`中提取出`SpanContext`.

以Golang语言为例:
``` go
// Tracer is a simple, thin interface for Span creation and SpanContext
// propagation.
type Tracer interface {

	// Create, start, and return a new Span with the given `operationName` and
	// incorporate the given StartSpanOption `opts`. (Note that `opts` borrows
	// from the "functional options" pattern, per
	// http://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)
	//
	// A Span with no SpanReference options (e.g., opentracing.ChildOf() or
	// opentracing.FollowsFrom()) becomes the root of its own trace.
	//
	// Examples:
	//
	//     var tracer opentracing.Tracer = ...
	//
	//     // The root-span case:
	//     sp := tracer.StartSpan("GetFeed")
	//
	//     // The vanilla child span case:
	//     sp := tracer.StartSpan(
	//         "GetFeed",
	//         opentracing.ChildOf(parentSpan.Context()))
	//
	//     // All the bells and whistles:
	//     sp := tracer.StartSpan(
	//         "GetFeed",
	//         opentracing.ChildOf(parentSpan.Context()),
	//         opentracing.Tag{"user_agent", loggedReq.UserAgent},
	//         opentracing.StartTime(loggedReq.Timestamp),
	//     )
	//
	StartSpan(operationName string, opts ...StartSpanOption) Span

	// Inject() takes the `sm` SpanContext instance and injects it for
	// propagation within `carrier`. The actual type of `carrier` depends on
	// the value of `format`.
	//
	// OpenTracing defines a common set of `format` values (see BuiltinFormat),
	// and each has an expected carrier type.
	//
	// Other packages may declare their own `format` values, much like the keys
	// used by `context.Context` (see https://godoc.org/context#WithValue).
	//
	// Example usage (sans error handling):
	//
	//     carrier := opentracing.HTTPHeadersCarrier(httpReq.Header)
	//     err := tracer.Inject(
	//         span.Context(),
	//         opentracing.HTTPHeaders,
	//         carrier)
	//
	// NOTE: All opentracing.Tracer implementations MUST support all
	// BuiltinFormats.
	//
	// Implementations may return opentracing.ErrUnsupportedFormat if `format`
	// is not supported by (or not known by) the implementation.
	//
	// Implementations may return opentracing.ErrInvalidCarrier or any other
	// implementation-specific error if the format is supported but injection
	// fails anyway.
	//
	// See Tracer.Extract().
	Inject(sm SpanContext, format interface{}, carrier interface{}) error

	// Extract() returns a SpanContext instance given `format` and `carrier`.
	//
	// OpenTracing defines a common set of `format` values (see BuiltinFormat),
	// and each has an expected carrier type.
	//
	// Other packages may declare their own `format` values, much like the keys
	// used by `context.Context` (see
	// https://godoc.org/golang.org/x/net/context#WithValue).
	//
	// Example usage (with StartSpan):
	//
	//
	//     carrier := opentracing.HTTPHeadersCarrier(httpReq.Header)
	//     clientContext, err := tracer.Extract(opentracing.HTTPHeaders, carrier)
	//
	//     // ... assuming the ultimate goal here is to resume the trace with a
	//     // server-side Span:
	//     var serverSpan opentracing.Span
	//     if err == nil {
	//         span = tracer.StartSpan(
	//             rpcMethodName, ext.RPCServerOption(clientContext))
	//     } else {
	//         span = tracer.StartSpan(rpcMethodName)
	//     }
	//
	//
	// NOTE: All opentracing.Tracer implementations MUST support all
	// BuiltinFormats.
	//
	// Return values:
	//  - A successful Extract returns a SpanContext instance and a nil error
	//  - If there was simply no SpanContext to extract in `carrier`, Extract()
	//    returns (nil, opentracing.ErrSpanContextNotFound)
	//  - If `format` is unsupported or unrecognized, Extract() returns (nil,
	//    opentracing.ErrUnsupportedFormat)
	//  - If there are more fundamental problems with the `carrier` object,
	//    Extract() may return opentracing.ErrInvalidCarrier,
	//    opentracing.ErrSpanContextCorrupted, or implementation-specific
	//    errors.
	//
	// See Tracer.Inject().
	Extract(format interface{}, carrier interface{}) (SpanContext, error)
}
```

### 设置Tracer
实现了`Tracer`接口的对象,主要用来记录`Spans`并发布到某个位置.应用程序如何处理Tracer对象取决于开发人员:可以直接在整个应用程序中使用它,或将其存储在`GlobalTracer`中.

不同的Tracer实现在初始化时接收参数的方式和接收的参数有所不同,如下:
* 应用程序的追踪组件名称.
* 分布式追踪系统的Endpoint.
* 分布式追踪系统的安全连接.
* 采样策略.

一旦Tracer对象实例被创建出来,就可以用来手工创建`Span`,或传递该对象到框架或库中.

为了不强制用户传递`Tracer`对象,提供了一个全局的`GlobalTracer`实例来存储`Tracer`对象,在任何地方都可以通过该全局实例来获取`Tracer`对象.
``` go
type registeredTracer struct {
	tracer       Tracer
	isRegistered bool
}

var (
	globalTracer = registeredTracer{NoopTracer{}, false}
)

// SetGlobalTracer sets the [singleton] opentracing.Tracer returned by
// GlobalTracer(). Those who use GlobalTracer (rather than directly manage an
// opentracing.Tracer instance) should call SetGlobalTracer as early as
// possible in main(), prior to calling the `StartSpan` global func below.
// Prior to calling `SetGlobalTracer`, any Spans started via the `StartSpan`
// (etc) globals are noops.
func SetGlobalTracer(tracer Tracer) {
	globalTracer = registeredTracer{tracer, true}
}

// GlobalTracer returns the global singleton `Tracer` implementation.
// Before `SetGlobalTracer()` is called, the `GlobalTracer()` is a noop
// implementation that drops all data handed to it.
func GlobalTracer() Tracer {
	return globalTracer.tracer
}
```

### 开启一个新的Trace
当创建一个新的`Span`且该`Span`没有关联到一个父`Span`时,一个新的trace就开启了.当创建一个新的`Span`时,需要为其定义一个`operation name`,主要用来帮助确定`Span`与代码的关联关系.

`Span`之间的关联关系目前支持`ChildOf`和`FollowsFrom`:
* `ChildOf`,表示两个`Span`之间存在父子关系.子`Span`是在父`Span`内执行的一个子流程.
* `FollowsFrom`,表示两个`Span`之间是独立的,父`Span`不依赖新的`Span`的执行结果,主要用于pipiline.
``` go
// ChildOfRef refers to a parent Span that caused *and* somehow depends
// upon the new child Span. Often (but not always), the parent Span cannot
// finish until the child Span does.
//
// An timing diagram for a ChildOfRef that's blocked on the new Span:
//
//     [-Parent Span---------]
//          [-Child Span----]
//
// See http://opentracing.io/spec/
//
// See opentracing.ChildOf()
ChildOfRef SpanReferenceType = iota

// FollowsFromRef refers to a parent Span that does not depend in any way
// on the result of the new child Span. For instance, one might use
// FollowsFromRefs to describe pipeline stages separated by queues,
// or a fire-and-forget cache insert at the tail end of a web request.
//
// A FollowsFromRef Span is part of the same logical trace as the new Span:
// i.e., the new Span is somehow caused by the work of its FollowsFromRef.
//
// All of the following could be valid timing diagrams for children that
// "FollowFrom" a parent.
//
//     [-Parent Span-]  [-Child Span-]
//
//
//     [-Parent Span--]
//      [-Child Span-]
//
//
//     [-Parent Span-]
//                 [-Child Span-]
//
// See http://opentracing.io/spec/
//
// See opentracing.FollowsFrom()
FollowsFromRef
```

### 传播追踪信息
为了在分布式系统中跨进程边界进行追踪,服务需要具备继续追踪每个被客户端注入追踪信息的请求.OpenTracing通过提供了`Inject`和`Extract`方法来实现此目标,将`Span`的上下文编码为载体.`Inject`方法可以将`SpanContext`传递到`carrier`中.举例,传递追踪信息到客户端请求中,这样下游服务就能继续进行跟踪了.`Extract`方法作用是相反的,从`carrier`中提取出`SpanContext`.
![跨进程追踪](/images/tracing_extract.png "跨进程边界追踪")

## Inject和Extract
开发人员在添加跨进程边界的追踪代码时必须懂得OpenTracing规范中定义的`Tracer.Inject`和`Tracer.Extract`的能力.它们在概念上很强大,允许开发人员编写正确和通用的跨进程传播代码,而不用绑定到某种特定的OpenTracing实现上.

无论特定的OpenTracing语言或具体的实现如何,下面会简要介绍Inject和Extract的设计以及正确使用.

### 用于追踪传播的全景图
对于分布式追踪系统来说最困难的部分是分布式.任何追踪系统都需要一种了解许多不同进程中活动之间的因果关系的方式,不论这些进程是通过RPC框架、订阅/发布系统、通用消息队列、HTTP调用、UDP或其它方式连接的.

有些分布式追踪系统(2003年的[Project5](http://dl.acm.org/citation.cfm?id=945454),或2006年的[WAP5](http://www.2006.org/programme/item.php?id=2033)或2014年的[The Mystery Machine](https://www.usenix.org/node/186168))可以推断出跨进程边界的因果关系.

### OpenTracing传播方案的要求
为了使`Inject`和`Extract`方案有效,必须满足以下所有条件:
* 使用OpenTracing在跨进程传播时必须不能依赖特定分布式追踪系统的代码.
* 实现OpenTracing规范的系统必须不能为每种已知的进程间通信机制做特殊处理,否则会有太多的工作,甚至定义不明确.
* 传播机制为了优化可扩展.

### 基本元素:Inject,Extract和Carriers
trace中的任何`SpanContext`都可以注入到OpenTracing称为`Carriers`之中,`Carriers`可以是接口或者结构,可用于进程间通信(IPC).把trace的状态从一个进程传递到另一个进程.OpenTracing规范包含两种`Carriesrs`格式,但也可以自定义格式.

类似的,给定一个被注入了trace的`Carriers`,可以被提取出来从而生成一个`SpanContext`实例,该实例在语义上与被注入到`Carriers`中的保持一致.

**Inject代码**
``` go
carrier := make(opentracing.TextMapCarrier)
err := tracer.Inject(span.Context(), opentracing.TextMap, carrier)
```

**Extract代码**
``` go
carrier := make(opentracing.TextMapCarrier)
for k, v := range md {
    carrier.Set(k, v[0])
}

spanctx, err := tracer.Extract(opentracing.TextMap, carrier)
span := tracer.StartSpan(info.FullMethod, opentracing.ChildOf(spanctx))
```

### Inject/Extract格式
支持OpenTracing规范的所有追踪系统都必须支持两个格式:`text map`格式和`binary`格式.
* text map格式是一个string->string的映射.
* binary格式是不透明的字节数组(并且可能更紧凑和高效).

OpenTracing规范并没有规定怎么去存储这些`Carriers`,但前提是要找到一种方法对传播的SpanContext的trace状态进行编码(例如,在Dapper中定义了`trace_id`,`span_id`,还有采样状态掩码)以及任何key:value的Baggage Items.

不能指望不同的分布式追踪系统(实现了OpenTracing规范的)以兼容的方式注入和提取`SpanContext`,虽然OpenTracing对于跨整个分布式系统的跟踪的具体实现是不可知的,但对于传播双方的进程都使用相同的实现.

**一个端到端的传播例子**
* 客户端进程拥有一个`SpanContext`实例,准备发起一个基于HTTP协议的RPC请求.
* 客户端调用`Tracer.Inject(...)`,传递当前的`SpanContext`实例,采用`text map`格式,把其作为参数.
* Inject把`text map`注入到Carrier中,客户端程序把数据编码写入HTTP协议中(一般是放入headers中).
* 发起HTTP请求,数据跨进程边界传输.
* 在服务端,应用程序从HTTP协议中提取text map数据,并初始化为一个Carrier.
* 服务端程序调用`Tracer.Extract(...)`,传入text map格式的名称和上面生成的Carrier.
* 在没有数据损坏或其它错误的情况下,服务端获取了一个`SpanContext`实例,和客户端的是同一个.

