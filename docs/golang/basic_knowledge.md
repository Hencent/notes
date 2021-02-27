# 环境
## go modules
很久很久以前，go 的所有项目，不管是第三方的还是自己的，都必须要放到 GOPATH 下，统一维护和管理,代码中任何import的路径均从GOPATH为根目录开始。  

当有多个项目时，不同项目对于依赖库的版本需求不一致时，无法在一个GOPATH下面放置不同版本的依赖项。典型的例子：当有多项目时候，A项目依赖C 1.0.0，B项目依赖C 2.0.0，由于没有依赖项版本的概念，C 1.0.0和C 2.0.0无法同时在GOPATH下共存，解决办法是分别为A项目和B项目设置GOPATH，将不同版本的C源代码放在两个GOPATH中，彼此独立（编译时切换），或者C 1.0.0和C 2.0.0两个版本更改包名。无论哪种解决方法，都需要人工判断更正，不便利性。  

在Go Modules之前，没有任何语义化的数据可以知道当前项目的所有依赖项，需要手动找出所有依赖  

于是，Go Modules诞生了。Go Modules是语义化版本管理的依赖项的包管理工具

1. 使用第三方的项目而言：
    以往 go get 的使用会把所有代码和第三方工具全部下载到 GOPATH 的 src 下，和本地项目混杂在一起。ugly且无法对依赖包进行版本控制，只有master分支能代表一个包的稳定版本。 

    有了包管理工具之后，依赖的第三方包被下载到了 `GOPATH/pkg/mod` 路径， `GOPATH/pkg/mod` 里可以保存相同包的不同版本。且在自己的项目中可以使用指定版本的包。（使用 go.mod 文件。）  

    go.mod 提供了module, require、replace和exclude四个命令  
    + module语句指定包的名字
    + require语句指定的依赖项模块 （可以特定版本了）
    + replace语句可以替换依赖项模块 （如果在代码调试过程中，涉及到修改其他依赖项目代码，或者其他原因需要引用本地包，可以采用 replace 机制）
        ```go
        require (
            golang.org/x/crypto v0.0.0
        )
        replace golang.org/x/crypto v0.0.0 => ../crypto
        ```
    + exclude语句可以忽略依赖项模块

2. 自己的项目作为 package 而言：
    + `go mod init github.com/Hencent/XXX` 创建 modules。会在目录下生成 go.sum 文件。
    + 在将代码推送至Github之后，其他人可以使用 `go get github.com/Hencent/XXX` 下载本项目
    + 我们可以通过使用git版本标签来发布版本或修改
3. go mod 更新/下载  
    可使用 go list 查看版本，可使用 go get 下载/更新

# 特性
## 栈逃逸机制
GO 函数中允许返回局部变量的地址。Go编译器使用“栈逃逸”机制将这种局部变量的空间分配在堆上：  
```go
    func sum(a, b int) *int {
        sum := a+b
        return &sum
    }
```

## 闭包
闭包 = 函数 + 引用环境。也可以说，闭包是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。实际上，这是利用了“栈逃逸”机制。  

一般来说，由 匿名函数引用外部函数的局部变量或者是包全局变量 构成一个闭包。闭包通常用于减少全局变量，避免污染全局空间。  
例如下列函数可以生成一个从 i 开始的自增器，又不污染全局空间。**所以不推荐在闭包中引用全局变量**
```go
func getSequence(i int) func() int {
    return func() int {
        a := i
        i+=1
        return a  
    }
}
increaser := getSequence(0)
a := increaser()
b := increaser()
c := increaser()
// a b c 分别为 0 1 2
```  
当闭包所引用的外部变量是函数的局部变量或是参数时：  
多次调用生成匿名函数的函数，返回的多个闭包所引用的外部变量（例如 i）是多个副本，各自拥有独立的内存地址。但是多次调用一个闭包函数多次，闭包函数内对其引用的外部变量的操作会影响到该闭包所关联的副本。  
另外需要注意的是，同一个函数若是同时返回了多个闭包，这些闭包共享该函数的局部变量。  

当闭包所引用的外部变量是全局变量时： 
每次操作都会影响全局变量。

# 基本语法
## 切片
当由 数组/切片 创建切片的时候，切片的低层空间与原有的 数组/切片 共用，所以修改要谨慎。但是如果随便加入元素，导致拓展的话，就会不共享底层空间了。

