# 一些 Web 构建技术
## CGI
[https://zhuanlan.zhihu.com/p/25013398](https://zhuanlan.zhihu.com/p/25013398)
[https://blog.csdn.net/guodongxiaren/article/details/50569675](https://blog.csdn.net/guodongxiaren/article/details/50569675)

极其古老的动态网页语言。归根结底 CGI就是一个接口协议，各种语言都可以编写CGI。CGI程序通常部署到 Web 服务器上，Web服务器然后调用CGI程序。

我们看到的网页不是静态网页，也就是说在服务端是没有这个网页文件，它是在网页请求的时候动态生成的，比如PHP/JSP网页。依据你请求的参数不同，所返回的内容不同。同理，如果是请求一个CGI程序的时候（ 比如在浏览器直接输入CGI程序的URL，或者提交表单的时候发送给CGI程序），CGI程序负责解析从前端传递过来的参数，理解它的意图然后返回数据，比如返回HTML、XML或JSON等。

CGI有一大硬伤：每次 HTTP 请求 CGI，Web 服务器都有启动一个新的进程去执行这个CGI程序，即颇具 Unix 特色的 fork-and-execute。当用户请求量大的时候，这个 fork-and-execute 的操作会严重拖慢Web服务器的性能。

## Servlet
[https://www.zhihu.com/question/21416727/answer/690289895](https://www.zhihu.com/question/21416727/answer/690289895)
[https://www.zhihu.com/question/21416727/answer/339012081](https://www.zhihu.com/question/21416727/answer/339012081)

事实上，servlet 就是一个Java 接口。servlet 接口定义的是一套处理网络请求的规范，所有实现 servlet 的类，都需要实现它那五个方法，其中最主要的是两个生命周期方法 init() 和destroy()，还有一个处理请求的 service()，也就是说，所有实现servlet接口的类，或者说，所有想要处理网络请求的类，都需要回答这三个问题：你初始化时要做什么、你销毁时要做什么、你接受到请求时要做什么。这是Java给的一种规范。

> servlet 不会直接和客户端打交道。那请求怎么来到servlet呢？答案是 servlet 容器，比如我们最常用的 tomcat。tomcat 才是与客户端直接打交道的家伙，他监听了端口，请求过来后，根据 url 等信息，确定要将请求交给哪个 servlet 去处理，然后调用那个 servlet 的 service 方法，service 方法返回一个 response 对象，tomcat 再把这个 response 返回给客户端。
> 
> 我们为什么能通过Web服务器映射的URL访问资源？肯定需要写程序处理请求，主要3个过程：
>  + 接收请求
>  + 处理请求
>  + 响应请求
> 
> 需要说明的是，tomcat 是 Web 服务器和 Servlet 容器的组合体。
> + 接收请求和响应请求是共性功能，且没有差异性，完成这一部分工作的是 Web 服务器。
> + 处理请求的逻辑是不同的，这里抽取出来做成Servlet，交给程序员自己编写。

Servlet 对 CGI 的最主要优势在于一个 Servlet 运行于 Servlet 容器中，等待以后的请求。每个请求将生成一个新的线程，而不是一个完整的进程。多个客户能够在同一个进程中同时得到服务。
