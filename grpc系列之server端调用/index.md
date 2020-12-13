# gRPC系列之server端调用


## NewServer创建gRPC服务对象
主要是基于grpc-go的1.33.1版本Unary RPC来分析.
``` go
// NewServer creates a gRPC server which has no service registered and has not
// started to accept requests yet.
func NewServer(opt ...ServerOption) *Server {
    // 处理Server对象的一些定制化参数.在Go中推荐采用Option的方式从外部来影响对象内的行为.
	opts := defaultServerOptions
	for _, o := range opt {
		o.apply(&opts)
    }
    
    // 生成Server对象.
	s := &Server{
        lis:      make(map[net.Listener]bool),
        opts:     opts,
		conns:    make(map[transport.ServerTransport]bool),
		services: make(map[string]*serviceInfo),
		quit:     grpcsync.NewEvent(),
		done:     grpcsync.NewEvent(),
		czData:   new(channelzData),
    }
    
    // 处理一元服务端拦截器,支持链式多拦截器.
    chainUnaryServerInterceptors(s)
    
    // 处理流式服务端拦截器,支持链式多拦截器.
	chainStreamServerInterceptors(s)
	s.cv = sync.NewCond(&s.mu)
	if EnableTracing {
		_, file, line, _ := runtime.Caller(1)
		s.events = trace.NewEventLog("grpc.Server", fmt.Sprintf("%s:%d", file, line))
	}

    // 用来控制处理连接的goroutine的数量,为0时表示不控制.
	if s.opts.numServerWorkers > 0 {
		s.initServerWorkers()
	}

	if channelz.IsOn() {
		s.channelzID = channelz.RegisterServer(&channelzServer{s}, "")
	}
	return s
}
```

## ServerOption自定义参数
**Creds**
主要用来设置服务端认证相关的参数.
``` go
// 用来设置TLS.
// Creds returns a ServerOption that sets credentials for server connections.
func Creds(c credentials.TransportCredentials) ServerOption {
	return newFuncServerOption(func(o *serverOptions) {
		o.creds = c
	})
}

// gRPC中的credentials包,已定义相关的TransportCredentials.
// NewServerTLSFromFile constructs TLS credentials from the input certificate file and key
// file for server.
func NewServerTLSFromFile(certFile, keyFile string) (TransportCredentials, error) {
	cert, err := tls.LoadX509KeyPair(certFile, keyFile)
	if err != nil {
		return nil, err
	}
	return NewTLS(&tls.Config{Certificates: []tls.Certificate{cert}}), nil
}
```

**UnaryInterceptor**
主要用来设置服务端的拦截器.
``` go
// UnaryInterceptor returns a ServerOption that sets the UnaryServerInterceptor for the
// server. Only one unary interceptor can be installed. The construction of multiple
// interceptors (e.g., chaining) can be implemented at the caller.
func UnaryInterceptor(i UnaryServerInterceptor) ServerOption {
	return newFuncServerOption(func(o *serverOptions) {
		if o.unaryInt != nil {
			panic("The unary server interceptor was already set and may not be reset.")
		}
		o.unaryInt = i
	})
}

// 具体的自定义拦截器需要实现UnaryServerInterceptor函数原型.
// UnaryServerInterceptor provides a hook to intercept the execution of a unary RPC on the server. info
// contains all the information of this RPC the interceptor can operate on. And handler is the wrapper
// of the service method implementation. It is the responsibility of the interceptor to invoke handler
// to complete the RPC.
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

**ChainUnaryInterceptor**
主要用来设置服务端的链式拦截器.
``` go
// 支持同时设置多个拦截器.
// ChainUnaryInterceptor returns a ServerOption that specifies the chained interceptor
// for unary RPCs. The first interceptor will be the outer most,
// while the last interceptor will be the inner most wrapper around the real call.
// All unary interceptors added by this method will be chained.
func ChainUnaryInterceptor(interceptors ...UnaryServerInterceptor) ServerOption {
	return newFuncServerOption(func(o *serverOptions) {
		o.chainUnaryInts = append(o.chainUnaryInts, interceptors...)
	})
}
```

## 注册RPC对象到Server中
``` go
// 注册HelloServiceServer对象到gRPC对象中.
func RegisterHelloServiceServer(s *grpc.Server, srv HelloServiceServer) {
	s.RegisterService(&_HelloService_serviceDesc, srv)
}

