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

import "fmt"

func main() {
    go print()
}

func print() {
    fmt.Println("Hello, 世界")
}
```

- java通过线程发起并发，需要先定义实现`Runnable`接口的类，同时将待执行逻辑放入到run方法中，并通过创建类对应的对象去运行。

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
        System.out.Println("Hello, 世界");
    }
}

```
## 1.3 对比分析
- go语法更加简洁，相比较于java更加灵活，java不太好处理发起并发，但是需要通过参数传递信息到run方法里的情况。
- go性能上更加出色，go底层使用协程的方式，java使用线程。

# 2. 并发控制

