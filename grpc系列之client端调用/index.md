# gRPC系列之client端调用


## Dail/DailContext与服务端建立连接
``` go
// 直接调用的DialContext.
// Dial creates a client connection to the given target.
func Dial(target string, opts ...DialOption) (*ClientConn, error) {
	return DialContext(context.Background(), target, opts...)
}

// DialContext默认是非阻塞的,不会等待连接成功,而是在后台处理连接逻辑.可以用grpc.WithBlock()来设置为阻塞的.
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
    // 生成conn.
	cc := &ClientConn{
		target:            target,
		csMgr:             &connectivityStateManager{},
		conns:             make(map[*addrConn]struct{}),
		dopts:             defaultDialOptions(),
		blockingpicker:    newPickerWrapper(),
		czData:            new(channelzData),
		firstResolveEvent: grpcsync.NewEvent(),
	}
	cc.retryThrottler.Store((*retryThrottler)(nil))
	cc.ctx, cc.cancel = context.WithCancel(context.Background())

    // 处理DiaoOption,来设置自定义参数.
	for _, opt := range opts {
		opt.apply(&cc.dopts)
	}

    // 处理一元RPC的拦截器.
	chainUnaryClientInterceptors(cc)
    // 处理流式RPC的拦截器.
	chainStreamClientInterceptors(cc)

	defer func() {
		if err != nil {
			cc.Close()
		}
	}()

	if channelz.IsOn() {
        ......
	}

	if !cc.dopts.insecure {
        // 校验认证相关.
        ......
	} else {
		if cc.dopts.copts.TransportCredentials != nil || cc.dopts.copts.CredsBundle != nil {
			return nil, errCredentialsConflict
        }
        // PerRPCCredentials是指设置了自定义认证.
		for _, cd := range cc.dopts.copts.PerRPCCredentials {
            // 判断自定义认证是否依赖TLS.
			if cd.RequireTransportSecurity() {
				return nil, errTransportCredentialsMissing
			}
		}
	}

    ......

    // 超时控制.
	if cc.dopts.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, cc.dopts.timeout)
		defer cancel()
	}
	defer func() {
		select {
		case <-ctx.Done():
			switch {
			case ctx.Err() == err:
				conn = nil
			case err == nil || !cc.dopts.returnLastError:
				conn, err = nil, ctx.Err()
			default:
				conn, err = nil, fmt.Errorf("%v: %v", ctx.Err(), err)
			}
		default:
		}
	}()

    ......

    // 解析target的格式,识别出对应的Scheme和Endpoint.
	// Determine the resolver to use.
	cc.parsedTarget = grpcutil.ParseTarget(cc.target)
	unixScheme := strings.HasPrefix(cc.target, "unix:")
    channelz.Infof(logger, cc.channelzID, "parsed scheme: %q", cc.parsedTarget.Scheme)
    
    // 根据Scheme来获取对应的Resolver.
    // 如gRPC默认支持的dns,就是在dns包的init函数中调用了resolver.Register()来注册dns Resolver.
	resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)
	if resolverBuilder == nil {
        // 如果获取失败,就采用默认的Resolver来解析,即passthrough.
		// If resolver builder is still nil, the parsed target's scheme is
		// not registered. Fallback to default resolver and set Endpoint to
		// the original target.
		channelz.Infof(logger, cc.channelzID, "scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)
		cc.parsedTarget = resolver.Target{
			Scheme:   resolver.GetDefaultScheme(),
			Endpoint: target,
		}
		resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)
		if resolverBuilder == nil {
			return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
		}
	}

    ......
    // 认证相关.
    // banlancer的相关Option.
	cc.balancerBuildOpts = balancer.BuildOptions{
		DialCreds:        credsClone,
		CredsBundle:      cc.dopts.copts.CredsBundle,
		Dialer:           cc.dopts.copts.Dialer,
		ChannelzParentID: cc.channelzID,
		Target:           cc.parsedTarget,
	}

    // 调用Resolver对象的Build方法,在Build内部会调用cc.UpdateState来更新服务端地址.
	// Build the resolver.
	rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}
	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()

    // 当调用grpc.WithBlock()后会阻塞直到连接成功建立.
	// A blocking dial blocks until the clientConn is ready.
	if cc.dopts.block {
		for {
            // 循环判断连接的状态是否为Ready.
			s := cc.GetState()
			if s == connectivity.Ready {
				break
			} else if cc.dopts.copts.FailOnNonTempDialError && s == connectivity.TransientFailure {
				if err = cc.connectionError(); err != nil {
					terr, ok := err.(interface {
						Temporary() bool
					})
					if ok && !terr.Temporary() {
						return nil, err
					}
				}
			}
			if !cc.WaitForStateChange(ctx, s) {
				// ctx got timeout or canceled.
				if err = cc.connectionError(); err != nil && cc.dopts.returnLastError {
					return nil, err
				}
				return nil, ctx.Err()
			}
		}
	}

	return cc, nil
}

// newCCResolverWrapper uses the resolver.Builder to build a Resolver and
// returns a ccResolverWrapper object which wraps the newly built resolver.
func newCCResolverWrapper(cc *ClientConn, rb resolver.Builder) (*ccResolverWrapper, error) {
	ccr := &ccResolverWrapper{
		cc:   cc,
		done: grpcsync.NewEvent(),
	}

	var credsClone credentials.TransportCredentials
	if creds := cc.dopts.copts.TransportCredentials; creds != nil {
		credsClone = creds.Clone()
	}
	rbo := resolver.BuildOptions{
		DisableServiceConfig: cc.dopts.disableServiceConfig,
		DialCreds:            credsClone,
		CredsBundle:          cc.dopts.copts.CredsBundle,
		Dialer:               cc.dopts.copts.Dialer,
	}

	var err error
	// We need to hold the lock here while we assign to the ccr.resolver field
	// to guard against a data race caused by the following code path,
	// rb.Build-->ccr.ReportError-->ccr.poll-->ccr.resolveNow, would end up
	// accessing ccr.resolver which is being assigned here.
	ccr.resolverMu.Lock()
    defer ccr.resolverMu.Unlock()
    // 主要是调用对应的Resolver的Build方法.
	ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, rbo)
	if err != nil {
		return nil, err
	}
	return ccr, nil
}

// 下面以gRPC默认的dns Resolver举例.
// Build creates and starts a DNS resolver that watches the name resolution of the target.
func (b *dnsBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	host, port, err := parseTarget(target.Endpoint, defaultPort)
	if err != nil {
		return nil, err
	}

    // 支持IP地址形式的target.
	// IP address.
	if ipAddr, ok := formatIP(host); ok {
        addr := []resolver.Address{{Addr: ipAddr + ":" + port}}
        // 调用cc.UpdateState,cc实际上是ccResolverWrapper对象.
		cc.UpdateState(resolver.State{Addresses: addr})
		return deadResolver{}, nil
	}

    // 需要进行dns解析.
	// DNS address (non-IP).
	ctx, cancel := context.WithCancel(context.Background())
	d := &dnsResolver{
		host:                 host,
		port:                 port,
		ctx:                  ctx,
		cancel:               cancel,
		cc:                   cc,
		rn:                   make(chan struct{}, 1),
		disableServiceConfig: opts.DisableServiceConfig,
	}

	if target.Authority == "" {
		d.resolver = defaultResolver
	} else {
		d.resolver, err = customAuthorityResolver(target.Authority)
		if err != nil {
			return nil, err
		}
	}

    d.wg.Add(1)
    // 具体的解析过程在watcher中,解析成功后也会调用cc.UpdateState.
	go d.watcher()
	d.ResolveNow(resolver.ResolveNowOptions{})
	return d, nil
}

// 所有Resolver的Build方法都会来调用到UpdateState方法.
func (ccr *ccResolverWrapper) UpdateState(s resolver.State) {
	if ccr.done.HasFired() {
		return
	}
	channelz.Infof(logger, ccr.cc.channelzID, "ccResolverWrapper: sending update to cc: %v", s)
	if channelz.IsOn() {
		ccr.addChannelzTraceEvent(s)
	}
    ccr.curState = s
    // 主要关注updateResolverState,ccr.cc是ClientConn对象.
	ccr.poll(ccr.cc.updateResolverState(ccr.curState, nil))
}

func (cc *ClientConn) updateResolverState(s resolver.State, err error) error {
	defer cc.firstResolveEvent.Fire()
    cc.mu.Lock()

    ......

	var ret error
	if cc.dopts.disableServiceConfig || s.ServiceConfig == nil {
        // 没有设置ServiceConfig,进入到该分支,主要处理balancer对象.
		cc.maybeApplyDefaultServiceConfig(s.Addresses)
		// TODO: do we need to apply a failing LB policy if there is no
		// default, per the error handling design?
	} else {
        ......
	}

	var balCfg serviceconfig.LoadBalancingConfig
	if cc.dopts.balancerBuilder == nil && cc.sc != nil && cc.sc.lbConfig != nil {
		balCfg = cc.sc.lbConfig.cfg
	}

	cbn := cc.curBalancerName
	bw := cc.balancerWrapper
	cc.mu.Unlock()
	if cbn != grpclbName {
		// Filter any grpclb addresses since we don't have the grpclb balancer.
		for i := 0; i < len(s.Addresses); {
			if s.Addresses[i].Type == resolver.GRPCLB {
				copy(s.Addresses[i:], s.Addresses[i+1:])
				s.Addresses = s.Addresses[:len(s.Addresses)-1]
				continue
			}
			i++
		}
    }
    // 主要处理连接的逻辑,在后面要重点关注.
	uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
	if ret == nil {
		ret = uccsErr // prefer ErrBadResolver state since any other error is
		// currently meaningless to the caller.
	}
	return ret
}

func (cc *ClientConn) maybeApplyDefaultServiceConfig(addrs []resolver.Address) {
	if cc.sc != nil {
		cc.applyServiceConfigAndBalancer(cc.sc, addrs)
		return
	}
	if cc.dopts.defaultServiceConfig != nil {
		cc.applyServiceConfigAndBalancer(cc.dopts.defaultServiceConfig, addrs)
	} else {
        // 默认会走到该分支.
		cc.applyServiceConfigAndBalancer(emptyServiceConfig, addrs)
	}
}

func (cc *ClientConn) applyServiceConfigAndBalancer(sc *ServiceConfig, addrs []resolver.Address) {
	if sc == nil {
		// should never reach here.
		return
	}
	cc.sc = sc

    ......

	if cc.dopts.balancerBuilder == nil {
		// Only look at balancer types and switch balancer if balancer dial
		// option is not set.
        ......
	} else if cc.balancerWrapper == nil {
		// Balancer dial option was set, and this is the first time handling
        // resolved addresses. Build a balancer with dopts.balancerBuilder.
        // 调用grpc.WithBalancerName()设置了balancerBuiler,进入该分支.
		cc.curBalancerName = cc.dopts.balancerBuilder.Name()
		cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
	}
}

func newCCBalancerWrapper(cc *ClientConn, b balancer.Builder, bopts balancer.BuildOptions) *ccBalancerWrapper {
	ccb := &ccBalancerWrapper{
		cc:       cc,
		scBuffer: buffer.NewUnbounded(),
		done:     grpcsync.NewEvent(),
		subConns: make(map[*acBalancerWrapper]struct{}),
    }
    // watcher会持续观察连接的状态变更,重点关注.
    go ccb.watcher()
    // 调用banlancer.Builder对象的Build方法来构建banlancer.
    // 注意banlancer.Builder和base.PickerBuilder这两个接口.
    // 自定义的负载均衡算法一般都是实现PickerBuilder或V2PickerBuilder接口,再通过base.NewBalancerBuilder包装成banlancer.Builder接口,实际对象是base.baseBuilder,这里Build返回的实际是base.baseBalancer对象(其实现了balancer.Balancer接口).
	ccb.balancer = b.Build(ccb, bopts)
	return ccb
}

func (bb *baseBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
    // 返回的是baseBalancer对象.
	bal := &baseBalancer{
        // 这里cc对象是ccBalancerWrapper.
		cc:              cc,
		pickerBuilder:   bb.pickerBuilder,
		v2PickerBuilder: bb.v2PickerBuilder,

		subConns: make(map[resolver.Address]balancer.SubConn),
		scStates: make(map[balancer.SubConn]connectivity.State),
		csEvltr:  &balancer.ConnectivityStateEvaluator{},
		config:   bb.config,
	}
	// Initialize picker to a picker that always returns
	// ErrNoSubConnAvailable, because when state of a SubConn changes, we
	// may call UpdateState with this picker.
	if bb.pickerBuilder != nil {
        // 当前还没有与服务端建立连接,先设置一个返回error的Picker对象.
		bal.picker = NewErrPicker(balancer.ErrNoSubConnAvailable)
	} else {
		bal.v2Picker = NewErrPickerV2(balancer.ErrNoSubConnAvailable)
	}
	return bal
}

func (ccb *ccBalancerWrapper) updateClientConnState(ccs *balancer.ClientConnState) error {
	ccb.balancerMu.Lock()
    defer ccb.balancerMu.Unlock()
    // 调用了Balancer接口的方法,这里实际上是base.baseBalancer对象.
	return ccb.balancer.UpdateClientConnState(*ccs)
}

func (b *baseBalancer) UpdateClientConnState(s balancer.ClientConnState) error {
	// TODO: handle s.ResolverState.ServiceConfig?
	if logger.V(2) {
		logger.Info("base.baseBalancer: got new ClientConn state: ", s)
	}
	// Successful resolution; clear resolver error and ensure we return nil.
	b.resolverErr = nil
	// addrsSet is the set converted from addrs, it's used for quick lookup of an address.
	addrsSet := make(map[resolver.Address]struct{})
	for _, a := range s.ResolverState.Addresses {
		addrsSet[a] = struct{}{}
		if _, ok := b.subConns[a]; !ok {
            // 创建连接,会调用到ccBalancerWrapper的NewSubConn方法,返回的实际上acBalancerWrapper对象.
			// a is a new address (not existing in b.subConns).
			sc, err := b.cc.NewSubConn([]resolver.Address{a}, balancer.NewSubConnOptions{HealthCheckEnabled: b.config.HealthCheck})
			if err != nil {
				logger.Warningf("base.baseBalancer: failed to create new SubConn: %v", err)
				continue
			}
            b.subConns[a] = sc
            // 初始状态默认为空闲.
            b.scStates[sc] = connectivity.Idle
            // 连接服务端.
			sc.Connect()
		}
	}
	for a, sc := range b.subConns {
		// a was removed by resolver.
		if _, ok := addrsSet[a]; !ok {
			b.cc.RemoveSubConn(sc)
			delete(b.subConns, a)
			// Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
			// The entry will be deleted in UpdateSubConnState.
		}
	}
	// If resolver state contains no addresses, return an error so ClientConn
	// will trigger re-resolve. Also records this as an resolver error, so when
	// the overall state turns transient failure, the error message will have
	// the zero address information.
	if len(s.ResolverState.Addresses) == 0 {
		b.ResolverError(errors.New("produced zero addresses"))
		return balancer.ErrBadResolverState
	}
	return nil
}

// 建立连接.
func (acbw *acBalancerWrapper) Connect() {
	acbw.mu.Lock()
	defer acbw.mu.Unlock()
	acbw.ac.connect()
}

// 最终会调用到这个connect方法.
// connect starts creating a transport.
// It does nothing if the ac is not IDLE.
// TODO(bar) Move this to the addrConn section.
func (ac *addrConn) connect() error {
	ac.mu.Lock()
	if ac.state == connectivity.Shutdown {
		ac.mu.Unlock()
		return errConnClosing
	}
	if ac.state != connectivity.Idle {
		ac.mu.Unlock()
		return nil
	}
	// Update connectivity state within the lock to prevent subsequent or
	// concurrent calls from resetting the transport more than once.
	ac.updateConnectivityState(connectivity.Connecting, nil)
	ac.mu.Unlock()

    // 采用异步的方式来与服务端建立连接的.
    // Start a goroutine connecting to the server asynchronously.
	go ac.resetTransport()
	return nil
}

func (ac *addrConn) resetTransport() {
	for i := 0; ; i++ {

        ......

        // 尝试去建立连接.
		newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)
		if err != nil {
            // 错误处理.
            ......
			continue
		}

		ac.mu.Lock()
		if ac.state == connectivity.Shutdown {
			ac.mu.Unlock()
			newTr.Close()
			return
		}
		ac.curAddr = addr
		ac.transport = newTr
		ac.backoffIdx = 0

        hctx, hcancel := context.WithCancel(ac.ctx)
        // 重点关注startHealthCheck方法.
		ac.startHealthCheck(hctx)
		ac.mu.Unlock()

		// Block until the created transport is down. And when this happens,
		// we restart from the top of the addr list.
		<-reconnect.Done()
		hcancel()
		// restart connecting - the top of the loop will set state to
		// CONNECTING.  This is against the current connectivity semantics doc,
		// however it allows for graceful behavior for RPCs not yet dispatched
		// - unfortunate timing would otherwise lead to the RPC failing even
		// though the TRANSIENT_FAILURE state (called for by the doc) would be
		// instantaneous.
		//
		// Ideally we should transition to Idle here and block until there is
		// RPC activity that leads to the balancer requesting a reconnect of
		// the associated SubConn.
	}
}

// startHealthCheck starts the health checking stream (RPC) to watch the health
// stats of this connection if health checking is requested and configured.
//
// LB channel health checking is enabled when all requirements below are met:
// 1. it is not disabled by the user with the WithDisableHealthCheck DialOption
// 2. internal.HealthCheckFunc is set by importing the grpc/health package
// 3. a service config with non-empty healthCheckConfig field is provided
// 4. the load balancer requests it
//
// It sets addrConn to READY if the health checking stream is not started.
//
// Caller must hold ac.mu.
func (ac *addrConn) startHealthCheck(ctx context.Context) {
    // 整个函数的重点是调用ac.updateConnecttivityState来设置状态.
	var healthcheckManagingState bool
	defer func() {
		if !healthcheckManagingState {
            // 设置连接状态为Ready.
			ac.updateConnectivityState(connectivity.Ready, nil)
		}
	}()

	if ac.cc.dopts.disableHealthCheck {
		return
    }
    // 若没有设置healthCheckConfig直接返回.
	healthCheckConfig := ac.cc.healthCheckConfig()
	if healthCheckConfig == nil {
		return
	}
	if !ac.scopts.HealthCheckEnabled {
		return
	}
	healthCheckFunc := ac.cc.dopts.healthCheckFunc
	if healthCheckFunc == nil {
		// The health package is not imported to set health check function.
		//
		// TODO: add a link to the health check doc in the error message.
		channelz.Error(logger, ac.channelzID, "Health check is requested but health check function is not set.")
		return
	}

	healthcheckManagingState = true

	// Set up the health check helper functions.
	currentTr := ac.transport
	newStream := func(method string) (interface{}, error) {
		ac.mu.Lock()
		if ac.transport != currentTr {
			ac.mu.Unlock()
			return nil, status.Error(codes.Canceled, "the provided transport is no longer valid to use")
		}
		ac.mu.Unlock()
		return newNonRetryClientStream(ctx, &StreamDesc{ServerStreams: true}, method, currentTr, ac)
    }
    // headlthCheckFunc最终会调用到下面的函数,来设置连接状态.
	setConnectivityState := func(s connectivity.State, lastErr error) {
		ac.mu.Lock()
		defer ac.mu.Unlock()
		if ac.transport != currentTr {
			return
		}
		ac.updateConnectivityState(s, lastErr)
	}
	// Start the health checking stream.
	go func() {
        // healthCheckFunc默认值是health.clientHealthCheck函数(health/client.go文件中).
		err := ac.cc.dopts.healthCheckFunc(ctx, newStream, setConnectivityState, healthCheckConfig.ServiceName)
		if err != nil {
			if status.Code(err) == codes.Unimplemented {
				channelz.Error(logger, ac.channelzID, "Subchannel health check is unimplemented at server side, thus health check is disabled")
			} else {
				channelz.Errorf(logger, ac.channelzID, "HealthCheckFunc exits with unexpected error %v", err)
			}
		}
	}()
}

// Note: this requires a lock on ac.mu.
func (ac *addrConn) updateConnectivityState(s connectivity.State, lastErr error) {
	if ac.state == s {
		return
	}
	ac.state = s
	channelz.Infof(logger, ac.channelzID, "Subchannel Connectivity change to %v", s)
	ac.cc.handleSubConnStateChange(ac.acbw, s, lastErr)
}

func (cc *ClientConn) handleSubConnStateChange(sc balancer.SubConn, s connectivity.State, err error) {
	cc.mu.Lock()
	if cc.conns == nil {
		cc.mu.Unlock()
		return
	}
	// TODO(bar switching) send updates to all balancer wrappers when balancer
	// gracefully switching is supported.
	cc.balancerWrapper.handleSubConnStateChange(sc, s, err)
	cc.mu.Unlock()
}

func (ccb *ccBalancerWrapper) handleSubConnStateChange(sc balancer.SubConn, s connectivity.State, err error) {
	// When updating addresses for a SubConn, if the address in use is not in
	// the new addresses, the old ac will be tearDown() and a new ac will be
	// created. tearDown() generates a state change with Shutdown state, we
	// don't want the balancer to receive this state change. So before
	// tearDown() on the old ac, ac.acbw (acWrapper) will be set to nil, and
	// this function will be called with (nil, Shutdown). We don't need to call
	// balancer method in this case.
	if sc == nil {
		return
    }
    // 最终会向scBuffer里写入连接的状态信息,还记得之前需要重点关注的ccBalancerWrapper对象的watcher方法么.
	ccb.scBuffer.Put(&scStateUpdate{
		sc:    sc,
		state: s,
		err:   err,
	})
}

// watcher balancer functions sequentially, so the balancer can be implemented
// lock-free.
func (ccb *ccBalancerWrapper) watcher() {
	for {
		select {
        case t := <-ccb.scBuffer.Get():
            // 从scBuffer中获取数据.
			ccb.scBuffer.Load()
			if ccb.done.HasFired() {
				break
			}
			ccb.balancerMu.Lock()
            su := t.(*scStateUpdate)
            // 调用Balancer接口的UpdateSubConnState来设置连接的状态,这里balancer对象是base.baseBalancer对象.
			ccb.balancer.UpdateSubConnState(su.sc, balancer.SubConnState{ConnectivityState: su.state, ConnectionError: su.err})
			ccb.balancerMu.Unlock()
		case <-ccb.done.Done():
        }

		if ccb.done.HasFired() {
			ccb.balancer.Close()
			ccb.mu.Lock()
			scs := ccb.subConns
			ccb.subConns = nil
			ccb.mu.Unlock()
			for acbw := range scs {
				ccb.cc.removeAddrConn(acbw.getAddrConn(), errConnDrain)
			}
			ccb.UpdateState(balancer.State{ConnectivityState: connectivity.Connecting, Picker: nil})
			return
		}
	}
}

func (b *baseBalancer) UpdateSubConnState(sc balancer.SubConn, state balancer.SubConnState) {
    s := state.ConnectivityState

    ......

    // 中间代码忽略,重点关注下面的.
	// Regenerate picker when one of the following happens:
	//  - this sc entered or left ready
	//  - the aggregated state of balancer is TransientFailure
	//    (may need to update error message)
	if (s == connectivity.Ready) != (oldS == connectivity.Ready) ||
		b.state == connectivity.TransientFailure {
        // 获取Picker接口的对象.
		b.regeneratePicker()
	}

    // 这里cc对象是ccBalancerWrapper对象(实现了balancer.ClientConn接口).
	b.cc.UpdateState(balancer.State{ConnectivityState: b.state, Picker: b.picker})
}

// regeneratePicker takes a snapshot of the balancer, and generates a picker
// from it. The picker is
//  - errPicker if the balancer is in TransientFailure,
//  - built by the pickerBuilder with all READY SubConns otherwise.
func (b *baseBalancer) regeneratePicker() {
	if b.state == connectivity.TransientFailure {
		b.picker = NewErrPicker(b.mergeErrors())
		return
    }
    // 记录状态为Ready的连接.
	readySCs := make(map[balancer.SubConn]SubConnInfo)

	// Filter out all ready SCs from full subConn map.
	for addr, sc := range b.subConns {
		if st, ok := b.scStates[sc]; ok && st == connectivity.Ready {
			readySCs[sc] = SubConnInfo{Address: addr}
		}
    }
    // 调用PickerBuilder接口的Build方法来生成Picker接口,供后续负载均衡使用,一般自定义负载均衡算法是需要实现这两个接口的.
	b.picker = b.pickerBuilder.Build(PickerBuildInfo{ReadySCs: readySCs})
}

func (ccb *ccBalancerWrapper) UpdateState(s balancer.State) {
	ccb.mu.Lock()
	defer ccb.mu.Unlock()
	if ccb.subConns == nil {
		return
    }
    
    // 设置ClientConn的blockingpicker对象的picker字段.
	// Update picker before updating state.  Even though the ordering here does
	// not matter, it can lead to multiple calls of Pick in the common start-up
	// case where we wait for ready and then perform an RPC.  If the picker is
	// updated later, we could call the "connecting" picker when the state is
	// updated, and then call the "ready" picker after the picker gets updated.
	ccb.cc.blockingpicker.updatePicker(s.Picker)
	ccb.cc.csMgr.updateState(s.ConnectivityState)
}
```