// _HelloService_serviceDesc主要用来描述RPC对象的信息.
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

// 注册RPC服务对象.
// RegisterService registers a service and its implementation to the gRPC
// server. It is called from the IDL generated code. This must be called before
// invoking Serve. If ss is non-nil (for legacy code), its type is checked to
// ensure it implements sd.HandlerType.
func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
    // 主要用来判断接口类型是否一致.
	if ss != nil {
		ht := reflect.TypeOf(sd.HandlerType).Elem()
		st := reflect.TypeOf(ss)
		if !st.Implements(ht) {
			logger.Fatalf("grpc: Server.RegisterService found the handler of type %v that does not satisfy %v", st, ht)
		}
	}
	s.register(sd, ss)
}

func (s *Server) register(sd *ServiceDesc, ss interface{}) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.printf("RegisterService(%q)", sd.ServiceName)
	if s.serve {
		logger.Fatalf("grpc: Server.RegisterService after Server.Serve for %q", sd.ServiceName)
    }
    
    // 判断服务名是否已经注册过了.
	if _, ok := s.services[sd.ServiceName]; ok {
		logger.Fatalf("grpc: Server.RegisterService found duplicate service registration for %q", sd.ServiceName)
    }
    
    // 创建serviceInfo对象.
	info := &serviceInfo{
        // 接口的实现对象.
        serviceImpl: ss,
        
        // 具体的方法描述信息.
		methods:     make(map[string]*MethodDesc),
        streams:     make(map[string]*StreamDesc),
        
        // 元数据.
		mdata:       sd.Metadata,
    }
    
    // 保存具体的方法.
	for i := range sd.Methods {
		d := &sd.Methods[i]
		info.methods[d.MethodName] = d
	}
	for i := range sd.Streams {
		d := &sd.Streams[i]
		info.streams[d.StreamName] = d
	}
	s.services[sd.ServiceName] = info
}
```

## Server启动监听等待连接
``` go
// Serve accepts incoming connections on the listener lis, creating a new
// ServerTransport and service goroutine for each. The service goroutines
// read gRPC requests and then call the registered handlers to reply to them.
// Serve returns when lis.Accept fails with fatal errors.  lis will be closed when
// this method returns.
// Serve will return a non-nil error unless Stop or GracefulStop is called.
func (s *Server) Serve(lis net.Listener) error {
    s.mu.Lock()

    ......

	ls := &listenSocket{Listener: lis}
	s.lis[ls] = true

    ......

	s.mu.Unlock()

	defer func() {
		s.mu.Lock()
		if s.lis != nil && s.lis[ls] {
			ls.Close()
			delete(s.lis, ls)
		}
		s.mu.Unlock()
	}()

	var tempDelay time.Duration // how long to sleep on accept failure

	for {
        // 等待连接.
		rawConn, err := lis.Accept()
		if err != nil {
            // 当返回错误时,会尝试重新调用Accept,时间间隔从5毫秒开始,每重试一次时间翻倍,直到1秒.
			if ne, ok := err.(interface {
				Temporary() bool
			}); ok && ne.Temporary() {
                // 计算时间间隔的逻辑.
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				s.mu.Lock()
				s.printf("Accept error: %v; retrying in %v", err, tempDelay)
				s.mu.Unlock()
				timer := time.NewTimer(tempDelay)
				select {
				case <-timer.C:
				case <-s.quit.Done():
					timer.Stop()
					return nil
				}
				continue
			}
			s.mu.Lock()
			s.printf("done serving; Accept = %v", err)
			s.mu.Unlock()

			if s.quit.HasFired() {
				return nil
			}
			return err
        }
        
        // Accept正常后,重置时间为0.
		tempDelay = 0
		// Start a new goroutine to deal with rawConn so we don't stall this Accept
		// loop goroutine.
		//
		// Make sure we account for the goroutine so GracefulStop doesn't nil out
        // s.conns before this conn can be added.
        // 主要是为了优雅的关闭,在关闭前所有的连接必须被处理完了.
		s.serveWG.Add(1)
		go func() {
            // 启动一个新的goroutine来处理新的连接.
			s.handleRawConn(rawConn)
			s.serveWG.Done()
		}()
	}
}
```

## 业务处理逻辑
``` go
// handleRawConn forks a goroutine to handle a just-accepted connection that
// has not had any I/O performed on it yet.
func (s *Server) handleRawConn(rawConn net.Conn) {
	if s.quit.HasFired() {
		rawConn.Close()
		return
    }
    
    // 设置超时时间,默认是120秒.
    rawConn.SetDeadline(time.Now().Add(s.opts.connectionTimeout))
    
    // 处理TLS认证.
	conn, authInfo, err := s.useTransportAuthenticator(rawConn)
	if err != nil {
		// ErrConnDispatched means that the connection was dispatched away from
		// gRPC; those connections should be left open.
		if err != credentials.ErrConnDispatched {
			s.mu.Lock()
			s.errorf("ServerHandshake(%q) failed: %v", rawConn.RemoteAddr(), err)
			s.mu.Unlock()
			channelz.Warningf(logger, s.channelzID, "grpc: Server.Serve failed to complete security handshake from %q: %v", rawConn.RemoteAddr(), err)
			rawConn.Close()
		}
		rawConn.SetDeadline(time.Time{})
		return
	}

    // 开启HTTP/2协议.
	// Finish handshaking (HTTP2)
	st := s.newHTTP2Transport(conn, authInfo)
	if st == nil {
		return
	}

	rawConn.SetDeadline(time.Time{})
	if !s.addConn(st) {
		return
	}
	go func() {
        // 开启新的goroutine来处理业务数据.
		s.serveStreams(st)
		s.removeConn(st)
	}()
}

