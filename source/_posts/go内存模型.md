---
title: go内存模型
date: 2021-01-06 20:34:11
tags:
---

**本文严重参考：[The Go Memory Model](https://golang.org/ref/mem)**

因为个人英文水平以及翻译过程中会掺杂个人的注解，所以建议读者仅仅将本文作为参考，阅读原文更佳。


### 前言

go内存模型描述的是：当某个goroutine对某个变量进行写操作后，在什么样的条件下，另一个goroutine可以观测到该变量的改变。

### 建议
当多个goroutine同时修改一份数据时，这种操作必须序列化。

在go中的channel以及其他同步元语，比如`sync`包就实现了这样的序列化

### Happens Before
我们可以简单的定义两个名词：编写顺序和执行顺序
- 编写顺序就是代码在编写过程中所定义的顺序
- 执行顺序就是该代码在实际执行过程中的顺序

之所为分为这两种顺序是因为：编译器和CPU是会对代码进行reorder。

单独的goroutine中，读操作和写操作<font color="red">**所表现**</font>的执行顺序必须和编写顺序一致，也就是说编译器以及CPU对代码进行Reorder时是不会造成执行顺序相对于编写顺序错乱的。reorder是不改变原有语义的情况下对指令进行排序。

比如：

```go
// 例子1
func main() {
    a := 1  // 1
    b := 2  // 2
    fmt.Println(a, b)  // 3
}
```

在单goroutine下我们的编写顺序是：1，2，3，但实际的执行顺序是不确定的，有可能是1，2，3；也有可能是2，1，3。

go可以保证的是执行顺序中3肯定是在1和2的后面。

reorder也导致了在多goroutine并发的情况下，一个goroutine观测到的执行顺序可能与另一个goroutine的执行顺序不一致。

我们编写的程序对内存的操作无非就是读/写，Happens Before就定义了内存读/写时的顺序。如果`e1` Happens before `e2`，那么我们就可以说`e2` Happens After `e1`，如果`e1`既不Happens Before`e2`也不Happens After `e2`，那么我们就说`e1` Happens concurrently `e2`。

在单个goroutine中，我们的编写顺序就是**Happens-Before顺序**。但需要注意的是：<font color="red">Happens-Before顺序定义的并不是执行顺序，`e1`Happens-Before`e2`并不一定代表`e1`比`e2`先执行，表示`e2`执行时可以观测到`e1`对内存的操作</font>

可能上面加红字的依然比较晦涩，那么这里我大胆的把Happens-Before重新定义一下(仅仅是个人总结，如果有误欢迎讨论)：
1. 如果`e1`的执行对`e2`执行有影响且`e1`Happens-Before`e2`，那么`e1`必定先于`e2`执行
2. 如果`e1`的执行对`e2`执行无影响则即使`e1`Happens-Before`e2`，真正的执行顺序依然不确定

对于上面的例子1来说，步骤1Happens Before步骤3，且步骤1的执行对步骤3存在影响，则1必定先于3执行。2与3也是同理。而步骤1Happens Before步骤2，但是步骤1并不影响步骤2的执行，所以步骤1和步骤2无法确定顺序。

说到“执行有影响”，在程序里似乎就是写了同一段内存地址，或者说同一个变量。在原文中也有这样的说法：

> A read r of a variable v is allowed to observe a write w to v if both of the following hold:

> 如果同时满足以下两个条件，则允许变量 v 的读操作 r 可以观察到对 v 的写操作 w，但并不保证能观察到：

> - r does not happen before w.
> - There is no other write w' to v that happens after w but before r.

> - r 不能Happens Before w
> - 在 r 与 w 操作之间没有其他的对变量v的写操作

> To guarantee that a read r of a variable v observes a particular write w to v, ensure that w is the only write r is allowed to observe. That is, r is guaranteed to observe w if both of the following hold:

> 为了保证能观察到，并且保证r只能观察到w，则需要满足以下两个条件：

> - w happens before r.
> - Any other write to the shared variable v either happens before w or after r.
> - w Happens Before r
> - 任何对共享变量 v 的其他写操作，都发生在 w 之前或 r 之后。

> This pair of conditions is stronger than the first pair; it requires that there are no other writes happening concurrently with w or r.

> 下面的条件相对于上面的条件来说要严格一些，需要没有其他的写操作与w操作和r操作happens concurrently

<font color="green">上面所描述的都是对同一个变量的写操作。所以我们可以再细化一点，把有影响改为同一个变量的写操作，也就是：
1. 如果`e1`修改了变量`e`而`e2`要读或者写`e`且`e1`Happens-Before`e2`，那么`e1`必定先于`e2`执行
2. 如果`e1`与`e2`对应的不是同一个变量或者`e1`和`e2`都是读操作，则即使`e1`Happens-Before`e2`，真正的执行顺序依然不确定。</font>


### 同步

#### 初始化

程序初始化是由单个goroutine执行的，在初始化后该goroutine可能会创建其他goroutine，这些goroutine并发运行。

如果package p导入了package q，那么q的init函数Happens Before p的所有函数。

所有包的init函数Happens Before main.main函数

#### goroutine的创建

`go f()` 这样的语句Happens Before f对应的goroutine

举个例子：
```go
var a string

func f() {
	print(a) // 3
}

func hello() { 
	a = "hello, world"  // 1
	go f()  // 2
}
```

因为1 Happens Before 2，而 2 Happens Before 3，所以1 Happens Before 3，所以你打印的肯定是`hello world`。

#### goroutine的销毁
gouroutine exit时不Happens Before任何其他事件。
#### channel通信

> A send on a channel happens before the corresponding receive from that channel completes.

1. channel的send操作Happens Before其receive操作。注意，这里的channel指的是buffered channel

```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world" // 4
	c <- 0  // 5
}

func main() {
	go f() // 1
	<-c // 2
	print(a) // 3
}
```
注：下面的hp都是指Happens Before

1hp4，1hp2，4hp5，5hp2，所以执行顺序为：1，4，5，2，3。也就是打印结果必定为`hello world`

> The closing of a channel happens before a receive that returns a zero value because the channel is closed.

2. 通道的关闭Happens Before关闭的通道返回的零值被接收前。

在前面的示例中，用 `close(c)` 替换 `c <- 0` 也能达到相同的保证。

> A receive from an unbuffered channel happens before the send on that channel completes.

3. unbuffered channel的receive操作Happens Before其send操作
```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world" // 4
	<-c  // 5
}

func main() {
	go f() // 1
	c <- 0 // 2
	print(a) // 3
}
```
上面的例子中将c换为unbuffered channel后：1hp2，1hp3，1hp4，4hp5，5hp2。所以执行顺序为1，4，5，2，3。打印结果依然为`hello world`

> The kth receive on a channel with capacity C happens before the k+Cth send from that channel completes.

4. 对于容量为C的channel来说，第k次receive操作Happens Before第h+C次send操作
这个规则是前面规则的扩展，我们可以使用channel来实现一个计数器来限制并发，这个channel的容量就是最大可同时执行的任务数。

```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```
这个例子中，最大的同时运行的任务数量为3。

#### Locks

sync包中实现了两种锁：`sync.Mutex` 和 `sync.RWMutex`

对于这两种锁来说，假如存在l为这两种锁的任意实例且 n < m，第n次的`l.Unlock()`操作Happens Before第m次的`l.Luck()`操作。

```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world" // 5
	l.Unlock()  // 6
}

func main() {
	l.Lock() // 1
	go f()  // 2
	l.Lock() // 3
	print(a) // 4
}
```

上面的例子中，1hp2，1hp3，1hp4，2hp5，5hp6，6hp3，3hp4。所以打印结果为`hello world`

特别的，对于`RWMutex`实例l来说，第n次的`l.RLock()`Happens After第n次的`l.Unlock()`，与之对应的`l.RUnlock()`Happens Before第n+1次的`l.Lock()`

#### Once

sync包提供了Once结构，当我们执行`once.Do(f)`时，只有当f()返回时，其他的`once.Do(f)`才会从阻塞中恢复过来并返回。

执行once.Do(f)时，f的返回 Happens Before 所有once.Do(f)调用返回。

```
var a string
var once sync.Once

func setup() {
	a = "hello, world" // 4
}

func doprint() {
	once.Do(setup) // 3
	print(a)  // 4
}

func twoprint() {
	go doprint() // 1
	go doprint() // 2
}
```

4hp3，输出两次`hello world`。

### 不正确的同步

```go
var a, b int

func f() {
	a = 1 // 3
	b = 2 // 4
}

func g() {
	print(b) // 5
	print(a) // 6
}

func main() {
	go f() // 1
	g()  // 2
}
```
因为尽管1Happens Before2，但是我们无法确定3，4与5，6之间的Happens关系，所以结果无法确定。

为了优化性能我们常常会采取Double-Check：
```go
var a string
var done bool

func setup() {
	a = "hello, world" // 3
	done = true  // 4 
}

func doprint() {
	if !done { // 5
		once.Do(setup) // 6
	}
	print(a) // 7
}

func twoprint() {
	go doprint() // 1
	go doprint() // 2
}
```
这里面3和4的执行顺序无法保证，所以在5中如果done是true，a也不一定是`hello world`。

另一个例子是忙等
```go
var a string
var done bool

func setup() {
	a = "hello, world" // 4
	done = true  // 5
}

func main() {
	go setup() // 1
	for !done {  // 2
	}
	print(a)  // 3
}
```

这个也是因为4和5的执行顺序无法确定导致的。

还有一些变种：

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}

```
译者注：这个例子按照官方文档上面是说不能保证输出`hello world`的，但是个人在mac上试是100%打印`hello world`的。另外这个代码在单核处理器中可能会死循环。


