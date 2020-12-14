# Jaeger指南


## 系统架构

### 组件
jaeger即可以作为单体应用来部署(所有jaeger后台组件全部运行在一个进程内),也可以作为一个可扩展的分布式系统来部署.如下所述.有两个主要的部署选项:
* Collectors直接把数据写入到存储中.
* Collectors把数据写入到kafka中,作为缓冲,再异步写入存储.

第一种情况,直接写入存储:
![架构图1](/images/jaeger-architecture-v1.png "直接写入存储的架构图")

第二种情况,先写入kafka:
![架构图2](/images/jaeger-architecture-v2.png "写入kafka的架构图")

### 客户端库
jaeger客户端是基于OpenTracing API的特定语言实现.它们可以为应用程序提供分布式追踪的能力,并能与已经集成OpenTracing的开源框架(如Flask,Dropwizard,gRPC等等)一起很好的协作.

当服务接收到新请求时会同时收到附加的上下文信息(trace id,span id和baggage),服务会基于上下文创建Span,然后继续往下游发送.在请求中仅仅相关id和baggage会被传播;而其它数据,如operation name,timing,tags和logs都不会被传播.这些数据会在后台被异步发送给jaeger服务.

为了降低负载,jager客户端提供了各种不同的采样策略.当一个trace被采样,其所有的span数据都会被抓取并被发送到jaeger服务.如果trace没有被采样,其所有的数据都不会被收集,并会对OpenTracing API实行熔断以减少开销.默认情况下,客户端的采样率为0.1%,并能从jaeger服务获取到采样策略(见上面架构图的adaptive sampling).
![传播](/images/jaeger-context-prop.png "传播")

### Agent
agent是一个网络代理服务,主要基于UDP协议来接收Span,然后把Span分批的发送给收集器.它是作为基础结构组件部署到所有主机上.该代理抽象的将收集器的路由和发现从客户端分离.

### Collector
collector从agent接收追踪数据,然后基于数据进行一系列的操作,当前操作有验证数据,建立索引,执行相关转换,最后存储.

存储是基于插件化设计的,当前支持Cassandra, Elasticsearch and Kafka.

### Query
查询服务,从存储中检索trace,然后在UI上展示.

### Ingester
从kafka中读取数据,然后写入其它的存储中(如Cassandra, Elasticsearch)

## Reporting APIs
Agent和Collector这两个组件都能接收Span数据.目前它们支持两组不重叠的API.

