# gRPC系列之client和server的timetou机制


## 客户端超时的处理
客户端在调用rpc接口时带timeout的context是如何传递给服务端的.
``` go
// 在调用对应的rpc方法时设置了超时时间为3秒.
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
reply, err := client.SayHello(ctx, &pb.HelloRequest{Name: "zhou"})

// 调用SayHello方法最终会调用到invoke函数(call.go文件中).
func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
	cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
	if err != nil {
		return err
	}
	if err := cs.SendMsg(req); err != nil {
		return err
	}
	return cs.RecvMsg(reply)
}

// 在newClientStream方法中会创建clientStream对象,ctx会赋值给该对象的ctx字段.
func newClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, opts ...CallOption) (_ ClientStream, err error) {
    ......
    // 前面代码都不关注,这里生成clientStream对象,关注其ctx字段.
	cs := &clientStream{
		callHdr:      callHdr,
		ctx:          ctx,
		methodConfig: &mc,
		opts:         opts,
		callInfo:     c,
		cc:           cc,
		desc:         desc,
		codec:        c.codec,
		cp:           cp,
		comp:         comp,
		cancel:       cancel,
		beginTime:    beginTime,
		firstAttempt: true,
	}
	if !cc.dopts.disableRetry {
		cs.retryThrottler = cc.retryThrottler.Load().(*retryThrottler)
	}
	cs.binlog = binarylog.GetMethodLogger(method)

    // 在newAttemptLocked中会通过负载均衡算法来选择Ready状态的连接.
	// Only this initial attempt has stats/tracing.
	// TODO(dfawley): move to newAttempt when per-attempt stats are implemented.
	if err := cs.newAttemptLocked(sh, trInfo); err != nil {
		cs.finish(err)
		return nil, err
	}

    // 在withRetry中最终会调用op函数,该函数会调用到csAttempt.newStream方法.
	op := func(a *csAttempt) error { return a.newStream() }
	if err := cs.withRetry(op, func() { cs.bufferForRetryLocked(0, op) }); err != nil {
		cs.finish(err)
		return nil, err
	}

    ......
    // 下面的也不用关注了.
	return cs, nil
}

func (a *csAttempt) newStream() error {
	cs := a.cs
    cs.callHdr.PreviousAttempts = cs.numRetries
    // 这里会调用NewStream方法,t指向的是http2Client对象,cs是指向clientStream对象的.
	s, err := a.t.NewStream(cs.ctx, cs.callHdr)
	if err != nil {
		if _, ok := err.(transport.PerformedIOError); ok {
			// Return without converting to an RPC error so retry code can
			// inspect.
			return err
		}
		return toRPCErr(err)
	}
	cs.attempt.s = s
	cs.attempt.p = &parser{r: s}
	return nil
}

// NewStream creates a stream and registers it into the transport as "active"
// streams.
func (t *http2Client) NewStream(ctx context.Context, callHdr *CallHdr) (_ *Stream, err error) {
    ctx = peer.NewContext(ctx, t.getPeer())
    // 重点关注createHeaderFields,这个方法会处理HEADERS的数据,来传播表头数据.
    headerFields, err := t.createHeaderFields(ctx, callHdr)
    
    ......
    // 下面的代码不关注.
}

func (t *http2Client) createHeaderFields(ctx context.Context, callHdr *CallHdr) ([]hpack.HeaderField, error) {
    // 主要处理HEADERS头部数据.
    ......

    // 支持的字段.
	headerFields := make([]hpack.HeaderField, 0, hfLen)
	headerFields = append(headerFields, hpack.HeaderField{Name: ":method", Value: "POST"})
	headerFields = append(headerFields, hpack.HeaderField{Name: ":scheme", Value: t.scheme})
	headerFields = append(headerFields, hpack.HeaderField{Name: ":path", Value: callHdr.Method})
	headerFields = append(headerFields, hpack.HeaderField{Name: ":authority", Value: callHdr.Host})
	headerFields = append(headerFields, hpack.HeaderField{Name: "content-type", Value: grpcutil.ContentType(callHdr.ContentSubtype)})
	headerFields = append(headerFields, hpack.HeaderField{Name: "user-agent", Value: t.userAgent})
	headerFields = append(headerFields, hpack.HeaderField{Name: "te", Value: "trailers"})
	if callHdr.PreviousAttempts > 0 {
		headerFields = append(headerFields, hpack.HeaderField{Name: "grpc-previous-rpc-attempts", Value: strconv.Itoa(callHdr.PreviousAttempts)})
	}

	if callHdr.SendCompress != "" {
		headerFields = append(headerFields, hpack.HeaderField{Name: "grpc-encoding", Value: callHdr.SendCompress})
		headerFields = append(headerFields, hpack.HeaderField{Name: "grpc-accept-encoding", Value: callHdr.SendCompress})
    }
    
    // 重点在这里,获取ctx的deadline时间,然后放入头部中的grpc-timeout字段中,客户端就是利用这个来吧超时时间传递到服务端的.
	if dl, ok := ctx.Deadline(); ok {
		// Send out timeout regardless its value. The server can detect timeout context by itself.
		// TODO(mmukhi): Perhaps this field should be updated when actually writing out to the wire.
		timeout := time.Until(dl)
		headerFields = append(headerFields, hpack.HeaderField{Name: "grpc-timeout", Value: grpcutil.EncodeDuration(timeout)})
	}
	for k, v := range authData {
		headerFields = append(headerFields, hpack.HeaderField{Name: k, Value: encodeMetadataHeader(k, v)})
	}
	for k, v := range callAuthData {
		headerFields = append(headerFields, hpack.HeaderField{Name: k, Value: encodeMetadataHeader(k, v)})
	}
	if b := stats.OutgoingTags(ctx); b != nil {
		headerFields = append(headerFields, hpack.HeaderField{Name: "grpc-tags-bin", Value: encodeBinHeader(b)})
	}
	if b := stats.OutgoingTrace(ctx); b != nil {
		headerFields = append(headerFields, hpack.HeaderField{Name: "grpc-trace-bin", Value: encodeBinHeader(b)})
	}

	if md, added, ok := metadata.FromOutgoingContextRaw(ctx); ok {
		var k string
		for k, vv := range md {
			// HTTP doesn't allow you to set pseudoheaders after non pseudoheaders were set.
			if isReservedHeader(k) {
				continue
			}
			for _, v := range vv {
				headerFields = append(headerFields, hpack.HeaderField{Name: k, Value: encodeMetadataHeader(k, v)})
			}
		}
		for _, vv := range added {
			for i, v := range vv {
				if i%2 == 0 {
					k = strings.ToLower(v)
					continue
				}
				// HTTP doesn't allow you to set pseudoheaders after non pseudoheaders were set.
				if isReservedHeader(k) {
					continue
				}
				headerFields = append(headerFields, hpack.HeaderField{Name: k, Value: encodeMetadataHeader(k, v)})
			}
		}
	}
	if md, ok := t.md.(*metadata.MD); ok {
		for k, vv := range *md {
			if isReservedHeader(k) {
				continue
			}
			for _, v := range vv {
				headerFields = append(headerFields, hpack.HeaderField{Name: k, Value: encodeMetadataHeader(k, v)})
			}
		}
	}
	return headerFields, nil
}
```
**总结**
1. 在调用rpc接口时设置了超时的context,会最终传递到http2Client对象中,然后在处理HEADERS时会把deadline时间设置到头部参数`grpc-timeout`中,传递到服务端.
2. http2Client对象是在调用`Dial`或`DialContext`过程中生成的,每个Endpoint对应一个该对象.