## map
map 是引用类型，复制与传参时，会共享底层的空间。实际上传递的内容可以看作是指针值。  

如果 map 的 value type 是一个复杂结构，不要直接修改复杂结构中的值，只能重新赋值新的复杂结构。

## struct
内部连续存放，有对齐要求。

# 函数
## 不定参数
使用 `param ...type` 来声明。

## defer
defer 注册延时调用，在本函数返回前，按照注册的顺序，先注册的后执行。 defer 只能注册方法与函数，不能注册语句，可以使用匿名函数规避此问题。  
defer 可以用于避免资源泄露，例如及时关闭管道避免遗忘，关闭数据库连接等等。  
defer 中避免对有名的返回值操作。

## 底层实现
Go　函数使用的是　caller-save 模式，即由调用者负责保存栈寄存器，在主调函数调用被调函数的头尾会有保存现场和回复现场的操作。
+ 多值返回　　
    函数调用前先为返回值和参数分配空间。先分配返回值空间，再分配参数空间。需要返回多少个参数就提前分配多少空间，所以多值返回的实质就是在栈上开辟多个地址空间分别存放返回值。如果返回值要存在栈上，那么多了一个复制的动作，复制过去而已。
+ 闭包实现　　
    Go 的闭包通过如下结构实现
    ```go
    type Closure struct {
        F uintptr
        env *Type
    }
    ```  
    F是返回的匿名函数指针，env是对外部环境变量的引用集合。检测到闭包之后编译器会生成上述结构的内存空间并存储，将闭包所引用的外部变量复制到闭包对象副本的位置中去（引用函数局部变量或是参数时），之后函数内部对外部环境变量的操作会在这个副本上进行。

## 函数类型
有名函数与匿名函数**的类型**都属于未命名类型，也叫作**函数字面量类型**，而使用 `type` 定义的函数类型叫做函数命名类型。  
例如 `func add(int, int) int{}` 和 `func (int, int) int{}` 分别是有名函数和匿名函数，他们的类型就是函数签名：  函数签名是不包含函数名的“字面量类型”，如 `func (int, int) int`  
而 `函数声明=函数名+函数签名`, 如 `func FuncName (int, int) int` 

## 方法集
一个类型的方法集包括，接受者是值类型 T 的方法集S，接受者为指针类型 *T 的方法集 *S。  
T类型的方法集是 S， *T类型的方法集是 S 和 *S。实际上，使用类型实例调用方法时，编译器会进行自动转换，转换成合适的类型。但是需要注意的是，**类型实例传递给接口的时候，编译器不会进行自动转换**，而是会进行严格的方法集校验。

# 类型系统
## 命名类型、未命名类型、底层类型
命名类型指的是类型可以用标识符来表示，包含 20 个预声明类型和用户自定义类型。  
未命名类型也称为类型字面量。一般而言，这些基本都是复合类型，如 map, 数组，切片等。  
底层类型：预声明类型和类型字面量（未命名类型）的底层类型是他们本身。自定义类型则需要逐层递归查找。

这里添加一个便于理解的例子。
```go
*int, []int, map[string]int 都是未命名类型，也叫作类型字面量
而 type Map map[string]int, type IntPoint ×int 定义之后，Map和 IntPoint 都是命名类型中的用户自定义类型。
```

## 类型赋值
命名类型与非命名类型一定不相同，而满足下列两个条件时类型相同。
+ 命名类型的类型声明语句相同时
+ 未命名类型的类型字面量结构与内部元素类型相同时

当 a 是类型 T1.赋值 `var b T2 = a` 合法的情况是：
+ T1 与 T2 的类型相同
+ T1, T2 有相同的底层类型，且其中至少有一个是未命名类型

## 类型方法
使用如下方式定义一个命名方法：
```go
func (t TypeName) MethodName(ParamList) (ReturnList) {}
func (t *TypeName) MethodName(ParamList) (ReturnList) {}
```
Go语言的类型方法本质上就是一个函数，显示传递了对象实例或指针且可以自己命名，而不是像C++那样用this指针。  
只能为同一个包中的命名类型增加方法，所以不能为未命名类型和内置预定义类型增加方法。  
使用 type 定义的新类型不能调用原有类型的方法，但是底层类型支持的运算可以被新类型继承
```go
// 下面的 Map 可使用底层支持的运算 range
type Map map[string]int
func (m Map) MethodName() {
    for index, key := range m {
        .....
    }
}
```

