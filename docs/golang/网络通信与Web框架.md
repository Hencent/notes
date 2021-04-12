# 网络通信与Web框架
本内容是学习 7days golang - web framework 项目的记录与笔记。[原项目](https://github.com/geektutu/7days-golang/tree/master/gee-web)是一个非常棒的项目。

# Day 1: http.Handler
Go语言内置了 net/http库，封装了HTTP网络编程的基础的接口。

首先来看内置的函数如何注册路由：
```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```
这个函数就是注册路由的函数。第一个参数是请求路径，第二个参数是一个函数类型，表示这个请求需要处理的事情。这里直接将我们传入的处理函数给 DefaultServeMux 处理。DefaultServeMux 是 ServeMux 一个全局实例，在这里可以用来处理路由注册。注册路由主要涉及到两个结构：ServeMux 多路路由器 和 muxEntry 具体路由。
```go
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux

type ServeMux struct {
    mu    sync.RWMutex  // 锁
	m     map[string]muxEntry  // 路由集合
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

type muxEntry struct {
	h       Handler // 路由处理逻辑 是一个接口实例  在每次匹配的时候，调用此接口的方法
	pattern string  // 请求路径
}
```
DefaultServeMux.HandleFunc 其实也是直接调用了 **路由注册函数：mux.Handle**，其实内部就是在一个 map 上新增了 请求路径 到 处理函数 的映射关系。
```go
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

注意到，注册路由时传入的函数类型为 `func(ResponseWriter, *Request)`，在这里转换为 HandlerFunc 适配器类型（类似于接口？**函数类型符合**的函数都可以转换为这个类型），实现了接口 Handler，其实也就是调用用户传入的这个处理函数本身
```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}

type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
这个接口就是 HTTP 处理器。

我们最后启动了服务：
```go
log.Fatal(http.ListenAndServe(":9000", nil))

func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```
第一个参数是地址，:9999表示在 9999 端口监听。而第二个参数则代表处理所有的 HTTP 请求的实例，[**nil 代表使用标准库中的实例处理**](https://github.com/golang/go/blob/8752454ece0c4516769e1260a14763cf9fe86770/src/net/http/server.go#L2881)。自定义框架，实际上就是要自定义第二个参数，即，如何处理 HTTP 请求。

第二个参数的是 Handler，是一个接口，需要实现方法 ServeHTTP ，也就是说，只要传入任何实现了 ServerHTTP 接口的实例，所有的HTTP请求，就都交给了该实例处理了。例如：
```go
type Engine struct {}

func (e *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/":
		fmt.Fprintf(w, "URL_Path=%q\n", req.URL.Path)
	case "/hello":
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q]=%q\n", k, v)
		}
	default:
		fmt.Fprintf(w, "404\n")
	}
}
```
之前只用默认处理器时，我们需要调用 http.HandleFunc 实现了路由和 Handler 的映射，进行静态路由注册。**但是在实现 Engine 之后，我们拦截了所有的 HTTP 请求，拥有了统一的控制入口。** 在这里我们可以自由定义路由映射的规则，也可以统一添加一些处理逻辑，例如日志、异常处理等。

接下来，要实现一个框架，只要丰富这个自定义 Engine 就可以了。

# Day 2: Context 与 Router
## Context
Web 服务实际的任务就是，根据请求 *http.Request，构造响应 http.ResponseWriter。为什么一定要有 Context 呢？

首先，请求信息包含请求首部、请求体等信息。对于请求的查询、解析，很有必要进行封装，避免无谓的重复，防止错误出现，也便于使用。

其次，后端的相应，也不可避免的需要设置状态码、消息类型等内容，也有比较进行封装。对于常见的返回内容：HTML、JSON、String，应该提供一个快捷操作的接口。

其次，为每一次请求与响应提供一个 Context，可以便于处理动态路由、便于加入中间件，**和当前请求强相关的信息都应由 Context 承载**。

```go
type Context struct {
	Writer http.ResponseWriter
	Req *http.Request
	// request info
	Path string
	Method string
	// response info
	StatusCode int
}
```

## 独立的 Router
Router 可以被单独提取出来成为一个模块，方便后续进行动态路由等调整。