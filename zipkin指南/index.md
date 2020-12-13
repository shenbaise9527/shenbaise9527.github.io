# Zipkin指南


## 快速启动
可以通过`http://your_host:9411`去访问zipkin UI.

**Docker**
`docker run -d -p 9411:9411 openzipkin/zipkin`

**Java**
需要Java8或更高版本.
``` bash
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

**Source**
可以通过源码来安装并运行.
``` bash
# get the latest source
git clone https://github.com/openzipkin/zipkin
cd zipkin
# Build the server and also make its dependencies
./mvnw -DskipTests --also-make -pl zipkin-server clean install
# Run the server
java -jar ./zipkin-server/target/zipkin-server-*exec.jar
```

## 架构
整体架构如下图所示(来源于[官网](https://zipkin.io/pages/architecture.html)):
![架构](/images/zk-architecture-1.png "架构图")

zipkin已支持的平台和语言[列表](https://zipkin.io/pages/tracers_instrumentation).

## 流程示例
标示符会在服务之间传播,而详细信息会被发送到zipkin.在这两种情况下,追踪库都负责创建有效的追踪并呈现它们.追踪库会确保两种数据之间保持奇偶校验一致性.

下面是http追踪的示例,其中用户代码调用了资源`/foo`.这是一个单独的`Span`,在用户代码收到响应后会被异步发送到zipkin中.
``` 
┌─────────────┐ ┌───────────────────────┐  ┌─────────────┐  ┌──────────────────┐
│ User Code   │ │ Trace Instrumentation │  │ Http Client │  │ Zipkin Collector │
└─────────────┘ └───────────────────────┘  └─────────────┘  └──────────────────┘
       │                 │                         │                 │
           ┌─────────┐
       │ ──┤GET /foo ├─▶ │ ────┐                   │                 │
           └─────────┘         │ record tags
       │                 │ ◀───┘                   │                 │
                           ────┐
       │                 │     │ add trace headers │                 │
                           ◀───┘
       │                 │ ────┐                   │                 │
                               │ record timestamp
       │                 │ ◀───┘                   │                 │
                             ┌─────────────────┐
       │                 │ ──┤GET /foo         ├─▶ │                 │
                             │X-B3-TraceId: aa │     ────┐
       │                 │   │X-B3-SpanId: 6b  │   │     │           │
                             └─────────────────┘         │ invoke
       │                 │                         │     │ request   │
                                                         │
       │                 │                         │     │           │
                                 ┌────────┐          ◀───┘
       │                 │ ◀─────┤200 OK  ├─────── │                 │
                           ────┐ └────────┘
       │                 │     │ record duration   │                 │
            ┌────────┐     ◀───┘
       │ ◀──┤200 OK  ├── │                         │                 │
            └────────┘       ┌────────────────────────────────┐
       │                 │ ──┤ asynchronously report span     ├────▶ │
                             │                                │
                             │{                               │
                             │  "traceId": "aa",              │
                             │  "id": "6b",                   │
                             │  "name": "get",                │
                             │  "timestamp": 1483945573944000,│
                             │  "duration": 386000,           │
                             │  "annotations": [              │
                             │--snip--                        │
                             └────────────────────────────────┘
```

追踪库会异步发送`Span`,是为了防止与追踪系统有关的延迟或故障导致用户代码的延迟或破坏.

## Transport
追踪库发送`Span`时,必须要从被追踪系统传输到Zipkin的collectors组件.目前主要有三种方式传输:HTTP,kafka和Scribe.

## 组件
主要包含四个组件:
* collector
* storage
* query service
* web UI

### Collector
当数据到达zipkin的收集器守护程序后,将对其进行验证、存储及索引,供zipkin收集器进行查找.

**HTTP**
默认HTTP方式是可用的,URI为`POST /api/v1/spans`和`POST /api/v2/spans`,目前主要使用v2版本.支持如下配置项:
|属性|环境变量|描述|
|:--|:--|:--|
|zipkin.collector.http.enabled|COLLECTOR_HTTP_ENABLED|`false`禁用HTTP方式,默认为`true`|

**Kafka**
当参数`KAFKA_BOOTSTRAP_SERVERS`设置为v0.10+版本的Kafka时,该收集器就会启用.支持如下配置项:
|变量|新Consumer配置|描述|
|:--|:--|:--|
|COLLECTOR_KAFKA_ENABLED|N/A|`false`禁用Kafka收集器,默认为`true`|
|KAFKA_BOOTSTRAP_SERVERS|bootstrap.servers|以逗号分隔的broker地址,如127.0.0.1:9092.没有默认值|
|KAFKA_GROUP_ID|group.id|此过程代表的消费组.默认为zipkin|
|KAFKA_TOPIC|N/A|以逗号分隔的topic列表,默认为zipkin|
|KAFKA_STREAMS|N/A|消费topic的线程数,默认为1|

启动命令:`KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:9092 java -jar zipkin.jar`

也可以设置其它[Kafka的conusmer属性](https://kafka.apache.org/documentation/#newconsumerconfigs)

举例:
``` bash
# 容器方式启动kafka
$ export KAFKA_BOOTSTRAP_SERVERS=$(docker-machine ip `docker-machine active`)
# Run Kafka in the background
$ docker run -d -p 9092:9092 \
    --env ADVERTISED_HOST=$KAFKA_BOOTSTRAP_SERVERS \
    --env AUTO_CREATE_TOPICS=true \
    spotify/kafka
# Start the zipkin server, which reads $KAFKA_BOOTSTRAP_SERVERS
$ java -jar zipkin.jar

# 设置多个broker地址
$ KAFKA_BOOTSTRAP_SERVERS=broker1.local:9092,broker2.local:9092 java -jar zipkin.jar

# 备用topic名称
$ KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:9092 java -Dzipkin.collector.kafka.topic=zapkin,zipken -jar zipkin.jar

# 使用系统属性取代环境变量KAFKA_BOOTSTRAP_SERVERS.
$ java -Dzipkin.collector.kafka.bootstrap-servers=127.0.0.1:9092 -jar zipkin.jar
```

**RabbitMQ**
支持如下配置项:
|属性|环境变量|描述|
|:--|:--|:--|
|zipkin.collector.rabbitmq.concurrency|RABBIT_CONCURRENCY|当前消费者数量,默认为1|
|zipkin.collector.rabbitmq.connection-timeout|RABBIT_CONNECTION_TIMEOUT|等待建立连接的超时时间,单位为毫秒,默认为60000(1分钟)|
|zipkin.collector.rabbitmq.queue|RABBIT_QUEUE|队列名称,默认为zipkin|
|zipkin.collector.rabbitmq.uri|RABBIT_URI|rabbitmq完整的uri,如:`amqp://user:pass@host:10000/vhost`|
如果uri被设置了,下面的配置项将会被忽略:
|属性|环境变量|描述|
|:--|:--|:--|
|zipkin.collector.rabbitmq.addresses|RABBIT_ADDRESSES|逗号分隔的rabbitmq地址,如:`localhost:5672,localhost:5673`|
|zipkin.collector.rabbitmq.password|RABBIT_PASSWORD|连接rabbitmq的密码,默认为guest|
|zipkin.collector.rabbitmq.username|RABBIT_USER|连接rabbitmq的用户名,默认为guest|
|zipkin.collector.rabbitmq.virtual-host|RABBIT_VIRTUAL_HOST|rabbitmq的虚拟主机名,默认为`/`|
|zipkin.collector.rabbitmq.use-ssl|RABBIT_USE_SSL|设置为`true`,表示使用ssl连接到rabbitmq|

队列会被申明为持久化的,收集器使用单个conn连接到rabbitmq,通过配置的`concurrency`数量的线程(每个线程一个channel)来消费消息.消费消息时`autoAck`设置为`on`了,表示消费者收到消息后rabbitmq就会在队列中自动删除该消息,若消费者出现异常是不能再重复消费该消息的.

启动命令:`RABBIT_ADDRESSES=localhost java -jar zipkin.jar`

**ActiveMQ**
支持ActiveMQ v5.x版本.
|属性|环境变量|描述|
|:--|:--|:--|
|zipkin.collector.activemq.enabled|COLLECTOR_ACTIVEMQ_ENABLED|`false`表示禁用该收集器,默认为`true`|
|zipkin.collector.activemq.url|ACTIVEMQ_URL|ActiveMQ broker地址,如:tcp://localhost:61616或者故障转移:(tcp://localhost:61616,tcp://remotehost:61616)|
|zipkin.collector.activemq.queue|ACTIVEMQ_QUEUE|队列名,默认为zipkin|
|zipkin.collector.activemq.client-id-prefix|ACTIVEMQ_CLIENT_ID_PREFIX|消费者的客户端ID的前缀,默认为zipkin|
|zipkin.collector.activemq.concurrency|ACTIVEMQ_CONCURRENCY|消费者数量,默认为1|S
|zipkin.collector.activemq.username|ACTIVEMQ_USERNAME|连接到ActiveMQ时的用户名,可选|
|zipkin.collector.activemq.password|ACTIVEMQ_PASSWORD|连接到ActiveMQ时的密码,可选|

启动命令:`ACTIVEMQ_URL=tcp://localhost:61616 java -jar zipkin.jar`

### Storage
存储组件是采用插件化的方式实现,目前支持InMemory、Cassandra、ElasticSearch和MySQL,还有其它第三方实现的.

**InMemory**
默认启动的就是In-Memory方式,所有数据全部保存在内存中,没有持久化功能.
启动方式:
``` bash
# 默认启动方式
java -jar zipkin.jar

# 指定STORAGE_TYPE为mem
STORAGE_TYPE=mem java -jar zipkin.jar
```

提供了参数`MEM_MAX_SPANS`来控制`Span`占用的内存大小.当碰到`out-of-memory`错误时,可以增大该参数或者调整堆大小(-Xmx).
``` bash
`MEM_MAX_SPANS`: Oldest traces (and their spans) will be purged first when this limit is exceeded. Default 500000

# 调整内存大小.
MEM_MAX_SPANS=1000000 java -Xmx1G -jar zipkin.jar
```

**MySQL**
基于MySQL5.7版本,需要先建库建表,[脚本文件地址](https://github.com/openzipkin/zipkin/tree/master/zipkin-storage/mysql-v1).
``` sql
# install the schema and indexes
$ mysql -uroot -e "create database if not exists zipkin"
$ mysql -uroot -Dzipkin < zipkin-storage/mysql-v1/src/main/resources/mysql.sql

# 根据trace id来查询
select * from zipkin_spans where trace_id = x'27960dafb1ea7454'
```

启动时可设置的相关参数
* `MYSQL_DB`: The database to use. Defaults to "zipkin".
* `MYSQL_USER` and `MYSQL_PASS`: MySQL authentication, which defaults to empty string.
* `MYSQL_HOST`: Defaults to localhost
* `MYSQL_TCP_PORT`: Defaults to 3306
* `MYSQL_MAX_CONNECTIONS`: Maximum concurrent connections, defaults to 10
* `MYSQL_USE_SSL`: Requires `javax.net.ssl.trustStore` and `javax.net.ssl.trustStorePassword`, defaults to false.

启动命令:`STORAGE_TYPE=mysql MYSQL_USER=root java -jar zipkin.jar`

**Cassandra**

**Elasticsearch**
启动时可设置的相关参数
* `ES_HOSTS`: A comma separated list of elasticsearch base urls to connect to ex. http://host:9200. Defaults to "http://localhost:9200".
* `ES_PIPELINE`: Indicates the ingest pipeline used before spans are indexed. No default.
* `ES_TIMEOUT`: Controls the connect, read and write socket timeouts (in milliseconds) for Elasticsearch API. Defaults to 10000 (10 seconds)
* `ES_INDEX`: The index prefix to use when generating daily index names. Defaults to zipkin.
* `ES_DATE_SEPARATOR`: The date separator to use when generating daily index names. Defaults to '-'.
* `ES_INDEX_SHARDS`: The number of shards to split the index into. Each shard and its replicas are assigned to a machine in the cluster. Increasing the number of shards and machines in the cluster will improve read and write performance. Number of shards cannot be changed for existing indices, but new daily indices will pick up changes to the setting. Defaults to 5.
* `ES_INDEX_REPLICAS`: The number of replica copies of each shard in the index. Each shard and its replicas are assigned to a machine in the cluster. Increasing the number of replicas and machines in the cluster will improve read performance, but not write performance. Number of replicas can be changed for existing indices. Defaults to 1. It is highly discouraged to set this to 0 as it would mean a machine failure results in data loss.
* `ES_ENSURE_TEMPLATES`: Installs Zipkin index templates when missing. Setting this to false can lead to corrupted data when index templates mismatch expectations. If you set this to false, you choose to troubleshoot your own data or migration problems as opposed to relying on the community for this. Defaults to true.
* `ES_USERNAME` and `ES_PASSWORD`: Elasticsearch basic authentication, which defaults to empty string. Use when X-Pack security (formerly Shield) is in place.
* `ES_CREDENTIALS_FILE`: The location of a file containing Elasticsearch basic authentication credentials, as properties. The username property is `zipkin.storage.elasticsearch.username`, password `zipkin.storage.elasticsearch.password`.This file is reloaded periodically, using `ES_CREDENTIALS_REFRESH_INTERVAL` as the interval. This parameter takes precedence over ES_USERNAME and ES_PASSWORD when specified.
* `ES_CREDENTIALS_REFRESH_INTERVAL`: Credentials refresh interval in seconds, which defaults to 1 second. This is the maximum amount of time spans will drop due to stale credentials. Any errors reading the credentials file occur in logs at this rate.
* `ES_HTTP_LOGGING`: When set, controls the volume of HTTP logging of the Elasticsearch API. Options are BASIC, HEADERS, BODY
* `ES_SSL_NO_VERIFY`: When true, disables the verification of server's key certificate chain. This is not appropriate for production. Defaults to false.
* `ES_TEMPLATE_PRIORITY`: The priority value of the composable index templates. This is only applicable for ES version 7.8 or above. Must be set, even to 0, to use composable template

启动命令:`STORAGE_TYPE=elasticsearch ES_HOSTS=http://myhost:9200 java -jar zipkin.jar`

### Query Service
查询服务提供简单的`JSON API`来查询和检索追踪信息,主要是供Web UI使用.

### Web UI
以网页的形式来展示追踪数据.

## 客户端代码
基于OpenTracing来使用zipkin,以[Golang](https://golang.org)为例.
依赖于[zipkin-go](https://github.com/openzipkin/zipkin-go)、[zipkin-go-opentracing](https://github.com/openzipkin-contrib/zipkin-go-opentracing)和[opentracing-go](https://github.com/opentracing/opentracing-go)这三个库.

### Reporter接口
接口定义:
``` go
// Tracer依赖该接口来发送Span数据.
type Reporter interface {
    Send(model.SpanModel)
    Close() error
}
```

具体实现:
**[noopReporter](https://github.com/openzipkin/zipkin-go/blob/master/reporter/reporter.go)**
具体实现都是空的,不做任何处理.
``` go
// 创建接口.
reporter := zipkin.NewNoopReporter()
```

**[httpReporter](https://github.com/openzipkin/zipkin-go/blob/master/reporter/http/http.go)**
基于HTTP协议来发送Span数据.

函数原型:
``` go
// url表示zipkin服务端接收数据的端点,如http://127.0.0.1:9411/api/v2/spans.
func NewReporter(url string, opts ...ReporterOption) reporter.Reporter
```

采用了`Option`设计方式来对相关参数进行自定义处理.
``` go
// 设置http的超时时间,默认为5秒.
func Timeout(duration time.Duration) ReporterOption

// 设置最大的待发送数量,当到达此阈值后就会触发收集操作,默认为100.
func BatchSize(n int) ReporterOption

// 设置触发收集操作的时间间隔,默认为1秒.可见触发收集操作有两个因素,当间隔时间达到指定时间或者待发送数量超过batchsize,就会立即触发.
func BatchInterval(d time.Duration) ReporterOption

// 设置最大的缓存数量,当超过此阈值,从队列开头到超出数量的Span将会被丢弃,默认为1000.
func MaxBacklog(n int) ReporterOption

// 设置回调,在发送span到zipkin之前会被调用,默认为nil,回调定义是type RequestCallbackFn func(*http.Request).
func RequestCallback(rc RequestCallbackFn) ReporterOption

// 设置日志对象,默认为Stderr
func Logger(l *log.Logger) ReporterOption

// 设置span的序列化方式,默认实现的是json格式.
func Serializer(serializer reporter.SpanSerializer) ReporterOption

// 设置发送请求到zipkin收集器的方式,默认为http.Client.
func Client(client HTTPDoer) ReporterOption
```

**[rmqReporter](https://github.com/openzipkin/zipkin-go/blob/master/reporter/amqp/amqp.go)**
把Span数据发送到[rabbitmq](https://www.rabbitmq.com/)消息总线上.

函数原型:
``` go
// address表示rabbitmq的地址,如:amqp://guest:guest@localhost:5672/test
func NewReporter(address string, opts ...ReporterOption) (reporter.Reporter, error)
```

采用了`Option`设计方式来对相关参数进行自定义处理.
``` go
// 设置日志对象,默认为Stderr
func Logger(logger *log.Logger) ReporterOption

// 设置rabbitmq中交换器的名字,默认为zipkin.
func Exchange(exchange string) ReporterOption

// 设置rabbitmq队列的名字,默认为zipkin.
func Queue(queue string) ReporterOption

// 设置channel通道对象,默认为nil.
func Channel(ch *amqp.Channel) ReporterOption

// 设置与rabbitmq连接的对象,默认为nil.
func Connection(conn *amqp.Connection) ReporterOption
```

rabbitmq的交换器类型默认为`direct`,交换器和队列默认都是持久化的,非独占的.

**[kafkaReporter](https://github.com/openzipkin/zipkin-go/blob/master/reporter/kafka/kafka.go)**
把Span数据发送到[kafka](https://kafka.apache.org/)中.

函数原型:
``` go
// address为broker地址列表.
func NewReporter(address []string, options ...ReporterOption) (reporter.Reporter, error) {
```

采用了`Option`设计方式来对相关参数进行自定义处理.
``` go
// 设置日志对象,默认为Stderr
func Logger(logger *log.Logger) ReporterOption

// 设置生产者.
func Producer(p sarama.AsyncProducer) ReporterOption

// 设置topic名字,默认为zipkin
func Topic(t string) ReporterOption

// 设置span的序列化方式,默认实现的是json格式.
func Serializer(serializer reporter.SpanSerializer) ReporterOption
```

**[logReporter](https://github.com/openzipkin/zipkin-go/blob/master/reporter/log/log.go)**
把Span数据发送到日志对象中.

函数原型:
``` go
func NewReporter(l *log.Logger) reporter.Reporter
```

只是把Span数据记录到指定的日志对象中,并不会发送到zipkin中.

### NewTracer函数
函数原型:
``` go
// rep指定Reporter.
func NewTracer(rep reporter.Reporter, opts ...TracerOption) (*Tracer, error)
```
当指定的rep为nil时,会默认创建为noopRepoerter.

采用了`Option`设计方式来对相关参数进行自定义处理.
``` go
// 设置被追踪服务的本地endpoint,可调用zipkin.NewEndpoint来生成对应的Endpoint对象,默认为nil.
func WithLocalEndpoint(e *model.Endpoint) TracerOption

// 设置采样策略,默认为AlwaysSample.
func WithSampler(sampler Sampler) TracerOption
```

针对采样策略,`Sampler`原型为`type Sampler func(id uint64) bool`,可按照需求来自定义采样策略.官方默认提供了如下几种方式:
**AlwaysSample**
``` go
// 所有span都会被发送到zipkin.
func AlwaysSample(_ uint64) bool { return true }
```

**NewModuloSampler**
``` go
// 当mod小于2时采用AlwaysSmaple.当mod大于2时,若trace id对mod求余为0就会被发送zipkin.
func NewModuloSampler(mod uint64) Sampler
```

**NewBoundarySampler**
``` go
// BoundarySampler适用于提供随机trace id且仅做出一次采样决定的高流量场景.它可以防止集群中的节点选择完全相同的ID.
func NewBoundarySampler(rate float64, salt int64) (Sampler, error)
```

**NewCountingSampler**
``` go
// CountingSampler适用于低流量或不提供随机trace id的场景,由于采样决策不是幂等的(根据traceid一致),因此不适用于收集器.
func NewCountingSampler(rate float64) (Sampler, error)
```

### 完整例子
``` go
import (
	"io"

	"github.com/opentracing/opentracing-go"
	zipkinot "github.com/openzipkin-contrib/zipkin-go-opentracing"
	"github.com/openzipkin/zipkin-go"
	zipkinhttp "github.com/openzipkin/zipkin-go/reporter/http"
)

type noopZkCloser struct{}

func (noopZkCloser) Close() error {
	return nil
}

// newTracer 创建基于zipkin的tracer对象.
func newTracer(tracingURL, serverName, localEndpoint string) (opentracing.Tracer, io.Closer, error) {
    // 基于HTTP的Reporter.
    zipkinReporter := zipkinhttp.NewReporter(tracingURL)
    
    // 创建localendpoint对象.
	endpoint, err := zipkin.NewEndpoint(serverName, localEndpoint)
	if err != nil {
		return nil, nil, err
	}

    // 创建zipkin原生的tracer对象,采样策略使用默认的AlwaysSample.
	nativeTracer, err := zipkin.NewTracer(zipkinReporter, zipkin.WithLocalEndpoint(endpoint))
	if err != nil {
		return nil, nil, err
	}

    // 把zipkin原生tracer对象包装成OpenTracing的tracer对象.
	return zipkinot.Wrap(nativeTracer), noopZkCloser{}, nil
}
```

