# Etcd操作指南


## 简介
etcd是强一致性的,分布式KV存储,为分布式系统或集群提供可靠的数据存储.底层基于[raft](https://raft.github.io/raft.pdf)共识算法,可以在网络分区情况下正常进行leader选举,即便是leader节点出现故障.满足CAP中的CP.

## 部署

### 基于源码
**编译**

``` bash
# 下载源码.
git clone git@github.com:etcd-io/etcd.git

# 编译,可执行文件放在bin目录下.
cd etcd
make build
```

**集群方式运行**

在本机上起3个etcd实例,放在如下目录:
``` bash
# sky @ sky-HP in ~/data/etcdcluster [11:10:09] 
$ ll
总用量 23M
drwxr-xr-x 3 sky sky 4.0K 12月 20 11:09 etcd1
drwxr-xr-x 3 sky sky 4.0K 12月 20 11:09 etcd2
drwxr-xr-x 3 sky sky 4.0K 12月 20 11:09 etcd3
-rwxr-xr-x 1 sky sky  23M 12月 19 12:51 etcdctl
-rwxr-xr-x 1 sky sky 2.0K 12月 20 11:07 start.sh

# sky @ sky-HP in ~/data/etcdcluster [11:28:14] 
$ cd etcd1/

# sky @ sky-HP in ~/data/etcdcluster/etcd1 [11:29:47] 
$ ll
总用量 29M
drwxr-xr-x 3 sky sky 4.0K 12月 20 11:07 data.etcd
-rwxr-xr-x 1 sky sky  29M 12月 19 12:51 etcd
```
用脚本`start.sh`来启动etcd实例.
``` bash
#!/bin/bash

TOKEN=token-1
CLUSTER_STATE=new
NAME_1=etcd-1
NAME_2=etcd-2
NAME_3=etcd-3
HOST_1=192.168.3.5
HOST_2=192.168.3.5
HOST_3=192.168.3.5
PEER_PORT_1=2380
PEER_PORT_2=3380
PEER_PORT_3=4380
CLIENT_PORT_1=2379
CLIENT_PORT_2=3379
CLIENT_PORT_3=4379
CLUSTER=${NAME_1}=http://${HOST_1}:${PEER_PORT_1},${NAME_2}=http://${HOST_2}:${PEER_PORT_2},${NAME_3}=http://${HOST_3}:${PEER_PORT_3}
 
nohup ./etcd1/etcd --data-dir=./etcd1/data.etcd --name ${NAME_1} \
    --initial-advertise-peer-urls http://${HOST_1}:${PEER_PORT_1} --listen-peer-urls http://${HOST_1}:${PEER_PORT_1} \
    --advertise-client-urls http://${HOST_1}:${CLIENT_PORT_1} --listen-client-urls http://${HOST_1}:${CLIENT_PORT_1} \
    --initial-cluster ${CLUSTER} \
    --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN} > etcd1.log 2>&1 &

nohup ./etcd2/etcd --data-dir=./etcd2/data.etcd --name ${NAME_2} \
    --initial-advertise-peer-urls http://${HOST_2}:${PEER_PORT_2} --listen-peer-urls http://${HOST_2}:${PEER_PORT_2} \
    --advertise-client-urls http://${HOST_2}:${CLIENT_PORT_2} --listen-client-urls http://${HOST_2}:${CLIENT_PORT_2} \
    --initial-cluster ${CLUSTER} \
    --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN} > etcd2.log 2>&1 &
 
nohup ./etcd3/etcd --data-dir=./etcd3/data.etcd --name ${NAME_3} \
    --initial-advertise-peer-urls http://${HOST_3}:${PEER_PORT_3} --listen-peer-urls http://${HOST_3}:${PEER_PORT_3} \
    --advertise-client-urls http://${HOST_3}:${CLIENT_PORT_3} --listen-client-urls http://${HOST_3}:${CLIENT_PORT_3} \
    --initial-cluster ${CLUSTER} \
    --initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN} > etcd3.log 2>&1 &
```

**查看集群状态**

``` bash
# sky @ sky-HP in ~/data/etcdcluster [11:32:54] 
$ export ETCDCTL_API=3  

# sky @ sky-HP in ~/data/etcdcluster [11:09:51] 
$ ENDPOINTS=192.168.3.5:2379,192.168.3.5:3379,192.168.3.5:4379

# sky @ sky-HP in ~/data/etcdcluster [11:10:47] 
$ ./etcdctl --endpoints=$ENDPOINTS --write-out=table member list
+------------------+---------+--------+-------------------------+-------------------------+------------+
|        ID        | STATUS  |  NAME  |       PEER ADDRS        |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+--------+-------------------------+-------------------------+------------+
| 1dd6eece19e43a7e | started | etcd-1 | http://192.168.3.5:2380 | http://192.168.3.5:2379 |      false |
| 2be980070f5fd239 | started | etcd-2 | http://192.168.3.5:3380 | http://192.168.3.5:3379 |      false |
| e85f82dc23644097 | started | etcd-3 | http://192.168.3.5:4380 | http://192.168.3.5:4379 |      false |
+------------------+---------+--------+-------------------------+-------------------------+------------+

# sky @ sky-HP in ~/data/etcdcluster [11:10:57] 
$ ./etcdctl --endpoints=$ENDPOINTS --write-out=table endpoint status
+------------------+------------------+-----------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT     |        ID        |  VERSION  | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+------------------+------------------+-----------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.3.5:2379 | 1dd6eece19e43a7e | 3.5.0-pre |   25 kB |     false |      false |         9 |         56 |                 56 |        |
| 192.168.3.5:3379 | 2be980070f5fd239 | 3.5.0-pre |   25 kB |      true |      false |         9 |         56 |                 56 |        |
| 192.168.3.5:4379 | e85f82dc23644097 | 3.5.0-pre |   25 kB |     false |      false |         9 |         56 |                 56 |        |
+------------------+------------------+-----------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

**相关操作**

``` bash
# sky @ sky-HP in ~/data/etcdcluster [11:36:23] 
$ ./etcdctl --endpoints=$ENDPOINTS put key1 value1
OK

# sky @ sky-HP in ~/data/etcdcluster [11:36:49] 
$ ./etcdctl --endpoints=$ENDPOINTS put key2 value2
OK

# sky @ sky-HP in ~/data/etcdcluster [11:36:55] 
$ ./etcdctl --endpoints=$ENDPOINTS put key3 value3
OK

# sky @ sky-HP in ~/data/etcdcluster [11:36:59] 
$ ./etcdctl --endpoints=$ENDPOINTS get key1
key1
value1

# sky @ sky-HP in ~/data/etcdcluster [11:37:09] 
$ ./etcdctl --endpoints=$ENDPOINTS get key --prefix
key1
value1
key2
value2
key3
value3

# sky @ sky-HP in ~/data/etcdcluster [11:37:15] 
$ ./etcdctl --endpoints=$ENDPOINTS del key1
1

# sky @ sky-HP in ~/data/etcdcluster [11:38:10] 
$ ./etcdctl --endpoints=$ENDPOINTS del key --prefix
2

# 终端１,监听key为stock1的操作,watch会阻塞.
# sky @ sky-HP in ~/data/etcdcluster [11:38:14] 
$ ./etcdctl --endpoints=$ENDPOINTS watch stock1

# 终端２,put操作
# sky @ sky-HP in ~/data/etcdcluster [13:19:57] 
$ ./etcdctl --endpoints=$ENDPOINTS put stock1 100
OK

# 终端１,会实时打印出终端2的操作.
# sky @ sky-HP in ~/data/etcdcluster [11:38:14] 
$ ./etcdctl --endpoints=$ENDPOINTS watch stock1
PUT
stock1
100

# 终端2,删除stock1
# sky @ sky-HP in ~/data/etcdcluster [13:22:56] 
$ ./etcdctl --endpoints=$ENDPOINTS del stock1
1

# 终端1,实时打印.
# sky @ sky-HP in ~/data/etcdcluster [11:38:14] 
$ ./etcdctl --endpoints=$ENDPOINTS watch stock1
PUT
stock1
100
DELETE
stock1

# 终端1,监听前缀为stock的操作.
# sky @ sky-HP in ~/data/etcdcluster [13:24:30] C:130
$ ./etcdctl --endpoints=$ENDPOINTS watch stock --prefix

# 终端2.
# sky @ sky-HP in ~/data/etcdcluster [13:24:32] 
$ ./etcdctl --endpoints=$ENDPOINTS put stock1 100
OK

# sky @ sky-HP in ~/data/etcdcluster [13:24:42] 
$ ./etcdctl --endpoints=$ENDPOINTS put stock2 100      
OK

# 终端1,实时打印.
# sky @ sky-HP in ~/data/etcdcluster [13:24:30] C:130
$ ./etcdctl --endpoints=$ENDPOINTS watch stock --prefix
PUT
stock1
100
PUT
stock2
100

# lease相关操作.
# sky @ sky-HP in ~/data/etcdcluster [13:27:11] C:1
$ ./etcdctl --endpoints=$ENDPOINTS lease grant 300
lease 4097767e1dfcb207 granted with TTL(300s)

# sky @ sky-HP in ~/data/etcdcluster [13:27:22] 
$ ./etcdctl --endpoints=$ENDPOINTS put sample value --lease=4097767e1dfcb207
OK

# sky @ sky-HP in ~/data/etcdcluster [13:27:59] 
$ ./etcdctl --endpoints=$ENDPOINTS get sample
sample
value

# 300s之后再次访问,sample已不存在.
# sky @ sky-HP in ~/data/etcdcluster [13:34:26] 
$ ./etcdctl --endpoints=$ENDPOINTS get sample

# sky @ sky-HP in ~/data/etcdcluster [13:36:02] C:1
$ ./etcdctl --endpoints=$ENDPOINTS lease grant 300                         
lease 4097767e1dfcb20b granted with TTL(300s)

# sky @ sky-HP in ~/data/etcdcluster [13:36:13] 
$ ./etcdctl --endpoints=$ENDPOINTS put sample value --lease=4097767e1dfcb20b
OK

# sky @ sky-HP in ~/data/etcdcluster [13:37:30] C:130
$ ./etcdctl --endpoints=$ENDPOINTS lease keep-alive 4097767e1dfcb20b
lease 4097767e1dfcb20b keepalived with TTL(300)
^C

# sky @ sky-HP in ~/data/etcdcluster [13:37:35] C:130
$ ./etcdctl --endpoints=$ENDPOINTS lease revoke 4097767e1dfcb20b
lease 4097767e1dfcb20b revoked

# sample已不存在.
# sky @ sky-HP in ~/data/etcdcluster [13:37:46] 
$ ./etcdctl --endpoints=$ENDPOINTS get sample

# Distributed lock操作.
# 终端1,执行lock操作.
# sky @ sky-HP in ~/data/etcdcluster [13:38:00] 
$ ./etcdctl --endpoints=$ENDPOINTS lock mutex
mutex/3a7e767e1dfcbd0e

# 终端2,会被阻塞.
# sky @ sky-HP in ~/data/etcdcluster [13:26:44] C:130
$ ./etcdctl --endpoints=$ENDPOINTS lock mutex

# 终端1,终止lock操作之后,终端2才能获取.
# sky @ sky-HP in ~/data/etcdcluster [13:26:44] C:130
$ ./etcdctl --endpoints=$ENDPOINTS lock mutex
mutex/3a7e767e1dfcbd11

# snapshot快照相关操作,只能针对单一节点操作.
# sky @ sky-HP in ~/data/etcdcluster [13:48:21] 
$ ENDPOINTS=192.168.3.5:2379

# sky @ sky-HP in ~/data/etcdcluster [13:48:33] 
$ ./etcdctl --endpoints=$ENDPOINTS snapshot save my.db
{"level":"info","ts":1608443327.4592857,"caller":"snapshot/v3_snapshot.go:67","msg":"created temporary db file","path":"my.db.part"}
{"level":"info","ts":"2020-12-20T13:48:47.459+0800","caller":"v3/maintenance.go:211","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1608443327.460071,"caller":"snapshot/v3_snapshot.go:75","msg":"fetching snapshot","endpoint":"192.168.3.5:2379"}
{"level":"info","ts":"2020-12-20T13:48:47.463+0800","caller":"v3/maintenance.go:219","msg":"completed snapshot read; closing"}
{"level":"info","ts":1608443327.4667196,"caller":"snapshot/v3_snapshot.go:90","msg":"fetched snapshot","endpoint":"192.168.3.5:2379","size":"25 kB","took":"now"}
{"level":"info","ts":1608443327.4670007,"caller":"snapshot/v3_snapshot.go:99","msg":"saved","path":"my.db"}
Snapshot saved at my.db

# sky @ sky-HP in ~/data/etcdcluster [13:49:52] 
$ ./etcdctl --endpoints=$ENDPOINTS --write-out=table snapshot status my.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 48b52149 |       45 |         55 |      25 kB |
+----------+----------+------------+------------+
```

### 基于Docker
使用docker-compose来统一部署,基于`quay.io/coreos/etcd:v3.4.14`镜像.
``` yml
version: "3.5"
services:
    etcd1:
        hostname: etcd1
        image: quay.io/coreos/etcd:v3.4.14
        container_name: etcd1
        restart: unless-stopped
        ports:
            - "2379:2379"
            - "2380:2380"
        volumes:
            - "./etcd1.data:/etcd_data"
        environment:
            - "ETCD_ADVERTISE_CLIENT_URLS=http://192.168.20.151:2379"
            - "ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379"
            - "ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380"
            - "ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd1:2380"
            - "ALLOW_NONE_AUTHENTICATION=yes"
            - "ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
            - "ETCD_NAME=etcd1"
            - "ETCD_DATA_DIR=/etcd_data"
            - "ETCD_INITIAL_CLUSTER_STATE=new"
            - "ETCD_INITIAL_CLUSTER_TOKEN=token-1"
            - "TZ=Asia/Shanghai"
        networks:
            - etcdcluster
    etcd2:
        hostname: etcd2
        image: quay.io/coreos/etcd:v3.4.14
        container_name: etcd2
        restart: unless-stopped
        ports:
            - "12379:2379"
            - "12380:2380"
        volumes:
            - "./etcd2.data:/etcd_data"
        environment:
            - "ETCD_ADVERTISE_CLIENT_URLS=http://192.168.20.151:12379"
            - "ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379"
            - "ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380"
            - "ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd2:2380"
            - "ALLOW_NONE_AUTHENTICATION=yes"
            - "ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
            - "ETCD_NAME=etcd2"
            - "ETCD_DATA_DIR=/etcd_data"
            - "ETCD_INITIAL_CLUSTER_STATE=new"
            - "ETCD_INITIAL_CLUSTER_TOKEN=token-1"
            - "TZ=Asia/Shanghai"
        networks:
            - etcdcluster
    etcd3:
        hostname: etcd3
        image: quay.io/coreos/etcd:v3.4.14
        container_name: etcd3
        restart: unless-stopped
        ports:
            - "22379:2379"
            - "22380:2380"
        volumes:
            - "./etcd3.data:/etcd_data"
        environment:
            - "ETCD_ADVERTISE_CLIENT_URLS=http://192.168.20.151:22379"
            - "ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379"
            - "ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380"
            - "ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd3:2380"
            - "ALLOW_NONE_AUTHENTICATION=yes"
            - "ETCD_INITIAL_CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
            - "ETCD_NAME=etcd3"
            - "ETCD_DATA_DIR=/etcd_data"
            - "ETCD_INITIAL_CLUSTER_STATE=new"
            - "ETCD_INITIAL_CLUSTER_TOKEN=token-1"
            - "TZ=Asia/Shanghai"
        networks:
            - etcdcluster

networks:
    etcdcluster:
        name: etcdcluster
```
需要注意环境变量`ETCD_ADVERTISE_CLIENT_URLS`,被配置成了真实IP+映射后的端口,在客户端获取`MemberList`时会返回该变量的值,是可在外部访问的地址.

客户端会定时去同步参数,代码详见`etcd/clientv3/client.go:autoSync`,是通过定时器触发,调用`MemberList`方法来获取成员列表,并取其`ClientURLs`字段,来初始化客户端连接.

查看集群状态
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS --write-out=table endpoint status
+----------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-------
-+|       ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS
 |+----------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-------
-+|  192.168.20.151:2379 | 7ff9976e030eb51c |  3.4.14 |   25 kB |     false |      false |         2 |         43 |                 43 |       
 || 192.168.20.151:12379 |  c8f871bdf8305be |  3.4.14 |   25 kB |     false |      false |         2 |         43 |                 43 |       
 || 192.168.20.151:22379 | b8dbeeea8e10767d |  3.4.14 |   25 kB |      true |      false |         2 |         43 |                 43 |       
 |+----------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-------
-+

$ ./etcdctl --endpoints=$ENDPOINTS --write-out=table member list
+------------------+---------+-------+-------------------+-----------------------------+------------+
|        ID        | STATUS  | NAME  |    PEER ADDRS     |        CLIENT ADDRS         | IS LEARNER |
+------------------+---------+-------+-------------------+-----------------------------+------------+
|  c8f871bdf8305be | started | etcd2 | http://etcd2:2380 | http://192.168.20.151:12379 |      false |
| 7ff9976e030eb51c | started | etcd1 | http://etcd1:2380 |  http://192.168.20.151:2379 |      false |
| b8dbeeea8e10767d | started | etcd3 | http://etcd3:2380 | http://192.168.20.151:22379 |      false |
+------------------+---------+-------+-------------------+-----------------------------+------------+
```

## 基于clientv3与etcd交互
需要注意,etcd和grpc版本兼容性可能有问题,当前使用的版本:
``` go
module etcdexample

go 1.14

require (
	github.com/gogo/protobuf v1.3.1 // indirect
	github.com/google/uuid v1.1.2 // indirect
	go.etcd.io/etcd v3.3.25+incompatible
	go.uber.org/zap v1.16.0 // indirect
	google.golang.org/grpc v1.34.0 // indirect
)

replace go.etcd.io/etcd v3.3.25+incompatible => go.etcd.io/etcd v0.0.0-20200402134248-51bdeb39e698

replace google.golang.org/grpc v1.34.0 => google.golang.org/grpc v1.29.1
```

### 基本增删查改等操作
``` go
unc main() {
	flag.Parse()
	endpoints := strings.Split(*addr, ",")

	// 创建etcd Client对象.
	cli, err := clientv3.New(
		clientv3.Config{
			Endpoints:        endpoints,
			RejectOldCluster: true,
		})
	if err != nil {
		fmt.Println(err)

		return
	}

	defer cli.Close()
	ctx := context.Background()

	// 设置租约为50秒.
	lease, err := cli.Grant(ctx, 50)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 使用租约来设置key1-value1.
	_, err = cli.Put(ctx, "key1", "value1", clientv3.WithLease(lease.ID))
	if err != nil {
		fmt.Println(err)

		return
	}

	// 使用前缀key,来获取数据.
	resp, err := cli.Get(ctx, "key", clientv3.WithPrefix())
	if err != nil {
		fmt.Println(err)

		return
	}

	// 打印etcd返回的key-value.
	for i := range resp.Kvs {
		fmt.Println("key:", string(resp.Kvs[i].Key), " value:", string(resp.Kvs[i].Value))
	}

	// 根据租约ID来查询租约当前详细信息.
	tresp, err := cli.TimeToLive(ctx, lease.ID)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 打印设置的ttl值和当前的ttl值.
	fmt.Println(tresp.GrantedTTL, tresp.TTL)

	// 根据租约ID来刷新当前的ttl,更新为所设置的ttl值.
	_, err = cli.KeepAliveOnce(ctx, lease.ID)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 修改.
	_, err = cli.Put(ctx, "key1", "value2", clientv3.WithLease(lease.ID))
	if err != nil {
		fmt.Println(err)

		return
	}

	// 删除key1,是精确匹配,也可采用前缀模式.
	_, err = cli.Delete(ctx, "key1")
	if err != nil {
		fmt.Println(err)

		return
    }
}
```

### Watch操作
主要是对感兴趣的Key进行监听,事件包括增删改.
``` go
wchan := cli.Watch(ctx, "key", clientv3.WithPrefix())
for resp := range wchan {
    err = resp.Err()
    if err != nil {
        fmt.Println(err)

        break
    }

    if resp.IsProgressNotify() {
        continue
    }

    for _, event := range resp.Events {
        if event.IsCreate() {
            fmt.Println("创建:", string(event.Kv.Key), string(event.Kv.Value), event.Type.String())
        } else if event.IsModify() {
            if event.PrevKv != nil {
                fmt.Println("修改前:", string(event.PrevKv.Key), string(event.PrevKv.Value))
            }

            fmt.Println("修改后:", string(event.Kv.Key), string(event.Kv.Value), event.Type.String())
        } else {
            fmt.Println("删除:", string(event.Kv.Key), string(event.Kv.Value), event.Type.String())
        }
    }
}
```
***注意:当修改时,`Event`对象中`PrevKv`字段的值为`nil`,而根据字段定义,该字段是保存事件发生之前的KV值,这里实际情况和字段定义不符.***

### 事务操作
etcd支持事务,提供了在一个事务中对多个key的更新功能,这一组key的操作要么全部成功,要么全部失败.是基于CAS方式来实现的.

主要提供的方法为:
``` go
// 创建一个事务对象.
Txn(ctx context.Context) Txn

type Txn interface {
	// If takes a list of comparison. If all comparisons passed in succeed,
	// the operations passed into Then() will be executed. Or the operations
    // passed into Else() will be executed.
    // 条件.
	If(cs ...Cmp) Txn

	// Then takes a list of operations. The Ops list will be executed, if the
    // comparisons passed in If() succeed.
    // 条件成功会执行.
	Then(ops ...Op) Txn

	// Else takes a list of operations. The Ops list will be executed, if the
    // comparisons passed in If() fail.
    // 条件失败会执行.
	Else(ops ...Op) Txn

    // Commit tries to commit the transaction.
    // 提交事务.
	Commit() (*TxnResponse, error)
}
```

具体调用代码如下:
``` go
func txnTransfer(cli *clientv3.Client, from, to string, amount int64) {
	ctx := context.Background()
	// 在一个事务中同时获取from和to的数据.
	resp, err := cli.Txn(ctx).Then(clientv3.OpGet(from), clientv3.OpGet(to)).Commit()
	if err != nil {
		fmt.Println(err)

		return
	}

	// 获取键为from的数据.
	fromKV := resp.Responses[0].GetResponseRange().Kvs[0]

	// 获取对应的值,转换为int64.
	fromAmount, err := strconv.ParseInt(string(fromKV.Value), 10, 64)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 获取键为to的数据.
	toKV := resp.Responses[1].GetResponseRange().Kvs[0]

	// 获取对应的值,转换为int64.
	toAmount, err := strconv.ParseInt(string(toKV.Value), 10, 64)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 判断from账户的钱是否充足.
	if fromAmount < amount {
		fmt.Println("money is not enough")

		return
	}

	// 开启事务,根据If中的来判断,若满足条件则可以执行Then里的操作,Then里的操作要么全部成功要么全部失败.
	putresp, err := cli.Txn(ctx).
		If(
			// 判断etcd中from的ModRevision是否和之前查询出来的相同.
			clientv3.Compare(clientv3.ModRevision(from), "=", fromKV.ModRevision),
			// 判断etcd中to的ModRevision是否和之前查询出来的相同.
			clientv3.Compare(clientv3.ModRevision(to), "=", toKV.ModRevision)).
		Then(
			// 把from对应的值减去amount.
			clientv3.OpPut(from, strconv.Itoa(int(fromAmount-amount))),
			// 把to对应的值加上amount.
			clientv3.OpPut(to, strconv.Itoa(int(toAmount+amount)))).
		Commit()
	if err != nil {
		fmt.Println(err)

		return
	}

	if putresp.Succeeded {
		fmt.Println("执行成功")
	} else {
		fmt.Println("执行失败")
	}
}
```

### 高级扩展操作
#### 分布式锁
主要提供的方法为:
``` go
// 初始化sync.Locker对象.
func NewLocker(s *Session, pfx string) sync.Locker

// 申请锁.
func (lm *lockerMutex) Lock()

// 释放锁.
func (lm *lockerMutex) Unlock()
```

具体调用代码如下:
``` go
func uselock(cli *clientv3.Client, name string) {
	// 创建Session对象.
	sess, err := concurrency.NewSession(cli)
	if err != nil {
		fmt.Println(err)

		return
	}

	defer sess.Close()

	// 根据Session来创建sync.Locker对象.
	locker := concurrency.NewLocker(sess, name)

	// 加锁.
	locker.Lock()

	// 模拟业务逻辑处理.
	time.Sleep(30 * time.Second)

	// 释放锁.
	locker.Unlock()
}
```
在两个终端上调用该函数时,第一个能顺利申请到锁,第二个会被阻塞直到第一个占用的锁被释放.

关注etcd中的数据.
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS get test-lock --prefix
test-lock/351c76848abb3131

test-lock/5be76848abafc1d
```

在调用`concurrency.NewSession`时,会设置ttl,默认为60秒,`Session`对象会持有对应的`LeaseID`,并会调用`KeepAlive`来续期,使得锁在`Unlock`之前一直是有效的,其它想抢占分布式锁的程序只能是等待.假如第一个终端里的程序在调用`Unlock`之前被提前终止了,所持有的锁实际并不会马上被释放,而是要等到租约到期,之后第二个终端的程序才能正常获取到分布式锁.

#### Mutex
主要提供的方法为:
``` go
// 初始化Mutex.
func NewMutex(s *Session, pfx string) *Mutex

// 尝试获取锁,若锁被占用获取失败会立即返回.
func (m *Mutex) TryLock(ctx context.Context) error

// 申请锁.
func (m *Mutex) Lock(ctx context.Context) error

// 释放锁.
func (m *Mutex) Unlock(ctx context.Context) error

// 是否是当前持有的锁.
func (m *Mutex) IsOwner() v3.Cmp

// 获取锁对应的key.
func (m *Mutex) Key() string

// 获取从etcd返回的数据的协议头.
func (m *Mutex) Header() *pb.ResponseHeader
```

具体调用代码如下:
``` go
func useMutex(cli *clientv3.Client, name string) {
	// 创建Session对象.
	sess, err := concurrency.NewSession(cli)
	if err != nil {
		fmt.Println(err)

		return
	}

	defer sess.Close()

	// 创建Mutex对象.
	m := concurrency.NewMutex(sess, name)
	ctx := context.Background()

	// 不是标准的sync.Locker接口,需要传入Context对象,在获取锁时可以设置超时时间,或主动取消请求.
	err = m.Lock(ctx)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 获取Mutex对象的key.
	key := m.Key()
	fmt.Println(key)

	// 模拟业务逻辑处理.
	time.Sleep(30 * time.Second)

	// 释放锁.
	err = m.Unlock(ctx)
	if err != nil {
		fmt.Println(err)
	}
}
```
`Mutex`提供的`TryLock`方法,会尝试获取锁,如果该锁没有被其它`Session`获取,则加锁成功;如果已经被占用,则立马返回并报错`mutex: Locked by another session`.

#### 读写锁
主要提供的方法为:
``` go
// 初始化读写锁对象.
func NewRWMutex(s *concurrency.Session, prefix string) *RWMutex

// 申请读锁.
func (rwm *RWMutex) RLock() error

// 申请写锁.
func (rwm *RWMutex) Lock()

// 释放读锁.
func (rwm *RWMutex) RUnlock() error

// 释放写锁.
func (rwm *RWMutex) Unlock() error
```

具体调用代码如下:
``` go
// 加读锁.
func useRWMutex(cli *clientv3.Client, name string) {
	sess, err := concurrency.NewSession(cli)
	if err != nil {
		fmt.Println(err)

		return
	}

	defer sess.Close()

	// 生成RWMutex读写锁对象.
	rw := recipe.NewRWMutex(sess, name)

	// 加读锁.
	err = rw.RLock()
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println("加读锁成功")

	// 模拟业务逻辑处理.
	time.Sleep(30 * time.Second)

	// 释放锁.
	err = rw.RUnlock()
	if err != nil {
		fmt.Println(err)
	}
}

// 加写锁.
func useRWMutexW(cli *clientv3.Client, name string) {
	time.Sleep(10 * time.Second)
	sess, err := concurrency.NewSession(cli)
	if err != nil {
		fmt.Println(err)

		return
	}

	defer sess.Close()

	// 生成RWMutex读写锁对象.
	rw := recipe.NewRWMutex(sess, name)

	// 加写锁.
	err = rw.Lock()
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println("加写锁成功")

	// 模拟业务逻辑处理.
	time.Sleep(30 * time.Second)

	// 释放锁.
	err = rw.Unlock()
	if err != nil {
		fmt.Println(err)
	}
}
```
同时启动3个goroutine,2个执行`useRWMutex`加读锁,1个执行`useRWMutexW`加写锁(先阻塞10秒,使读锁先加成功).

观察etcd的数据,总共有三个锁,两个读一个写,写被读阻塞.当读锁都被释放后,加写锁成功.
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS get test-lock --prefix
test-lock/read/1608689345032840000

test-lock/read/1608689345033840100

test-lock/write/1608689355033412000
```

***分布式读写锁在加锁时是写锁优先还是读锁优先?***

在等待锁的过程中是按照先进先出的原则来依次获取锁的.当前第一个Session获取到了读锁,第二个Session申请获取写锁被阻塞,第三个Session申请获取读锁也会被阻塞,第四个Session申请获取写锁也被阻塞.当第一个Session释放了读锁后,第二个Session获取到写锁,第三个和第四个Session仍然处于被阻塞状态.当第二个Session释放了写锁后,第三个Session获取到读锁,第四个Session仍然处于被阻塞状态.当第三个Session释放读锁后,第四个Session获取到写锁.

#### 分布式队列
**分布式队列(支持多读多写)**

先进先出,主要提供的方法为:
``` go
// 初始化队列对象.
func NewQueue(client *v3.Client, keyPrefix string) *Queue

// 入队.
func (q *Queue) Enqueue(val string) error

// 出队.
func (q *Queue) Dequeue() (string, error)
```

具体调用代码:
``` go
// 入队.
func useQueueEn(cli *clientv3.Client, name string) {
	// 创建分布式队列.
	q := recipe.NewQueue(cli, name)

	// 入队.
	err := q.Enqueue("key1")
	if err != nil {
		fmt.Println(err)

		return
	}

	// 入队.
	err = q.Enqueue("en2")
	if err != nil {
		fmt.Println(err)
	}
}

// 出队.
func useQueueDe(cli *clientv3.Client, name string) {
	// 创建队列.
	q := recipe.NewQueue(cli, name)

	// 出队,注意当队列为空时,该操作会被阻塞.
	key, err := q.Dequeue()
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println(key)
}
```
当执行完入队操作之后,观察etcd中的数据如下,也是采用的KV方式存储的,key的前缀为队列名,后缀为当前的时间戳(纳秒级,`time.Now().UnixNano()`),注意队列中的数据并没有设置租约.
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS get test_queue --prefix
test_queue/1608692476200480900
key1
test_queue/1608692476203481100
en2
```

当执行出队操作之后,etcd中数据如下,第一个key被删除.
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS get test_queue --prefix
test_queue/1608692476203481100
en2
```

**优先级队列(支持多读多写)**

按照优先级先进先出,主要提供的方法为:
``` go
// 初始化优先级队列对象.
func NewPriorityQueue(client *v3.Client, key string) *PriorityQueue

// 入队,并设置优先级.
func (q *PriorityQueue) Enqueue(val string, pr uint16) error

// 出队.
func (q *PriorityQueue) Dequeue() (string, error)
```

具体调用代码:
``` go
func usePriorityQueueEn(cli *clientv3.Client, name string) {
	// 创建优先级队列.
	q := recipe.NewPriorityQueue(cli, name)

	// 入队.
	err := q.Enqueue("key1", 10)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 入队.
	err = q.Enqueue("other1", 10)
	if err != nil {
		fmt.Println(err)

		return
	}

	// 入队.
	err = q.Enqueue("en2", 5)
	if err != nil {
		fmt.Println(err)
	}
}

func usePriorityQueueDe(cli *clientv3.Client, name string) {
	// 创建优先级队列.
	q := recipe.NewPriorityQueue(cli, name)

	// 出队,注意:当队列为空时,出队操作会被阻塞.
	value, err := q.Dequeue()
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println(value)
}
```
在执行入队操作之后,观察etcd数据如下,key的规则是:队列名称+优先级(5位)+序列号(16位),当优先级相同时序列号会从0开始递增.数字越小优先级越高.
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS get test_queue --prefix
test_queue/00005/0000000000000000
en2
test_queue/00010/0000000000000000
key1
test_queue/00010/0000000000000001
other1
```

#### 分布式栅栏
**Barrier:分布式栅栏**

主要提供的方法为:
``` go
// 初始化Barrier对象.
func NewBarrier(client *v3.Client, key string) *Barrier

// 在etcd中创建键为key值为""的数据对,会导致其它节点阻塞在Wait操作上.
func (b *Barrier) Hold() error

// 在etcd中删除数据对,会导致其它节点解除阻塞状态.
func (b *Barrier) Release() error

// key已被创建,则会阻塞直到key被删除;若key未创建,立即返回.
func (b *Barrier) Wait() error
```

适用场景:一组节点协同工作,等待同一个信号,在信号未出现前,该组节点会被阻塞,而一旦信号出现,该组节点就会同时开始继续执行下一步的任务.如果持有Barrier的节点释放了它,所有等待这个Barrier的节点就不会被阻塞,而是继续执行.
``` go
func useBarrier(cli *clientv3.Client, name string) {
	// 创建Barrier对象.
	b := recipe.NewBarrier(cli, name)

	// 创建以name为key,其值为""的KV对保存在etcd中,其上并没有设置租约.
	err := b.Hold()
	if err != nil {
		fmt.Println(err)

		return
	}

	// 模拟业务逻辑.
	time.Sleep(15 * time.Second)

	// 删除key为name的KV对.
	err = b.Release()
	if err != nil {
		fmt.Println(err)

		return
	}
}

func useBarrierWait(cli *clientv3.Client, name string) {
	time.Sleep(4 * time.Second)
	// 创建Barrier对象.
	b := recipe.NewBarrier(cli, name)
	fmt.Println("开启wait")

	// 若etcd中存在key为name的KV对,则会阻塞,并针对该key进行watch,当key被删除时会解除阻塞,继续往下执行.
	err := b.Wait()
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println("结束wait")
}
```
注意:当调用`Hold`的节点在`Release`之前由于异常原因挂掉,会导致`Wait`的节点一直会处于阻塞状态.这是由于KV对上并没有设置租约,如果没有主动调用`Release`,该KV对会一直存在.

在调用`Hold`之后且调用`Release`之前,观察etcd数据.在调用`Release`之后会被删除.
``` bash
$ ./etcdctl --endpoints=$ENDPOINTS get test_barrier --prefix
test_barrier

```

**DoubleBarrier:计数型栅栏**

主要提供的方法为:
``` go
// 初始化DoubleBarrier对象.
func NewDoubleBarrier(s *concurrency.Session, key string, count int) *DoubleBarrier

// 阻塞,直到有count个进程或goroutine持有该barrier时,才会返回.
func (b *DoubleBarrier) Enter() error

// 阻塞,直到有count个进程或goroutine释放该barrier时,才会返回.
func (b *DoubleBarrier) Leave() error
```
在初始化计数型栅栏时,必须提供参与节点的数量,当这些数量的节点都Enter或Leave时,这个栅栏就会放开.

适用场景:需要一组N个节点同时开始执行某些任务(等待N个节点都调用`Enter`后,同时开始执行任务),并且这些任务要同时完成(等待N个节点都调用`Leave`后,同时结束任务),然后再继续往下执行.
``` go
func useDoubleBarrier(cli *clientv3.Client, name string) {
	// 创建Session.
	sess, err := concurrency.NewSession(cli)
	if err != nil {
		fmt.Println(err)

		return
	}

	defer sess.Close()
	// 初始化DoubleBarrier对象.
	db := recipe.NewDoubleBarrier(sess, name, 3)

	// 模拟不同goroutine在不同的时间点调用Enter.
	time.Sleep(time.Duration(rand.Intn(60)) * time.Second)
	// 等待,直到有3个goroutine调用Enter为止.
	err = db.Enter()
	if err != nil {
		fmt.Println(err)

		return
	}

	// 同时开始执行任务.
	fmt.Println("开始工作")

	// 模拟任务的执行,所耗时各不相同.
	time.Sleep(time.Duration(rand.Intn(60)) * time.Second)

	// 等待,直到有3个goroutine调用Leave为止.
	err = db.Leave()
	// 任务同时执行完成,继续往下.
	if err != nil {
		fmt.Println(err)

		return
	}

	fmt.Println("工作结束")
}
```

#### STM(Soft Transactional Memory)
主要提供的方法为:
``` go
// 开启一个事务,执行apply定义的操作.
func NewSTM(c *v3.Client, apply func(STM) error, so ...stmOption) (*v3.TxnResponse, error)

// STM is an interface for software transactional memory.
type STM interface {
	// Get returns the value for a key and inserts the key in the txn's read set.
	// If Get fails, it aborts the transaction with an error, never returning.
	Get(key ...string) string
	// Put adds a value for a key to the write set.
	Put(key, val string, opts ...v3.OpOption)
	// Rev returns the revision of a key in the read set.
	Rev(key string) int64
	// Del deletes a key.
	Del(key string)

	// commit attempts to apply the txn's changes to the server.
	commit() *v3.TxnResponse
	reset()
}
```

具体调用代码:
``` go
func useSTM(cli *clientv3.Client, from, to string, amount int64) {
	// 定义操作函数,里面逻辑相当于是原子执行.
	apply := func(stm concurrency.STM) error {
		fromV, toV := stm.Get(from), stm.Get(to)
		fromInt, _ := strconv.ParseInt(fromV, 10, 64)
		toInt, _ := strconv.ParseInt(toV, 10, 64)
		if fromInt < amount {
			return errors.New("money is not enough")
		}

		fromInt, toInt = fromInt-amount, toInt+amount

		stm.Put(from, fmt.Sprintf("%d", fromInt))
		stm.Put(to, fmt.Sprintf("%d", toInt))

		return nil
	}

	// 开启事务,执行transfer中定义的业务逻辑.
	resp, err := concurrency.NewSTM(cli, apply)
	if err != nil {
		fmt.Println(err)

		return
	}

	if resp.Succeeded {
		fmt.Println("操作成功")
	} else {
		fmt.Println("操作失败")
	}
}
```

