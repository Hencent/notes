# what is protobuf
protobuf 指 Protocol Buffers，是 Google 公司内部的混合语言数据标准。他们用于 RPC 系统和持续数据存储系统。

Protocol Buffers 是一种轻便高效的**结构化数据存储格式**，可以用于**结构化数据串行化**，或者说**序列化**。它很适合做数据存储或 RPC 数据交换格式。

OK，现在有人好奇了，json 难道不香吗？嗯？我也觉得很香吧。
> [https://geektutu.com/post/quick-go-protobuf.html](https://geektutu.com/post/quick-go-protobuf.html)
> 
> [https://www.zhihu.com/question/29378878](https://www.zhihu.com/question/29378878)

通过参考上述链接，得出了以下的原因：
1. json 更加注重可读性，简单，易用！JSON 作为一种模型友好可亲，也有工具在 JSON 上面做 schema 描述、校验等，可以满足各种层次的工程要求。而相反，PB 序列化为二进制格式，不可读，而且需要编写 proto 文件，非常麻烦。此外，PB 并非支持全部语言，会有一定麻烦。
2. 因为两边都要有 proto 文件，使得其成为了一个接口强规范的数据格式。它的优势在于**接口的一致性**。
   > pb的核心价值，是作为远程系统调用时的API contract，保证接口的向后兼容，杜绝系统的向前兼容，避免某个模块接口自己魔改而导致整个调用链的不可控，也能降低开发者和接口设计者的心智负担。

   JSON虽然也可以有Schema，但是用的地方不多。在超大规模的项目上，弱类型或者没有类型是很糟糕的。例如后端逻辑中，没法保证 response 里一定有 fieldA，并且 fieldA 的类型和预期的一致。即便用 TypeScript, 也只能在编译期间保证客户端代码本身的类型一致，但是无法保证和服务端保持一致。
3. PB 的另一大优势在于，其二进制格式下，数据量少，传输效率高，提高了性能。
   > 不过不少人认为：绝大多数业务性能都不是瓶颈，相比之下可读性最重要。脱开功能谈性能是很没有意义的，而且大部分人根本意识不到这个性能问题，或者说性能根本不是一个问题。

# protobuf 的简明使用
鉴于上述分析的原因，暂缓学习其使用。

JSON yyds！