# HTTP 的用户身份认证
## BASIC 认证  
HTTP 1.0 时代的认证方式。当资源需要认证时，服务器返回带有 WWW-Authenticate 首部的 401 响应，要求认证。客户端将字符串 `username:password` 用 Base64 编码之后传输给服务器。服务器再进行校验。

base64 编码并非加密，任何人都可以解码，所以不安全。此外，一般浏览器不支持注销操作。

## DIGEST 认证
HTTP 1.1 开始有的。服务器先发送一个质询码，用户用凭证在本地计算之后得到响应码，直接传输响应码。相比 BASIC 认证更安全。

## SSL 客户端认证
相较于凭证的泄露风险，SSL 客户端认证借助**客户端证书**可以防范这种问题。这一步实际上验证客户端的机器是否合法，但是仍不能确保使用者是用户本人。

## 表单认证
并不是由 HTTP 协议所定义的认证方式。将用户的ID密码等登录信息以表单形式提交给后端服务器。现有的认证基本都是基于表单的认证。

表单认证通常和 session/token 结合起来。
[https://blog.csdn.net/whl190412/article/details/90024671?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-2&spm=1001.2101.3001.4242]()
+ session: 服务器验证之后，给该用户生成一个**识别码** session 返回给用户，并且将该 session 记录在服务端。客户端收到 session 后存在本地，之后的请求里都需要带上 session 用于证明自己的身份。服务器通过查询 session 得到对应的用户名、认证状态等信息。
+ token: 服务器验证用户的用户名密码之后，发送一个 token 给客户端，token 内包含用户名等自定义信息。token 的生成过程是私有的。后续的用户活动都需要带上 token，服务器**解析该 token** 来验证请求是否合法，是否有该权限。

需要说明的是，如果 session/token 被盗，服务器是无法阻止的。