## 组合
命名结构类型可以嵌套其他命名字段，在Go中，叫做组合而不是继承，是 "has a" 关系而不是 “is a”关系。
### 一般组合
直接将其他的类型作为自己的内部命名成员
```go
type Wifi struct {
    Name string
}

type Phone struct {
     WifiMdl Wifi
}
```
### 匿名组合
匿名组合也可以叫做内嵌一个类型。

> 当我们嵌入一个类型，这个类型的方法就变成了外部类型的方法，但是当它被调用时，方法的接受者是内部类型(嵌入类型)，而非外部类型。

内部类型的成员名可理解为内部类型名，在外部类型未定义相同名称的属性和方法时，内部方法可提升为外部类型的属性和方法，但实际上，属性和方法的所有者和调用的接受仍然为内部类型。

```go
type Wifi struct {
    Name string
}

type Phone struct {
     Wifi
}
可以使用 Phone 对象直接调用 Wifi 的方法。  
```

需要注意的是， 外层可以定义和内嵌字段方法同名的方法，使得 struct 能够重写内嵌字段的方法。


# 接口
## 基本概念
接口变量有的 方法声明 与 方法值变量。  
接口声明中包含数个**方法声明**（是带有方法名的，不是方法签名）  

接口变量可以绑定到某个具体类型实例上 ，（该实例实现了该方法，也就是该实例的方法集是接口方法集的超集），接口调用方法实际上就是调用其绑定的实例的方法。  

接口有动态类型和静态类型的概念。接口的动态类型指的是接口绑定的具体实例的类型是动态的，具体实例的类型称为接口的动态类型。而接口能绑定什么实例是由接口的方法签名集合（？签名还是声明，带不带函数名？）所决定的，这称为接口的静态类型。  

接口赋值，也就是接口初始化时会进行**静态类型检查**，具体类型实例的方法集必须是接口方法集的超集 **（特别是指针类的类型方法，如果直接赋值一个实例会导致方法集不全，比如没有 set ）** 。用接口变量赋值同理。

## 空接口
没有任何方法的接口称为空接口，任何类型都符合空接口的要求。  

空接口可用于 泛型（参数化类型，空接口用于接收任意类型参数） 与 反射。  

空接口并不是真的是空的，实际上它有两个字段：绑定的实例的类型 和 绑定的实例的指针。当且仅当这两个都是 nil 的时候，空接口才是 nil。所以不能用 `接口变量 == nil` 来判断接口绑定的实例指针是不是空，有可能空接口绑定的是某类型的空指针。

## 接口类型检查
接口变量可以绑定实例（实例值或者实例指针值）
### 类型断言
```go
value, ok := i.(TypeName)
```

1. TypeName 是具体类型名，则类型断言用于判断接口变量 i 绑定的实例的类型是不是 TypeName
2. TypeName 是接口类型名，则类型断言用于判断接口变量 i 绑定的**实例**是否**实现了接口** TypeName
  
i 必须是接口变量而不是具体类型变量。 ok 为 `true` or `false`，表示断言的结果。
1. 如果断言为真，则 value 是接口绑定的实例值的副本（如果实例是指针值，那么就是指针值的副本）；或 value 是接口类型 TypeName 变量，且其底层绑定的实例是 i 绑定的实例的副本（指针同理）。
2. 如果断言为假，则 value TypeName 类型的零值，没有任何意义，不应当使用它。

如果确信断言结果为真，可以简写为 `value := i.(TypeName)`，但如果结果为假就会导致抛出 panic

### 类型查询
```go
switch v := i.(type) {
case type1:
    xxx
case type2:
    xxx
case nil:  // 例如空的接口变量
    xxx
default:
    xxx
}
```
其中 type1, type2 ... 可以是接口类型名或实例类型名。需要注意的是，case 后面可以跟着多个 type，只要有一个符合，就等价于 v:= o，比较奇怪，需要留意一下。

## 接口的内部实现
### 数据结构
非空接口的底层数据结构是 iface。非空接口初始化的过程就是初始化一个 iface 的过程。
```go
type iface struct {
    tab *itab
    data unsafe.Pointer
}
```
+ itab 用来存放 **接口自身类型** 和 **绑定的实例的类型** 以及 **实例相关的函数指针**  
+ data 是个指针，指向接口绑定的**实例的副本**或者是**指针的副本**，接口的初始化也是一种值拷贝（哪怕传递的是指针，也是赋值了一个指针）。

