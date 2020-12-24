# go性能分析


## 性能分析
Go语言项目中的性能分析主要有以下几个方面:
* CPU profile: CPU使用情况,按照一定频率去采集应用程序在CPU和寄存器上面的数据.
* Memory profile(Heap profile): 报告程序的内存使用情况.
* Block Profiling: 报告goroutines不在运行状态的情况,可用来分析和查找死锁等性能瓶颈.
* Goroutine Profiling: 报告goroutines的使用情况,有哪些goroutines,调用关系是怎么样的?

## 数据采集
Go语言内置了获取程序运行数据的工具,包括两个标准库:
1. `runtime/pprof`: 采集工具型应用的运行数据进行分析
``` go
import "runtime/pprof"

// 开启CPU性能分析.
pprof.StartCPUProfile(w io.Writer)

// 关闭CPU性能分析.
pprof.StopCPUProfile()

// 记录程序堆栈信息.
pprof.WriteHeapProfile(w io.Writer)
```
pprof开启后,每隔一段时间(10ms)就会收集下当前的堆栈信息,获取各个函数占用的CPU以及内存资源,最后通过采样数据分析,形成性能分析报告.

2. `net/http/pprof`: 采集服务型应用的运行数据进行分析
``` go
// 在web server端导入pprof库.
import _ "net/http/pprof"

// 如果使用自定义Mux,需要手动注册路由规则.
r.HandleFunc("/debug/pprof", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)

// 如果使用gin框架,推荐使用'github.com/DeanThompson/ginpprof'
```
http服务会多出`/debug/pprof`的endpoint:
* /debug/pprof/profile: 访问这个链接会自动进行CPU Profiling,持续30s,并生成文件供下载.
* /debug/pprof/heap: Memory Profiling.
* /debug/pprof/block: Block Profiling.
* /debug/pprof/goroutines: 运行的goroutines列表以及调用关系.

3. profiling数据是动态的,要想获得有效的数据,请保证应用处于较大的负载,否则如果处于空闲状态,得到的结果可能没有任何意义

## 数据分析
### go tool pprof命令
可以通过命令`go tool pprof --help`查看命令的具体使用方法.
``` 
$ go tool pprof --help
usage:

Produce output in the specified format.

   pprof <format> [options] [binary] <source> ...

Omit the format to get an interactive shell whose commands can be used
to generate various views of a profile

   pprof [options] [binary] <source> ...

Omit the format and provide the "-http" flag to get an interactive web
interface at the specified host:port that can be used to navigate through
various views of a profile.

   pprof -http [host]:[port] [options] [binary] <source> ...

Details:
  Output formats (select at most one):
```

### 图形化
1. 安装[graphviz](https://graphviz.gitlab.io/),windows是还需要把安装目录下的`bin`文件夹添加到`PATH`环境变量中.
2. 使用`dot -version`命令查看graphviz安装是否成功.
3. 安装go-torch,使用`go get -v github.com/uber/go-torch`命令安装.
    当`go-torch`不带任何参数时,会默认从`http://localhost:8080/debug/pprof/profile`获取profiling数据.
    ``` 
    $ go-torch --help
    Usage:
    go-torch [options] [binary] <profile source>

    pprof Options:
    -u, --url=         Base URL of your Go program (default: http://localhost:8080)
        --suffix=      URL path of pprof profile (default: /debug/pprof/profile)
    -b, --binaryinput= File path of previously saved binary profile. (binary profile is anything accepted by https://golang.org/cmd/pprof)
        --binaryname=  File path of the binary that the binaryinput is for, used for pprof inputs
    -t, --seconds=     Number of seconds to profile for (default: 30)
        --pprofArgs=   Extra arguments for pprof
    ```
4. 安装[perl](https://www.perl.org/get.html),FlameGraph需要perl支持.
5. 安装FlameGraph,使用`git clone https://github.com/brendangregg/FlameGraph.git`命令安装.
    windows平台下,需要把`go-torch/render/flamegraph.go`文件中的`GenerateFlameGraph`按如下方式修改,然后在`go-torch`目录下执行`go install`命令.
    ``` go
    // GenerateFlameGraph runs the flamegraph script to generate a flame graph SVG. func GenerateFlameGraph(graphInput []byte, args ...string) ([]byte, error) {
    flameGraph := findInPath(flameGraphScripts)
    if flameGraph == "" {
        return nil, errNoPerlScript
    }
    if runtime.GOOS == "windows" {
        return runScript("perl", append([]string{flameGraph}, args...), graphInput)
    }
    return runScript(flameGraph, args, graphInput)
    }
    ```
6. 安装go-wrk,使用`go get -v https://github.com/adjust/go-wrk`命令安装.
    ``` 
    $ go-wrk --help
    Usage of go-wrk:
    -CA string
            A PEM eoncoded CA's certificate file. (default "someCertCAFile")
    -H string
            the http headers sent separated by '\n' (default "User-Agent: go-wrk 0.1 benchmark\nContent-Type: text/html;")
    -b string
            the http request body
    -c int
            the max numbers of connections used (default 100)
    -cert string
            A PEM eoncoded certificate file. (default "someCertFile")
    -d string
            dist mode
    -f string
            json config file
    -i	TLS checks are disabled
    -k	if keep-alives are disabled (default true)
    -key string
            A PEM encoded private key file. (default "someKeyFile")
    -m string
            the http request method (default "GET")
    -n int
            the total number of calls processed (default 1000)
    -p string
            the http request body data file
    -r	in the case of having stream or file in the response,
            it reads all response body to calculate the response size
    -s string
            if specified, it counts how often the searched string s is contained in the responses
    -t int
            the numbers of threads used (default 1)
    ```
7. 使用方式
    * 使用go-wrk压测,使用命令`go-wrk -n 50000 http://127.0.0.1:8080/*/*`在某个接口进行压测
    * 使用go-torch收集数据,使用命令`go-torch -u http://127.0.0.1:8080 -t 30`,30秒之后终端会出现如下提示: `Writing svg to torch.svg`,然后使用浏览器打开`torch.svg`,就能看到火焰图.

8. perf
    * Linux下使用命令`perf record -a -g -p pid -- sleep 30`,对指定进程采样30秒.
    * 使用命令`perf script -i ../perf.data | ./stackcollapse-perf.pl --all | ./flamegraph.pl > app.svg`,其中`../perf.data`为`perf record`生成的采样数据,然后切换到FlameGraph的目录来执行上述命令,就能得到火焰图了(`stackcollapse-perf.pl`脚本是合并调用栈信息,`flamegraph.pl`脚本是生成火焰图).

## 参考资料
* [Go pprof性能调优](https://www.cnblogs.com/Dr-wei/p/11742414.html).


