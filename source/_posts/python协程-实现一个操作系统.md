---
title: 'python协程:实现一个操作系统'
date: 2017-03-11 20:34:08
tags:
- python
- yield
categories:
- python
---

本文参考：[A Curious Course on Coroutines and Concurrency](http://www.dabeaz.com/coroutines/)

### 缘起：
　　本人最近在学习python的协程。偶然发现了David Beazley的coroutine课程，花了几天时间读完后，为了加深理解就把其中个人认为最为精华的部分摘下来并加上个人理解写了本篇博客。

 

**扯一些淡：**

　　既然要搞一个操作系统，那我们就先来设一个目标吧！就像找女朋友，我们不可能随随便便的是个女的就上，肯定要对女方有一定的要求，比如肤白貌美气质佳…… 

所以，我们对这个' 姑娘 '的要求是这样的：

1. 使用纯Python代码开发 (必须是人形，非狗非猫非其他)　
2. 真正的操作系统，不仅仅能调度任务，还提供了许许多多的系统调用，比如说新建一个进程，kill一个进程，这些我们也要实现！！(所以说就算是找个充气的也必须是有多功能的)
3. 可以处理多任务  (洗衣做饭啥都能干)

 
虽然这要求有点低，但也凑活着用了……  好，不扯淡了。　　

因为我们的主要目的是为了了解协程如何应用，所以我们不使用子进程或者线程模块，我们要使用协程(coroutine)！！！ 

ps：这里的协程指的是由Python中原生的generator衍生出来的coroutine，不是greenlet等库！

 

### 大体分析：
1. 为了让coroutine/generator更像是task，我们定义一个Task类，用来封装coroutine。我们把一个task模仿为一个进程或者线程。
2. 既然要处理多任务，我们需要搞一个调度器(scheduler)用来调度任务的执行。
3. 需要定义系统调用类。　　

好，接下来我们撸起袖子就是干

### 实现多任务处理
#### 1 先来实现一个Task类

- 每一个task都拥有一个唯一的tid
- 为Task定义一个run()方法，用来向generator中传递值

```python
class Task(object):
    '''
    generator的wrapper
    '''
    tid = 0
    def __init__(self,target):
        '''
        :param target:  一个generator对象
        '''
        Task.tid += 1
        self.tid = Task.tid     # 用来唯一地标识一个task
        self.target = target
        self.sendval = None

    def run(self):
        return self.target.send(self.sendval)

```

#### 2 定义一个scheduler

1. 首先我们需要开辟一段数据结构来保存所有的task，这里使用dict来保存，使用tid作为key，task对象作为value
2. 因为我们要实现多任务处理，所以我们需要有一个有序的序列来存储每一个处于ready状态(也就是可运行)的task，在这里我们选择使用队列queue
3. 我们需要一个方法来创建task
4. 我们需要一个方法来处理运行结束的task
5. 我们需要一个方法来将ready状态的task放入queue中
6. 然后就是搞一个loop，不断的从queue中获取task并运行。

```python
class Scheduler(object):
    def __init__(self):
        self.task_map = {}
        self.ready = queue.Queue()
    def new(self,target):
        '''
        新建一个task
        :param target:  generator对象
        :return: 新建task的tid
        '''
        task = Task(target)
        self.task_map[task.tid] = task
        self.schedule(task)
        return task.tid
    def schedule(self,task):
        '''
        将task放入ready队列中
        :param task:  Task对象
        :return:
        '''
        self.ready.put(task)

    def exit(self,tid):
        '''
        处理运行结束的task
        :param task: Task对象
        :return:
        '''
        del self.task_map[tid]

    def main_loop(self):
        '''
        启动loop
        :return:
        '''
        while True:
            task = self.ready.get()
            try:
                result = task.run()
            except StopIteration:
                self.exit(task.tid)
                continue
            self.schedule(task)
```

试着运行一下：

```python
def Laundry():
    for i in range(5):
        yield
        print('I am doing the laundry')

def Cook():
    for i in range(10):
        yield
        print('I am cooking')

s = Scheduler()
s.new(Cook())
s.new(Laundry())
s.main_loop()
```

在上面的代码中：

1. 每当进行task.run()就会执行某个逻辑，然后yield。  (在yield之前的这段时间task拥有cpu控制权)
2. 当task进行yield时，scheduler取代task获得cpu执行权，然后将运行了一次yield的task重新加入到queue中
3. 然后scheduler获取queue中的下一个task，执行task.run()，这就表示新的task获得了cpu控制权。

是不是很像上下文切换？

到现在为止，我们已经实现了多任务处理，接下来搞定系统调用。

我们知道当某个进程进行系统调用时，表示内核获得cpu控制权，内核代替该进程执行相关操作。 这里和上面的yield是不是很类似？

### 实现系统调用：
#### 1 首先我们定义一个所有系统调用的基类：SystemCall

1. 我们得知道是哪个task调用了该系统调用吧，所以增加一个属性self.task
2. 我们得知道当前调度器是谁，因为系统调用不仅要进行相关操作，还需要在相关操作完成后决定如何调度该task(比如说结束该task，继续执行该task……)，所以增加一个属性self.schedule
3. 我们还要知道该如何运行，所以我们统一定义一个handle方法来进行相关操作

根据以上分析，我们这么定义：
```python
class SystemCall(object):
    def __init__(self):
        self.task = None
        self.scheduler = None

    def handler(self):
        pass

```

#### 2 我们可以使用yield 来传递或者说调用 系统调用

那么回想一下：系统调用(准确的说是yield返回的值)是在哪里被捕获的呢？

```python
class Scheduler(object):
    ......
    def main_loop(self):
        while True:
            task = self.ready.get()
            try:
                result = task.run()   # 就是这！！！！
            except StopIteration:
                self.exit(task)
                continue
            self.schedule(task)
```
所以我们需要对 result 进行一些处理。修改main_loop()方法：

```python
def main_loop(self):
        '''
        启动loop
        :return:
        '''
        while True:
            task = self.ready.get()
            try:
                result = task.run()
                if isinstance(result, SystemCall):  # 判断result是否是系统调用
                    result.task = task  # 保存当前环境： 当前task
                    result.scheduler = self  # 保存当前环境： 当前调度器
                    result.handler()  # 运行该系统调用的handler方法
                    continue
            except StopIteration:
                self.exit(task.tid)
                continue
            self.schedule(task)
```
#### 3 真正的操作系统有什么系统调用？

##### 1 GetTid：获取当前tid
```python
class GetTid(SystemCall):
    def __init__(self):
        super().__init__()

    def handler(self):
        self.task.sendval = self.task.tid
        self.scheduler.ready.put(self.task)

```
实例：
```python
def Laundry():
    while True:
        tid = yield GetTid()
        print('I am doing the laundry,my tid is %s'% tid)

def Cook():
    for i in range(10):
        tid = yield GetTid()
        print('I am cooking,my tid is %s'% tid)
s = Scheduler()
s.new(Laundry())
s.new(Cook())
s.main_loop()
```
##### 2 New: 新建一个task
```python
class New(SystemCall):
    def __init__(self,target):
        super().__init__()
        self.target = target
    def handler(self):
        tid = self.scheduler.new(self.target)
        self.task.sendval = tid
        self.scheduler.schedule(self.task)
```
实例：
```python
def foo():
    for i in range(5):
        tid = yield GetTid()
        print('I am foo,my tid is %s'% tid)

def bar():
    for i in range(10):
        tid = yield GetTid()
        print('I am bar,my tid is %s'% tid)
    r = yield New(foo())
    if r:
        print('create a new task,the tid of the new task is %s'% r)

s = Scheduler()
s.new(bar())
s.main_loop()
```
##### 3  kill一个task
```python
class Kill(SystemCall):
    def __init__(self,tid):
        super().__init__()
        self.kill_tid = tid

    def handler(self):
        target_task = self.scheduler.task_map.get(self.kill_tid, None)
        if target_task:
            target_task.target.close()
            self.task.sendval = True
            self.scheduler.schedule(self.task)
```
实例
```python
def foo():
    for i in range(5):
        tid = yield GetTid()
        print('I am foo,my tid is %s'% tid)

def bar():
    for i in range(10):
        tid = yield GetTid()
        print('I am bar,my tid is %s'% tid)
    r = yield New(foo())
    if r:
        print('create a new task,the tid of the new task is %s'% r)
    yield
    r = yield Kill(r)
    if r:
        print('killed success')

s = Scheduler()
s.new(bar())
s.main_loop()
```

##### 4  Task waiting
就是说，a task需要等待b task执行完成后才能继续运行
先来讲一下大体思路：
1. 我们为调度程序设置一个字典，用来存储waiting task(相当于a task) 与 wait for task(相当于b task)的关系
2. 字典的键值对该如何设计呢？
    - a方案：假如是 a:[b1,b2...] 这样设计，那么a可以有很多的wait for task，但缺点就是当某个wait for task运行完后，我们需要遍历该字典，并进行相关检测，才能知道可以运行哪些task
    - b方案： 假如是 b:[a1,a2...] 这样设计，那么当b完成后我们可以很快的知道哪些task可以运行了，缺点就是无法为a设置多个wait for task。(当然也可以设置多个，但很麻烦)
3. 我们偷个懒：每个task仅允许一个wait for task 。所以我们选择b方案。为调度器设立一个新的属性 self.exit_wating = {}
```python
class Scheduler(object):

    def __init__(self):
        self.task_map = {}
        self.ready = queue.Queue()
        self.exit_waiting = {}
```
4. 当一个task运行完成后(也就是调度器执行了exit(task))，我们需要检查该task是否有关联的waiting task
    - 如果有的话： 将waiting task放入ready队列
    - 没有： do noting
```python
def exit(self,tid):
    '''
    处理运行结束的task
    :param task: Task对象
    :return:
    '''
    del self.task_map[tid]
    waiting_tasks = self.exit_waiting.pop(tid, None)
    if waiting_tasks:
        for task in waiting_tasks:
            self.schedule(task)
```
5. 为调度器添加一个方法wait_for_exit：该方法接收两个参数：waiting_task wait_for_task_tid
    - 该方法会在调度器的exit_waiting中生成/修改对应的键值对
    - 返回是否成功
```python
 def wait_for_exit(self,wait_tid,waiting_task):
    '''
    为task设置wait for task
    :param wait_tid: wait for task的tid
    :param waiting_task: waiting task
    :return:
    '''
    if self.task_map.get(wait_tid,None):
        self.exit_waiting.setdefault(wait_tid,[]).append(waiting_task)
        return True
    else: return False
```
6. 创建一个系统调用 TaskWait
    - 该系统调用接受一个参数，接收wait for task的tid
    - 运行调度器的wait_for_exit方法
    - 若失败则调度该task
```python

class TaskWait(SystemCall):
    def __init__(self,wait_tid):
        super().__init__()
        self.wait_tid = wait_tid

    def handler(self):
        r = self.scheduler.wait_for_exit(self.wait_tid,self.task)
        if not r:
            self.scheduler.schedule(self.task)
```

实例：
```python
def foo():
    for i in range(5):
        tid = yield GetTid()
        print('I am foo,my tid is %s'% tid)

def bar():
    for i in range(10):
        tid = yield GetTid()
        print('I am bar,my tid is %s'% tid)
    r = yield New(foo())
    if r:
        print('create a new task,the tid of the new task is %s'% r)
    yield
    r = yield TaskWait(r)
    yield
    if r:
        print('stop bar')
s = Scheduler()
s.new(bar())
s.main_loop()
```

好了到现在为止：

我们已经实现了多任务处理和系统调用。

大家可能对generator/coroutine的用途有了一点点的认知----> 我们可以自己来定义task的切换(也就是在用户空间级别进行task切换！)， 但是！！！ 这有什么用呢？

### 继续深入
所谓交浅言深：不要怪别人太深，只能怪自己太短！！ 所以，继续成长并深入吧，骚年！！

#### 首先根据我们构建的系统搞一个web服务器

```python
def handle_client(client, addr):
    print("Connection from", addr)
    while True:
        data = client.recv(65536)
        if not data:
            break
        client.send(data)
    client.close()
    print("Client closed")
    yield  # Make the function a generator/coroutine


def server(port):
    print("Server starting")
    sock = socket.socket()
    sock.bind(("", port))
    sock.listen(5)
    while True:
        # 阻塞
        client, addr = sock.accept()
        yield New(handle_client(client, addr))


def alive():
    while True:
        print("I'm alive!")
        yield


sched = Scheduler()
sched.new(alive())
sched.new(server(45000))
sched.main_loop()

# 结果：
# I'm alive!
# Server starting
# server阻塞了整个scheduler，无法运行其他task(也就是alive)
```

当某个task需要进行I/O时(也就是阻塞时)，会阻塞整个scheduler，其他task无法运行

这很不科学啊，真正的操作系统可不会这样，所以我们要改进。

#### 改进：

1. 我们增加一个属性(dict格式)，用来存放文件描述符与task的对应关系

```python
class Scheduler(object):

    def __init__(self):
        self.task_map = {}
        self.ready = queue.Queue()
        self.exit_waiting = {}
        self.read_waiting = {}    # fd可读
        self.write_waiting = {}    # fd可写
```

2. 给调度器增加两个方法，用来将对应的fd与task增加到read_waiting或者write_waiting中

```python
def wait_for_read(self, fd, task):
    self.read_waiting[fd] = task

def wait_for_write(self, fd, task):
    self.write_waiting[fd] = task

```
3. 再加一些系统调用，当task调用该系统调用时，代表该task需要等待某个文件描述符就绪。在文件描述符没有准备就绪时，将task放入dict中，不加入ready队列。 这样一个task阻塞就不会导致整体阻塞了。
```python
class ReadWait(SystemCall):
    def __init__(self,fd):
        super().__init__()
        self.fd = fd.fileno() or fd
    def handler(self):
        self.scheduler.wait_for_read(self.fd, self.task)

class WriteWait(SystemCall):
    def __init__(self,fd):
        super().__init__()
        self.fd = fd.fileno() or fd
    def handler(self):
        self.scheduler.wait_for_read(self.fd, self.task)
```
4. 然后我们要解决文件描述符就绪后唤醒task的问题。我们引入select模块用来监控文件描述符，当某个文件描述符就绪时，将其所对应的task放入ready队列中。
```python
def ioloop(self,timeout):
    '''
    检测当前是否有文件描述符就绪，若有则将对应的task放入调度队列中
    :param timeout:  超时时间
    :return:
    '''
    if self.write_waiting or self.read_waiting:
        r, w, e = select.select(self.read_waiting, self.write_waiting, [], timeout)
        for i in r:
            task = self.read_waiting.pop(i,None)
            if task: self.schedule(task)
        for i in w:
            task = self.write_waiting.pop(i, None)
            if task: self.schedule(task)
```
5. 然后我们要考虑什么时候监控？
    - 方式一： 我们可以在每次从ready队列中取出task之前进行运行监控方法。
    - 方式二： 我们可以将监控方法作为一个task，放入ready队列中。
    - 比较优劣： 方式一执行监控方法过于频繁，如果ready队列中task过多，则很浪费cpu资源。，而且每一次运行select()就会导致一次真正意义上的内核上下文切换！所以方式二是较为可行的：
```python
def io_task(self):
    while True:
        if self.ready.empty():
            self.ioloop(None)
        else:
            self.ioloop(0)
        yield
def main_loop(self):
    '''
    启动loop
    :return:
    '''
    self.new(self.io_task())  # 在这里添加
    while True:
        task = self.ready.get()
        try:
        ...
```
6. 修改web服务器

```python
def handle_client(client, addr):
    print("Connection from", addr)
    try:
        while True:
            yield ReadWait(client)
            data = client.recv(65536)
            client.sendall(data)
    except ConnectionResetError:
        client.close()
        print("Client closed")

def server(port):
    print("Server starting")
    sock = socket.socket()
    sock.bind(("127.0.0.1", port))
    sock.listen(5)
    while True:
        yield ReadWait(sock)
        client, addr = sock.accept()
        yield New(handle_client(client, addr))

def alive():
    while True:
        print("I'm alive!")
        yield

sched = Scheduler()
# sched.new(alive())
sched.new(server(8888))
sched.main_loop()
```
来一个客户端：
```python
import socket

def client():
    s = socket.socket()
    s.connect(('127.0.0.1',8888))
    try:
        while True:
            input_data = input('please input message').encode()
            s.sendall(input_data)
            print(s.recv(200).decode())
    except:
        s.close()

client()
```

### 总结：

我们好像是搞了一个纯Python开发的' 操作系统 '   这个操作系统：

1. 采用协程与I/O多路复用相结合
2. 加上一个修改过的简单 web 服务器

就可以处理多个连接！！，而且最重要的是： 我们仅仅使用了一个进程/线程！！  好了，是时候 自舔一波了！！--(tm我太帅了……)

 
1. 多进程/多线程网络编程都是一个进程或者线程处理一个task，当task过多时，就会导致巨量的进程/线程。 
2. 巨量的进程/线程会导致 上下文切换极其频繁！  大家知道：上下文切换是要消耗cpu资源的 所以当进程/线程数量过多时，cpu资源就得不到有效利用
3. 而协程实际上就是：在用户空间实现task的上下文切换！ 这种上下文切换消耗的代价相较而言微乎其微。这就是协程的优势！
4. 当然协程也有劣势：就是无法利用多核cpu，但是我们有解决办法：多进程 + 协程 

 

### The end

示例代码：[地址](https://github.com/mncu/build_an_os_with_python)
