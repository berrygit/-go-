# 1. goroutine
## 1.1 什么是goroutine
go语言运行一个并发的基本单位是`goroutine`，一般叫做go的协程。在操作系统层面只有进程和线程的概念（线程其实是一种特殊的进程），而协程可以理解为用户态的线程。具体而言，go使用`m:n`的模型，也就是m个goroutine运行在n个操作系统线程上，n的数量一般为cpu的超线程个数。go有自己的调度器处理每个goroutine具体运行在哪个操作系统的线程上，比如一个goroutine当前处于sleep状态，go调度器可以挂起该goroutine，用底层的线程去运行其它需要运行的goroutine，这样做的好处是可以节省操作系统线程调度开销。

![image](https://github.com/berrygit/fast-start-golang/assets/13058540/93eaf23d-3bd9-49ff-b03e-ed30bc461d57)

注：图片来源于网络

如上图所示为cpu的缓存结构，在操作系统层面的线程调度，当线程数量多于cpu的超线程个数时，操作系统调度器需要分时间片给不同的线程运行。每次线程切换时，需要将当前线程在cpu的缓存（如寄存器）内的上下文保存到主存，同时将需要被调度的线程上下文，从主存保存到cpu的缓存中，由于这一操作需要访问内存，时间开销会比较大。

go底层使用的线程数量等于cpu的超线程个数，操作系统不需要做线程切换，相关的goroutine调度由go自己完成，性能会优于直接使用线程。同时，线程初始化的栈空间为固定的2MB，goroutine可以按需分配，从而节省内存，goroutine更加轻量化，支持创建的数量会更多。

## 1.2 如何启动goroutine
- go启动goroutine比较简单，只需要在待执行方法前加`go`关键字即可。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    go print()
    time.Sleep(1 * time.Second)
}

func print() {
    fmt.Println("Hello, 世界")
}
```

- java通过线程发起并发，需要先定义实现`Runnable`接口的类，同时将待执行逻辑放入到run方法中，并通过创建类对应的对象去运行（启动线程方式有多种，这里只举例一种）。
```java
package main;

public class Main {
    public static void main(String[] args) {
        Thread thread = new Thread(new MyThread());
        thread.start();
    }
}

class MyThread implements Runnable{
    @Override
    public void run() {
        System.out.println("Hello, 世界");
    }
}
```
## 1.3 与java对比
- go语法更加简洁，相比较于java更加灵活，java不太好处理发起并发，但是需要通过参数传递信息到run方法里的情况。
- go性能上更加出色，go底层使用协程的方式，java使用线程。

## 1.4 常见陷阱
在for循环中，将实例传递到goroutine，由于闭包每个goroutine读取到的变量可能是同一个，没有完全隔离：
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    array := []int{1, 2, 3, 4, 5}
    for _, i := range array {
        go func() {
            fmt.Printf("%d", i) // 读取到的是同一个变量
        }()
    }
    time.Sleep(time.Second)
}
```
改进版：
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    array := []int{1, 2, 3, 4, 5}
    for _, i := range array {
        j := i  // 循环每次迭代生成一个新的变量 
        go func() {
            fmt.Printf("%d", j)
        }()
    }
    time.Sleep(time.Second)
}
```

# 2. 并发协同

## 2.1 等待完成
当启动多个并发时，需要等待每个并发完成后退出，go使用`sync.WaitGroup`实现。
```go
package main

var (
    wg        sync.WaitGroup
)

func main() {
    wg.Add(2) // 设置并发数量
    go Inc()  // 异步执行
    go Inc()  // 异步执行
    wg.Wait() // 等待完成
}

func Inc() {
    wg.Done() // 确认完成
}
```

java使用`CountDownLatch`实现。

```java
package main;

import java.util.concurrent.CountDownLatch;
import static main.Main.latch;

public class Main {
    public static CountDownLatch latch = new CountDownLatch(2); // 设置并发数量

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(new MyThread());
        Thread thread2 = new Thread(new MyThread());
        thread1.start(); 	// 异步执行
        thread2.start(); 	// 异步执行
        latch.await(); 		// 等待完成
    }
}

class MyThread implements Runnable{
    @Override
    public void run() {
        latch.countDown(); // 确认完成
    }
}
```

## 2.2 互斥
go使用`sync.Mutex`做并发访问临界区的控制，获得锁的goroutine继续执行，其余goroutine阻塞，直到锁释放后重新竞争获得锁方可执行。
```go
package main

