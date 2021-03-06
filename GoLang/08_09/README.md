# 1. 并发编程

## 1.1 goroutine

示例：

```go
func main() {
	for i := 0; i < 10; i++ {
		go func(i int) {
            // 开启的这个多任务可以一直输出打印
			for {
				fmt.Printf("hello from goroutine %d \n", i)
			}
		}(i)
	}
    // 让 mian 函数延迟结束，否则创建的多任务还没有来得及输出 mian 函数就结束运行了
	time.Sleep(time.Millisecond)
}
```

输出：

```sh
hello from goroutine 2 
hello from goroutine 2 
hello from goroutine 6 
hello from goroutine 6 
... ...
hello from goroutine 7 
hello from goroutine 7 
```

## 1.2 goroutine 的本质 —— 协程 Coroutine

如果我们将上面的示例改为循环1000次，程序仍能够运行，但是熟悉操作系统的同学可能知道，开启1000个线程是根本不现实的，所以 go 语言的并发操作开启的并非是多个线程，而是多个协程，其有如下几个优点：

- 协程是一个轻量级的线程
- 非抢占式的多任务处理，由协程主动交出控制权（系统可以主动切出线程）
- 编译器/解释器/虚拟机层面的多任务（线程是由操作系统层面的多任务）

- 多个协程可以在一个或多个线程上运行

>  子程序是携程中的一个特例。

普通函数：在一个线程中，main 函数掌握控制权执行代码语句，当遇到 doWork 函数，将执行权传递给 doWork 函数，然后等待 doWork 的执行结果返回给 main 函数

协程：mian 和 doWork 的数据可以双向流通，控制权也可以双向流通，相当于并发执行的两个线程