## 调用RPC接口
``` go
func (c *helloServiceClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
    out := new(HelloReply)
    // 这里cc是ClientConn对象,调用其Invoke方法,定义在call.go文件里.
	err := c.cc.Invoke(ctx, "/HelloService/SayHello", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

// Invoke sends the RPC request on the wire and returns after response is
// received.  This is typically called by generated code.
//
// All errors returned by Invoke are compatible with the status package.
func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
	// allow interceptor to see all applicable call options, which means those
	// configured as defaults from dial option as well as per-call options
	opts = combine(cc.dopts.callOptions, opts)

	if cc.dopts.unaryInt != nil {
        // 如果有设置客户端拦截器,最终也会调用到invoke函数.
		return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
    }
    // 没有就直接调用invoke函数.
	return invoke(ctx, method, args, reply, cc, opts...)
}

func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
    // 该函数内部会使用负载均衡算法,来选出一个连接.
	cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
	if err != nil {
		return err
	}
	if err := cs.SendMsg(req); err != nil {
		return err
	}
	return cs.RecvMsg(reply)
}

func newClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, opts ...CallOption) (_ ClientStream, err error) {
    // 前面代码可以不关注.
    ......

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

    // 重点关注newAttempLocked.
	// Only this initial attempt has stats/tracing.
	// TODO(dfawley): move to newAttempt when per-attempt stats are implemented.
	if err := cs.newAttemptLocked(sh, trInfo); err != nil {
		cs.finish(err)
		return nil, err
	}

	op := func(a *csAttempt) error { return a.newStream() }
	if err := cs.withRetry(op, func() { cs.bufferForRetryLocked(0, op) }); err != nil {
		cs.finish(err)
		return nil, err
	}

	if cs.binlog != nil {
		md, _ := metadata.FromOutgoingContext(ctx)
		logEntry := &binarylog.ClientHeader{
			OnClientSide: true,
			Header:       md,
			MethodName:   method,
			Authority:    cs.cc.authority,
		}
		if deadline, ok := ctx.Deadline(); ok {
			logEntry.Timeout = time.Until(deadline)
			if logEntry.Timeout < 0 {
				logEntry.Timeout = 0
			}
		}
		cs.binlog.Log(logEntry)
	}

	if desc != unaryStreamDesc {
		// Listen on cc and stream contexts to cleanup when the user closes the
		// ClientConn or cancels the stream context.  In all other cases, an error
		// should already be injected into the recv buffer by the transport, which
		// the client will eventually receive, and then we will cancel the stream's
		// context in clientStream.finish.
		go func() {
			select {
			case <-cc.ctx.Done():
				cs.finish(ErrClientConnClosing)
			case <-ctx.Done():
				cs.finish(toRPCErr(ctx.Err()))
			}
		}()
	}
	return cs, nil
}

// newAttemptLocked creates a new attempt with a transport.
// If it succeeds, then it replaces clientStream's attempt with this new attempt.
func (cs *clientStream) newAttemptLocked(sh stats.Handler, trInfo *traceInfo) (retErr error) {
	newAttempt := &csAttempt{
		cs:           cs,
		dc:           cs.cc.dopts.dc,
		statsHandler: sh,
		trInfo:       trInfo,
    }
    
    ......

    // 重点关注getTransport.
	t, done, err := cs.cc.getTransport(ctx, cs.callInfo.failFast, cs.callHdr.Method)
	if err != nil {
		return err
	}
	if trInfo != nil {
		trInfo.firstLine.SetRemoteAddr(t.RemoteAddr())
	}
	newAttempt.t = t
	newAttempt.done = done
	cs.attempt = newAttempt
	return nil
}

func (cc *ClientConn) getTransport(ctx context.Context, failfast bool, method string) (transport.ClientTransport, func(balancer.DoneInfo), error) {
    // 调用pick方法来负载均衡.
	t, done, err := cc.blockingpicker.pick(ctx, failfast, balancer.PickInfo{
		Ctx:            ctx,
		FullMethodName: method,
	})
	if err != nil {
		return nil, nil, toRPCErr(err)
	}
	return t, done, nil
}

// pick returns the transport that will be used for the RPC.
// It may block in the following cases:
// - there's no picker
// - the current picker returns ErrNoSubConnAvailable
// - the current picker returns other errors and failfast is false.
// - the subConn returned by the current picker is not READY
// When one of these situations happens, pick blocks until the picker gets updated.
func (pw *pickerWrapper) pick(ctx context.Context, failfast bool, info balancer.PickInfo) (transport.ClientTransport, func(balancer.DoneInfo), error) {
	var ch chan struct{}

	var lastPickErr error
	for {
		pw.mu.Lock()
		if pw.done {
			pw.mu.Unlock()
			return nil, nil, ErrClientConnClosing
		}

		if pw.picker == nil {
			ch = pw.blockingCh
		}
		if ch == pw.blockingCh {

            ......

			continue
		}

		ch = pw.blockingCh
		p := pw.picker
		pw.mu.Unlock()

        // 调用Pick方法负载均衡,可以是自定义负载均衡算法,gRPC默认的是轮询.
		pickResult, err := p.Pick(info)

		if err != nil {
            // 返回错误时的处理.
            ......
		}

        // 把conn显示的转化为acBalancerWrapper.
		acw, ok := pickResult.SubConn.(*acBalancerWrapper)
		if !ok {
			logger.Error("subconn returned from pick is not *acBalancerWrapper")
			continue
        }
        
        // 调用getReadyTransport,来判断状态是否为Ready,若为Idle则发起connect.
		if t, ok := acw.getAddrConn().getReadyTransport(); ok {
			if channelz.IsOn() {
				return t, doneChannelzWrapper(acw, pickResult.Done), nil
			}
			return t, pickResult.Done, nil
        }
        // 状态不为Ready,则返回重新进行pick.
		if pickResult.Done != nil {
			// Calling done with nil error, no bytes sent and no bytes received.
			// DoneInfo with default value works.
			pickResult.Done(balancer.DoneInfo{})
		}
		logger.Infof("blockingPicker: the picked transport is not ready, loop back to repick")
		// If ok == false, ac.state is not READY.
		// A valid picker always returns READY subConn. This means the state of ac
		// just changed, and picker will be updated shortly.
		// continue back to the beginning of the for loop to repick.
	}
}
```

