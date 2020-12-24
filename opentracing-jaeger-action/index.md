# Jaeger实战


## 部署

### 基于Docker
所有组件基于Docker,后端存储采用Elasticsearch.docker-compose.yml文件如下:
``` yml
version: '2.2'
services:
    elasticsearch:
        image: elasticsearch:6.8.13
        volumes:
            - "data01:/usr/share/elasticsearch/data"
        ports:
            - "9200:9200"
        networks:
            - elastic-jaeger
        environment:
            - TZ=Asia/Shanghai
            - node.name=es01
            - cluster.name=es-docker-cluster
            - discovery.type=single-node
            - http.host=0.0.0.0
            - transport.host=127.0.0.1
            - ES_JAVA_OPTS=-Xms512m -Xmx512m
            - xpack.security.enabled=false
        restart: unless-stopped
    jaeger-collector:
        image: jaegertracing/jaeger-collector:latest
        ports:
            - 14268:14268
            - 14250:14250
        networks:
            - elastic-jaeger
        environment:
            - TZ=Asia/Shanghai
            - SPAN_STORAGE_TYPE=elasticsearch
        command: [
            "--es.server-urls=http://elasticsearch:9200", 
            "--es.num-shards=1", 
            "--es.num-replicas=0", 
            "--log-level=debug"
        ]
        restart: unless-stopped
        depends_on: 
            - elasticsearch
    jaeger-agent:
        image: jaegertracing/jaeger-agent:latest
        ports:
            - 6831:6831/udp
            - 6832:6832/udp
            - 5778:5778
        networks:
            - elastic-jaeger
        environment:
            - TZ=Asia/Shanghai
            - SPAN_STORAGE_TYPE=elasticsearch
        command: [
            "--reporter.grpc.host-port=jaeger-collector:14250", 
            "--log-level=debug"
        ]
        restart: unless-stopped
        depends_on: 
            - jaeger-collector
    jaeger-query:
        image: jaegertracing/jaeger-query:latest
        ports:
            - 16686:16686
            - 16687:16687
        networks:
            - elastic-jaeger
        environment:
            - TZ=Asia/Shanghai
            - SPAN_STORAGE_TYPE=elasticsearch
            - no_proxy=localhost
        command: [
            "--es.server-urls=http://elasticsearch:9200", 
            "--span-storage.type=elasticsearch", 
            "--log-level=debug"
        ]
        restart: unless-stopped
        depends_on: 
            - jaeger-agent

volumes:
    data01:
        driver: local
        driver_opts:
            type: none
            device: ./es/data
            o: bind

networks:
    elastic-jaeger:
        driver: bridge
```

**注意:Elasticsearch是以单节点方式启动的**

执行如下命令:
``` bash
# 先启动Elasticsearch,等服务启动成功,通过docker logs jaeger_elasticsearch_1来观察.
$ docker-compose up -d elasticsearch   
Creating network "jaeger_elastic-jaeger" with driver "bridge"
Creating jaeger_elasticsearch_1 ... done

# 再启动其它服务.
$ docker-compose up -d  
jaeger_elasticsearch_1 is up-to-date
Creating jaeger_jaeger-collector_1 ... done
Creating jaeger_jaeger-agent_1     ... done
Creating jaeger_jaeger-query_1     ... done

# 查看状态.
$ docker-compose ps 
          Name                         Command               State                                    Ports                                  
---------------------------------------------------------------------------------------------------------------------------------------------
jaeger_elasticsearch_1      /usr/local/bin/docker-entr ...   Up      0.0.0.0:9200->9200/tcp, 9300/tcp
jaeger_jaeger-agent_1       /go/bin/agent-linux --repo ...   Up      5775/udp, 0.0.0.0:5778->5778/tcp, 0.0.0.0:6831->6831/udp, 0.0.0.0:6832->6832/udp
jaeger_jaeger-collector_1   /go/bin/collector-linux -- ...   Up      0.0.0.0:14250->14250/tcp, 0.0.0.0:14268->14268/tcp
jaeger_jaeger-query_1       /go/bin/query-linux --es.s ...   Up      0.0.0.0:16686->16686/tcp, 0.0.0.0:16687->16687/tcp
```

如下地址需要注意:
* `http://elasticsearch:9200`,是Elasticsearch暴露的地址,供Collector和Query服务使用,通过参数`es.server-urls`来设置.
* `http://127.0.0.1:16686`,是Query服务对外暴露的地址,用来查看Jaeger UI.
* `jaeger-collector:14250`,是Collector暴露的gRPC端口,供Agent发送Span数据到Collector,通过参数`reporter.grpc.host-port`来设置.
* `http://127.0.0.1:14268`,是Collector暴露的http端口,供应用程序直接通过HTTP协议发送Span数据到Collector,端点为`/api/traces`.
* `127.0.0.1:6831`,是Agent对外暴露的UDP端口,供应用程序通过UDP协议发送Span数据到Agent.