import (
    "fmt"
    "sync"
)

var (
    count = 0
    wg    sync.WaitGroup
    mu    sync.Mutex
)

func main() {
    wg.Add(2) // 设置并发数量
    go Inc()  // 异步执行
    go Inc()  // 异步执行
    wg.Wait() // 等待完成
    fmt.Println(count)
}

func Inc() {
    mu.Lock()         // 加锁
    defer mu.Unlock() // 释放锁
    defer wg.Done()   // 确认完成
    count = count + 1
}
```

java使用`ReentrantLock`实现互斥。
```java
package main;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

import static main.Main.*;

public class Main {
    public static CountDownLatch latch = new CountDownLatch(2); // 设置并发数量
    public static Lock lock = new ReentrantLock();
    public static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(new MyThread());
        Thread thread2 = new Thread(new MyThread());
        thread1.start(); 	// 异步执行
        thread2.start(); 	// 异步执行
        latch.await(); 		// 等待完成
        System.out.println(count);
    }
}

class MyThread implements Runnable{
    @Override
    public void run() {
        lock.lock(); 		// 抢锁
        count = count + 1;
        latch.countDown(); 	// 确认完成
        lock.unlock(); 		// 释放锁
    }
}
```
类似的，对于同时有读有写的情况，go使用`sync.RWMutex`实现互斥，java使用`ReentrantReadWriteLock`实现互斥，这里不再赘述。

## 2.3 单例

对于一些初始化比较耗时或占用资源比较多的实例，一般使用单例的方式生成。go的单例使用`sync.Once`实现：
```go
package main

import (
    "sync"
)

var (
    once     sync.Once
    instance *singleton
)

type singleton struct {				// 非导出的，不允许在包外创建
}

func main() {
    GetInstance()
}

func GetInstance() *singleton {
    once.Do(func() {			// 只执行一次
        instance = &singleton{}
    })
    return instance
}
```
java有多种实现方式，这里只举例一种：
```java
package main;

public class Singleton {

    private Singleton(){} // 私有化构造器，防止被外部初始化

    private static volatile Singleton instance; // 静态变量全局只有一份，volatile保证多个线程同时可见

    public static Singleton getInstance() {
        if (instance  != null) {            // 如果已经初始化，直接返回
            return instance;
        }

        synchronized(Singleton.class) {     // 防止并发，此处上锁
            if (instance == null) {         // 该临界区只会有一个线程访问，再次检查
                instance = new Singleton(); // 如果未被实例化，执行
            }
        }
        return instance;
    }
}
```

# 3. 通道channel

## 3.1 通道特点

直观理解通道（channel）就是一个队列，用于不同goroutine间传递信息，具有以下特性：
- **通道有特定的类型**，只能传递特定类型的数据。
- **通道有接收方和发送方**，接收方从通道中拿数据，发送方从通道中发送数据。
- **通道具有原子性**，发送或接收数据时，通道可以保证信息是完整的。
- **通道有容量限制**，可以指定通道中瞬时可存放数据的数量大小。
- **通道可以做同步控制**，当通道中没有数据时，接收方会被阻塞；当通道中数据放满时，发送方会被阻塞。

## 3.2 通道语法

定义一个通道，使用make函数创建，类型支持自定义，这里是空类型`struct{}`，容量大小为3。
```go
ch := make(chan struct{}, 3)
```
向通道中写入数据，使用操作符`<-`
```go
ch <- struct{}{}
```
向通道中读取数据，也是使用操作符`<-`，只是通道在操作符的右边
```go
x := <- ch	// 将通道返回的内容放入到变量中
<- ch		// 忽略通道返回的内容
```
单向通道，通过类型限定，只允许读或写一种操作的通道。
```go
chan<-    // 仅允许写入的通道
writech := make(chan<- struct{}, 3)

<-chan    // 仅允许读取的通道
readch := make(<-chan struct{}, 3)

