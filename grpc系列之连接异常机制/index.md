# gRPC系列之连接异常机制


## 连接server端失败的处理
### 重试机制
``` go
// 重点关注addrConn.resetTransport方法.
func (ac *addrConn) resetTransport() {
    // 代码逻辑放在一个死循环里的.
	for i := 0; ; i++ {
		if i > 0 {
			ac.cc.resolveNow(resolver.ResolveNowOptions{})
		}

        ac.mu.Lock()
        // 当连接关闭时直接返回.
		if ac.state == connectivity.Shutdown {
			ac.mu.Unlock()
			return
		}

        addrs := ac.addrs
        // 获取backoff时间,根据重试的次数会计算出不同的时间,算法后面重点关注.
		backoffFor := ac.dopts.bs.Backoff(ac.backoffIdx)
        // This will be the duration that dial gets to finish.
        // 超时时间,默认20秒.
		dialDuration := minConnectTimeout
		if ac.dopts.minConnectTimeout != nil {
			dialDuration = ac.dopts.minConnectTimeout()
		}

		if dialDuration < backoffFor {
			// Give dial more time as we keep failing to connect.
			dialDuration = backoffFor
		}
		// We can potentially spend all the time trying the first address, and
		// if the server accepts the connection and then hangs, the following
		// addresses will never be tried.
		//
		// The spec doesn't mention what should be done for multiple addresses.
		// https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md#proposed-backoff-algorithm
		connectDeadline := time.Now().Add(dialDuration)

        // 更新状态为连接中.
		ac.updateConnectivityState(connectivity.Connecting, nil)
		ac.transport = nil
		ac.mu.Unlock()

        // 尝试连接服务端.
		newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)
		if err != nil {
			// After exhausting all addresses, the addrConn enters
			// TRANSIENT_FAILURE.
            ac.mu.Lock()
            // 如果失败了,且状态为Shutdown直接返回.
			if ac.state == connectivity.Shutdown {
				ac.mu.Unlock()
				return
            }
            // 标记状态为失败.
			ac.updateConnectivityState(connectivity.TransientFailure, err)

			// Backoff.
			b := ac.resetBackoff
			ac.mu.Unlock()

            // 根据backoff时间创建定时器.
			timer := time.NewTimer(backoffFor)
			select {
            case <-timer.C:
                // backoff时间到,增加backoff次数,继续循环去尝试连接.
				ac.mu.Lock()
				ac.backoffIdx++
				ac.mu.Unlock()
            case <-b:
                // 外部重置了backoff,马上重新循环去尝试连接.
				timer.Stop()
            case <-ac.ctx.Done():
                // context取消了或超时了,直接返回.
				timer.Stop()
				return
			}
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
```
**总结**
1. 当连接失败后会等待一段时间之后再尝试重连,时间间隔的算法依赖于backoff.Strategy接口的Backoff方法.
2. 利用context的超时控制或取消机制,直接结束.

### Backoff算法
``` go
// 在DialContext函数中,当没有设置bs自定义参数时,会默认设置为DefaultExponential.
	if cc.dopts.bs == nil {
		cc.dopts.bs = backoff.DefaultExponential
    }
    
// internal/backoffG/backoff.go
var DefaultExponential = Exponential{Config: grpcbackoff.DefaultConfig}

// backoff/backoff.go
// DefaultConfig is a backoff configuration with the default values specfied
// at https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md.
//
// This should be useful for callers who want to configure backoff with
// non-default values only for a subset of the options.
var DefaultConfig = Config{
    // 第一次失败之后的延迟时间.
    BaseDelay:  1.0 * time.Second,
    // 多次失败之后的时间乘数.
    Multiplier: 1.6,
    // 随机因子.
    Jitter:     0.2,
    // 最大延迟时间.
	MaxDelay:   120 * time.Second,
}

// Backoff returns the amount of time to wait before the next retry given the
// number of retries.
func (bc Exponential) Backoff(retries int) time.Duration {
    // 当重试次数为0时直接返回BaseDelay,为1秒.
	if retries == 0 {
		return bc.Config.BaseDelay
	}
	backoff, max := float64(bc.Config.BaseDelay), float64(bc.Config.MaxDelay)
	for backoff < max && retries > 0 {
        // 当backoff小于max且重试次数大于0时不断的乘以Multiplier.
		backoff *= bc.Config.Multiplier
		retries--
	}
	if backoff > max {
		backoff = max
    }
    // 对时间加上一个随机数.
	// Randomize backoff delays so that if a cluster of requests start at
	// the same time, they won't operate in lockstep.
	backoff *= 1 + bc.Config.Jitter*(grpcrand.Float64()*2-1)
	if backoff < 0 {
		return 0
	}
	return time.Duration(backoff)
}
```
**总结**
1. 可以通过grpc.WithConnectParams和grpc.WithBackoff来设置自定义的backoff策略,在自定义策略里可以定义重试的时间间隔.
2. 默认的backoff策略,第一次重试间隔为1秒,第二次为1\*1.6+随机数...第N次为1\*1.6^N +随机数(其中1\*1.6^N最大不能超过120秒).

