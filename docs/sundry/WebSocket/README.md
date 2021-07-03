# why we need WebSocket
> 参考
> 
> [https://www.jianshu.com/p/3fc3646fad80](https://www.jianshu.com/p/3fc3646fad80)
> 
> [https://www.zhihu.com/question/20215561](https://www.zhihu.com/question/20215561)


HTTP 1.0 时代：HTTP 的生命周期通过 Request 来界定，也就是一个 Request 一个Response。（为了进行一起 HTTP 请求，也完成一次 TCP 的建立与释放）。注意，这里只能是客户端单向的向服务端发送请求，服务端不能主动发送请求，因此某些需要客户端主动通知服务端的场景就受限了（比如 Web 聊天室（注意，我强调了是 Web 环境下，否则直接用 Socket 了）中，服务端需要主动推送消息到客户端）。为了解决这个问题，只能采用轮询的办法，即客户端不停的去问服务器有没有新消息。这占用了大量带宽与服务器资源。

这有一个非常 naive 的解决方案，所谓的长轮询：轮询时，服务器发现没有什么要发送的东西时，采取阻塞的策略，就把链接挂起，一直不返回 Response 给客户端，直到超时或有消息。
+ 对于客户端来说，不管是长轮询还是短轮询，客户端的动作都是一样的，就是不停的去请求
+ 不同的是服务端，短轮询情况下服务端每次请求不管有没有变化都会立即返回结果，而长轮询情况下，如果有变化才会立即返回结果，而没有变化的话，则不会再立即给客户端返回结果，直到超时为止。这样一来，客户端的请求次数将会大量减少，这也就意味着节省了网络流量、节省了服务端的处理资源。但是仍然没有很好的解决服务端需要主动推送的情况，而且由于要保持链接，占用了很多内存等资源。

但是这样还是有问题啊喂！你每次都要建立 TCP 连接，很浪费时间的啦。于是有了 HTTP1.1。HTTP1.1 中，有一个字段 `Connection: keep-alive`，所谓的长连接，其实本质上就是复用 TCP 连接。一个 TCP 连接中，可以有多次 HTTP 请求，这就节省了很多TCP连接建立和断开的消耗。比如请求一个网页，CSS、JS、图片等等一系列资源，可以在一个 TCP 连接上通过多个 HTTP 请求得到。

可是还有问题啊喂！虽然复用了 TCP 连接，但是每次 HTTP 连接都有大量的 header 需要解析，真正的数据部分很少啊。可是这是人家 HTTP 的标准啊，又不能改。不如另起炉灶，我们**基于TCP** 搞一个新的协议吧！ **Websocket** 协议通过第一个 request 建立了 TCP 连接之后，之后的数据交换，没有 HTTP 协议那么多固定的报头，且不用重复建立连接[参考](
https://www.zhihu.com/question/20215561/answer/543897762)。等一下，既然我们改了协议，那就抛弃掉 HTTP 的那种，仅能由客户端单独发起的 request-response 模式好了，就像 Socket 一样搞一个全双工、双向通信的协议好了。于是就有了 **Websocket**。

顺带一提，有答主认为，HTTP 是删除了 TCP 协议的长连接全双工双向特性，WebSocket 把它加回来了罢了。所以 WebSocket 实现的功能是 HTTP 的超集？这是个有点 [意思的理解](
https://www.zhihu.com/question/20215561/answer/58593827)

# WebSocket 和 HTTP 有啥关系啊
答：关系不大。

+ WebSocket 和 HTTP 都是基于 TCP 的**应用层协议**。
+ 最初主要用于 Web 的数据传输（不过之后非 Web 用上了）
+ WebSocket 建立 TCP 连接时，通常借助 HTTP 进行，且在过程中利用 `Upgrade: websocket 和 Connection: Upgrade` 将协议切换到 WebSocket。

# WebSocket 是啥玩意
+ Websocket 是一个**全双工、双向通信**的协议。
+ 除最初建立连接外，直接基于 TCP 完成通信
+ WS连接建立之后，数据的传输使用帧来传递，不需要 Request 消息。
+ WebSocket 使用了自定义的二进制分帧格式，把每个应用消息切分成一或多个帧，发送到目的地之后再组装起来，等到接收到完整的消息后再通知接收端。
+ 所有WebSocket 消息都会按照它们在客户端排队的次序逐个发送。因此，大量排队的消息，甚至一个大消息，都可能导致排在它后面的消息延迟——队首阻塞！

> 参考 (https://mp.weixin.qq.com/s/7aXMdnajINt0C5dcJy2USg?)[https://mp.weixin.qq.com/s/7aXMdnajINt0C5dcJy2USg?]

