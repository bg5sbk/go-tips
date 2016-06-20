# 贴士1 － io包

Go语言作为定位服务端编程的语言，处理文件和网络通讯是它主要的应用场景，不论是处理文件还是处理网络通讯，它们都被称之为IO编程，即－对计算机的输入输出设备进行编程。
Go的运行时有一个名叫io的包，从命名可想而知它在Go语言的实际应用中有多么重要的地位，Go语言的所有IO编程都绕不过这一个包。

所以正确的理解这个包，在Go语言的工程实践中是非常重要的，不论你是准备用Go语言处理文件还是处理网络通讯，请务必先看这个包。

io包中大部分是接口定义，其中`io.Reader`和`io.Writer`最为关键。在Go语言中，文件、套接字等一切输入设备抽象，都会实现`io.Reader`接口，而一切输出设备抽象，都会实现`io.Writer`接口。

`io.Reader`的定义如下：

```go
type Reader interface {
        Read(p []byte) (n int, err error)
}
```

其中文档的说明非常重要，文档中详细描述了`Read`方法的各种返回可能性。

文档描述中有一个要点，就是`n`可能小于等于`len(p)`，也就是说Go在读IO的时候，是不会保证一次读取预期的所有数据的。

如果我们要确保一次读取我们所需的所有数据，就需要在一个循环里调用`Read`，累加每次返回的`n`并小心设置下次`Read`时`p`的偏移量，直到`n`的累加值达到我们的预期。

因为上述需求实在太常见了，所以Go在io包中提供了一个`ReadFull`函数来做到一次读取要求的所有数据，通过阅读`ReadFull`函数的代码，也可以反过来帮助大家理解`io.Reader`是怎么运作的。

我们稍微跳跃一下，看一下`net`包中的`Conn`接口，`net.Conn`接口的第一个方法就是`Read`，方法签名跟`io.Reader`接口要求的一样。

要做好Go语言的网络编程，在理解`net.Conn`之前，先得理解`io.Reader`，否则该用`io.ReadFull`的地方没有用，就可能出现奇怪的协议解析错误，并且很难排查。

这就是我说`io.Reader`很重要的原因。

接着我们再看`io.Writer`接口：

```go
type Writer interface {
        Write(p []byte) (n int, err error)
}
```

文档中最关键的信息是：如果`n`小于`len(p)`，`err`必须不为`nil`。

也就是说`io.Writer`在每次写数据时，要保证数据的完整写入，这个特性跟`io.Reader`正好是相反的。

基于`io.Writer`的这一特性，我们可以推断，当我们往一个`net.Conn`写入数据时（是的，它当然也实现了这个接口），调用会阻塞直到数据完整写完或者发生IO错误，这甚至不需要我们阅读任何Go的底层代码就可以做出这个判断，如果你去阅读`net`包和`runtime`包的底层代码，可以进一步确认这个事实。

基于这个事实，我们在设计网络应用或其它IO应用的时候就要小心我们的应用场景中IO阻塞是否是可接受的，如果IO阻塞会影响到业务处理，那我们就需要想办法让IO变成异步行为。

Go语言中要让一个事情变成异步很简单，举个例子：

```go
type Session struct {
        conn     net.Conn
        sendChan chan []byte
}

func NewSession(conn net.Conn, sendChanSize int) *Session {
        s := &Session{
                conn:     conn,
                sendChan: make(chan []byte, sendChanSize),
        }
        go s.sendLoop()
        return s
}

func (s *Session) sendLoop() {
        for {
                p := <-s.sendChan
                _, err := s.conn.Write(p)
                if err != nil {
                        return
                }
        }
}

func (s *Session) Send(p []byte) error {
        select {
        case s.sendChan <- p:
        default:
                return errors.New("Send Chan Blocked!")
        }
        return nil
}
```

以上代码是利用Goroutine和chan来实现异步发送消息，Go的chan在缓冲区满之前是不会阻塞的，缓冲区满了再继续往里写就会阻塞。

为了防止极端情况下chan的缓冲区满发生阻塞影响到业务，我们利用select语法的特性来做到不阻塞并返回错误。

io包除了`io.Reader`和`io.Writer`外，还有很多很有用的内容，比如`io.Copy`，通过阅读底层实现代码，你会发现`io.Copy`不仅仅是简单的从一个`io.Reader`读区数据写入`io.Writer`，当要通过`net.Conn`发送一个文件时，它其实会利用Linux的`SendFile`机制做到零拷贝。

这篇文章没办法一一例举和说明io包的所有内容，也无法代替包的文档，请大家务必要自己仔细阅读包的文档（其实文档第二段就非常关键，不要假定这些调用是线程安全的）。