func (s *Server) serveStreams(st transport.ServerTransport) {
	defer st.Close()
	var wg sync.WaitGroup

    var roundRobinCounter uint32
    // HandleStreams主要是接收数据,并生成Stream对象,在调用下面的匿名函数来处理具体的业务逻辑.
	st.HandleStreams(func(stream *transport.Stream) {
        wg.Add(1)
        // 判断是否有设置numServerWorkers.
		if s.opts.numServerWorkers > 0 {
			data := &serverWorkerData{st: st, wg: &wg, stream: stream}
			select {
            // 发送数据到指定的channel中.
			case s.serverWorkerChannels[atomic.AddUint32(&roundRobinCounter, 1)%s.opts.numServerWorkers] <- data:
			default:
				// If all stream workers are busy, fallback to the default code path.
				go func() {
                    // 若所有workerchannel都在忙,则单独创建goroutine.
					s.handleStream(st, stream, s.traceInfo(st, stream))
					wg.Done()
				}()
			}
		} else {
			go func() {
                // 没有限制worker大小,则单独创建goroutine.
				defer wg.Done()
				s.handleStream(st, stream, s.traceInfo(st, stream))
			}()
		}
	}, func(ctx context.Context, method string) context.Context {
		if !EnableTracing {
			return ctx
		}
		tr := trace.New("grpc.Recv."+methodFamily(method), method)
		return trace.NewContext(ctx, tr)
	})
	wg.Wait()
}

