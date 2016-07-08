# 贴士2 － 协议解析

今天这个小贴士主要介绍协议解析的一些知识，Go语言作为服务端编程语言，免不了要涉及到通讯协议解析，即便不是做网络通讯，也难免会涉及到文件解析，其实它们的知识点都是一样的。
现实应用场景中，通讯协议按通常可以分为两类：二进制协议和文本协议。Go语言内置的gob格式就是一种二进制协议，而JSON、XML等则是文本协议。

假设我们要发送123这个数值，用二进制协议只需要一个字节，因为一个字节（byte）有8个二进制位（bit），2的8次方是256，一个字节可以表达0-255之间的任意值，共256种可能性。

如果我们用文本协议发送123这个数值，则需要至少三个字节，因为123这个数字需要转换成字符'1'、'2'、'3'这三个ASCII字符，存入三个字节中。

所以同样一个数据，用二进制协议表达的体积通常会小于用文本协议表达的体积。这个特性体现到网络应用中，可能就是网络带宽需求的差异。

换个角度看，当我们用二进制协议把123这个数值写入一个文件以后，我们用文本编辑器打开它，看到的会'{'这个字符，因为这个字符的ASCII值正好是123。而当我们用文本协议存储数据时，我们可以用文本编辑器直接读到123这个数值。

所以通常二进制协议比较不利于阅读，而文本协议方便阅读。这个特性体现到开发中的时候，可能就是调试难易度的差异。

二进制数据和文本数据还有个差异是执行效率差异。以123这个值为例，二进制序列化时候只需要直接对一个字节进行赋值，而用用文本格式的时候，则需要计算出‘个’、‘十’、‘百’位上的值，并转成ASCII码，再赋值给三个字节，反序列化的时候也是如此。

以上分析了二进制协议和文本协议的一些特性，并没有说哪个是最优方案，因为不同的应用场景会需要不同的技术方案。比如TCP/IP协议是二进制协议，在TCP/IP之上构建的HTTP协议则是文本协议，它们各有各的应用场景，所以会出现技术上的差异。

再说回协议解析，当我们要解析二进制数据的时候，通常会需要用到Go语言内置的`encoding/binary`这个包，这个包内置了大端序和小端序的二进制数据操作。

什么是大端序和小端序呢？以数值256为例，上面我们有提到，一个字节可以表达0-255之间的任意值，但是当我们要表达256这个值的时候怎么表达呢？

255用二进制表达就是`1111 1111`，再加1就是`1 0000 0000`，多了一个1出来，显然我们需要再用额外的一个字节来存放这个1，但是这个1要存放在第一个字节还是第二个字节呢？这时候因为人们选择的不同，就出现了大端序和小端序的差异。

当我们把这个1放在第一个字节的时候，就称之为大端序格式。当我们把1放在第二个字节的时候，就称之为小端序格式。

这两种格式显然没办法说谁更好，所以两个格式一直都各自的支持者，如果是按标准实现一个通讯协议，那就得严格按照标准上说的字节序来实现。如果是自定义的二进制协议，选择哪个格式按自己喜好就可以了。

`encoding/binary`包中的全局变量`BigEndian`用于操作大端序数据，`LittleEndian`用于操作小端序数据，这两个变量所对应的数据类型都实行了`ByteOrder`接口：

```go
type ByteOrder interface {
        Uint16([]byte) uint16
        Uint32([]byte) uint32
        Uint64([]byte) uint64
        PutUint16([]byte, uint16)
        PutUint32([]byte, uint32)
        PutUint64([]byte, uint64)
        String() string
}
```

其中，前三个方法用于读取数据，后三个方法用于写入数据。

大家可能会注意到，上面的方法操作的都是无符号整型，如果我们要操作有符号整型的时候怎么办呢？很简单，强制转换就可以了，比如这样：

```go
func PutInt32(b []byte, v int32) {
        binary.BigEndian.PutUint32(b, uint32(v))
}
```

大家可能还注意到，上面提供的方法都是操作整型值。原因是浮点数的实现在不同编程语言中可能会不一样，没办法在运行时库中给出一个普适的标准。如果我们要写入和读取浮点数怎么办呢？

项目实践上有两种做法，一种是在协议上约定好一个取整精度，比如小数点后多少位，然后把浮点数转成对应精度的整数，这样的做法跨语言兼容性最好。