itab 数据结构是接口内部实现的核心和基础，itab的信息存放在静态分配的存储空间中，不受到GC的限制，不会被回收。itab 表示 interface 和 实际类型的转换信息。对于每个 interface 和实际类型，只要在代码中存在引用关系， go 就会在运行时为这一对具体的 <Interface, Type> 生成 itab 信息。
```go
type itab struct {
    inter *interfacetype  // 接口自身的静态类型
    _type *_type  // 接口对应的具体实例的类型，也就是接口的动态类型
    hash uint32
    _ [4]byte
    fun [1]uintptr
}
```
+ inter 是接口自身的静态类型的信息（接口类型的元信息）
+ _type 是接口对应的具体类型的信息，也就是接口的动态类型的信息（具体实例的元信息）。注意的是，这里存放的对应的具体类型的类型信息，而 iface 中的 data 指向的是该类型的值。
+ hash 具体类型的 Hash 值，在 _type 中也有一份，用于接口断言或是类型查询
+ fun 是函数指针，有点像 CPP 中的虚函数指针。

_type ：Go是强类型语言，编译器在编译时会进行严格的类型检查，所以需要为每一个类型维护一个类型的元信息。所有其他类型都是以 _type 为内嵌字段封装而成的结构体。**_type 是类型的类型元信息**。  
Go语言的类型元信息由编译器负责构建，并以表的形式存放在编译后的对象文件中，再由链接器在链接时进行段合并、符号重定位。这些类型信息会在接口的动态调用与反射中被**运行中引用**

上面说的是类型的类型元信息。下面看一下接口类型的元信息
```go
type interfacetype struct {
    typ _type
    pkgpath name  // 接口所属包的信息
    mhdir []imethod  // 接口的方法
}
```

### 接口的调用过程与代价
接口调用过程：
1. 构建 iface 动态数据结构。在接口实例化的时候完成该过程。
2. 通过函数指针间接调用接口绑定的实例方法

接口调用代价：
1. 接口实例化的过程，也就是 iface 的创建过程。一旦实例化完成之后，这个接口和具体类型的 itab 数据接口是可以复用的
2. 接口的方法调用，是函数指针的间接调用。需要注意的是，这里的调用是一种动态的计算后的调用，会导致CPU缓存与分支预测失效，带来损耗

### 空接口的数据结构
空接口是没有任何方法集的接口，所以空接口内部不需要维护和动态相关的数据结构 itab。空接口只关心存放的具体类型是什么，具体类型的值是什么。所以空接口的底层数据结构 eface 如下：
```go
type eface struct {
    _type *type
    data unsafe.Pointer
}
```
空接口自身没有方法集，所以空接口实例化之后的真正用途并不是接口方法的动态调用。空接口真正的意义是支持多态，这一步需要将空接口类型还原，方法有：
+ 接口类型断言
+ 接口类型查询
+ 反射

# 并发 与 通信
## goroutine
routine 指例程，这里就是 Go 例程。go routine 是用户级线程，是 Go 的并发执行体。通过 `go + 匿名函数/命名函数` 的方式启动一个 goroutine。goroutine 的特点是：
+ 返回值被忽略
+ 调度器不保证多个 goroutine 的执行顺序
+ 没有父子 goroutine 的概念，每个 goroutine 之间是平等的调度与执行的
+ 程序执行时，为 main 创建一个 goroutine，遇到其他 go 关键字再创建别的 goroutinue
+ go 没有暴露 goroutine id，难以在一个 goroutine 里面显示操作另一个 goroutine，可使用 runtime 包进行有限的操作，包括但不限于：
  + func GOMAXPROCS 用于设置和查询可以并发执行的 goroutine 数目
  + func Goexit 结束当前 goroutine 的执行，结束前会调用已经注册的 defer
  + func Gosched 放弃当前调度执行的机会，将当前的 goroutine 放到队列中等待下一次的调度

## chan
*不要通过共享内存来通信，而是通过通信来共享内存*  
chan 即 channel。Go 通过 make 来创建通道， `close(chan)` 之后，缓冲中是数据不会丢失，除非 channel 的生命周期结束。

```go
make(chan datatype)  // 通道元素是 datatype 的通道, 可以用于同步功能
make(chan datatype, 10) // 通道元素是 datatype 的通道，有 10 个缓冲
```
+ 向未初始化的通道读或写数据会导致永久阻塞
+ 向缓冲区已满的通道写数据会导致 goroutine 阻塞
+ 通道中没有数据的时候，读取通道会导致 goroutine 阻塞

