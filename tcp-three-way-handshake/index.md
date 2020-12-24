# tcp连接过程


## tcp建连接三次握手
* 客户端发送SYN到服务器发起握手.
* 服务器收到SYN后回复SYN+ACK给客户端.
* 客户端收到SYN+ACK后,回复服务器一个ACK表示收到了,此时客户端的端口状态已经是established.
* tcp握手的详细过程,[图片来源](http://www.cnxct.com/something-about-phpfpm-s-backlog/)
![tcp建立连接的流程和队列](/images/tcp_accept_queue.png "tcp连接的流程")

客户端作为主动发起连接方,首先要发送SYN(Synchronize Sequence Nubers,同步序列号)包,若客户端长时间收不到服务端的ACK报文,客户端就会重发SYN包,重传次数是受内核参数`/proc/sys/net/ipv4/tcp_syn_retries`控制.
```
# 系统为centos7.2.1511,默认为6次.
$ cat /proc/sys/net/ipv4/tcp_syn_retries 
6
```
通常第一次超时重传为1秒,第二次超时重传为2秒,第三次超时重传为4秒,第四次为8秒,第五次为16秒,第五次超时重传之后还会再等待32秒,如果服务端仍然没有回应ACK,客户端就会终止三次握手.总耗时为63秒.

## 半连接队列
syns queue就是半连接队列,server收到client的syn后会把连接信息放入该队列.

半连接队列的大小为max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog).
```
# 在ubuntu18.04机器上为128.
$ cat /proc/sys/net/ipv4/tcp_max_syn_backlog 
128
```

syn floods攻击就是针对半连接队列的,攻击方不停的建立连接,收到server的syn+ack就丢弃什么也不做,导致server的半连接队列满而其它正常连接无法进来.

## 全连接队列
* accept queue就是全连接队列,server再收到client的ack后会把连接信息放入该队列.
* 全连接队列的大小为min(backlog, /proc/sys/net/core/somaxconn).
* backlog是指`listen(int sockfd, int backlog)`函数中的backlog大小.
```
# 在ubuntu18.04机器上为128.
$ cat /proc/sys/net/core/somaxconn
128
```

## 如何观察队列溢出
**netstat -s**
```
# 查看半连接队列溢出.
sky-HP# netstat -s | egrep "SYNs to LISTEN"
667399 SYNs to LISTEN sockets ignored
# 上面看到的667399就是半连接队列溢出次数,隔几秒执行下,如果这个数字一直在变大肯定就是半连接队列溢出了.

# 查看全连接队列溢出.
sky-HP# netstat -s | grep "overflowed"
667399 times the listen queue of a socket overflowed
```

**ss -lntp**
```
# l表示处于LISTEN状态 n表示不反向解析 t表示tcp协议 p表示进程信息.
sky-HP# ss -lntp
State       Recv-Q       Send-Q              Local Address:Port              Peer Address:Port                                                        
LISTEN      0            128                       0.0.0.0:8388                   0.0.0.0:*           users:(("haproxy",pid=1465,fd=6))               
LISTEN      0            128                 127.0.0.53%lo:53                     0.0.0.0:*           users:(("systemd-resolve",pid=841,fd=13))       
LISTEN      0            128                       0.0.0.0:22                     0.0.0.0:*           users:(("sshd",pid=1476,fd=3))                  
LISTEN      0            5                       127.0.0.1:631                    0.0.0.0:*           users:(("cupsd",pid=5112,fd=7))                 
LISTEN      0            128                     127.0.0.1:1080                   0.0.0.0:*           users:(("trojan",pid=1940,fd=6))                
LISTEN      0            128                     127.0.0.1:8123                   0.0.0.0:*           users:(("polipo",pid=1461,fd=4))                
```
Send-Q就是表示全连接队列的允许最大长度,Recv-Q表示当前全连接队列的长度.这是套接字处于LISTEN状态时.
需要注意,当套接字处于Established状态时Recv-Q表示套接字缓冲区还没有被应用取走的字节数(接收队列长度),Send-Q表示还没有被远端主机确认的字节数(发送队列长度).

## 溢出行为的控制
**半连接队列控制**
当半连接队列满时,只能丢弃连接?

并不是这样的,Linux提供了syncookies功能,可以在不适用半连接队列的情况下成功建立连接.
syncookies原理:服务器会根据当前的状态计算出一个值,放入己方的SYN+ACK报文中发送给客户端,当客户端返回ACK报文时,取出该值验证,如果合法,就认为连接建立成功.

