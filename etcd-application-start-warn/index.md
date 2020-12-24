# 服务启动时连接etcd集群的告警分析


## 问题起因
通过`docker-compose`部署了3个节点的etcd集群,服务在启动时会随机报0个、1个或2个告警信息,信息如下:
``` json
{
  "level": "warn",
  "ts": "2020-12-21T14:43:33.380+0800",
  "caller": "clientv3/retry_interceptor.go:62",
  "msg": "retrying of unary invoker failed",
  "target": "passthrough:///192.168.20.151:22379",
  "attempt": 0,
  "error": "rpc error: code = Canceled desc = context canceled"
}
```

## 问题分析
服务是基于[go-zero](https://github.com/tal-tech/go-zero)框架的,etcd配置如下:
``` yml
Etcd:
  Hosts:
  - 192.168.20.151:2379
  - 192.168.20.151:12379
  - 192.168.20.151:22379
  Key: xxx.rpc
```

由于是服务启动时报的,追踪服务的启动过程.在启动时会向etcd注册服务的信息.首先是创建etcd客户端:
``` go
// go-zero@v1.0.28/core/discov/internal/registry.go
func DialClient(endpoints []string) (EtcdClient, error) {
	return clientv3.New(clientv3.Config{
		Endpoints:            endpoints,
		AutoSyncInterval:     autoSyncInterval,
		DialTimeout:          DialTimeout,
		DialKeepAliveTime:    dialKeepAliveTime,
		DialKeepAliveTimeout: DialTimeout,
		RejectOldCluster:     true,
	})
}
```

继续追踪`clientv3.New`方法,最终会调用`newClient`:
``` go
// etcd/cleintv3/client.go
func newClient(cfg *Config) (*Client, error) {
	if cfg == nil {
		cfg = &Config{}
    }
    
    // 中间一大串可以忽略.
    ......

    // 前面该参数是被设置为true的,就会调用到checkVersion.
	if cfg.RejectOldCluster {
		if err := client.checkVersion(); err != nil {
			client.Close()
			return nil, err
		}
	}

	go client.autoSync()
	return client, nil
}

func (c *Client) checkVersion() (err error) {
	var wg sync.WaitGroup

	eps := c.Endpoints()
	errc := make(chan error, len(eps))
	ctx, cancel := context.WithCancel(c.ctx)
	if c.cfg.DialTimeout > 0 {
		cancel()
		ctx, cancel = context.WithTimeout(c.ctx, c.cfg.DialTimeout)
	}

    wg.Add(len(eps))
    // 由于配置了3个etcd节点的地址,这里会起3个goroutine.
	for _, ep := range eps {
		// if cluster is current, any endpoint gives a recent version
		go func(e string) {
            defer wg.Done()
            // 查询状态.
			resp, rerr := c.Status(ctx, e)
			if rerr != nil {
				errc <- rerr
				return
            }
            // 解析版本号.
			vs := strings.Split(resp.Version, ".")
			maj, min := 0, 0
			if len(vs) >= 2 {
				var serr error
				if maj, serr = strconv.Atoi(vs[0]); serr != nil {
					errc <- serr
					return
				}
				if min, serr = strconv.Atoi(vs[1]); serr != nil {
					errc <- serr
					return
				}
			}
			if maj < 3 || (maj == 3 && min < 2) {
				rerr = ErrOldCluster
			}
			errc <- rerr
		}(ep)
	}
	// wait for success
	for range eps {
        // 只有errc这个channel内有数据且错误是nil,就会跳出循环.
		if err = <-errc; err == nil {
			break
		}
    }
    // 跳出后就会调用cancel函数取消其它goroutine的查询状态操作.
    // 这里就会导致有rpc操作被取消了,对应到本文的错误信息.
    // 告警信息为什么有时候是0个,有时候又是1个或2个列?主要是看这里3个操作完成的时间,如果同时完成,cancel就没作用,就不会有告警信息了.
	cancel()
	wg.Wait()
	return err
}
```

## 问题总结
1. 当连接到是etcd集群且不允许连接老版本的集群,则这里的告警信息是正常的.
2. 当碰到问题时多看源码.