### go select 与 fan in/out
#### select
select语句只能用于信道的读写操作, 会随机选择一个可以执行的信道操作，如果没有可以操作的信道，则阻塞。空的 select{} 会带来死锁。
```go
select {
case ch1 <- data:
    // 如果成功向 ch1 信道成功发送数据，则执行该分支代码
case ch2 <- data:
    // 如果成功向 ch2 信道成功发送数据，则执行该分支代码
default:
    // 如果上面都没有成功，则进入 default 分支处理流程
}
```

#### fan in/out
扇入是指将多路通道聚合到一路通道之中，例如 select。扇出是指将一条通道发送到多条通道中处理，例如使用多个 goroutine。当生产者速度很慢，可以使用多个生产者并且扇入聚合来满足消费者需求，例如耗时的加密解密服务。当消费者速度很慢的时候，需要使用扇出技术，比如 Web 并发。

### 退出通知机制
+ 读取已经关闭的通道不会阻塞或 panic，而是会返回该通道的零值 **!important**
+ 关闭 select 的某个监听，select 能够立即感知，进行相应处理，不再选择该分支

## sync 与 WaitGroup 的同步
```go
var wg sync.WaitGroup

// 每个 go routinue 中
wg.Add()
defer wg.Done()

// 等待 其他goroutinue 执行完毕的 goroutinue
wg.Wait()
```

## 并发范式
### 生成器
1. 多个 goroutine 增强型生成器  
    ```go
    func GenerateIntA() chan int {
        ch := make(chan int, 10)
        go func() {
            for {
                ch <- rand.Int()
            }
        }()
        return ch
    }
    func GenerateIntB() chan int {
        ch := make(chan int, 10)
        go func() {
            for {
                ch <- rand.Int()
            }
        }()
        return ch
    }
    func GenerateInt() chan int {
        ch := make(chan int, 20)
        go func() {
            chA := GenerateIntA()
            chB := GenerateIntB()
            for {
                select {
                    case ch <- <- chA:
                    case ch <- <- chB:
                }
            }
        }()
        return ch
    }
    ```
2. 借助 Go 通道的退出通知机制，使生成器自动退出
    ```go
    func Generater (done chan struct{}) chan int {
        ch := make(chan int)
        go func() {
            Lable:
                for {
                    select {
                    case ch <- rand.Int():
                    case <- done:
                            break Lable
                    }
                }
                close(ch)
        }()
        return ch
    }

    func main() {
        done := make(chan struct{})
        ch := Generater(done)

        fmt.Println(<-ch)
        fmt.Println(<-ch)

        close(done)

        fmt.Println(<-ch)
        println("num of goruntine:",runtime.NumGoroutine())
    }
    ```
3. 融合了并发、缓冲、退出通知的多重特性生成器，此处略

### 每个请求一个 goroutine
每来一个请求就启动一个 goroutine 去处理，典型代表就是 Go 中的 HTTPserver

### woker 工作池
构建固定数目的 goroutinues 作为工作池。

对于 routine, 除了 main routine，还包括了 分发任务的 goroutine。程序中有两个通道，分别是：传递 task 任务的通道，接受 task 结果的通道。

可以结合 ```select i.(type)``` 一起用

> Tip:这里简化了书上的例子，假设系统无限制工作，不考虑结束任务处理携程

```go
const (
    WORKER_NUMBER = 10
    MAX_TASK_CACHE = 10000
    MAX_RESULT_CACHE = 10000
)

type task struct {
    taskID  // 自定义的需要的数据结构
    result  chan result_type // 用于任务处理完成后，返回信息
}

func (t *task) do() {
    // handle task
}

func main() {

    // 待处理工作，可以当成是 消息队列？
    taskchan := make(chan task, MAX_TASK_CACHE)

    // task 处理完成后的结果消息队列
    resultchan := make(chan result_type, MAX_RESULT_CACHE)

    // worker 信号通道
    done := make(chan struct{}, MAX_TASK_CACHE)

    generate_task(taskchan)  // 生成任务的函数，自定义

    go generate_worker(taskchan)

    go result_handle(resultchan)
}

func generate_worker(taskchan) {
    for i := 0; i < WORKER_NUMBER; ++i {
        go Process_task(taskchan)
    }
}

func Process_task(taskchan) {
    for t : range(taskchan) {
        t.do()
    }
}

func result_handle(resultchan) {
    for t : range(resultchan) {
        // 处理每一个结果，也可以改成并发的形式
    }
}

```

