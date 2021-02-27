# 基于 HTTP 扩展的协议/技术
Http 存在一些固有的瓶颈。如：一次连接只能处理一次 http 请求、只能由客户端主动向服务器发起请求，服务器只能被动响应、首部臃肿重复、数据并非强制压缩。

## Ajax
Ajax 指：异步 Javascript 及 XML 技术，是一种利用了 JS 和 DOM 的操作，局部替换 Web 界面的技术。通过 XMLHttpRequest API ，可以在加载完成的页面上发起请求，局部更新界面

## Comet
客户端发起试探性的页面更新检测请求，服务器挂起该请求，直到有页面更新，达到了一种假的“推送”效果。然而，维持链接需要消耗资源

## WebSocket
WebSocket 建立在 TCP 之上。WebSocket 协议是一种全双工的通信协议，通信过程中可以传输格式的数据。发起方使用 HTTP 请求建立连接，然后双方切换到 WebSocket 上进行通信。
WebSocket 可以实现
+ 推送功能：服务器直接发送数据
+ 减少通信录：建立连接之后一直保持连接状态，不需要每次传输都重新建立连接。且首部信息很小，也减少了通信量

> WebSocket 的特点可以用于心跳检测

[http://www.ruanyifeng.com/blog/2017/05/websocket.html](http://www.ruanyifeng.com/blog/2017/05/websocket.html)
[https://www.jianshu.com/p/65ef71ddb910](https://www.jianshu.com/p/65ef71ddb910)