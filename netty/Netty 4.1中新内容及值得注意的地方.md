# [Netty 4.1中新内容及值得注意的地方](https://netty.io/wiki/new-and-noteworthy-in-4.1.html)

**此文档将会带你了解4.1相对于4.0的新特性及值得注意的地方**

## 简述(TL;DR)

尽管我们尽我们最大的努力保持对4.0的向后兼容，4.1依然包含许多不能与4.0保持完美兼容的特性。请重新编译你的应用以确认新版本(4.1)。

当你重新编译你的应用时，你可能发现一些异常警告。请使用推荐操作修复它们，这样在你使用新版本时会遇到更少的麻烦。



## 核心改变

### 安卓支持

考虑到以下原因:

 - 移动设备将变得越来越重要
 - 大多数已知的关于NIO和SSLEngine的issues从安卓4.0已经被修复
 - 人民更希望在它们的移动设备中重新启用解码器和处理器

我们觉得正式支持安卓(4.0+)。

无论如何，我们还没有针对安卓的自动化测试工具。如果你发现任何关于安卓的问题，请自由的创建一个issue。同样你可以捐献到我们以保证安卓测试成为项目构建中的一部分。

###`ChannelHandlerContext.attr(..)` == `Channel.attr(..)`

[`Channel`](http://netty.io/4.1/api/io/netty/channel/Channel.html)和[`ChannelHandlerContext`](http://netty.io/4.1/api/io/netty/channel/ChannelHandlerContext.html)都实现了`AttributeMap` 接口用来附加用户的自定义数据。使我们困惑的是`Channel`和`ChannelHandlerContext`使用各自的存储来存放用户的自定义数据。**例如**，假使你使用`Channel.attr(KEY_X).set(valueX)`来设置`KEY_X`，但是你不能使用`ChannelHandlerContext.attr(KEY_X).get()`来读取数据。这种行为不仅使人迷惑而且浪费内存。

为了解决这个issue，我们决定在每个`Channel`内部只保留一个Map。[`AttributeMap`](http://netty.io/4.1/api/io/netty/util/AttributeMap.html) 总是使用[`AttributeKey`](http://netty.io/4.1/api/io/netty/util/AttributeKey.html)来作为它的Key。`AttributeKey`确保对于每个Key保持唯一性，因而确保对于每个`Channel`拥有不超过一个的AttributeMap。只要用户为它们自己的`ChannelHandler`定义私有、静态、不变的`AttributeKey`,那么对于key的重复是没有风险的。

###`Channel.hasAttr(...)`

它现在可以检测一个属性是否存在或有效。

### 更快更明确的缓冲区泄漏跟踪

以前，查找缓冲区泄漏的原因是一件不容易的事，而且泄漏警告也不是那么有帮助。我们现在有了一个更先进但会增加开销的泄漏报告机制。

查看[Reference counted objects](https://netty.io/wiki/reference-counted-objects.html) 查看更多信息。由于它的重要性，这个特性同样会移植到`4.0.14.Final`版本。

### `PooledByteBufAllocator`将作为默认分配器

在4.x版本时代，`UnpooledByteBufAllocator`移植作为默认分配器而不顾它的诸多限制。现在`PooledByteBufAllocator`已经逐渐回归自然并且我们拥有了更先进的泄漏跟踪机制，是时候让它作为默认分配器了。

### 全局唯一通道ID

每个`Channel`会拥有一个全局唯一ID,ID会从以下信息为基础生成:

- MAC地址 (EUI-48 or EUI-64)，优先全局唯一的一个
- 当前进程号PID
- 系统毫秒数`System#currentTimeMillis()`
- 系统纳秒数`System#nanoTime()`
- 一个随机的32位的int
- 一个递增的32位的int

`Channel`的ID可以由方法`Channel.id()`获得。

### `EmbeddedChannel`可用性

[`EmbeddedChannel`](http://netty.io/4.1/api/io/netty/channel/embedded/EmbeddedChannel.html) 中的`readInbound()`和`readOutbound()`方法将会返回一个指定类型参数，你不需要再强转它们的返回值。这将会减少你相当一部分的单元测试代码。

```java
EmbeddedChannel ch = ...;

// BEFORE:
FullHttpRequest req = (FullHttpRequest) ch.readInbound();

// AFTER:
FullHttpRequest req = ch.readInbound();
```

### 使用`Executor`代替`ThreadFactory`

一些应用需要在指定的`Executor`总运行它们的任务。4.x版本中需要用户创建一个事件处理单元时指定一个`ThreadFactory`，但是这都将是过去时了。

关于这个改进的更多信息，请参见[the pull request #1762](https://github.com/netty/netty/pull/1762).

###对类加载器友好

一些类型对运行在容器环境的应用不友好，例如`AttributeKey`，但这种情况也不会再发生了。

###`ByteBufAllocator.calculateNewCapacity()`

计算可扩容`ByteBuf容量大小`的逻辑已经从`AbstractByteBuf`移到了`ByteBufAllocator`，因为`ByteBufAllocator`更清楚的知道它所管理的缓存区的大小。

### 新编/解码器及处理器

- 二进制缓存协议编/解码器
- 压缩编/解码器
  - BZip2
  - FastLZ
  - LZ4
  - LZF
- DNS协议编/解码器
- HAProxy协议编/解码器
- MQTT协议编/解码器
- SPDY/3.1协议编/解码器
- STOMP协议编/解码器
- 支持版本4，4a，5的SOCKSx协议编/解码器，查看socksx包
- 支持XML文档流的[`XmlFrameDecoder`](http://netty.io/4.1/api/io/netty/handler/codec/xml/XmlFrameDecoder.html) 
- 支持JSON对象流的[`JsonObjectDecoder`](http://netty.io/4.1/api/io/netty/handler/codec/json/JsonObjectDecoder.html) 
- IP过滤处理器

## 其他编码变化

### AsciiString

[`AsciiString`](http://netty.io/4.1/api/io/netty/handler/codec/AsciiString.html) 是一个新的只包含一个字节的CharSequence的实现。在处理`US-ASCII`和`ISO-8859-1` 字符串时你会发现它很有用。

举例来说，`HTTP`和`STOMP`编/解码器在Netty中使用`AsciiString`来代表请求头名。因为`AsciiString`转换到`ByteBuf`不需要任何代价，它保证比使用`String`性能更好。

### TextHeaders

[`TextHeaders`](http://netty.io/4.1/api/io/netty/handler/codec/TextHeaders.html) 提供一种通用的数据结构来存储HTTP的头数据就像多重映射的字符串。`HttpHeaders`同样使用`TextHeaders`重写。

### MessageAggregator

[`MessageAggregator`](http://netty.io/4.1/api/io/netty/handler/codec/MessageAggregator.html)提供了通用的函数来合并多个小消息成一个大消息，就像`HttpObjectAggregator`做的一样。`HttpObjectAggregator`同样被`MessageAggregator`重写。

### `HttpObjectAggregator`对容量超限消息更好的处理机制

在4.0中，没有办法在客户端发送内容之前拒绝一个容量超限的HTTP消息，即使这个内容需要分割成100个。

在这次发版中，添加一个可重写的`handleOversizedMessage`方法，这个方法可以帮助用户使用自己的方法处理容量超限消息。默认情况下，对于容量超限消息，服务端会返回`413 Request Entity Too Large` 并断开连接。

### `ChunkedInput` and `ChunkedWriteHandler`

[`ChunkedInput`](http://netty.io/4.1/api/io/netty/handler/stream/ChunkedInput.html)有两个新方法，`progress()` 返回当前转换的处理器，`length()`返回各自流的总长度， [`ChunkedWriteHandler`](http://netty.io/4.1/api/io/netty/handler/stream/ChunkedWriteHandler.html)将会使用这两个信息通知 [`ChannelProgressiveFutureListener`](http://netty.io/4.1/api/io/netty/channel/ChannelProgressiveFutureListener.html).

### `SnappyFramedEncoder` and `SnappyFramedDecoder`

这两个类已经被重命名成`SnappyFramedEncoder` 和 `SnappyFramedDecoder`，老版本已经被标记成删除，并且老版本实际上是新版本的子类。



Last retrieved on 27-Jul-2018