syncookies功能由内核参数`/proc/sys/net/ipv4/tcp_syncookies`来控制.
* 值为0时,表示关闭该功能.
* 值为1时,表示仅当半连接队列满时,再启用该功能.1为默认值,默认开启.
* 值为2时,表示无条件开启该功能.

**全连接队列控制**
内核参数`/proc/sys/net/ipv4/tcp_abort_on_overflow`决定当溢出后系统如何处理.
* 为0时表示server扔掉client发过来的ack.server会认为连接还未建立.server过段时间会继续向client发送syn+ack.重传会经历1、2、4、8、16、32秒(若重传为5次),如果服务端仍没有收到ack,才会关闭连接,总共需63秒.
  - 内核参数`/proc/sys/net/ipv4/tcp_synack_retries`控制重试次数.如果client超时时间比较短,client就容易异常.
* 为1时表示server发送一个reset包给client,表示废掉这个握手过程和连接.client会看到connection reset by peer的错误.
```
# 在ubuntu18.04机器上的默认值.
sky-HP# cat /proc/sys/net/ipv4/tcp_abort_on_overflow 
0
sky-HP# cat /proc/sys/net/ipv4/tcp_synack_retries 
5
```

可以通过`sysctl -w`来修改这些内核参数,重启之后修改无效.
```
sky-HP# sysctl -w net.ipv4.tcp_synack_retries=2
net.ipv4.tcp_synack_retries = 2
sky-HP# cat /proc/sys/net/ipv4/tcp_synack_retries 
2

sky-HP# sysctl -w net.ipv4.tcp_abort_on_overflow=1
net.ipv4.tcp_abort_on_overflow = 1
sky-HP# cat /proc/sys/net/ipv4/tcp_abort_on_overflow 
1
```

修改配置文件`/etc/sysctl.conf`,然后再`sysctl -p`来触发,重启之后仍生效.

## 绕过三次握手
三次握手建立连接的后果就是,数据请求必须在一个RTT(从客户端到服务器一个往返时间)后才能发送.

在Linux3.7内核版本之后,提供了TCP Fast Open功能,可以减少TCP连接建立的时延.
![Fast Open](/images/fast.jpg "Linux TCP Fast Open")

客户端首次建立连接时仍然需要三次握手.
1. 客户端发送SYN报文,报文包含Fast Open选项,且该选项的Cookie为空,表名客户端请求Fast Open Cookie.
2. 支持TCP Fast Open的服务器生成Cookie,并置于SYN-ACK数据包中的Fast Open选项发回给客户端.
3. 客户端收到SYN-ACK后,本地缓存Fast Open选项中的Cookie.

当客户端再次与服务器建立连接时就可以利用Cookie来绕过三次握手过程.
1. 客户端发送SYN报文,该报文包含之前缓存的Cookie及业务数据报文.
2. 支持TCP Fast Open的服务器会对收到的Cookie进行校验.
   - 如果合法,服务器将在SYN-ACK报文中对SYN和业务数据进行确认,服务器随后把业务数据传递给应用程序;
   - 如果不合法,服务器将丢弃SYN报文中包含的业务数据,且在SYN-ACK报文中只确认SYN的序列号.
3. 若服务器接受了SYN报文中的业务数据,即在握手完成之前发送了数据,这就减少了握手带来的一个RTT的时间消耗.
4. 客户端将发送ACK确认服务器发回的SYN及数据.若客户端在初始的SYN报文中的数据未被确认,则客户端会重新发送这些数据.
5. 此后的TCP连接的数据传输过程和非TCP Fast Open的正常情况是一致的.

TCP Fast Open功能受内核参数`/proc/sys/net/ipv4/tcp_fastopen`控制.
* 值为0时,表示关闭该功能.
* 值为1时,作为客户端使用Fast Open功能.
* 值为2时,作为服务端使用Fast Open功能.
* 值为3时,作为客户端和服务端都可以使用Fast Open功能.

## 参考文章
* [TCP 三次握手原理，你真的理解吗？](https://mp.weixin.qq.com/s/yH3PzGEFopbpA-jw4MythQ)
* [关于 Linux 网络，你必须知道这些](https://time.geekbang.org/column/article/81057)
* [看完这篇，再不懂TCP我也没办法了](https://www.cnblogs.com/otis/p/13070877.html)
* [TCP SOCKET中backlog参数的用途是什么？](https://www.cnxct.com/something-about-phpfpm-s-backlog/)