### future 模式
在一个流程中需要调用多个子调用，这些子调用之间没有任何依赖，如果串行调用会很耗时，可以考虑使用 future 模式。future 模式的基本工作原理：
+ 使用 chan 作为函数参数，并通过 chan 传入参数
+ 启动 goroutinue 调用函数，可以做其他可以并行处理的事情
+ 通过 chan 异步获取结果
```go
type query struct {
    sql chan string // (实际上可以是更复杂的数据结构，而不只是 string)
    result chan string // (实际上可以是更复杂的数据结构，而不只是 string)
}

func execQuery(q query) {
    go func() {
        sql := <- q.sql;

        res := operation// 进行数据库操作

        q.result <- res
    }
}

func main() {
    q := query(make(chan string, 1), make(chan string, 1))

    go execQuery(q)

    q.sql <- "select * from table"

    // 做其他事

    // 获取结果
    fmt.Println(<- q.result)
}
```
future 可以将函数的同步调用转换为异步调用，适用于一个交易需要多个子调用且这些子调用之间没有依赖的场景。

## context 标准库
GO 中的 goroutinue 之间没有父子关系，也没有所谓的子进程退出后的通知机制，多个 routinue 平行调度，所以涉及到通信、同步、通知和退出。
+ 通信： chan 通道
+ 同步： 不带缓冲的 chan 或者 sync.WaitGroup
+ 通知： 通信中指的是业务数据。通知指的是管理、控制数据。可以绑定多个 chan 来解决
+ 退出： routinue 之间没有父子关系。可使用一个单独的通道实现退出

go 的 context 标准库可以解决这个问题，提供了两种功能：退出通知 和 元数据传递。 context 库的目的是跟踪 routinue 调用，在内部维护一个调用树，在其中传递通知和元数据，来实现：
+ 退出通知机制——传递给 routine 调用树上的每一个 routine
+ 传递数据——传递给 routine 调用树上的每一个 routine

个人认为 上下文 这个翻译真的精准独到