// HandleStreams receives incoming streams using the given handler. This is
// typically run in a separate goroutine.
// traceCtx attaches trace to ctx and returns the new context.
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
	defer close(t.readerDone)
	for {
        t.controlBuf.throttle()
        // 读取HTTP/2协议中的frame数据.
		frame, err := t.framer.fr.ReadFrame()
		atomic.StoreInt64(&t.lastRead, time.Now().UnixNano())
		if err != nil {
            // 错误相关.
            ......
			t.Close()
			return
        }
        // 针对不同类型的frame的处理,可以和之前的抓包对应起来.
		switch frame := frame.(type) {
		case *http2.MetaHeadersFrame:
			if t.operateHeaders(frame, handle, traceCtx) {
				t.Close()
				break
			}
		case *http2.DataFrame:
			t.handleData(frame)
		case *http2.RSTStreamFrame:
			t.handleRSTStream(frame)
		case *http2.SettingsFrame:
			t.handleSettings(frame)
		case *http2.PingFrame:
			t.handlePing(frame)
		case *http2.WindowUpdateFrame:
			t.handleWindowUpdate(frame)
		case *http2.GoAwayFrame:
			// TODO: Handle GoAway from the client appropriately.
		default:
			if logger.V(logLevel) {
				logger.Errorf("transport: http2Server.HandleStreams found unhandled frame type %v.", frame)
			}
		}
	}
}

func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
    // 解析方法名,如/HelloService/SayHello.
	sm := stream.Method()
	if sm != "" && sm[0] == '/' {
		sm = sm[1:]
	}
	pos := strings.LastIndex(sm, "/")
	if pos == -1 {
        // 错误处理.
        ......
		return
    }
    // service服务名等于HelloService.
    service := sm[:pos]
    // 方法名等于SyaHello.
	method := sm[pos+1:]

    // 从已注册的service中查找.
	srv, knownService := s.services[service]
	if knownService {
		if md, ok := srv.methods[method]; ok {
            // 若方法存在,在调用processUnaryRPC.此时md已经指向注册时的Handler了,如SyaHello方法对应的_HelloService_SayHello_Handler.
			s.processUnaryRPC(t, stream, srv, md, trInfo)
			return
		}
		if sd, ok := srv.streams[method]; ok {
            // 调用流式处理.
			s.processStreamingRPC(t, stream, srv, sd, trInfo)
			return
		}
    }
    // 若不存在,调用unknown.
	// Unknown service, or known server unknown method.
	if unknownDesc := s.opts.unknownStreamDesc; unknownDesc != nil {
		s.processStreamingRPC(t, stream, nil, unknownDesc, trInfo)
		return
    }
    
    ......
}

// processUnaryRPC实质上就是先从stream中读取一个完整的message,然后再调用md的Handler,来执行具体的业务代码,最后再sendResponse.

// 具体的业务逻辑代码,*.pb.go文件中.
func _HelloService_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
    // 解析请求参数.
	in := new(HelloRequest)
	if err := dec(in); err != nil {
		return nil, err
    }
    
    // 如果没有设置拦截器,就直接调用SayHello.
	if interceptor == nil {
		return srv.(HelloServiceServer).SayHello(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/HelloService/SayHello",
    }
    
    // UnaryHandler,具体的业务逻辑.
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(HelloServiceServer).SayHello(ctx, req.(*HelloRequest))
    }
    
    // 调用拦截器,最后再执行handler,来处理业务逻辑.
	return interceptor(ctx, in, info, handler)
}
```

## 总结
* 通过`ServerOption`来设置自定义参数,最主要的包括`grpc.Creds`(用于设置服务端认证)和`grpc.UnaryInterceptor`(用于设置服务端拦截器).
* 在*.pb.go中主要是通过`grpc.ServiceDesc`来描述rpc接口的信息,最终调用会指向其`Handler`字段.
* 在整个处理过程中会涉及到的goroutine.
  - 当`Accept`接收到一个新连接时就会启用一个goroutine,主要用来处理认证及HTTP/2相关的初始化.
  - 接着会启用一个goroutine用来接收HTTP/2协议的数据.
  - 每接收到一个完整请求包时,会再启用一个goroutine用来处理新的消息包.注意:此处如果设置了`numServerWorkers`,会优先使用workchannel.