如果是Go语言开发的应用之间的二进制数据交换，或者是符合IEEE 754浮点数标准的编程语言，则可以用`math`包里的这几个函数：

```go
func Float32bits(f float32) uint32
func Float32frombits(b uint32) float32

func Float64bits(f float64) uint64
func Float64frombits(b uint64) float64
```

上面说到的都是数值类型的操作，但是实际应用场景中文本，列表，字典等各种复杂数据要怎么在二进制协议中实现呢？

复杂的二进制数据表达，最主要的一个问题就是数据分割问题，比如我们要将下面这个Go结构体进行二进制序列化：

```go
type MyStruct struct {
        Field1 int32
        Field2 string
        Field3 []int16
}
```

首先我们会遇到的问题就是怎么区别各个字段？

首先我们对结构体进行分析，其中第一个字段是`int32`类型的，这种数据类型固定都会被表达成4个字节，所以我们称之为定长类型。第二个字段是`string`类型的，字符串里面内容多上是不一定的，所以我们称之为变长数据类型。第三个字段是`[]int16`类型的，列表中元素个数是不一定的，但是每个元素的字节长度是固定的。

对于字符串，我们可以在字符串开始前用两个字节来存放它的长度，这样我们的字符串可以存放65536个字符（2的16次方）。对于数组，一样可以用两个字节来存放它的元素个数。

这样我们就可以得到以下的序列化和反序列化代码：

```go
package main

import (
	"fmt"
	"encoding/binary"
)

func main() {
        var s1 = MyStruct {123, "456", []int16{1,2,3}}
        var s2 MyStruct

        s2.Unmarshal(s1.Marshal())

        fmt.Println(s1, s2)
}

type MyStruct struct {
        Field1 int32
        Field2 string
        Field3 []int16
}

func (s *MyStruct) binarySize() int {
        return 4 +                    // Field1
               2 + len(s.Field2) +    // Len + Field2
               2 + 2 * len(s.Field3)  // Len + Field3
}

func (s *MyStruct) Marshal() []byte {
        b := make([]byte, s.binarySize())
        n := 0

        binary.BigEndian.PutUint32(b[n:], uint32(s.Field1))
        n += 4

        binary.BigEndian.PutUint16(b[n:], uint16(len(s.Field2)))
        n += 2

        copy(b[n:], s.Field2)
        n += len(s.Field2)

        binary.BigEndian.PutUint16(b[n:], uint16(len(s.Field3)))
        n += 2

        for i := 0; i < len(s.Field3); i ++ {
                binary.BigEndian.PutUint16(b[n:], uint16(s.Field3[i]))
                n += 2
        }
        
        return b
}

func (s *MyStruct) Unmarshal(b []byte) {
        n := 0

        s.Field1 = int32(binary.BigEndian.Uint32(b[n:]))
        n += 4

        x := int(binary.BigEndian.Uint16(b[n:]))
        n += 2
	
        s.Field2 = string(b[n : n + x])
        n += x

        s.Field3 = make([]int16, binary.BigEndian.Uint16(b[n:]))
        n += 2

        for i := 0; i < len(s.Field3); i ++ {
                s.Field3[i] = int16(binary.BigEndian.Uint16(b[n:]))
                n += 2
        }
}
```

上面用到的协议设计技巧，同样适用于不定长的消息包的发送。在很多应用场景中，消息包的长度是不固定的，就像上面的字符串字段一样。我们一样可以用开头固定的几个字节来存放消息长度，在解析通讯协议的时候就可以从字节流中截出一个个的消息包了，这样的操作通常叫做协议分包或者粘包处理。

贴个从Socket读取消息包的伪代码（没编译）：

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

上面的代码就用到了前一个小贴士中说到的`io.ReadFull`来确保一次读取完整数据。

要注意，这段代码不是线程安全的，如果有两个线程同时对一个`net.Conn`进行`ReadPacket`操作，很可能会发生严重错误，具体逻辑请自行分析。

从上面结构体序列化和反序列化的代码中，大家不难看出，实现一个二进制协议是挺繁琐和容易出BUG的，只要稍微有一个数值计算错就解析出错了。

所以在工程实践中，不推荐大家手写二进制协议的解析代码，项目中通常会用自动化的工具来辅助生成代码。

因为篇幅限制，这篇文章没办法进一步介绍文本协议相关的知识，因为之后有准备讲`bufio`和`json`，这两者都会涉及到文本协议相关的知识，所以就放到以后的文章中再介绍。