## 服务端超时处理
``` go
// HandleStreams receives incoming streams using the given handler. This is
// typically run in a separate goroutine.
// traceCtx attaches trace to ctx and returns the new context.
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
	defer close(t.readerDone)
	for {
        // 该函数主要接收客户端发送过来的数据,重点关注处理HEADERS部分的数据.
        ......

		switch frame := frame.(type) {
        case *http2.MetaHeadersFrame:
            // 当frame类型为HEADERS时的处理.
			if t.operateHeaders(frame, handle, traceCtx) {
				t.Close()
				break
            }
            
        // 其它暂不关注.
        ......
	}
}

// operateHeader takes action on the decoded headers.
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
	streamID := frame.Header().StreamID
	state := &decodeState{
		serverSide: true,
    }
    // 获取HEADERS里的数据.
	if h2code, err := state.decodeHeader(frame); err != nil {
		if _, ok := status.FromError(err); ok {
			t.controlBuf.put(&cleanupStream{
				streamID: streamID,
				rst:      true,
				rstCode:  h2code,
				onWrite:  func() {},
			})
		}
		return false
	}

	buf := newRecvBuffer()
	s := &Stream{
		id:             streamID,
		st:             t,
		buf:            buf,
		fc:             &inFlow{limit: uint32(t.initialWindowSize)},
		recvCompress:   state.data.encoding,
		method:         state.data.method,
		contentSubtype: state.data.contentSubtype,
	}
	if frame.StreamEnded() {
		// s is just created by the caller. No lock needed.
		s.state = streamReadDone
    }
    // 当有设置超时时,Stream对象的ctx设置为Timeout的.
	if state.data.timeoutSet {
		s.ctx, s.cancel = context.WithTimeout(t.ctx, state.data.timeout)
	} else {
		s.ctx, s.cancel = context.WithCancel(t.ctx)
    }
    
    // 其它暂不关注.
    ......
	return false
}

func (d *decodeState) decodeHeader(frame *http2.MetaHeadersFrame) (http2.ErrCode, error) {
	// frame.Truncated is set to true when framer detects that the current header
	// list size hits MaxHeaderListSize limit.
	if frame.Truncated {
		return http2.ErrCodeFrameSize, status.Error(codes.Internal, "peer header list size exceeded limit")
	}

    // 处理HEADERS.
	for _, hf := range frame.Fields {
		d.processHeaderField(hf)
	}

    // 其它暂不关注.
    ......

	return http2.ErrCodeProtocol, status.Error(code, d.constructHTTPErrMsg())
}

func (d *decodeState) processHeaderField(f hpack.HeaderField) {
	switch f.Name {
    // 其它暂不关注.
    ......

    case "grpc-timeout":
        // 如果有该字段,解析超时时间,在这里就和客户端联系起来了.
		d.data.timeoutSet = true
		var err error
		if d.data.timeout, err = decodeTimeout(f.Value); err != nil {
			d.data.grpcErr = status.Errorf(codes.Internal, "transport: malformed time-out: %v", err)
        }
    // 其它暂不关注.
    ......
	}
}

// 经过上述分析,带超时的context已经赋值给Stream的ctx字段了.
// 最后这个ctx会被传递给RPC接口的第一个参数,这样在服务端的接口中就能感知到超时了.
```
**总结**
1. 服务端在收到HEADERS之后,会解析所有参数,如果有`grpc-timeout`,就会设置一个带timeout的context,然后传递到rpc接口中.

## 抓包
![timeout](/images/grpc-timeout.png "带timeout的HEADERS")
从图中可以看到HEADERS中有参数`grpc-timeout`,值为2994000u,超时时间为2994毫秒.