### 基于源码
下载源码:
``` bash
# 从github中下载.
$ git clone git@github.com:jaegertracing/jaeger.git
Cloning into 'jaeger'...
remote: Enumerating objects: 34, done.
remote: Counting objects: 100% (34/34), done.
remote: Compressing objects: 100% (33/33), done.
remote: Total 15400 (delta 7), reused 14 (delta 1), pack-reused 15366
Receiving objects: 100% (15400/15400), 19.79 MiB | 16.00 KiB/s, done.
Resolving deltas: 100% (10130/10130), done.

# 使用v1.21.0tag所对应的版本.
$ git checkout -b 1.21.0 v1.21.0
Switched to a new branch '1.21.0'

# 加载子模块.
$ git submodule update --init --recursive
Submodule 'idl' (https://github.com/jaegertracing/jaeger-idl.git) registered for path 'idl'
Submodule 'jaeger-ui' (https://github.com/jaegertracing/jaeger-ui.git) registered for path 'jaeger-ui'
Cloning into 'D:/project/jaeger/idl'...
Cloning into 'D:/project/jaeger/jaeger-ui'...
```

依赖安装:
``` bash
# 安装yarn(facebook发布的一款取代npm的包管理工具),先要安装node.js.
$ npm install -g yarn

# 查看yarn版本.
$ yarn --version
1.22.10

# 配置淘宝源.
$ yarn config set registry https://registry.npm.taobao.org -g
yarn config v1.22.10
success Set "registry" to "https://registry.npm.taobao.org".
Done in 0.13s.

$ yarn config set sass_binary_site http://cdn.npm.taobao.org/dist/node-sass -g
yarn config v1.22.10
success Set "sass_binary_site" to "http://cdn.npm.taobao.org/dist/node-sass".
Done in 0.07s.

# 编译ui.
$ make build-ui
```

在Windows下编译各个服务:
``` bash
# 编译agent服务.
PS D:\project\jaeger> go build .\cmd\agent

# 编译collector服务.
PS D:\project\jaeger> go build .\cmd\collector

# 编译query服务.
PS D:\project\jaeger> go build -tags=ui .\cmd\query
```

在Windows下启动服务:
``` bash
# 启动Collector.
D:\project\jaeger>collector.exe --span-storage.type=elasticsearch --es.server-urls=http://192.168.20.151:9200 --es.num-shards=1 --es.num-replicas=0 --log-level=debug

# 启动Agent,指定Collector地址.
D:\project\jaeger>agent.exe --reporter.grpc.host-port=127.0.0.1:14250 --log-level=debug

# 启动Query.
D:\project\jaeger>query.exe --es.server-urls=http://192.168.20.151:9200 --log-level=debug
```

## 基于Golang使用Jaeger

### 初始化Jaeger Tracer
可以采用UDP协议连接到Agent上,也可以采用HTTP协议直连到Collector上,两个参数是互斥的.
``` go
cfg := jaegercfg.Configuration{
    ServiceName: "goods",
    // 采样策略,这里使用Const,全部采样.
    Sampler: &jaegercfg.SamplerConfig{
        Type:  "const",
        Param: 1.0,
    },
    Reporter: &jaegercfg.ReporterConfig{
        BufferFlushInterval: time.Second,
        LocalAgentHostPort:  "192.168.20.153:6831", // 采用UDP协议连接Agent.
        //CollectorEndpoint:   "http://192.168.20.153:14268/api/traces", // 采用HTTP协议直连Collector
    },
}

// 根据配置生成Tracer对象,启用Span的内存池.
tracer, closer, err := cfg.NewTracer(jaegercfg.PoolSpans(true))
if err != nil {
    fmt.Println(err)

    return
}

// 调用Close,释放资源.
defer closer.Close()

// 注册为opentracing里的GlobalTracer对象.
opentracing.SetGlobalTracer(tracer)
```

### 函数追踪
直接利用`opentracing.StartSpan`来创建Span,其OperationName为zerosql.
``` go
func tracedSQL() {
	span := opentracing.StartSpan("zerosql")
	defer span.Finish()
	dsn := "root:123456@tcp(127.0.0.1:3306)/xxx?charset=utf8&parseTime=true&loc=Local"
	db := NewZeroMysql(dsn)
	goodsql := "select * from goods where goodsid = ?"
	var goodsinfo GoodsInfo
	err := db.QueryRow(&goodsinfo, goodsql, 1)
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println(goodsinfo)
}
```

