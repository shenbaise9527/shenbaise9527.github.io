# gRPC系列之初识


## RPC
RPC指远程过程调用(Remote Procedure Call),让远程服务调用更加简单、透明.服务调用者可以像调用本地接口一样调用远程的服务提供者,而不需要关心底层通信细节和调用过程,RPC框架负责底层的传输方式、序列化方式和通信细节.

gRPC是一个高性能、开源和通用的RPC框架,面向服务端和移动端,特点如下:
* 支持多语言.
* 基于IDL文件定义服务,通过protoc工具生成指定语言的数据结构、服务端接口和客户端Stub.
* 通信协议基于HTTP/2设计,支持双向流、消息头压缩、单TCP的多路复用、服务端推送等特性.使得在移动端设备上更加省电和节省网络流量.
* 序列化支持[Protocol Buffer](https://github.com/protocolbuffers/protobuf)和JSON.

![gRPC调用示例](/images/grpc.png "gRPC调用")

## protoc工具
先安装相关工具,可以用来直接生成go和grpc的代码.
1. 安装[protoc](https://github.com/protocolbuffers/protobuf)
2. 安装[protoc-gen-go](https://github.com/protocolbuffers/protobuf-go)
3. 安装[protoc-gen-go-grpc](https://github.com/grpc/grpc-go)

## IDL文件
生成`hello.proto`文件,内容如下:
``` go
syntax = "proto3";

option go_package=".;pb";

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}

service HelloService {
    rpc SayHello (HelloRequest) returns (HelloReply);
}
```
* syntax,定义proto的版本,支持proto2和proto3,proto3才支持grpc.
* go_package,定义生成的go文件的包名(package name).
* message,定义数据结构.
* service,定义服务,可包含多个rpc函数.

## 生成go语言代码
使用工具`protoc`来生成对应的go文件,命令`protoc -I=./proto --go_out=plugins=grpc:./pb hello.proto`.
* `-I=./proto`,表示proto文件所在的目录.
* `--go_out`,表示生成go语言的代码,且存放go文件的目录,默认是不会生成grpc的代码的,需要显式声明`plugins=grpc`.
* `hello.proto`,表示proto文件的名字.

在目录`pb`中生成文件`hello.pb.go`,文件里面包含了grpc相关代码.
``` go
// Reference imports to suppress errors if they are not otherwise used.
var _ context.Context
var _ grpc.ClientConnInterface

// This is a compile-time assertion to ensure that this generated file
// is compatible with the grpc package it is being compiled against.
const _ = grpc.SupportPackageIsVersion6

// HelloServiceClient is the client API for HelloService service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.
type HelloServiceClient interface {
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}

type helloServiceClient struct {
	cc grpc.ClientConnInterface
}

func NewHelloServiceClient(cc grpc.ClientConnInterface) HelloServiceClient {
	return &helloServiceClient{cc}
}

func (c *helloServiceClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, "/HelloService/SayHello", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

// HelloServiceServer is the server API for HelloService service.
type HelloServiceServer interface {
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
}

// UnimplementedHelloServiceServer can be embedded to have forward compatible implementations.
type UnimplementedHelloServiceServer struct {
}

func (*UnimplementedHelloServiceServer) SayHello(context.Context, *HelloRequest) (*HelloReply, error) {
	return nil, status.Errorf(codes.Unimplemented, "method SayHello not implemented")
}

func RegisterHelloServiceServer(s *grpc.Server, srv HelloServiceServer) {
	s.RegisterService(&_HelloService_serviceDesc, srv)
}

func _HelloService_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(HelloRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(HelloServiceServer).SayHello(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/HelloService/SayHello",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(HelloServiceServer).SayHello(ctx, req.(*HelloRequest))
	}
	return interceptor(ctx, in, info, handler)
}

var _HelloService_serviceDesc = grpc.ServiceDesc{
	ServiceName: "HelloService",
	HandlerType: (*HelloServiceServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "SayHello",
			Handler:    _HelloService_SayHello_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "hello.proto",
}
```

## 服务端代码
``` go
// 监听tcp端口,用于接受客户端请求.
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
if err != nil {
	fmt.Printf("failed to listen: %v", err)

	return
}

// 创建gRPC服务实例对象.
grpcserver := grpc.NewServer()

// 把server对象注册到gRPC服务中,server对象实现了HelloServiceServer接口.
pb.RegisterHelloServiceServer(grpcserver, &server{})

// 阻塞等待客户端连接,直到进程被终止或Stop函数被调用.
err = grpcserver.Serve(lis)
if err != nil {
	fmt.Printf("failed to server: %v", err)
}
```

## 客户端代码
``` go
// 创建一个连接与服务端进行通信.
conn, err := grpc.Dial(*addr, grpc.WithInsecure())
if err != nil {
	fmt.Printf("failed to dail: %v", err)

	return
}

// 关闭连接.
defer conn.Close()

// 创建HelloService的Client stub.
client := pb.NewHelloServiceClient(conn)

// 调用对应的服务方法.
reply, err := client.SayHello(context.Background(), &pb.HelloRequest{Name: "zhou"})
if err != nil {
	fmt.Printf("failed to sayhello: %v", err)

	return
}

fmt.Println(reply)
```

## 调用分析
### wireshark抓包 
服务端监听在9090端口,用[wireshark](https://www.wireshark.org/)来进行抓包.
![抓包](/images/pcap1.png "wireshark抓包")
可以看到前三行是tcp三次握手的报文,后续的全部都解析成了tcp协议,gRPC是基于http/2,需要手工修改Protocol为http/2.
wireshark菜单栏-->分析(A)-->解码为(Decode As),在弹出的界面新增一行,然后修改"当前"列为HTTP2.
![设置http/2](/images/pcap2.png "设置协议为http2")

现在能正常解析为HTTP/2协议了,一次gRPC调用总览如下:
![HTTP/2协议](/images/pcap3.png "gRPC调用总览")
从上图大体可以看出,gRPC调用过程分为:`Magic(C->S) --> SETTINGS(S->C) --> SETTINGS(C->S) --> SETTINGS(S->C) --> SETTINGS,HEADERS,DATA(C->S) --> WINDOW_UPDATE,PING(S->C) --> HEADERS,DATA,HEADERS(S->C) --> PING,WINDOW_UPDATE,PING(C->S) --> PING(S->C)`

### Magic
![Magic](/images/grpc-magic.png "gRPC-Magic")
Magic帧的主要作用是建立HTTP/2请求的前言.在HTTP/2协议中,要求两端都要发送连接前言,来最终确认所使用的协议,并确定HTTP/2连接的初始设置.

而Magic帧是客户端的前言之一,内容为`PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`,以确定启用HTTP/2连接.

### SETTINGS
![SETTINGS1](/images/grpc-settings1.png "gRPC-SETTINGS")
由服务端发送给客户端,主要设置`Max Frame Size`为16384字节,作用域是整个连接而非单一的流.也是服务端的连接前言.

![SETTINGS2](/images/grpc-settings2.png "gRPC-SETTINGS")
由客户端发送给服务端,是客户端的连接前言(和Magic一起).

![SETTINGS3](/images/grpc-settings3.png "gRPC-SETTINGS")
由服务端发送给客户端,在发送完前言后,客户端和服务端还需要有一步互相确认的动作,对应的就是带有`ACK: True`的帧.

![SETTINGS4](/images/grpc-settings4.png "gRPC-SETTINGS")
由客户端发送给服务端,是带有`ACK: True`的帧.

### HEADERS
![HEADERS1](/images/grpc-headers1.png "gRPC-HEADERS")
主要是存储和传播HTTP的表头信息.

### DATA
![DATA1](/images/grpc-data1.png "gRPC-DATA")
DATA是数据帧,可以看到请求的protobuf结构只有1个字段,该字段的值为`zhou`(可以参见客户端代码).

### WINDOW_UPDATE
![WIN](/images/grpc-win.png "gRPC-WINDOW_UPDATE")
主要是管理流控制窗口的大小.

### HEADERS,DATA,HEADERS
![RSP1](/images/grpc-rsp1.png "gRPC-DATA")
![RSP2](/images/grpc-rsp2.png "gRPC-DATA")
服务端发送给客户端的响应,HEADERS frame记录的是HTTP响应状态(`200 OK`)和响应的内容格式(`application/grpc`).
响应的protobuf结构也是只有1个字段,字段的值是`zhou`.

### PING
主要是判断当前连接是否仍然可用,也常用于计算往返时间.

## 流式模式
gRPC支持UnaryRPC(一元PRC)和StreamRPC(流式RPC).
* UnaryRPC,上面介绍的都是基于UnaryRPC的,该模式是一个请求对应一个响应.
* StreamRPC流式模式的请求和响应是多对多的,又分为三种类型:
  - Server-side streaming RPC,服务端流式模式,即一个请求对应多个响应.
  - Client-side streaming RPC,客户端流式模式,即多个请求对应一个响应.
  - Bidirectional streaming RPC,双向流式模式,即多个请求对应多个响应.

流式模式需要用到关键字`stream`,如下proto文件.
``` go
syntax = "proto3";

option go_package=".;pb";

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}

service HelloService {
    rpc SayHello (HelloRequest) returns (HelloReply);
    rpc ServerSayHello (HelloRequest) returns (stream HelloReply);
    rpc ClientSayHello (stream HelloRequest) returns (HelloReply);
    rpc BidirSayHello (stream HelloRequest) returns (stream HelloReply);
}
```
`SayHello`是一元RPC模式.
`ServerSayHello`是服务端流式模式.
`ClientSayHello`是客户端流式模式.
`BidirSayHello`是双向流式模式.

## 参考资料
* [抓包gRPC的细节与分析](https://jingwei.link/2018/10/02/grpc-wireshark-analysis.html)
* [Hypertext Transfer Protocol Version 2](https://httpwg.org/specs/rfc7540.html)
* [从实践到原理，带你参透 gRPC](https://eddycjy.com/posts/go/talk/2019-06-29-talking-grpc/)