## 总结
* 要实现自定义Resolver,需要实现resolver.Builder接口和resolver.Resolver接口.在init函数中调用resolver.Register来注册自定义的Builder对象.在Builder接口的Build方法中要调用UpdateState来更新服务端地址信息.客户端在解析target时会根据scheme来获取对应的Resolver对象.
* 要实现自定义负载均衡算法,需要实现base.PickerBuilder接口和balancer.Picker接口(注意这两个接口有V2版的),通过base.NewBalancerBuilder包装成balancer.Builder接口,再在init函数中调用balancer.Register来注册该Builder对象.最后grpc.WithBalancerName把Builder包装成grpc.DialOption.
* 要实现自定义认证,需要实现credentials.PerRPCCredentials接口,然后调用grpc.WithPerRPCCredentials把对象包装成grpc.DialOption.
* 在客户端调用`Dial`或`DialContext`过程中,会先通过Resolver来解析出所有的Endpoint,然后会与每一个Endpoint建立连接(采用异步的方式),同时也会设置好对应的负载均衡Picker对象.注意如果使用了grpc.WithBlock()会同步等待连接建立成功(有一个成功就可以了),否则就会直接返回.
* 负载均衡算法会在调用具体的RPC函数过程中使用到,在invoke函数(call.go文件中)中会根据负载均衡算法来选定一个连接,然后发起请求.