### Thrift over UDP
Agent只能基于UDP协议接收Thrift格式的Span数据.主要的API是处理的UDP包,其中包含[jaeger.thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/jaeger.thrift) IDL文件中定义的Thrift编码的Batch结构,该IDL文件位于[jaeger-idl](https://github.com/jaegertracing/jaeger-idl/)代码库中.大多数客户端库使用Thrift紧凑模式编码,但有些客户端库不支持(如Node.js),其会使用Thrift二进制模式编码(发送到不同的UDP端口上).API都定义在[agent.thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/agent.thrift) IDL文件中.

由于历史原因,Agent还能接收Zipkin格式的Span数据,但只有老版本的客户端库才能发送该格式的数据,目前官方已经正式弃用了.
|Port|Protocol|Component|Function|
|:--|:--|:--|:--|
|5775|UDP|agent|接收`zipkin.thrift`的thrift协议的紧凑格式(已弃用,老版本客户端能使用)|
|6831|UDP|agent|接收`jaeger.thrift`的thrift协议的紧凑格式|
|6832|UDP|agent|接收`jaeger.thrift`的thrift协议的二进制格式|
|5778|HTTP|agent|服务配置|

### gRPC
在典型的jaeger部署中,Agent会从客户端接收Span数据然后将其转发给Collector.从jaeger 1.11版本开始,Agent与Collector之间的官方推荐协议是gRPC与Protobuf,协议定义在[collector.proto](https://github.com/jaegertracing/jaeger-idl/blob/master/proto/api_v2/collector.proto) IDL文件中.

### Thrift over HTTP
在某些情况下,将Agent和应用程序部署在一起是不可行的,比如当应用程序代码作为AWS的Lambda函数运行时.在这些场景下,jaeger客户端可以通过HTTP/HTTPS协议直接把Span数据发送给Collector.

基于相同[jaeger.thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/jaeger.thrift) IDL文件定义的数据可以以HTTP Post请求被提交给`/api/traces`端点,如`https://jaeger-collector:14268/api/traces`.Batch结构需要使用Thrift二进制格式编码,在HTTP头中要设置特殊的content type,`Content-Type: application/vnd.apache.thrift.binary`.
|Port|Protocol|Component|Function|
|:--|:--|:--|:--|
|14268|HTTP|collector|直接从客户端接收`jaeger.thrift`|

### Zipkin格式
Collector也可以接受来自Zipkin格式的Span数据,基于JSON v1/v2和Thrift.Collector需要配置为启用Zipkin HTTP服务,如在9411端口启用被当做Zipkin收集器.服务启用了两个端点来接受POST请求:
* `/api/v1/spans`,用来接受Zipkin的JSON v1或Thrift格式的数据.
* `/api/v2/spans`,用来接受Zipkin的JSON v2格式的数据.

|Port|Protocol|Component|Function|
|:--|:--|:--|:--|
|9411|HTTP|collector|适配Zipkin(可选)|

## 采样策略
Jaeger库实现了一致性的前端(或基于头部的)采样.举例,假设我们有一个简单的调用图,其中服务A调用服务B,服务B调用服务C:`A->B->C`.当服务A接收到一个没有包含追踪信息的请求时,会开启一个新的`trace`,并分配一个随机的trade ID,然后基于当前的采样策略会对该`trace`做出一个采样决策.这个决策会跟随这个请求传播到服务B和服务C,因此服务B和C不在需要进行采样决策,而是尊重顶级服务A的采样决策.这种方法保证了如果一个`trace`被采样了,其所有的`spans`都会被记录到Jaeger服务中.如果每个服务都在做自己的采样决策,那么在Jaeger服务中很少能获取到完整的追踪信息.

### 客户端采样配置
当使用配置对象去实例化一个tracer时,可以根据`sampler.type`和`sampler.param`属性来选择采样类型.Jaeger库支持如下采样策略:
* Constant(`sampler.type=const`),该采样策略会对所有trace采用相同的决策.如果`sampler.param=1`将对所有trace都采样,如果`sampler.param=0`将都不睬采样.
* Probabilistic(`sampler.type=probabilistic`),该采样策略会基于`sampler.param`属性值做出一个概率性的随机决策.例如,`sampler.param=0.1`时大约在10个trace中会有1个被采样.
* Rate Limiting(`sampler.type=ratelimiting`),该采样策略使用一个漏洞限流器来确保trace会按照一个恒定的速率被采样.例如,`sampler.param=2.0`时会按照每秒钟2个trace的速率来采样.
* Remote(`sampler.type=remote`,也是默认策略),会向Jaeger Agent咨询在当前服务中使用的合适的采样策略.这允许从Jaeger服务的中心配置来控制服务的采样策略,甚至是动态的(参见自适应采样).

### 自适应采样
自适应采样策略是一个组合策略,结合了两个功能:
* 它在每个操作的基础上进行采样决策,即基于span的操作名称.这在API服务中是特别有用的,这些服务的端点可能有非常不同的流量,并且对整个服务使用单一的概率采样策略可能会导致饿死(无法采样)一些低QPS的端点.
* 它支持最小保证的采样率,如总是允许每秒最多N次trace,然后以一定的概率对超过这个频率的任何数据进行采样(所有操作都是按操作而不是按服务进行的).

每个操作参数都可以静态配置或定期从Jaeger后端拉取(基于Remote采样策略).自适应采样策略是设计与Jaeger后端即将到来的自适应采样功能一起工作的.

### Collector采样配置
收集器可以使用静态采样策略(如果使用Remote采样配置,则将其传播到相应的服务),通过`--sampling.strategies-file`选项.此选项需要一个json文件的路径,该文件定义了采样策略.

如果没有相应的配置项,收集器会对所有服务返回默认的采样率为0.001(0.1%)的随机采样策略.

举例`strategies.json`:
``` json
{
  "service_strategies": [
    {
      "service": "foo",
      "type": "probabilistic",
      "param": 0.8,
      "operation_strategies": [
        {
          "operation": "op1",
          "type": "probabilistic",
          "param": 0.2
        },
        {
          "operation": "op2",
          "type": "probabilistic",
          "param": 0.4
        }
      ]
    },
    {
      "service": "bar",
      "type": "ratelimiting",
      "param": 5
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.5,
    "operation_strategies": [
      {
        "operation": "/health",
        "type": "probabilistic",
        "param": 0.0
      },
      {
        "operation": "/metrics",
        "type": "probabilistic",
        "param": 0.0
      }
    ]
  }
}
```
`service_strategies`元素定义了相关服务的特殊的采样策略,`operation_strategies`定义了相关操作的特殊的采样策略.这儿用到了两种策略:`probabilistic`和`ratelimiting`,可以参见[客户端采样配置](#客户端采样配置)(注意:`ratelimiting`不支持`operation_strategies`).`default_strategy`定义默认采样策略,适用于所有没有包含在`service_strategies`里的服务.

在上面的例子中:
* 服务`foo`的所有操作会按照0.8的概率被采用,除了操作`op1`和`op2`,`op1`会按照0.2的概率,`op2`会按照0.4的概率.
* 服务`bar`的所有操作会按照每秒5个trace的速率来采样.
* 其它任何服务都会按照`default_strategy`中定义的0.5的概率被采样.
* `default_strategy`还包括每个操作共享的策略.在这个例子中我们通过使用0概率来禁用了所有服务中对包含`/health`和`/metrics`端点的追踪.这些每个操作策略将应用于配置中未列出的任何新服务，以及`foo`和`bar`服务，除非它们为这两个操作定义了自己的策略。

## 部署
主要的Jaeger后端组件已经作为镜像发布在[Docker Hub](https://hub.docker.com)上:
|组件|仓库地址|
|:--|:--|
|jaeger-agent|hub.docker.com/r/jaegertracing/jaeger-agent|
|jaeger-collector|hub.docker.com/r/jaegertracing/jaeger-collector|
|jaeger-query|hub.docker.com/r/jaegertracing/jaeger-query|
|jaeger-ingester|hub.docker.com/r/jaegertracing/jaeger-ingester|

下面是为了运行Jaeger而编排的模板:
* Kubernetes: [jaeger-kubernetes](github.com/jaegertracing/jaeger-kubernetes)
* OpenShift: [jaeger-openshit](github.com/jaegertracing/jaeger-openshit)

### 配置选项
Jaeger二进制文件可以采用下面几种方式来配置(优先级递减):
* 命令行参数.
* 环境变量.
* 配置文件(JSON,TOML,YAML,HCL等格式)或Java属性格式.

要查看选项的完整列表,可以通过运行二进制文件的`help`命令或参考[CLI Flags](https://www.jaegertracing.io/docs/1.21/cli/)页以获得更多信息.只有在选择存储类型时,才会列出特定于某个存储后端的选项.当在使用`Cassandra`存储时查看所有可用的选项,使用命令`docker run --rm -e SPAN_STORAGE_TYPE=cassandra jaegertracing/jaeger-collector:1.21 help`.

为了通过环境变量来提供配置参数,查找相应的命令行选项并将其名称转换为大写格式.如:
|命令行选项|环境变量|
|:--|:--|
|--cassandra.connections-per-host|CASSANDRA_CONNECTIONS_PER_HOST|
|--metrics-backend|METRICS_BACKEND|

### Agent
Jaeger客户端库期望`jaeger-agent`进程在每个主机上本地运行.agent暴露的端口信息:
|端口号|协议|功能|
|:--|:--|:--|
|6831|UDP|接收紧凑格式的[jaeger.thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/jaeger.thrift)协议,大多数Jaeger客户端使用的|
|6832|UDP|接收二进制格式的[jaeger.thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/jaeger.thrift)协议,主要是Node.js客户端使用|
|5778|HTTP|服务配置,采样策略|
|5775|UDP|接收紧凑格式的[zipkin.thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/zipkincore.thrift)协议(已弃用,老客户端使用)|
|14271|HTTP|管理端口:健康检查`/`和指标`/metrics`|

可以在主机上直接运行或采用Docker方式运行:
``` bash
## make sure to expose only the ports you use in your deployment scenario!
docker run \
  --rm \
  -p6831:6831/udp \
  -p6832:6832/udp \
  -p5778:5778/tcp \
  -p5775:5775/udp \
  jaegertracing/jaeger-agent:1.21
```

#### 服务发现集成
Agents可以点对点连接到单个Collector上,也可以依赖其它基础组件(如NDS)在多个Collector之间负载均衡.Agent也可以配置一个静态的Collector服务地址的列表.

采用Docker启动,命令行如下:
``` bash
docker run \
  --rm \
  -p5775:5775/udp \
  -p6831:6831/udp \
  -p6832:6832/udp \
  -p5778:5778/tcp \
  jaegertracing/jaeger-agent:1.21 \
  --reporter.grpc.host-port=jaeger-collector.jaeger-infra.svc:14250
```

当使用gRPC时,在负载均衡和名称解析上有几个选项:
* 单个连接没有负载均衡.这是默认采用的方式`host:port`.(例如:`--reporter.grpc.host-port=jaeger-collector.jaeger-infra.svc:14250`)
* 静态主机名列表和轮训负载均衡.地址之间采用逗号分隔.(例如:`--reporter.grpc.host-port=jaeger-collector1:14250,jaeger-collector2:14250,jaeger-collector3:14250`)
* 动态DNS解析和轮训负载均衡.要获得这种能力,需要在地址前加上前缀`dns:///`,gRPC将尝试使用SRV记录(用于[外部负载均衡](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md))、TXT记录(用于服务配置)和A记录来解析主机名.参考[gRPC名称解析文档](https://github.com/grpc/grpc/blob/master/doc/naming.md)和[dns_resolver](https://github.com/grpc/grpc-go/blob/master/resolver/dns/dns_resolver.go)获取更多信息.(例如:`--reporter.grpc.host-port=dns:///jaeger-collector.jaeger-infra.svc:14250`)

#### 代理层标签
Jaeger支持代理层标签,这些标签可以添加到所有的通过Agent传输的`spans`中.通过命令行参数`--jaeger.tags=key1=value1,key2=value2,...,keyn=valuen`,也可以通过环境变量`--jaeger.tags=key=${envFlag:defaultValue}`,标签值将会被设置为`envFlag`环境变量的值,如没有设置,则标签值默认设置为`defaultValue`.

### 收集器
Collector是无状态的且可以并行的运行多个`jaeger-collector`实例.Collector大部分是不需要配置的,除了使用Cassandra集群时的`--cassandra.keyspace`和`--cassandra.servers`选项,或则使用Elasticsearch集群时的`--es.server-urls`选项,依赖于特定的存储.用如下命令查看所有命令行选项`go run ./cmd/collector/main.go -h`,或则没有源码时使用如下命令`docker run -it --rm jaegertracing/jaeger-collector:1.21 -h`.

Collector默认暴露的端口号:
|端口号|协议|功能|
|:--|:--|:--|
|14250|gRPC|`jaeger-agent`使用,用来发送model.proto格式的span到Collector|
|14268|HTTP|可以直接从客户端接受二进制的thrift协议的数据|
|9411|HTTP|可以接受Zipkin的span数据,支持Thrift,JSON和Proto(默认是禁用)|
|14269|HTTP|管理端口:在`/`的健康检查和在`/metrics`的指标|

### 后端存储
Collector需要一个支持持久化的后端存储,Cassandra和ElasticSearch是主要支持的存储后端,其它后端可参考[这里](https://github.com/jaegertracing/jaeger/issues/638).

存储类型是根据环境变量`SPAN_STORAGE_TYPE`来的,其值可以为`cassandra`,`elasticsearch`,`kafka`(仅用于缓存),`grpc-plugin`,`badger`(仅用于all-in-one)和`memory`(仅用于all-in-one).

在版本1.6.0之后,通过向环境变量`SPAN_STORAGE_TYPE`提供以逗号分隔的多个有效类型列表,可以同时使用多个存储类型.存储列表对应的所有存储都会写入,但只有列表第一个类型对应的存储负责提供读和归档.

对于大规模生产部署,Jaeger团队[推荐Elasticsearch后端超过Cassandra](https://www.jaegertracing.io/docs/1.21/faq/#what-is-the-recommended-storage-backend).

#### Memory
in-memory存储不是为了生产环境准备的,主要是为了快速搭建环境而使用,当程序终止时数据将丢失,不具备持久化功能.

默认情况下,在内存中是不会限制trace的数量的,但也可以通过参数`--memory.max-traces`来设置.

#### Badger - 本地存储
1.9版本之后的实验性功能.

[Badger](https://github.com/dgraph-io/badger)是嵌入式本地存储器,只适用于`all-in-one`发行版.默认情况下,它充当临时存储,使用临时文件系统.这些文件可以通过参数`--badger.ephemeral=false`来重写.
``` bash
docker run \
  -e SPAN_STORAGE_TYPE=badger \
  -e BADGER_EPHEMERAL=false \
  -e BADGER_DIRECTORY_VALUE=/badger/data \
  -e BADGER_DIRECTORY_KEY=/badger/key \
  -v <storage_dir_on_host>:/badger \
  -p 16686:16686 \
  jaegertracing/all-in-one:1.21
```

#### Cassandra
支持版本为3.4+.部署Cassandra可参考文档[Apache Cassandra Docs](https://cassandra.apache.org/doc/latest/).

**配置**

* 最简化命令:`docker run -e SPAN_STORAGE_TYPE=cassandra -e CASSANDRA_SERVERS=<...> jaegertracing/jaeger-collector:1.21`
* 所有选项,通过如下命令可查看所有配置选项:`docker run -e SPAN_STORAGE_TYPE=cassandra jaegertracing/jaeger-collector:1.21 --help`

**Schema脚本**

提供了一个使用Cassandra的交互式shell`cqlsh`来初始化Cassandra的keyspace和schema的脚本:`MODE=test sh ./plugin/storage/cassandra/schema/create.sh | cqlsh`

在生产环境下,传递`MODE=prod DATACENTER={datacenter}`参数给该脚本,其中`{datacenter}`是Cassandra配置/网络拓扑中使用的名称.

该脚本允许覆盖TTL,keyspace名称,复制因子等.运行不带参数的脚本,可查看可识别参数的完整列表.

**TLS支持**

Jaeger支持在客户端与所配置的Cassandra集群之间采用TLS连接.在例如`cqlsh`的验证之后,可使用如下命令配置Collector:
``` bash
docker run \
  -e CASSANDRA_SERVERS=<...> \
  -e CASSANDRA_TLS=true \
  -e CASSANDRA_TLS_SERVER_NAME="CN-in-certificate" \
  -e CASSANDRA_TLS_KEY=<path to client key file> \
  -e CASSANDRA_TLS_CERT=<path to client cert file> \
  -e CASSANDRA_TLS_CA=<path to your CA cert file> \
  jaegertracing/jaeger-collector:1.21
```

schema工具也支持TLS,可以使用如下自定义的cqlshrc文件:
``` bash
# Creating schema in a cassandra cluster requiring client TLS certificates.
#
# Create a volume for the schema docker container containing four files:
# cqlshrc: this file
# ca-cert: the cert authority for your keys
# client-key: the keyfile for your client
# client-cert: the cert file matching client-key
#
# if there is any sort of DNS mismatch and you want to ignore server validation
# issues, then uncomment validate = false below.
#
# When running the container, map this volume to /root/.cassandra and set the
# environment variable CQLSH_SSL=--ssl
[ssl]
certfile = ~/.cassandra/ca-cert
userkey = ~/.cassandra/client-key
usercert = ~/.cassandra/client-cert
# validate = false
```

#### Elasticsearch
从Jaeger0.6版本之后开始支持,支持5.x,6.x,7.x版本的Elasticsearch.

Elasticsearch版本会自动从根/ping端点检索.基于这些版本Jaeger使用兼容的索引映射和Elasticsearch REST API.版本可以通过参数`--es.version=`来指定.

除了安装和运行Elasticsearch外,Elasticsearch不需要初始化.一旦运行之后,会将正确的配置值传递给Jaeger收集器和查询服务.

**配置**

* 最简化命令:`docker run -e SPAN_STORAGE_TYPE=elasticsearch -e ES_SERVER_URLS=<...> jaegertracing/jaeger-collector:1.21`
* 所有选项,通过如下命令可查看所有配置选项:`docker run -e SPAN_STORAGE_TYPE=elasticsearch jaegertracing/jaeger-collector:1.21 --help`

**Elasticsearch索引的分片和副本**

分片和副本有些配置值是需要特别关注的,因为这些会决定索引的创建.[这篇文章](https://qbox.io/blog/optimizing-elasticsearch-how-many-shards-per-index)将详细介绍如何选择多少个分片在优化时.

#### Kafka
从Jaeger1.6.0版本之后开始支持,支持0.9+版本的Kafka.

Kafka主要是作为收集器和真实的存储之间的临时缓存.收集器被配置为`SPAN_STORAGE_TYPE=kafka`,使得收集器上接收到的所有spans会被写入到一个Kafka的topic中.在1.7.0版本新增的一个组件[Ingester](https://www.jaegertracing.io/docs/1.21/deployment/#ingester),用来从Kafka中读取然后存储spans到另一个存储后端(如Elasticsearch或Cassandra).

写入Kafka对于构建后处理数据管道特别有用.

**配置**

* 最简化命令:`docker run -e SPAN_STORAGE_TYPE=kafka -e KAFKA_PRODUCER_BROKERS=<...> -e KAFKA_TOPIC=<...> jaegertracing/jaeger-collector:1.21`
* 所有选项,通过如下命令可查看所有配置选项:`docker run -e SPAN_STORAGE_TYPE=kafka jaegertracing/jaeger-collector:1.21 --help`

**Topic和分区**

除非Kafka集群被配置为自动创建topic,否则应该提前创建好topic.可以参考[Kafka快速入门文档](https://kafka.apache.org/documentation/#quickstart_createtopic)来了解如何做到这一点.

在[官方文档](https://kafka.apache.org/documentation/#intro_topics)可了解到更多关于topic和分区的信息.[这篇文章](https://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)关于如何选择分区的数量有更多的细节.

#### 存储插件
Jaeger支持基于gRPC协议的存储插件.更多细节参考[jaeger/plugin/storage/grpc](https://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)

当前可用插件:
* [InfluxDB](https://github.com/influxdata/jaeger-influxdb/)
* [Logz.io](https://github.com/logzio/jaeger-logzio),安全,可扩展,可管理,基于云的ELK存储.

使用如下命令:
``` bash
docker run \
  -e SPAN_STORAGE_TYPE=grpc-plugin \
  -e GRPC_STORAGE_PLUGIN_BINARY=<...> \
  -e GRPC_STORAGE_PLUGIN_CONFIGURATION_FILE=<...> \
  jaegertracing/all-in-one:1.21
```

### Ingester
`jaeger-ingester`是一个服务用来从Kafka的topic中读取spans数据,然后写入到另一个后端存储中(Elasticsearch或Cassandra).
|端口号|协议|功能|
|:--|:--|:--|
|14270|HTTP|管理端口:在`/`端点的健康检查和在`/metrics`的指标|

使用如下命令查看所有配置选项:`docker run -e SPAN_STORAGE_TYPE=cassandra jaegertracing/jaeger-ingester:1.21 --help`

### Query Service & UI
`jaeger-query`服务提供API和一个React/Javascript的UI.服务是无状态的,经常是运行在负载均衡器之后,如[NGINX](https://www.nginx.com/).

有如下端口会暴露:
|端口号|协议|功能|
|:--|:--|:--|
|16686|HTTP|`/api/*`端点,UI在`/`|
|16686|gRPC|Protobuf/gRPC [查询服务](https://github.com/jaegertracing/jaeger-idl/blob/master/proto/api_v2/query.proto)|
|16687|HTTP|管理端口:在`/`端点的健康检查和在`/metrics`的指标|

**最小依赖例子(Elasticsearch为后端存储):**
``` bash
docker run -d --rm \
  -p 16686:16686 \
  -p 16687:16687 \
  -e SPAN_STORAGE_TYPE=elasticsearch \
  -e ES_SERVER_URLS=http://<ES_SERVER_IP>:<ES_SERVER_PORT> \
  jaegertracing/jaeger-query:1.21
```

**时钟偏移调整**

Jaeger后端会聚合来自不同主机的应用程序的追踪数据.主机上的硬件时钟经常会经历相对漂移,称为[时钟偏移效应](https://en.wikipedia.org/wiki/Clock_skew).时钟偏移会使得追踪数据的推断变得困难,假如,当一个服务的span有可能比客户端的span更早产生,这种情况是不可能的.查询服务实现了一个时钟偏移调整的[算法](https://github.com/jaegertracing/jaeger/blob/master/model/adjuster/clockskew.go),使用span之间的因果关系知识来校正时钟偏移.所有被调整的span在UI上会显示一个告警信息,提供应用其时间戳的精确时钟偏移增量.

有时这些调整本身使得追踪难以理解.例如,当在父span的范围内重新定位服务span时,Jaeger不知道请求和响应延迟之间的确切关系,因此假设两者相等,并将子span放在父span的中间(参见[issue #961](https://github.com/jaegertracing/jaeger/issues/961#issuecomment-453925244)).

查询服务支持一个配置选项`--query.max-clock-skew-adjustment`,用来控制多少时钟偏移调整是被允许的.如果该参数被设置为zero(0s)则会完全禁用时钟偏移调整.此设置适用于从给定查询服务检索的所有追踪.有一个开放[标签#197](https://github.com/jaegertracing/jaeger-ui/issues/197)来支持在UI中直接切换调整的开启和关闭.

**UI基础Path**

针对所有`jaeger-query`的HTTP路由的基础路径可以设置为一个非root的值,如`/jaeger`,会导致所有UI URLs会已`/jaeger`开头.当在反向代理后运行的`jaeger-query`是非常有用的.

可以通过命令行参数`--query.base-path`或环境变量`QUERY_BASE_PATH`来配置基础路径.

