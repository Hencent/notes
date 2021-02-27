# HTTP 概览
## 不保存状态
Http 是不保存状态的协议，即无状态协议。HTTP 协议自身不对请求和响应之间的通信状态进行保存。在 HTTP 这个级别，协议对于发送过的请求和响应都不做持久化处理。（之后引入了 cookie, token 等）

## Http 中的方法
1. GET：用于获取资源
2. POST：传输实体主体（虽然 GET 也可以用于传输，但是一般不用）
3. DELETE：删除文件
4. PUT：传输文件。和 POST 的区别在于其一般有幂等性 [https://www.zhihu.com/question/48482736](https://www.zhihu.com/question/48482736)
5. HEAD: 和 GET 一样，但是只获得报文首部，不返回主体部分。一般用于确认 URI 有效性和资源更新的时间日期等等
6. OPTIONS：询问支持的方法
7. TRACE：追踪路径
8. CONNECT：要求和代理服务器通信时建立隧道，和隧道进行 TCP 通信。通信时使用  SSL/TSL 加密。

## 持久连接以节省通信量：
   1. 持久连接：只有任意一端没有明确提出断开连接，则保持 TCP 连接状态。减少了 TCP 连接的重复建立和断开带来的开销。必须通信的双方都支持。
   2. 管线化：发送请求后不需要等待并收到响应，直接发送下一个请求。如请求一个包含 10 张图片的 HTML Web 界面，不需要传完一张图再传下一张图。

## URI 与 URL
[https://www.zhihu.com/question/21950864/answer/28847598](https://www.zhihu.com/question/21950864/answer/28847598)

URI 在于I(Identifier)是统一资源标示符，可以唯一标识一个资源。

URL在于Locater，一般来说（URL）统一资源定位符，可以提供找到该资源的路径，比如 `http://www.zhihu.com/question/21950864`，但URL又是URI，因为它可以标识一个资源，所以URL又是URI的子集。

举个是个URI但不是URL的例子：`urn:isbn:0-486-27557-4`，这个是一本书的isbn，可以唯一标识这本书，更确切说这个是URN。

总的来说，locators are also identifiers, so every URL is also a URI, but there are URIs