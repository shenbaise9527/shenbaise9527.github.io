# Golang Private Mod


## 前言
&ensp;&ensp;&ensp;&ensp;目前使用go的工程越来越多,每个工程都有很多重复的逻辑,需要把重复的进行提炼统一起来,就在bitbucket上新建了一个新的仓库`frame/goframe.git`,后续所有通用的基础组件都会放在`goframe`库中,由于库是私有的,就需要支持私有的`go mod`.

&ensp;&ensp;&ensp;&ensp;下面主要说明为了支持go工程能引用私有的`go mod`,而需要做的配置.

## 工作环境设置(个人)

### DNS设置
&ensp;&ensp;&ensp;&ensp;内网DNS地址: `172.16.30.243`, 内网域名: `go.goingint.io`

#### windows
![windows](/images/golang-private-windows.JPG "windows设置内网dns")

#### linux
&ensp;&ensp;&ensp;&ensp;编辑文件`/etc/resolv.conf`,新增一行`nameserver 172.16.30.243`
![linux](/images/golang-private-linux.JPG "linux设置内网dns")

#### 测试
&ensp;&ensp;&ensp;&ensp;可以使用命令`ping go.goingint.io`来测试dns是否生效,如果能ping通,就表示已经生效了
![ping](/images/golang-private-ping.JPG "测试dns是否生效")

### 配置git

#### 生成密钥
&ensp;&ensp;&ensp;&ensp;使用`ssh-keygen`工具来生成公私钥,使用命令`ssh-keygen -t rsa -C "steven.zhou@1quant.com" -f private`,如图:
![keygen](/images/golang-private-git.JPG "生成密钥")
注意:
1. `-C "steven.zhou@1quant.com"`换成自己的邮箱
2. `-f private`,`private`是文件名,生成的文件要放在`~/.ssh`目录下(***windows下目录为C:\Users\Admin\\.ssh***)

#### 配置多个SSH key
&ensp;&ensp;&ensp;&ensp;在目录`~/.ssh`下(***windows下目录为C:\Users\Admin\\.ssh***),生成文件`config`,注意文件名必须为`config`.

文件内容如下:
![config](/images/golang-private-config.JPG "config文件内容")

注意: 
如果在执行命令`git pull`时报错,信息如下: `Bad owner or permissions on /home/admin/.ssh/config`.

解决方案: 修改`~/.ssh/config`的权限,运行命令`chmod 0600 ~/.ssh/config`

最终目录`~/.ssh`下的文件情况:
![ssh](/images/golang-private-ssh.JPG "ssh目录下的文件")

注意文件`id_rsa`和`id_rsa.pub`是之前配置bitbucket用的.

#### 配置bitbucket
&ensp;&ensp;&ensp;&ensp;查看文件`private.pub`文件的内容:
![pub](/images/golang-private-pub.JPG "公钥内容")

把以上内容粘贴到bitbucket中,如下图:
![bitbucket](/images/golang-private-bitbucket.png "bitbucket中添加公钥")

#### 配置git config
&ensp;&ensp;&ensp;&ensp;使用如下命令: `git config --global url."git@go.goingint.io:".insteadOf "http://go.goingint.io/"`

### 配置golang环境变量
&ensp;&ensp;&ensp;&ensp;需要设置私有仓库,如下:
``` bash
go env -w GOINSECURE="go.goingint.io"
go env -w GOPRIVATE="go.goingint.io"
`````

## 后端相关设置
### 域名配置
&ensp;&ensp;&ensp;&ensp;由于内网的bitbucket的地址是`172.16.30.215:7999`,域名`go.goingint.io`实际指向的nginx,然后通过nginx转发到bitbucket.

nginx就需要支持转发ssh,如下配置:
``` nginx
stream {
    upstream ssh {
        server  172.16.30.215:7999;
    }

    server {
        listen       22;
        proxy_pass ssh;
        proxy_connect_timeout 1h;
        proxy_timeout 1h;
    }
}
```

在使用命令`go mod tidy`或`go mod download`下载依赖时,会调用`GET /frame/goframe?go-get=1`(可查看源码,`src/cmd/go/internal/vcs/vcs.go:urlForImportPath`),需要配置http,如下:
``` nginx
server {
        listen       80;
        server_name  go.goingint.io;
        if ($args ~* "^go-get=1") {
            set $condition goget;
        }

        if ($uri ~ ^/([a-zA-Z0-9_-]+)/([a-zA-Z0-9_-]+)$) {
            set $condition "${condition}path";
        }

        if ($condition = gogetpath) {
            return 200 "<!DOCTYPE html><html><head><meta content='go.goingint.io/$1/$2 git http://go.goingint.io/$1/$2.git' name='go-import'></head></html>";
        }

        location / {
            proxy_pass http://172.16.30.215:7999/;
        }
}
```

在nginx中对请求`/frame/goframe?go-get=1`直接进行拦截.

注意: 配置文件中的正则表达式可根据需要调整,还要把`go.goingint.io`换成自己的域名.

### goframe库设置
&ensp;&ensp;&ensp;&ensp;由于库的路径为`/frame/goframe`,在初始化时需要使用命令: `go mod init go.goingint.io/frame/goframe`,`go.mod`内容:
``` go
module go.goingint.io/frame/goframe

go 1.16
```

## 引用private mod
&ensp;&ensp;&ensp;&ensp;新建测试工程,`go mod init example`,然后新建`main.go`文件:
``` go
package main

import (
    "fmt"
    "go.goingint.io/frame/goframe/core/mq"
)

func main() {
    bus, err := mq.NewRabbitMQ("test", mq.WithRabbitMQUrl("amqp://guest:guest@172.16.60.12:45672/quote"))
    if err != nil {
        fmt.Printf("mq init err: %s\n", err.Error())

        return
    }

    fmt.Print("mq init successfully\n")
    defer bus.Close()
}
```

然后使用`go mod tidy`引入goframe私有包.


