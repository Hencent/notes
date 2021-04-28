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

# Day 3: 前缀树路由 & 动态路由
路由如果使用 map 来存储的话，那就只能实现动态路由，不能实现带参数的路由了。例如：

`/hello/:name` 可以匹配 `/hello/geektutu` 和 `hello/jack`，且把 name 赋值为 geektutu 或是 jack。

实现动态路由最常见的方法是字典树(前缀树、Trie 树)。HTTP 请求的路径恰好是由 / 分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们在树上查询，如果中间某一层的节点不满足条件，那么就说明无法完成匹配，查询结束。

本次实现的动态路由有两个功能：
+ 参数匹配: 例如 `/p/:lang/doc`，可以匹配 `/p/c/doc` 和 `/p/go/doc`
+ 通配 *： 例如 `/static/*filepath`，可以匹配 `/static/fav.ico`，也可以匹配 `/static/js/jQuery.js`。这种模式常用于静态服务器，能够递归地匹配子路径

## 字典树（路由所用到的树）的数据结构与实现
之前提到，每个由 `/` 分隔的段落都可以看作是一个节点。接下来我们构建每个节点的数据结构。首先，需要一个标识用来指示当前节点是不是一个真正存在的路由节点。虽然可以用一个 bool 值来标识，但是考虑到后面需要方便的获取完整的路由路径，我们干脆就使用一个字符串。非空字符串标识当前节点是一个真正存在的路由节点，且字符串值就是全路径；空字符串表示当前节点不是一个真正的路由节点，而是一个中间的虚节点。其次，我们需要一个字符串来表示当前这个节点是什么，即当前段是什么。再者，也显然需要一个数组来存放子节点指针。最后，需要一个 bool 来表示当前段是否需要动态路由绑定
```go
type node struct {
	pattern     string  // 全路径路由，只在真正的路由节点上才是非空字符串，中间的虚节点是空字符串
	part        string  // 当前段
	child       []*node
	isWildcard  bool
}
```

路由注册时，若出现冲突或不合法的路径，需要返回 error。例如，当前节点已经被其他相符的路由注册（通常出现在通配符混用时），或是 `*` 通配符之后还有别的子段，这都是不合理的。注册的其余流程和字典树相同，找到当前段相符合的节点，往深处继续查找，若是没找到，则新增子节点。

路由查询也是同样的过程，沿着相符合的子段逐个节点向深处查询。需要注意的是，即使当前 pattern 的所有 parts 已经匹配完了，也要保证当前 node 是一个真正被注册的路由节点，而不是中间的虚节点。

## Router 的修改
路由注册部分，除了在记录处理函数的 map 中添加记录，还需要在将 pattern 添加到字典树中

路由处理部分，除了从记录处理函数的 map 中取出相应的处理函数，还需要在路由前缀树上进行解析，判断路由是否存在、解析参数。解析出来的参数需要记录到本次请求的 Context 中。

## Context 的变化
增加一个子段，表示此次请求中解析到的参数。显然也需要新增一个方法来提供访问接口

# Day 4: 分组路由
实际业务逻辑中，通常有一系列相同前缀的路由需要进行相似的处理。例如：
+ 以/post开头的路由匿名可访问。
+ 以/admin开头的路由需要鉴权。
+ 以/api开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

此外，路由分组还需要能够实现嵌套，例如 /post 是一个分组，/post/a 和 /post/b 可以是子分组。

有了路由分组，就可以配合中间件，提供丰富的拓展能力。例如 /admin 的分组，可以应用鉴权中间件； / 分组应用日志中间件。

这里创建一个 GroupRouter 的结构。由于 engine 管理所有的资源，所以所有的 RouterGroup 指针共享同一个 engine 实例。engine 甚至可以当成是顶层的 GroupRouter，拥有一个匿名的 GroupRouter。
```go
type RouterGroup struct {
	preifx string
	middlewares []HandlerFunc
	engine *Engine
}
```

# Day 5:　中间件
中间件就是非业务的技术类组件。中间件可以由用户自定义，嵌入到框架中，**仿佛是框架的原生功能一样**。
> 插入点不能太底层，框架本身就是便捷使用的，太底层了用户用起来麻烦。也不能直接变成用户定义函数，否则和用户用户直接定义一组函数，每次在 Handler 中手工调用相比没有多大的优势了。

两个比较常见的需求是：
1. 框架接收到请求，生成了本次请求的 context之后，先进行一些处理（例如鉴权、记录日志），再让用户函数 handle
2. 用户函数 handle 完之后，再进行一些额外的操作（如记录处理时间等）

中间件是应用在RouterGroup上的，应用在最顶层的 Group，相当于作用于全局，所有的请求都会被中间件处理。

所以，框架的处理逻辑修订为：
1. 当接收到请求后，匹配路由，该请求的所有信息都保存在Context中
2. 查找所有应作用于该路由的中间件，保存在Context中。依次调用用户处理前的中间件
3. 用户处理函数
4. 执行用户处理后的中间件

中间件的调用函数函数如下。这样，如果在中间件执行过程中，调用c.Next()，就会执行下一个中间件！后续内容执行完毕后，再执行当前中间件剩余的部分。假设依次注册了 A、B 两个中间件，再将用户处理函数加入 handlers。执行流程为 
`part1 -> part3 -> Handler -> part 4 -> part2`
```go
func (c *Context) Next() {
	c.index++
	s := len(c.handlers)
	for ; c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}

func A(c *Context) {
    part1
    c.Next()
    part2
}
func B(c *Context) {
    part3
    c.Next()
    part4
}
```