![image.png](https://i.loli.net/2019/09/17/CdDqvfiZwnxFsaG.png)

Go语言由调度器来决定每个协程存放的位置：

![image.png](https://i.loli.net/2019/09/17/NKOZdDmY54t8Ukz.png)

由此可以得出 goroutine 的定义：

- 任何函数只要加上 go 关键字，就能送给调度器运行
- 不需要在定义时区分是否是异步函数（即在任意的一个函数前加上 `go` 就可以将其作为一个协程）
- 调度器在合适的点进行切换
- 使用 -racce 来检测数据访问冲突

goroutine 可能的切换点：

- I/O, select
- channel
- 等待锁
- 函数调用（有时）
- runtime.Gosched()
- 上述只是参考，不能保证切换、不能保证在其他点不会切换

# 1. channel

> goroutine 之间通信的通道就叫做 channel

![image.png](https://i.loli.net/2019/09/17/m3CLMFbzDQJoS6Y.png)

## 1.1 创建使用 channel

定义一个 channel 类型：

```go
var c chan int // c == nil
```

创建一个 channel：

```go
c := make(chan int)
```

向一个 channel 发送数据：

```go
c <- 1 // 将1发送给 channel 实例
```

接收一个 channel 发送的数据：

```go
n := <-c
```

综合示例如下：

```go
func chanDemo() {
	c := make(chan int)
	go func() {
		for {
			n := <-c
			fmt.Println(n)
		}
	}()
	c <- 1 // 如果发送一个channel却无人接收，那么就会产生死锁
	c <- 2
	time.Sleep(time.Millisecond)
}

func main() {
	chanDemo()
}
```

我们可以单独将 goroutine 抽离出来从而创建多个 goroutine，并建立多个 channel：

```go
func worker(id int, c chan int) {
	for {
		fmt.Printf("Worker %d received %d\n", id, <-c)
	}
}

func chanDemo() {
	var channels [10]chan int
	for i := 0; i < 10; i++ {
		channels[i] = make(chan int)
		go worker(i, channels[i])
	}
	for i := 0; i < 10; i++ {
		channels[i] <- i
	}
	time.Sleep(time.Millisecond)
}

func main() {
	chanDemo()
}
```

同时创建 goroutine 与生成一个 channel 的步骤可以合并为一个步骤，返回的对象为一个 channel：

```go
func createWorker(id int) chan int {
	c := make(chan int)
	go func() {
		for {
			fmt.Printf("Worker %d received %d\n", id, <-c)
		}
	}()
	return c
}
```

定义channel类型时，我们可以规定其只能接受数据或者只能发送数据：

```go
channel := make(chan<- int) // channel 只能接收数据
channel := make(<-chan int) // channel 只能发送
```

这样的操作通常用于返回一个仅可以接收数据的channel。

## 1.2 使用 BufferChanel

使用了 buffer channel 后，会创建一个缓冲区，不必等待 channel 的接收者，因此可以接受数条数据并不让系统发生死锁，如下的的程序是合法的：

```go
func bufferedChannel() {
	c := make(chan int, 3)
	c <- 1
	c <- 2
	c <- 3
    // c <- 4 超出缓冲区的部分却无接收对象的数据会让系统产生死锁
}
```

创建缓冲区可以优化性能，但是与平常的使用无异。

## 1.3 使用 close() 函数关闭 channel

使用 `close(c chan<- Type)` 方法可以用来关闭一个channel：

```go
func channelClose() {
	c := make(chan int)
	go worker(1, c) // worker 中创建了一个 goroutine，不断接收 channel 的值
	c <- 1
	c <- 2
	c <- 3
	c <- 4
	close(c)
}
```

关闭的 channel 会一直发送数据，但是发送的数据是 `0`:

```sh
Worker 1 received 1
Worker 1 received 2
Worker 1 received 3
Worker 1 received 4
Worker 1 received 0
Worker 1 received 0
... ...
Worker 1 received 0
```

有了 `close()` 方法我们加上对接受数据的判断就可以决定在何时结束对 channel 的持续接收：

```go
func worker(id int, c chan int) {
	for {
		n, ok := <-c
		// 检查接收的 channel 数据是否是由 close() 方法执行后发送过来的
		if !ok {
			break
		}
		fmt.Printf("Worker %d received %d\n", id, n)
	}
}
```

可以使用 range 来简化代码：

```go
func worker(id int, c chan int) {
	for n := range c {
		fmt.Printf("Worker %d received %d\n", id, n)
	}
}
```

> 不要通过共享内存来通信，要通过通信来共享内存。

> 何为使用共享内存?
>
> 如：完成一件事情后将标识符 flag 设置为 true，某一方法监听 flag 的状态，从而获取通信信息。

# 2. 使用 Channel 等待任务结束

## 2.1 在 goroutine 中创建 channel 向外通信 

我们先来整理一下之前的代码，看其做了什么事情：

```go
func createWorker(id int) chan<- int {
	c := make(chan int)
	go worker(id, c)
	return c
}

func chanDemo() {
	var channels [10]chan<- int
	for i := 0; i < 10; i++ {
		// 开启一个 goroutine（Worker） 并创建一个与其连接的Channel并返回
		channels[i] = createWorker(i)
	}
	// 并发向每个 channel 发送数据
	for i := 0; i < 10; i++ {
		channels[i] <- i + 'a'
	}
	for i := 0; i < 10; i++ {
		channels[i] <- i + 'A'
	}
    time.Sleep(time.Millisecond) // 一毫秒之后停止发送数据
}

func worker(id int, c chan int) {
	for n := range c {
		fmt.Printf("Worker %v received %c\n", id, n)
	}
}

func main() {
	chanDemo()
}
```

```sh
Worker 4 received e
Worker 2 received c
Worker 8 received i
Worker 3 received d
...
Worker 6 received g
Worker 9 received j
Worker 4 received E
Worker 1 received B
...
Worker 8 received I
Worker 7 received H
Worker 9 received J
```

上面的代码会生成 10 个 Channel 分别向 10 个 Worker 发送数据（这是一个并发的过程），然后等待 1ms 之后不管 Woker 是否打印完毕，就停止发送数据。这个 1ms 的时间是我们自己“猜”的，显然我们更希望当 Woker 在完成工作后通知 Channel 可以停止发送数据。

![image.png](https://i.loli.net/2019/09/24/w3mCcbjYoZ1Apta.png)

利用 Channel 可以实现向外部通信，只需要在 goroutine 中创建一个 Channel ，在任务执行完成后，从外部通过接收这个 Channel 的信息即可判定当前的任务是否完成。

![image.png](https://i.loli.net/2019/09/24/aptPCSwdsYfLqHG.png)

上述的流程由代码实现则为下：

```go
func doWork(id int, w worker) {
	for n := range w.in {
		fmt.Printf("Worker %v received %c\n", id, n)
		w.done <- true // 当任务结束后，向外发出一个 Channel 信息
	}
}

type worker struct {
	in   chan int
	done chan bool
}

func createWorker(id int) worker {
	w := worker{
		in:   make(chan int),
		done: make(chan bool),
	}
	go doWork(id, w)
	return w
}

func chanDemo() {
	var workers [10]worker
	for i := 0; i < 10; i++ {
		workers[i] = createWorker(i)
	}

	// 并发向每个 channel 发送数据
	for i, worker := range workers {
		worker.in <- i + 'a'
	}
	for _, worker := range workers {
		<-worker.done // 在这里会阻塞后方代码，直到接收完第一波由 Worker 发出的 channel 信号
	}

	for i, worker := range workers {
		worker.in <- i + 'A'
	}
	// 等待所有的返回结果
	for _, worker := range workers {
		<-worker.done
	}
}

func main() {
	chanDemo()
}
```

但是我们会发现上述的结果打印小写字母与大写字母的流程是同步的，如果我们想将其改为非阻塞的，则需要将接收的两波由 Worker 发来的信息的操作集中放置在后面处理：

```diff
func chanDemo() {
	var workers [10]worker
	for i := 0; i < 10; i++ {
		workers[i] = createWorker(i)
	}
	for i, worker := range workers {
		worker.in <- i + 'a'
	}
-   for _, worker := range workers {<-worker.done }
	for i, worker := range workers {
		worker.in <- i + 'A'
	}
-   for _, worker := range workers {<-worker.done}
+   for _, worker := range workers {
+   	<-worker.done
+   	<-worker.done
+   }
}
```

但是这样会造成死锁，这是因为由于接受 worker channel 的逻辑在后面，导致在 worker 发送第一批 channel 时还未有接收者就要发送第二批 channel 信息。我们可以将发送 channel 的步骤改为 goroutine 来发送信息：

```diff
for n := range w.in {
    fmt.Printf("Worker %v received %c\n", id, n)   
-   w.done <- true
+   go func() {
+       w.done <- true
+   }()
}
```

## 2.2 使用 WaitGroup 等待 Channel 通信

我们在创建多个 Channel 进行通信时，可以通过创建一个 `aync.WatiGroup` 对象，每个 `WaitGroup` 对象都有 `.Done()` 和 `.Wait()` 方法，当调用 `.Wait()` 方法时会对当前的代码进行阻塞，直到所有的 `WaitGroup` 对象都调用 `.Done` 方法时才结束阻塞。

利用 WaitGroup 的特性，我们可以重构如下代码：

```go
func doWork(id int, w worker) {
	for n := range w.in {
		fmt.Printf("Worker %v received %c\n", id, n)
		w.wg.Done()
	}
}

type worker struct {
	in chan int
	wg *sync.WaitGroup
}

func createWorker(id int, wg *sync.WaitGroup) worker {
	w := worker{
		in: make(chan int),
		wg: wg,
	}
	go doWork(id, w)
	return w
}

func chanDemo() {
	var wg sync.WaitGroup 
	wg.Add(20) // 创建 20 个任务

	var workers [10]worker
	for i := 0; i < 10; i++ {
		workers[i] = createWorker(i, &wg) // 向外派发
	}
	for i, worker := range workers {
		worker.in <- i + 'a'
	}
	for i, worker := range workers {
		worker.in <- i + 'A'
	}

	wg.Wait() // 直到接收玩所有的任务才结束运行代码（在此起阻塞效果）
}

func main() {
	chanDemo() 
}
```

# 3. 使用 Select 进行调度

由于 **Channel 发数据与接受数据都是阻塞式的** ，我们可以由以下代码证实：

```go
func send(c1 *chan int, c2 *chan int) {
	for {
		*c1 <- 3
		*c2 <- 5
	}
}

func main() {
	c1 := make(chan int)
	c2 := make(chan int)
	go send(&c1, &c2)
	time.Sleep(time.Millisecond)
	n1 := <-c1
	fmt.Println("n1:", n1)
	n2 := <-c2
	fmt.Println("n2", n2)
}
```

上述代码的大致流程如下：

![image.png](https://i.loli.net/2019/09/25/ObjnFqL9Bl3Iiet.png)

在控制台可以看出 c1 与 c2 是固定按照顺序打印出的，如果在发送 channel 信号时，对发送顺序进行调转，则会出现死锁。其原因是因为先发送 c2 再发送 c1 的话，c2 会一直等待接收者。而 c2 的接收者此时尚未出现，先出现的是 c1 的接收者，然而在此时刻 c1 的信号尚未发出，所以就会导致程序陷入死锁：

![image.png](https://i.loli.net/2019/09/25/OTXxe2kdU6a4rFD.png)

但是使用 select 可以并行接收数据，会同时接收 c1 与 c2 的数据，但是仅接收先来的数据：

```go
func send(c1 *chan int, c2 *chan int) {
	for {
		*c1 <- 3
		*c2 <- 5
	}
}

func main() {
	c1 := make(chan int)
	c2 := make(chan int)
	go send(&c1, &c2)
	//time.Sleep(time.Millisecond)
	select {
	case n := <-c1:
		fmt.Println("Received from c1", n)
	case n := <-c2:
		fmt.Println("Received from c2", n)
	}
}
```

由于发送 channel 还是同步的顺序，所以上面的代码总是仅接收第一个 channel 数据。

如下是一个完整的使用 select 调度的示例：

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func generator() chan int {
	out := make(chan int)
	go func() {
		i := 0
		for {
			// 不断向外部发出信号
			time.Sleep(
				time.Duration(rand.Intn(1500)) * time.Millisecond)
			out <- i
			i++
		}
	}()
	return out
}

func createWorker(id int) chan<- int {
	c := make(chan int)
	go worker(id, c)
	return c
}

func worker(id int, c chan int) {
	for n := range c {
		time.Sleep(1 * time.Second)
		fmt.Printf("Worker %d received %d\n", id, n)
	}
}

func main() {
	var c1, c2 = generator(), generator()
	var worker = createWorker(0)
	var values []int
	tm := time.After(10 * time.Second)
	tick := time.Tick(time.Second)
	for {
		var activeWorker chan<- int
		var activeValue int
		if len(values) > 0 {
			activeWorker = worker
			activeValue = values[0]
		}
		select {
		// 从外部接收 channel 信号
		case n := <-c1:
			values = append(values, n)
		case n := <-c2:
			values = append(values, n)
		//	将接受到的信号发送给 worker
		case activeWorker <- activeValue:
			values = values[1:]
		//	判断相邻的两个请求是否超时
		case <-time.After(800 * time.Millisecond):
			fmt.Println("timeout")
		//	每隔一秒输出队列长度
		case <-tick:
			fmt.Println("quene len =", len(values))
		//	超过 10 秒后终止循环
		case <-tm:
			fmt.Println("bye")
			return
		}
	}
}
```

> 以上称为 Go 语言的 csp 模型

# 4. 传统同步机制（很少使用）

- WaitGroup
- Mutex
- Cond

## 4.1 Mutex

复写一个原子化的操作：

```go
type atomicInt struct {
	value int
	lock  sync.Mutex
}

func (a *atomicInt) increment() {
	a.lock.Lock()
	defer a.lock.Unlock() // 延迟执行
	a.value++
}

func (a *atomicInt) get() int {
	a.lock.Lock()
	defer a.lock.Unlock() // 延迟执行
	return a.value
}

func main() {
	var a atomicInt
	a.increment()
	go func() {
		a.increment()
	}()
	time.Sleep(time.Millisecond)
	fmt.Println(a.get())
}
```
