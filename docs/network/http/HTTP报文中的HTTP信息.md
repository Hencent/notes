
# HTTP 报文中的 HTTP 信息
HTTP 报文用于 HTTP 协议交互信息。分为请求报文和响应报文。报文可分为**报文首部**和**报文主体**两部分。使用 CR+LF 符分开。

## 报文首部
请求报文分为：请求行，请求首部字段，通用首部字段，实体首部字段，其他  
响应报文分为：状态行，相应首部字段，通用首部字段，实体首部字段，其他  

+ 请求行：包含用于请求的方法，请求 URL 和 HTTP 版本，如：
  + `GET / HTTP/1.1`
+ 状态行：包含表明响应结果的状态码，原因短语和 HTTP 版本，如：
  + `HTTP/1.1 200 OK`
+ 首部字段：包含请求和响应的各种条件与属性的字段
+ 其他：一些 RFC 中未定义的首部信息，如 Cookie

## 报文编码与解码
HTTP 报文传输过程中，可以原样传输，但是可能比较大。也可以编码压缩之后传输，但是这样需要编码解码，消耗一定的 CPU 资源。这涉及到两个概念：

+ 报文：HTTP 基本通信单位，由 8-bit 字节流组成。
+ 实体：请求或相应的有效载荷数据，包括实体首部和实体主体

报文主体用于传输实体主体。在非压缩的情况下，`报文主体==实体主体`，编码之后会导致差异。HTTP协议中的内容编码功能会指明在实体内容上的编码格式，由接收端负责解码。常用的内容编码包括：`gzip` 等。

若实体资源没有传输完成，不能进行解码，所以传输大量内容时，浏览器会很长一段时间不能显示任何内容。所以，可以使用分割发送的**分块传输编码**，能够让浏览器逐个解码，逐渐显示页面。每一块都会标记块的大小，最后一块会有结束标志。

## 报文中的 多部分对象集合
HTTP 采用了**多部分对象集合**(multipart)，报文主体中含有多类型实体，可以容纳多种类型的多份实体内容。多部分对象集合包含的对象如下：
+ multipart/form-data：Web 表单文件上传时使用
  ```
  Content-Type: multipart/form-data; boundary=Aab03x

  --AaB03x
  Content-Disposition: form-data; name="field"

  Joe Blow
  --AaB03x
  Content-Disposition: form-data; name="pics"; filename="file.txt"
  Content-Type: text/plain

  …(file1.txt的数据)…
  --AaB03x--
  ```
+ multipart/byteranges：响应报文 206(Partial Content) 包含了多个范围的内容的时候是使用
  ```
  HTTP/1.1 206 Partial Context
  ……
  Content-Type: multipart/byteranges; boundary=THIS_STRING

  --THIS_STRING
  Content-Type: application/pdf
  Content-Range: bytes 500-999/8000

  …(指定范围的数据)…
  --THIS_STRING
  Content-Type: application/pdf
  Content-Range: bytes 7000-7999/8000

  …(指定范围的数据)…
  --THIS_STRING--
  ```

HTTP 报文中需要使用多部分对象集合时，需要在首部字段加上 Content-type 首部字段 和指定 boundary 字符串用来划分多部分对象集合的各类实体。多部分对象集合的每个部分类型中，都可以含有首部字段，或嵌套多类型对象集合。

## 范围请求：获取部分内容
为了支持断点续传，需要支持指定下载的实体范围，这种请求叫范围请求。
范围请求的响应如上节示例中的 `Content-Range: bytes 500-999/8000`。

范围请求会返回状态码 206 Partial Content，对于多重的范围请求HTTP 首部中标明 `Content-Type: multipart/byteranges`。如果服务器无法响应范围请求，会返回状态码 200 OK 和完整的实体内容。

## 内容协商：返回最合适的内容
同一个网站可能存在多份相同内容的页面，如中文版或英文版、PC 版或手机版。

通过内容协商机制，基于语言、字符集、编码方式等进行判断，返回最合适的内容。请求报文中的某些首部字段就有这样的作用：`Accept, Accept-Charset, Accept-Encoding, Accept-Language, Content-Language`