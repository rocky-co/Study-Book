# 1. Socket

常用的 Socket 类型有两种：流式 Socket（SOCK_STREAM）和数据报式 Socket（SOCK_DGRAM）。流式是一种面向连接的 Socket，**针对于面向连接的 TCP 服务应用**；数据报式 Socket 是一种无连接的 Socket，**对应于无连接的 UDP 服务应用**。

`Socket `是应用层与TCP/IP协议族通信的中间软件抽象层。在设计模式中，`Socket` 其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在`Socket`后面，对用户来说只需要调用Socket规定的相关函数，让 `Socket` 去组织符合指定的协议数据然后进行通信。

![image.png](https://i.loli.net/2019/10/14/c3UEdNgI9QhsuRt.png)

# 2. TCP Socket

要想在客户端与服务器端之间一个基于 TCP 的通信，通常分为以下几步：

- 服务器端部署监听端口，准备接收客户端发送的连接请求
- 客户端创建一个 TCP 连接，与服务器端进行确认（三次握手），互相获得一个连接对象
- 客户端发送数据，由服务器端接收数据
- 服务器端对数据进行处理，返回数据给客户端
- 断开两端的连接（四次挥手）

那么接下来，我们按照这个流程来创建一个基于 TCP 的：

## 2.1 TCP Client

**1. 客户端创建连接：**

我们先不考虑如何创建一个 TCP 服务器端，因为我们首先要创建一个客户端来发送链接请求。`net.DialTCP()` 可以由一个 IP 对象获得一个连接对象（同时服务器端接收请求也创建了一个连接对象），这个过程就是与服务器端建立连接的过程。创建连接方式的方法如下：

```go
tcpAddr, _ := net.ResolveTCPAddr("tcp4", "localhost:9090")
conn, _ := net.DialTCP("tcp", nil, tcpAddr)
defer conn.Close()
```

```go
func DialTCP(network string, laddr, raddr *TCPAddr) (*TCPConn, error)
```

- net 参数是 "tcp4"、"tcp6"、"tcp" 中的任意一个，分别表示 TCP (IPv4-only)、TCP (IPv6-only) 或者 TCP (IPv4, IPv6 的任意一个)
- laddr 表示本机地址，一般设置为 nil
- raddr 表示远程的服务地址

我们还可以使用 `net.Dial()` 方法 将上述过程简化：

```go
conn, err := net.Dial("tcp", "127.0.0.1:20000")
```

**2. 向连接写入数据：**

当获取数据后，服务器端就可以针对于连接对象写入数据了，这样服务器端就能够接收到数据：

```go
conn.Write([]byte("This is data from client")) // 正常发送的数据为一个 HTTP 报文
```

**3. 读取服务器端返回的数据：**

当服务器端对获取的数据进行处理后，会返回给客户端数据，获取客户端数据的方式为：

```go
result, err := ioutil.ReadAll(conn)
checkError(err)
fmt.Println(string(result))
```

**4. Demo**

接下来我们写一个简单的例子，模拟一个基于 HTTP 协议的客户端请求去连接一个 Web 服务端。我们要写一个简单的 http 请求头，格式类似如下：

```php
"HEAD / HTTP/1.0\r\n\r\n"
```

我们的客户端代码如下所示：

```go
// serve/main.go

package main

import (
	"fmt"
	"io/ioutil"
	"net"
	"os"
)

func checkError(err error) {
	if err != nil {
		_, _ = fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}
}

func main() {
	service := "localhost:9090"
	// 由 ip 地址字符串创建个 IP 对象
	tcpAddr, err := net.ResolveTCPAddr("tcp4", service)
	checkError(err)
    
	// 建立连接
	conn, err := net.DialTCP("tcp", nil, tcpAddr)
	checkError(err)
	defer conn.Close()
    
    // 向连接对象发送数据
	_, err = conn.Write([]byte("HEAD / HTTP/1.0\r\n\r\n"))
	checkError(err)
    
    // 读取连接对象返回的数据
	result, err := ioutil.ReadAll(conn)
	checkError(err)
	fmt.Println(string(result))
	os.Exit(0)
}
```

## 2.2 TCP Serve

**1. 创建监听端口：**

对于服务器端，要向接收到客户端发送的 TCP 连接请求，则需要创建一个监听端口来捕获信息，这个监听端口的创建方式如下：

```go
service := "localhost:9090"
tcpAddr, err := net.ResolveTCPAddr("tcp4", service)
checkError(err)
listener, err := net.ListenTCP("tcp", tcpAddr)
checkError(err)
```

当然，我们可以使用 `net.Listen()`  方法来简化创建过程：

```go
listener, err := net.Listen("tcp", service)
```

之后再开启一个 for 循环，因为我们要一直监听这个端口，不断获取连接对象，这个 for 循环过程中获取当前连接对象的方式为：

```go
conn, err := listener.Accept()
```

**2. 处理客户端数据：**

这一步对应客户端发送数据的过程，当客户端发送给服务器端数据后，客户端会等待服务器端的响应。但是如果服务器端没有处理客户端的信息就发送了数据，客户端的连接就会中断；相应的，如果客户端没有发送数据而服务器端却想要读取连接对象中的数据，客户端与服务器端的连接将会一直被挂起。**也就是说，服务器端与客户端的信息交流只要一端发送了，另一端就必须接受；如果一端没有发送数据，则另一端不能强行去读取一个空数据。**

处理客户端的数据通常使用一个 Reader 去读取连接对象：

```go
reader := bufio.NewReader(conn)
var buf [128]byte
n, err := reader.Read(buf[:]) // n为读取的字节数
```

**3. 服务器端向连接对象写入数据：**

与客户端一样，当服务器端也需要向连接对象写入数据，客户端才能够获取到数据，写入数据的方式与客户端一样：

```go
conn.Write([]byte(recvStr)) 
```

**4. Demo**

演示客户端接收服务器端的 HTTP 报文，然后向客户端返回数据：

```go
// client/main.go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
)

func checkError(err error) {
	if err != nil {
		_, _ = fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}
}

func handleClient(conn net.Conn) {
	defer conn.Close()

	reader := bufio.NewReader(conn)
	var buf [128]byte
	n, err := reader.Read(buf[:])
	if err != nil {
		fmt.Println("read from client failed, err:", err)
	}
	recvStr := string(buf[:n])
	fmt.Println("收到client端发来的数据：\n", recvStr)

	conn.Write([]byte("I'm ok")) // 发送数据
}

func main() {
	service := "localhost:9090"
	tcpAddr, err := net.ResolveTCPAddr("tcp4", service)
	checkError(err)
	listener, err := net.ListenTCP("tcp", tcpAddr)
	checkError(err)
	fmt.Println("serve start at:", service)
	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("connect err:", err)
			continue
		}
		go handleClient(conn)
	}
}
```

**4. 创建长连接：**

在上面创建的服务器端的连接实际上是有问题的，在服务器端接收客户端发来的数据时并非是阻塞的，如果客户端发送的是一条特别长的数据，那么在服务器端接收的数据是不完整的，很可能数据接收到一半就终止，这时客户端就会报错。

那么这就需要去创建 TCP 长连接来维持服务器端能够维持连接不被中断，我们可以在服务器端使用 for 循环来不断等待客户端发送数据，且当客户端发送一个数据的终止标识符后，服务器端就可以处理数据进而再返回数据：

```go
// serve/main.go
func handleClientLong(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)
	var buf [1024]byte
	for {
		n, err := reader.Read(buf[:])
        // 直到读取到终止符或者出现错误才终止连接
		if err == io.EOF {
			break 
		}
		if err != nil {
			fmt.Println("read from client failed, err:", err)
			break
		}
		recvStr := string(buf[:n])
		fmt.Println("收到client发来的数据：", recvStr)
	}
}
```

# 2. UDP Socket

Go 语言包中处理 UDP Socket 和 TCP Socket 不同的地方就是在服务器端处理多个客户端请求数据包的方式不同，UDP 缺少了对客户端连接请求的 Accept 函数。其他基本几乎一模一样，只有 TCP 换成了 UDP 而已。UDP 的几个主要函数如下所示：

```go
func ResolveUDPAddr(net, addr string) (*UDPAddr, os.Error)
func DialUDP(net string, laddr, raddr *UDPAddr) (c *UDPConn, err os.Error)
func ListenUDP(net string, laddr *UDPAddr) (c *UDPConn, err os.Error)
func (c *UDPConn) ReadFromUDP(b []byte) (n int, addr *UDPAddr, err os.Error)
func (c *UDPConn) WriteToUDP(b []byte, addr *UDPAddr) (n int, err os.Error)
```

一个 UDP 的客户端代码如下所示，我们可以看到不同的就是 TCP 换成了 UDP 而已：

```go
package main

import (
    "fmt"
    "net"
    "os"
)

func main() {
    if len(os.Args) != 2 {
        fmt.Fprintf(os.Stderr, "Usage: %s host:port", os.Args[0])
        os.Exit(1)
    }
    service := os.Args[1]
    udpAddr, err := net.ResolveUDPAddr("udp4", service)
    checkError(err)
    conn, err := net.DialUDP("udp", nil, udpAddr)
    checkError(err)
    _, err = conn.Write([]byte("anything"))
    checkError(err)
    var buf [512]byte
    n, err := conn.Read(buf[0:])
    checkError(err)
    fmt.Println(string(buf[0:n]))
    os.Exit(0)
}
func checkError(err error) {
    if err != nil {
        fmt.Fprintf(os.Stderr, "Fatal error ", err.Error())
        os.Exit(1)
    }
}
```

我们来看一下 UDP 服务器端如何来处理：

```go
package main

import (
    "fmt"
    "net"
    "os"
    "time"
)

func main() {
    service := ":1200"
    udpAddr, err := net.ResolveUDPAddr("udp4", service)
    checkError(err)
    conn, err := net.ListenUDP("udp", udpAddr)
    checkError(err)
    for {
        handleClient(conn)
    }
}
func handleClient(conn *net.UDPConn) {
    var buf [512]byte
    _, addr, err := conn.ReadFromUDP(buf[0:])
    if err != nil {
        return
    }
    daytime := time.Now().String()
    conn.WriteToUDP([]byte(daytime), addr)
}
func checkError(err error) {
    if err != nil {
        fmt.Fprintf(os.Stderr, "Fatal error ", err.Error())
        os.Exit(1)
    }
}
```

# 3. WebSocket

WebSocket 是 HTML5 的重要特性，它实现了基于浏览器的远程 socket，它使浏览器和服务器可以进行全双工通信，许多浏览器（Firefox、Google Chrom e 和 Safari）都已对此做了支持。

## 3.1 WebSocket Client

客户端跟服务端都可以双向发送消息与接受消息，我们先来演示如何利用 js 在客户端发送 webSocket 消息。

1. 首先要创建一个 Socket 对象：

   ```js
   var wsuri = "ws://127.0.0.1:9090"
   var sock = new WebSocket(wsuri)
   ```

2. 可以使用 `send()` 在客户端主动发送消息：

   ```js
   var msg = "client msg"
   sock.send(msg)
   ```

3. 同时还可以设置几个钩子函数，用于处理不同的情况：

   当建立连接时：

   ```js
   sock.onopen = function() {
       console.log("connected to " + wsuri);
   }
   ```

   当连接关闭时：

   ```js
   sock.onclose = function(e) {
       console.log("connection closed (" + e.code + ")");
   }
   ```

   当客户端收到消息时：

   ```js
   sock.onmessage = function(e) {
       console.log("message received: " + e.data);
   }
   ```

## 3.2 WebSocket Serve

服务端要接受客户端的消息首先要设置 webSocket 信息接收的路由，仍可以使用 `http.Handle()` 来处理路由连接对应的 Handle 函数，但是 Handle 函数必须由 `websocket.Handler()` 方法来转换，整体的流程如下：

```go
func main() {
    http.Handle("/", websocket.Handler(Echo))

    if err := http.ListenAndServe(":1234", nil); err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}
```

在 Handle 函数中接收一个 `*websocket.Conn` 连接对象，利用这个连接对象可以信息：

```go
var reply string
if err = websocket.Message.Receive(ws, &reply); err != nil {
    fmt.Println("Can't receive")
    break
}
fmt.Println("Received back from client: " + reply)
```

也可以主动发送信息：

```go
msg := "send message from serve"

if err = websocket.Message.Send(ws, msg); err != nil {
    fmt.Println("Can't send")
    break
}
```

同时在函数中开启一个 for 循环用于不断接受信息，所以整理的流程如下：

```go
func Echo(ws *websocket.Conn) {
    var err error

    for {
        var reply string

        if err = websocket.Message.Receive(ws, &reply); err != nil {
            fmt.Println("Can't receive")
            break
        }

        fmt.Println("Received back from client: " + reply)

        msg := "Received:  " + reply
        fmt.Println("Sending to client: " + msg)

        if err = websocket.Message.Send(ws, msg); err != nil {
            fmt.Println("Can't send")
            break
        }
    }
}
```