```

**可以使用通道协调多个goroutine，如若干goroutine向通道投递数据，若干goroutine从通道读取数据，不同goroutine不相互感知。**

java有和通道类似的效果的接口`BlockingQueue`

```java
package main;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class Main {
    public static void main(String[] args) {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);	// 指定队列的元素类型和容量
        queue.add("message");										// 队列中加入元素
        queue.poll();												// 队列中取出元素
    }
}
```

## 3.3 通道终止
当程序完成需要退出时，需要一个信号通知通道接收方退出，可以通过关闭通道实现：

1. 发送方**关闭通道**(使用close函数关闭)
```go
close(ch)
```
向一个关闭的通道发送数据，会报panic；从一个关闭的通道读取数据，如果通道内仍然有数据，可以正常读取，如果通道已经没有数据不会阻塞且只会读取到`通道绑定类型`的空值。

2. 接收方**判定通道读取完毕**，并退出
```go
x, ok := <- ch
if !ok {
    // exit
}
```
可以通过ok值判定，如果通道被关闭，且通道元素已经全部被消费完毕，ok值会返回false（如果通道关闭，但仍然有数据未被消费完毕，ok值为true）。

更加简单的，可以用for循环语句实现，会依次读取通道内的元素，直到通道关闭且通道内元素被全部消费完毕。
```go
for x := range ch {
    // do something
}
```

## 3.4 控制并发
如果无限制启动并发，往往会将系统资源耗尽，通常需要限制并发的最大数量。go使用带缓冲区通道的方式，来限制并发数量，可以将通道看做一个令牌桶，每次启动一个并发需要先将一个令牌放入桶中，成功则执行，否则等待。具体如下：
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    array := []int{1, 2, 3, 4, 5}            
    var tokens = make(chan struct{}, 3)    // 令牌桶，容量为3
    for _, i := range array {
        j := i
        tokens <- struct{}{}               // 将令牌放入桶中，如果桶满则阻塞，限制并发
        go func() {
            fmt.Printf("%d", j)
            <-tokens                       // 执行完毕，释放令牌空间
        }()
    }
    time.Sleep(time.Second)
}
```

java使用线程池的方式控制并发，相当于控制goroutine的数量，具体如下：
```java
package main;

import java.util.concurrent.Executors;
import java.util.concurrent.ExecutorService;

public class Main {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(3);  // 固定数量线程池
        for (int i = 0; i < 5; i++) {
            executorService.submit(new Task());     // 提交任务到线程池
        }
        executorService.shutdown();                 // 关闭线程池
    }
}

class Task implements Runnable {
    public void run() {
        System.out.println("Task executed by " + Thread.currentThread().getName());
    }
}
```

## 3.5 多通道
如果需要同时从多个通道读取数据，有一个通道数据就绪就立刻处理，这种情况可以使用select语句处理。具体如下：
```go
package main

func main() {
    ch1 := make(chan struct{})
    ch2 := make(chan struct{})
    select {
    case <-ch1:
    	// ch1有数据，则直接处理，并退出
    case <-ch2:
    	// ch2有数据，则直接处理，并退出
    default:
    	// ch1与ch2均无数据，做处理后退出
    }
}
```
如果多个通道数据同时就位，则随机选择一个分支执行。


# 4. context.Context

在实际项目工程中往往使用`context.Context`接口去处理goroutine的上下文信息传递，服务退出等操作。

## 4.1 接口定义
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)    // 可获取预期的截止时间，下游可根据该时间评估需要做的操作
    Done() <-chan struct{}                      // 只读通道，判定当前服务是否需要退出
    Err() error                                 // 退出的原因
    Value(key any) any                          // 根据特定的key获取value，多用于上下文信息传递，类似java的ThreadLocal
}
```

## 4.2 使用方式
Context是一个树状结构，可以生成子Context，如果父Context取消，会通知全部子Context取消。
1. 首先创建根节点的Context
```go
context.Background()
```
2. 向Context写入数据
```go
ctx = context.WithValue(ctx, "key", "value")
```
通过源码可以看到，这里会生成新的子节点
```go
func WithValue(parent Context, key, val any) Context {
    // 无关代码忽略
    return &valueCtx{parent, key, val}        // 当前节点作为父节点，生成了新的子节点
}
```
3. 需要时从Context读取数据
```go
val := ctx.Value("key")
```
具体看源码可以看到，Value方法会一直向树的根节点寻找匹配的值
```go
func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key)
}
func value(c Context, key any) any {
    for {
        switch ctx := c.(type) {
        case *valueCtx:
            if key == ctx.key {        // 如果匹配，则返回
                return ctx.val
            }
            c = ctx.Context           // 替换当前节点为父亲节点
        // xxx
        default:
            return c.Value(key)       // 继续寻找节点
        }
    }
}
```
4. 等待退出通知
```go
<-ctx.Done()
```


更多详细信息可参考go官方文档：https://blog.golang.org/context
