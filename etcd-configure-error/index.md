# Etcd配置错误后的分析


## 问题起因
在采用开源框架[go-zero](https://github.com/tal-tech/go-zero)开发的过程中,服务在启动一段时间后控制台一直在报如下警告信息:
``` json
{
  "level": "warn",
  "ts": "2020-12-15T16:43:21.709+0800",
  "caller": "clientv3/retry_interceptor.go:62",
  "msg": "retrying of unary invoker failed",
  "target": "endpoint://client-478bc374-adc9-4cec-9dcb-1c8245b9ad36/192.168.20.151:2379",
  "attempt": 0,
  "error": "rpc error: code = DeadlineExceeded desc = latest balancer error: all SubConns are in TransientFailure, latest connection error: c
onnection error: desc = \"transport: Error while dialing dial tcp 0.0.0.0:2379: connectex: No connection could be made because the target machine actively refused it.\""}
```
看信息是在连接`0.0.0.0:2379`的时候失败了,但target明明是`192.168.20.151:2379`,怎么会变成`0.0.0.0`的列?

## 问题分析
根据报错信息,有定位到是`clientv3/retry_interceptor.go:62`,那就在该文件打个断点看看到底是怎么回事?
![error](/images/etcd_errpicker.png "定位错误产生的位置")
可以看到在调用gRPC接口时,`Picker`接口实际指向了`errPicker`,导致错误的发生.

回头再去分析调用链上的`pickerWrapper.pick`函数(来自文件`grpc@v1.29.1/picker_wrapper.go`),有如下代码:
``` go
ch = pw.blockingCh
p := pw.picker
pw.mu.Unlock()

pickResult, err := p.Pick(info)
```
`Picker`来自于`pw.picker`字段,而该字段的变更只在如下代码中:
``` go
// updatePicker is called by UpdateBalancerState. It unblocks all blocked pick.
func (pw *pickerWrapper) updatePicker(p balancer.Picker) {
	pw.updatePickerV2(&v2PickerWrapper{picker: p, connErr: pw.connErr})
}

// updatePicker is called by UpdateBalancerState. It unblocks all blocked pick.
func (pw *pickerWrapper) updatePickerV2(p balancer.V2Picker) {
	pw.mu.Lock()
	if pw.done {
		pw.mu.Unlock()
		return
	}
	pw.picker = p
	// pw.blockingCh should never be nil.
	close(pw.blockingCh)
	pw.blockingCh = make(chan struct{})
	pw.mu.Unlock()
}
```

继续在`updatePicker`函数断点,看是什么导致的变更?
![updatePicker](/images/etcd_updatepicker.png "跟踪updatePicker")
调用链源头为`ccBalancerWrapper.watcher`函数(来自文件`grpc@v1.29.1/balancer_conn_wrappers.go`)
``` go
// watcher balancer functions sequentially, so the balancer can be implemented
// lock-free.
func (ccb *ccBalancerWrapper) watcher() {
	for {
		select {
		case t := <-ccb.scBuffer.Get():
			ccb.scBuffer.Load()
			if ccb.done.HasFired() {
				break
			}
			ccb.balancerMu.Lock()
			su := t.(*scStateUpdate)
			if ub, ok := ccb.balancer.(balancer.V2Balancer); ok {
				ub.UpdateSubConnState(su.sc, balancer.SubConnState{ConnectivityState: su.state, ConnectionError: su.err})
			} else {
				ccb.balancer.HandleSubConnStateChange(su.sc, su.state)
			}
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
```
变化来自channel对象`ccb.scBuffer`,搜索该对象只在`ccBalancerWrapper.handleSubConnStateChange`函数中有`Put`操作.
``` go
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
	ccb.scBuffer.Put(&scStateUpdate{
		sc:    sc,
		state: s,
		err:   err,
	})
}
```

在`Put`地方打上断点继续追踪
![resetTransport](/images/etcd_reset.png "Put消息")
在调用链入口查看变量:
![addrs](/images/etcd_addrs.png "addrs的值")
可以看到地址为`0.0.0.0:2379`,找到地址来源了,与错误信息中的可以匹配.

继续追踪`resetTransport`,只会在`addrConn.connect`函数被调用,在此断点:
![connect](/images/etcd_sync.png "追踪connect")
可以看到在`Sync`函数(在文件`etcd/clientv3/client.go`)中获取的地址就是`0.0.0.0:2379`(来自`Members`的`ClientURLs`字段),而`c.MemberList`最终调用到如下:(来自文件`etcd/etcdserver/etcdserverpb/rpc.pb.go`,**etcd当前最新版3.4.14的文件位置已发生变化**)
``` go
func (c *clusterClient) MemberList(ctx context.Context, in *MemberListRequest, opts ...grpc.CallOption) (*MemberListResponse, error) {
	out := new(MemberListResponse)
	err := grpc.Invoke(ctx, "/etcdserverpb.Cluster/MemberList", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```
调用gRPC接口`/etcdserverpb.Cluster/MemberList`获取etcd集群的成员列表.而etcd集群的成员列表是由环境变量`ETCD_ADVERTISE_CLIENT_URLS`或参数`--advertise-client-urls`来控制的.

查看etcd的部署文件`docker-compose.yml`,发现配置为`- "ETCD_ADVERTISE_CLIENT_URLS=http://0.0.0.0:2379"`,把配置修改为`http://192.168.20.151:2379`,该问题消失.

## 总结
### 配置文档
在[官网文档](https://etcd.io/docs/v3.4.0/faq/)上其实已经提到了关于配置项的:

**Configuration**

What is the difference between `listen-<client,peer>-urls`, `advertise-client-urls` or `initial-advertise-peer-urls`?

`listen-client-urls` and `listen-peer-urls` specify the local addresses etcd server binds to for accepting incoming connections. To listen on a port for all interfaces, specify `0.0.0.0` as the listen IP address.

`advertise-client-urls` and `initial-advertise-peer-urls` specify the addresses etcd clients or other etcd members should use to contact the etcd server. The advertise addresses must be reachable from the remote machines. Do not advertise addresses like `localhost` or `0.0.0.0` for a production setup since these addresses are unreachable from remote machines.

明确提到了不要把`advertise-client-urls`和`initial-advertise-peer-urls`设置成`localhost`或`0.0.0.0`,这些地址从远程机器是不能访问的,应该配置为具体的IP地址.

### 流程
![流程](/images/etcd_follow.png "流程图")

