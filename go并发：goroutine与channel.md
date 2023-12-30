# 1. 启动goroutine
go语言运行一个并发的基本单位是goroutine，一般叫做go的协程。在操作系统层面只有进程和线程（线程是一种特殊的进程）的概念，协程可以理解为用户态的线程，相比于线程，协程可以减少操作系统层面线程调度上的开销（线程被挂起时，内核需要处理用户态到内核态的切换，进行上下文保存，更新cpu上的寄存器信息）。
## 1.1 语法
- go发起并发比较简单，只需要在待执行方法前加`go`关键字即可。

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

- java通过线程发起，需要先定义实现`Runnable`接口的类，同时将待执行逻辑放入到run方法中，并通过创建类对应的对象去运行。

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
## 1.2 分析对比
- go语法更加简洁，相比较于java更加灵活，java不太好处理发起一个并发，但是需要传参数到run方法里的情况。
- go性能上更加出色，go底层使用协程的方式，java使用线程。