### context 的使用
#### context 树根节点
要创建 context 树，第一步是要有一个根结点。context.Background() 函数的返回值是一个空的 context，经常作为树的根结点，它一般由接收请求的第一个 routine 创建，不能被取消、没有值、也没有过期时间。
#### 创建子孙节点
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```
第一个参数都是父 context ，返回一个 Context 类型的值，这样就层层创建出不同的节点。子节点是从复制父节点得到的，并且根据接收的函数参数保存子节点的一些状态值，然后就可以将它传递给下层的 routine 了。
+ WithCancel 函数，返回一个额外的CancelFunc函数类型变量，该函数类型的定义为： `type CancelFunc func()`。调用 CancelFunc 对象将撤销对应的 Context 对象，这样父结点的所在的环境中，获得了撤销子节点 context 的权利，当触发某些条件时，可以调用 CancelFunc 对象来终止子结点树的所有 routine。在子节点的 routine 中，需要用类似下面的代码来判断何时退出 routine
    ```go
    select {
        case <-cxt.Done():
            // do some cleaning and return
    }
    ```
+ WithDeadline 和 WithTimeout 比 WithCancel 多了一个时间参数，它指示 context 存活的最长时间。如果超过了过期时间，会自动撤销它的子 context。所以 context 的生命期是由父 context 的 routine 和 deadline 共同决定的。
+ WithValue 返回 parent 的一个副本，该副本保存了传入的 key/value，而调用 Context 接口的 Value(key) 方法就可以得到 val。注意在同一个 context 中设置 key/value，若key相同，值会被覆盖


### context 的基本数据结构
第一个创建 Context 的 goroutine 称为 root 节点。root节点负责创建一个实现 Context 接口的具体对象，并将该对象作为参数传递到其新拉起的 goroutine，下游的 goroutine 可以在封装、再传递，最终形成一个树状的数据结构。使用位于 root 节点处的 Context 就可以遍历整个 Context 树，消息和通知就可以从 root 节点传递出去，实现上游 goroutine 对下游 goroutine 的消息传递。

#### Context 接口
Context 是一个基本接口，所有　Context 对象都要实现该接口，context 的使用者在调用接口中都使用 Context 作为类型参数。
```go
type Context interface {
    // 如果其实现了超时控制，deadline 为超时时间，ok 为 true；否则 ok 为false
    Deadline() (deadline time.Time, ok bool)

    // 后端被调的 context 应该监听该方法返回的 chan，以便及时释放资源
    Done() <- chan struct{}

    // Done 返回的 chan 收到通知的时候，才可以访问 Err 获知为什么取消
    Err() error

    // 访问上游传给下游 goroutine 的值
    Value(key interface{}) interface{}
}
```

#### cancaler 接口
canceler 接口是一个拓展接口，规定了取消通知的 Context 具体类型需要实现的接口。一个 Context 对象如果实现了 cancler 接口，则可以被取消。
```go
type cancler interface {
    // 创建 cancler 接口实例的 goroutine 调用 cancler 方法通知后续创建的 goroutine 退出。
    cancel(removeFromParent bool, err error)

    // Done 方法返回的 chan 需要后端 goroutine 来监听，并及时退出
    Done() <-chan struct{}
}
```

#### context package 构造 Context 的 root 节点
package 中有：
```go
var (
    background new(emptyCtx)
    todo new(emptyCtx)
)
func Background() Context {
    return background
}
func TODO() Context {
    return todo
}
```

#### empty Context 结构
emptyCtx 是一个具体类型，实现了 Context 接口，但是没有任何功能，所有其实现的方法都是空方法。其存在的目的是作为 Context 对象树的 root。context 包的使用思路就是不停的调用 context 包提供的包装函数来创建具有特殊功能的 Context 实例，每一个 Context 实例的创建都是以上一个 Context 对象作为参数。
```go
type emptyCtx int
func (*emptyCtx) Deadline() (deadline time.Time, ok book) {
    return
}
func (*emptyCtx) Done() <- chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}
```

#### cancelCtx
cancalCtx 是一个具体类型，实现了 Context 接口，同时实现了 canceler 接口。其具有退出通知方法，不但能通知自己，也能逐层通知其 children 节点。
```go
type cancelCtx struct {
    Context
    done chan struct{}  // closed by the first cancel call

    mu sync.Mutex
    children map[canceler]struct{}  // set to nil by the first cancel call
    err error  // set to non-nil by the first cancel call
}

func (c *cancelCtx) Done() <-chan struct{} {
    return c.done
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.error
}