### http中间件
``` go
// 主要记录http的statuscode.
type withHTTPCodeResponse struct {
	writer http.ResponseWriter
	code   int
}

func (w *withHTTPCodeResponse) Header() http.Header {
	return w.writer.Header()
}

func (w *withHTTPCodeResponse) Write(bytes []byte) (int, error) {
	return w.writer.Write(bytes)
}

func (w *withHTTPCodeResponse) WriteHeader(code int) {
	w.writer.WriteHeader(code)
	w.code = code
}

// HttpTracing http.Handler.
func HttpTracing(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
        // 获取全局tracer对象.
		tracer := opentracing.GlobalTracer()
        // 尝试从http的Header中提取上游的SpanContext.
		spanCtx, _ := tracer.Extract(
			opentracing.HTTPHeaders,
			opentracing.HTTPHeadersCarrier(r.Header))
		span := tracer.StartSpan(r.RequestURI, opentracing.ChildOf(spanCtx))
		ext.HTTPMethod.Set(span, r.Method)
		defer span.Finish()
		cw := &withHTTPCodeResponse{writer: w}
		rc := opentracing.ContextWithSpan(r.Context(), span)
		r = r.WithContext(rc)
		defer func() {
            // 设置statuscode和error.
			ext.HTTPStatusCode.Set(span, uint16(cw.code))
			if cw.code >= http.StatusBadRequest {
				ext.Error.Set(span, true)
			}
		}()

		next(cw, r)
	}
}
```

### gRPC一元客户端拦截器
``` go
// OpenTracingClientInterceptor grpc unary clientinterceptor.
func OpenTracingClientInterceptor() grpc.UnaryClientInterceptor {
	return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        // 从Context中尝试提取Span对象.
		span := ChildOfSpanFromContext(ctx, method)
        // 设置tag.
		ext.Component.Set(span, "grpc")
		ext.SpanKindRPCClient.Set(span)
		defer span.Finish()
		carrier := make(opentracing.TextMapCarrier)
		tracer := opentracing.GlobalTracer()
        // 把SpanContext注入到Carrier中.
		err := tracer.Inject(span.Context(), opentracing.TextMap, carrier)
		if err == nil {
			var pairs []string
			_ = carrier.ForeachKey(func(key, val string) error {
				pairs = append(pairs, key, val)
				return nil
			})

            // 然后把这些数据放入到gRPC的HEADER中.
			ctx = metadata.AppendToOutgoingContext(ctx, pairs...)
		}

		err = invoker(ctx, method, req, reply, cc, opts...)
		if err != nil {
            // 若返回错误,把错误日志记录到Span中.
			ext.LogError(span, err)
		}

		return err
	}
}

// ChildOfSpanFromContext 根据context中的span生成ChildOf的span.
func ChildOfSpanFromContext(ctx context.Context, operationName string) opentracing.Span {
	return newSubSpanFromContext(ctx, operationName, opentracing.ChildOf)
}

func newSubSpanFromContext(
	ctx context.Context,
	operationName string,
	op func(opentracing.SpanContext) opentracing.SpanReference) opentracing.Span {
	tracer := opentracing.GlobalTracer()
	span := opentracing.SpanFromContext(ctx)
	if span == nil {
		span = tracer.StartSpan(operationName)
	} else {
		span = tracer.StartSpan(operationName, op(span.Context()))
	}

	return span
}
```

### gRPC一元服务端拦截器
``` go
// OpenTracingServerInterceptor grpc unary serverinterceptor.
func OpenTracingServerInterceptor() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
		md, ok := metadata.FromIncomingContext(ctx)
		var spanCtx opentracing.SpanContext
		tracer := opentracing.GlobalTracer()
		if ok {
            // 从gRPC的元数据中提取SpanContext.
			carrier := make(opentracing.TextMapCarrier)
			for k, v := range md {
				carrier.Set(k, v[0])
			}

            // 提取成功,生成子Span对象.
			spanCtx, _ = tracer.Extract(opentracing.TextMap, carrier)
		}

		span := tracer.StartSpan(info.FullMethod, ext.RPCServerOption(spanCtx))
        // 设置tag.
		ext.Component.Set(span, "grpc")
		defer span.Finish()
        // 把Span对象放入到Context中,来传递到业务代码中.
		ctx = opentracing.ContextWithSpan(ctx, span)
		resp, err = handler(ctx, req)
		if err != nil {
			ext.LogError(span, err)
		}

		return
	}
}
```

## 参考
* [tracing库](https://github.com/shenbaise9527/tracing)

