# 贴士3 － bufio包

之前我们介绍了io包和协议解析，这次我们要来讲讲bufio包，这个包实现了在项目中很常用到的带缓冲的IO。
先从我们前一个小贴士中的分包代码讲起，重新贴一下这段代码：

```go
func ReadPacket(conn net.Conn) ([]byte, error) {
        var head [2]byte

        if _, err := io.ReadFull(conn, head[:]); err != nil {
                return err
        }

        size := binary.BigEndian.Uint16(head)
        packet := make([]byte, size)

        if _, err := io.ReadFull(conn, packet); err != nil {
                return err
        }

        return packet
}
```

这个分包逻辑，对conn执行了两次`io.ReadFull`调用，从小贴士1对io包的介绍中，大家可以知道`io.ReadFull`实际上是一个内部循环调用`conn.Read()`的过程，所以这段代码虽然很短，但是潜在的IO调用次数却挺多，在最理想情况下，也至少要调用两次`conn.Read()`。

IO调用的开销是什么呢？这得从Go的runtime实现分析起，假设我们这里用到的是一个TCP连接，从`TCPConn.Read()`为入口，我们可以定位到`fd_unix.go`这个文件中的`netFD.Read()`方法。

这个方法中有一个循环调用`syscall.Read()`和`pd.WaitRead()`的过程，这个过程有两个主要开销。

首先是`syscall.Read()`的开销，这个系统调用会在应用程序缓冲区和系统的Socket缓冲区之间复制数据。

其次是`pd.WaitRead()`，因为Go的核心是CSP模型，要让一个线程上可以跑多个Goroutine，其中的关键就是让需要等待IO的Goroutine让出执行线程，当IO事件到达的时候再重新唤醒Goroutine，这样来回切换是有一定开销的。

而我们的这个分包协议的包头很小，有极大的概率是包头和一部分包体甚至是整个包已经在Socket缓冲区等待我们读取，这种情况就很适合使用`bufio.Reader`来优化性能。

`bufio.Reader`的基本工作原理是使用一块预先分配好的内存作为缓冲区，发生真实IO的时候尽量填充缓冲区，调用者读取数据的时候先从缓冲区中读取，从而减少真实的IO调用次数，以起到优化作用。

举一个形象点的例子：你有一个不能移动的桶，一个杯子和一个水龙头（老式的，流量小，手拧开关），你要往桶里装满水并且不能让水龙头的水白白流走。这时候就需要拿着杯子在水龙头下接水，接满一杯立即关掉水龙头，把杯里水倒进桶里，再回来开水龙头接水，如此往复直到桶满。这个过程中很多时间浪费在开关水龙头和等杯子装满水。

如果这时候拿个桶放在水龙头下，水龙头就不用关了，每次先到水龙头下的桶里舀一杯水，如果桶里没水才去水龙头接，这样就省掉了开关水龙头和等杯子装满水的时间。`bufio.Reader`做的就是这样一个事情。

把一个`io.Reader`包装成`bufio.Reader`只需要一行代码，我们的代码可以改造成以下形式：

```go
type PacketConn struct {
        net.Conn
        reader *bufio.Reader
}

func NewPacketConn(conn net.Conn) *PacketConn {
        return &PacketConn{conn, bufio.NewReader(conn)}
}

func (conn *PacketConn) ReadPacket() []byte {
        var head [2]byte

        if _, err := io.ReadFull(conn.reader, head[:]); err != nil {
                return err
        }

        size := binary.BigEndian.Uint16(head)
        packet := make([]byte, size)

        if _, err := io.ReadFull(conn.reader, packet); err != nil {
                return err
        }

        return packet
}

func (conn *PacketConn) Read(p []byte) (int, error) {
        return conn.reader.Read(p)
}
```

代码逻辑是一样的，但是因为用了`bufio.Reader`，在理想状态下，`ReadPacket`在第一次`io.ReadFull`调用的时候就会把后续的数据读入缓冲区，第二次`io.ReadFull`不会有真实IO调用产生。

这里有一个细节需要注意，一旦有一个`io.Reader`被`bufio.Reader`包装并使用了以后，要从这个`io.Reader`读取数据就需要从同一个`bufio.Reader`读取，不能一会用原生`io.Reader`一会用`bufio.Reader`，也不能分别从两个`bufio.Reader`读取，因为每次读取都有可能缓存一部分后续数据在缓冲中，如果下次读取不是从缓冲区里的数据开始读，那么读到的数据意义就不一样了。

除了我们的这个二进制分包协议可以利用`bufio.Reader`来优化性能之外，文本协议的解析可以说几乎无法不适用`bufio.Reader`。

我们举个简单的文本协议例子，假设我们有个简单的文本协议是用'\n'换行符来作为一行数据的结尾，一行一行的发送文本数据。

我们在不使用`bufio.Reader`的情况下要怎样从`io.Reader`中一行一行的读取数据呢？

显然，我们会需要写一个循环（伪代码，没编译）：

```go
func ReadLine(reader io.Reader) (line []byte, err error) {
        var p = []byte{0}
        for {
                _, err := reader.Read(p)
                if err == io.EOF {
                        return line, err
                }
                if err != nil {
                        return nil, err
                }
                if p[0] == '\n' {
                        return line, nil
                }
                line = append(line, p[0])
        }
}
```

逐字节的调用Read方法显然效率会极低，所以显然这里需要用一个缓冲区来预读和解析以及缓冲残余数据。

这种情况很常见，比如HTTP协议就是一个基于换行的文本协议，所以`bufio.Reader`直接就内置了`ReadLine`等一些列用于文本协议解析的方法。

如果`bufio.Reader`还无法满足你的复杂协议解析需求，`bufio`还另外提供了`Scanner`来实现自定义的格式解析。

`bufio`包还提供了一个`Writer`类型，用于实现带缓冲区的写入，比如HTTP应用在输入一个HTML页面的时候，经常会分多个步骤输出HTML的文本内容，如果每次输出都真实发生一次IO调用，效率显然会很不好，先写入缓冲区，再一次性发送给客户端，这样就可以大量减少IO调用次数了。

本文无法替代`bufio`包的文档对所有内容一一做说明，更多内容请大家进一步阅读`bufio`包的文档。