func (c *cancelCtx) cancel(removeFromParent bool, err Error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.error = err  // 显示地通知自己
    close(c.done)
    // 调用每个 child 的 cancel。由于 parent 已经取消，所以此时调用的 calcel 传入 false
    for child := range children {
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

#### timerCtx
timerCtx 是一个实现了 Context 的具体类型，内部封装了 cancelCtx 类型实例，同时有一个 deadline 变量，用来实现定时退出通知
```go
type timerCtx struct {
    cancelCtx
    time *time.Timer  // under cancelCtx.mu.
    deadline time.Timer
}

func (c *timerCtx) Deadline() (deadline time.Timer, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) cancel (removeFromParent bool, err error) {
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

#### valueCtx
timerCtx 是一个实现了 Context 的具体类型，内部封装了 Context 类型实例，同时封装了一个 k/v 的存储变量。valueCtx 可以用来传递通知信息。context 上下文数据的存储就像一个树，每个结点只存储一个 key/value 对。
```go
type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.value(key)  // 若当前节点没有，则递归向父节点找
}
```
### context 用法
1. 创建一个 context 根对象
   （例如借助 func Background())
2. 包装上一步创建的 Context 对象，使其具有特定的功能
3. 将上一步包装创建的对象作为实参传递给后续启动的并发函数。每个并发函数内部还可以重复 2、3 来实现自己的功能
4. 顶端的 goroutine 在超时等需要退出时，调用 cancel 通知函数，通知子树中所有 goroutine 释放资源
5. 子树中的 goroutine 通过 select 监听 Context.Done() 返回的 chan，及时响应其父亲 goroutine 的退出通知。一般停止本次处理，释放所占用的资源。

| Tip: 也可以自己封装更多的 context，但是要实现很多东西很麻烦。

例子：
```go
func main() {
    ctxa, cancel := context.WithCancel(context.Background())
    go work(ctxa, "work1")

    tm := time.Now().Add(3 * time.Second)
    ctxb, _ := context.WithDeadline(ctxa, tm)
    go work(ctxbb, "work2")

    ctxc, _ := context.WithValue(ctxb, "key", "value of k/v")
    go workWithValue(ctxbc, "work3")

    time.Sleep(10 * time.Second)
    cancel()

    time.Sleep(5 * time.Second)
}

func work(ctx context.Context, name string) {
    for {
        select {
            case <- ctx.Done():
                fmt.printf("%s get msg to cancel\n", name)
                return
            default:
                fmt.printf("%s is running\n", name)
                time.Sleep(1 * time.Second)
        }
    }
}

func workWithValue(ctx context.Context, name string) {
    for {
        select {
            case <- ctx.Done():
                fmt.printf("%s get msg to cancel\n", name)
                return
            default:
                value := ctx.Value("key").(string)
                fmt.printf("%s is running, value = %s\n", name, value)
                time.Sleep(1 * time.Second)
        }
    }
}
```
实际上，程序维护了两条关系链。
+ 一条是 Context 的 children key 构成的根到叶子的引用关系，使得取消广播能够沿着链传递到下层节点，直至叶子结点。
+ 另一条是 Context 对象获取自身包裹的 Context 对象，自底向上查找。用于自身取消之后，把自己从广播树上清除。其实也用于 Value 的逐层查找。自 Context 可查到父 Context 的 key-value 信息。

## 并发模型
### 调度模型
+ 多进程模型  
  + 每个进程都有自己独立的内存空间，隔离性好，健壮性强
  + 进程比较重，切换开销大，进程间通信需要在内核区用户区之间复制数据
+ 多线程模型
  + 通过共享内存通信，线程切换代价小
  + 多线程共享内存，可能导致数据访问混乱。某个进程误操作可能会导致整个线程组挂掉，健壮性差  
    > Tip: 实际上，进程线程都需要上下文切换，只不过线程组间的内存资源指针指向同一块地址空间。这会导致线程的 cache/TLB 命中的概率比进程高很多，缺页中断少，数据写回少。这是线程调度开销少的根本原因。
+ 用户级多线程模型
  1. 分为 M:1 和 M:N 两种。前一种还是会存在一个线程阻塞导致所有线程阻塞，后一种若系统线程数量过多会导致操作系统调度开销过大，单个线程时间片太少。
  2. 新的概念：协程。  
    协程是一种用户级别的轻量级线程，协程的调度完全由用户态程序控制。协程拥有自己的寄存器上下文和栈，协程调度切换时，会保存恢复寄存器上下文(AX BX PC IR 等等)和栈。  
    **每个内核线程可以对应多个用户协程。当一个协程执行体阻塞，调度器会调度另一个协程执行。** 当然，其也可以使用 M:N 模型。  
    好处显而易见：
       + 控制了系统线程总数，使每个系统线程时间片充足
       + 调度层可以进行用户态的切换(避免内核态用户态频繁切换)，不会因为单个协程阻塞整个程序，减少程序上下文切换。

### GO 的 goroutine 调度模型
goroutine 调度模型有三个实体：M, P, G

G(Goroutine) 是对 goroutine 的抽象描述，存放了并发执行的代码入口、上下文、运行环境（关联的 M 和 P）、运行栈等信息。Go runtime 的监控线程会监控 G 的调度。G 新建或恢复时会加入运行队列，等待 M 取出并执行。为了减少对象分配回收，G 是可复用的。

M(Machine) 表示内核线程，是系统层面调度运行的实体。 M 不停的被唤醒/创建，然后执行。唤醒/创建时，会首先执行 Go runtime 管理代码，获取 G 和 P，然后执行调度。 Go runtime 有一个监控线程会对内存、调度监控控制。

P(Processer)是一个**数据模型**，只是一个 M 管理调度 G 的间接控制用数据结构。P 持有 G 的队列，隔离了调度。M 通过绑定 P 来调用一串 G。

M 和 P 构成了一个 runtime 环境。每个 P 持有一个可调度 G 队列，如果 P 中 G 空了，就去全局队列偷取一部分 G，如果全局队列也空了，就从其他 P 中偷取一部分 G。这就是所谓的 Work Stealing。

特殊的 M 和 G：m0 和 g0。m0 是启动程序之后的主线程，负责初始化和启动第一个 G(runtime.main)。每一个 M 都会有自己的管理堆栈 g0，用于 M 的执行管理和调度逻辑。