---
title: 【译】Go优雅重启
date: 2021-01-11 11:08:07
categories:
- go
tags:
- go
- endless
---

**原文 [Graceful Restart in Golang](https://grisha.org/blog/2014/06/03/graceful-restart-in-golang/)**

**作者 [grisha](https://grisha.org/)**

**声明：本文目的仅仅作为个人mark，所以在翻译的过程中参杂了自己的思想甚至改变了部分内容，其中有下划线的文字为译者添加。但由于译者水平有限，所写文字或者代码可能会误导读者，如发现文章有问题，请尽快告知，不胜感激。**

***

### 前言
Update (Apr 2015): [Florian von Bock](https://github.com/fvbock)已经根据本文实现了一个叫做[endless](https://github.com/fvbock/endless)的Go package

大家知道，当我们用Go写的web服务器需要修改配置或者需要升级代码的时候我们需要重启服务器，如果你（像我一样）已经将优雅的重启视为理所当然，因为使用Golang你需要自己动手来做这些操作，所以你可能会发现这个方式非常方便。

### 什么是优雅重启
本文中的优雅重启表现为两点
1. 进程在不关闭其所监听的端口的情况下重启
2. 重启过程中保证所有请求能被正确的处理

#### 1.进程在不关闭其所监听的端口的情况下重启
- fork一个子进程，该子进程继承了父进程所监听的socket
- 子进程执行初始化等操作，并最终开始接收该socket的请求
- 父进程停止接收请求并等待当前处理的请求终止

##### fork一个子进程
有不止一种方法fork一个子进程，但在这种情况下推荐[exec.Command](https://golang.org/pkg/os/exec/#Command)，因为[Cmd](https://golang.org/pkg/os/exec/#Cmd)结构提供了一个字段`ExtraFiles`，该字段(注意不支持windows)为子进程额外地指定了需要继承的额外的文件描述符，不包含`std_in, std_out, std_err`。
需要注意的是，`ExtraFiles`描述中有这样一句话：
> If non-nil, entry i becomes file descriptor 3+i

这句是说，索引位置为i的文件描述符传过去，最终会变为值为i+3的文件描述符。ie: 索引为0的文件描述符565, 最终变为文件描述符3

```go
file := netListener.File() // this returns a Dup()
path := "/path/to/executable"
args := []string{
    "-graceful"}

cmd := exec.Command(path, args...)
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
cmd.ExtraFiles = []*os.File{file}

err := cmd.Start()
if err != nil {
    log.Fatalf("gracefulRestart: Failed to launch, error: %v", err)
}
```
上面的代码中，`netListener`是一个[net.Listener](https://golang.org/pkg/net/#Listener)类型的指针，path变量则是我们要更新的新的可执行文件的路径。

需要注意的是：上面`netListener.File()`与[dup](http://pubs.opengroup.org/onlinepubs/009695399/functions/dup.html)函数类似，返回的是一个拷贝的文件描述符。另外，该文件描述符不应该设置[FD_CLOEXEC](http://pubs.opengroup.org/onlinepubs/009695399/functions/fcntl.html)标识，这将会导致出现我们不想要的结果：子进程的该文件描述符被关闭。

你可能会想到可以使用命令行参数把该文件描述符的值传递给子进程，但相对来说，我使用的这种方式更为简单

最终，`args`数组包含了一个`-graceful`选项，你的进程需要以某种方式通知子进程要复用父进程的描述符而不是新打开一个。

##### 子进程初始化
```go
server := &http.Server{Addr: "0.0.0.0:8888"}

var gracefulChild bool
var l net.Listever
var err error

flag.BoolVar(&gracefulChild, "graceful", false, "listen on fd open 3 (internal use only)")

if gracefulChild {
    log.Print("main: Listening to existing file descriptor 3.")
    f := os.NewFile(3, "")
    l, err = net.FileListener(f)
} else {
    log.Print("main: Listening on a new file descriptor.")
    l, err = net.Listen("tcp", server.Addr)
}
```
##### 通知父进程停止
```go
if gracefulChild {
    parent := syscall.Getppid()
    log.Printf("main: Killing parent pid: %v", parent)
    syscall.Kill(parent, syscall.SIGTERM)
}

server.Serve(l)
```

##### 父进程停止接收请求并等待当前所处理的所有请求结束
为了做到这一点我们需要使用[sync.WaitGroup](https://golang.org/pkg/sync/#WaitGroup)来保证对当前打开的连接的追踪，基本上就是：每当接收一个新的请求时，给wait group做原子性加法，当请求结束时给wait group做原子性减法。也就是说wait group存储了当前正在处理的请求的数量
```go
var httpWg sync.WaitGroup
```

匆匆一瞥，我发现go中的http标准库并没有为Accept()和Close()提供钩子函数，但这就到了`interface`展现其魔力的时候了(非常感谢[ Jeff R. Allen](http://nella.org/jra/)的这篇[文章](http://blog.nella.org/zero-downtime-upgrades-of-tcp-servers-in-go/))

下面是一个例子，该例子实现了每当执行Accept()的时候会原子性增加wait group。首先我们先继承`net.Listener`实现一个结构体
```go
type gracefulListener struct {
    net.Listener
    stop    chan error
    stopped bool
}

func (gl *gracefulListener) File() *os.File {
    tl := gl.Listener.(*net.TCPListener)
    fl, _ := tl.File()
    return fl
}
```
接下来我们覆盖Accept方法(暂时先忽略`gracefulConn`)
```go
func (gl *gracefulListener) Accept() (c net.Conn, err error) {
    c, err = gl.Listener.Accept()
    if err != nil {
        return
    }

    c = gracefulConn{Conn: c}

    httpWg.Add(1)
    return
}
```
我们还需要一个构造函数以及一个Close方法，构造函数中另起一个goroutine关闭，为什么要另起一个goroutine关闭，请看refer<sup>[1]</sup>
```go
func newGracefulListener(l net.Listener) (gl *gracefulListener) {
    gl = &gracefulListener{Listener: l, stop: make(chan error)}
    // 这里为什么使用go 另起一个goroutine关闭请看文章末尾
    go func() {
        _ = <-gl.stop
        gl.stopped = true
        gl.stop <- gl.Listener.Close()
    }()
    return
}

func (gl *gracefulListener) Close() error {
    if gl.stopped {
        return syscall.EINVAL
    }
    gl.stop <- nil
    return <-gl.stop
}
```
我们的`Close`方法简单的向stop chan中发送了一个nil，让构造函数中的goroutine解除阻塞状态并执行Close操作。最终，goroutine执行的函数释放了`net.TCPListener`文件描述符。

接下来，我们还需要一个`net.Conn`的变种来原子性的对wait group做减法
```go
type gracefulConn struct {
    net.Conn
}

func (w gracefulConn) Close() error {
    httpWg.Done()
    return w.Conn.Close()
}
```

为了让我们上面所写的优雅启动方案生效，我们需要替换`server.Serve(l)`行为:
```go
netListener = newGracefulListener(l)
server.Serve(netListener)
```
最后补充：我们还需要避免客户端长时间不关闭连接的情况，所以我们创建server的时候可以指定超时时间：
```go
server := &http.Server{
        Addr:           "0.0.0.0:8888",
        ReadTimeout:    10 * time.Second,
        WriteTimeout:   10 * time.Second,
        MaxHeaderBytes: 1 << 16}
```

#### 译者总结


#### 译者注:
1. refer<sup>[1]</sup>
<u>在上面的代码中使用goroutine的原因作者写了一部分，但我并没有读懂，但幸好在评论中，[jokingus](https://disqus.com/by/disqus_uypMWCNAEC/)问道：如果用下面的方式，是否就不需要在`newGracefulListener`中使用那个goroutine函数了</u>
```go
func (gl *gracefulListener) Close() error { 
    // some code
    gl.Listener.Close()
}
```
作者回复道：
> Honestly, I cannot fathom why there would need to be a goroutine for this, and simply doing gl.Listener.Close() like you suggest wouldn't work.... May be there is some reason that is escaping me presently, or perhaps I just didn't know what I was doing? If you get to the bottom of it, would you post here, so I can correct the post if this goroutine business is wrong?

<u>作者自己也较为疑惑，但表示像[jokingus](https://disqus.com/by/disqus_uypMWCNAEC/)所提到的这种方式是行不通的</u>

<u>译者的个人理解：在绝大多数情况下，需要一个goroutine(可以称之为主goroutine)来创建socket，监听该socket，并accept直到有请求到达，当请求到来之后再另起goroutine进行处理。首先因为accept一般处于主goroutine中，且其是一个阻塞操作，如果我们想在accept执行后关闭socket一般来说有两个方法：</u>

- <u>为accept设置一个超时时间，到达超时时间后，检测是否需要close socket，如果需要就关闭。但这样的话我们的超时时间可定不能设置太大，这样结束就不够灵敏，但设置的太小，就会对性能影响很大，总之来说不够优雅。</u>
- <u>accept方法可以一直阻塞，当我们需要close socket的时候，在另一个goroutine执行流中关闭socket，这样相对来说就比较优雅了，作者所使用的方法就是这种</u>

<u>另外，也可以参考：[Go中如何优雅地关闭net.Listener](http://blog.fatedier.com/2016/02/19/how-to-shutdown-go-net-dot-listeners-gracefully/)</u>
