<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/127_0.png'>
</p>

# HTTP/2 中的 HTTP 语义

HTTP/2 协议设计之初就为了与当前使用的 HTTP 尽可能兼容。这意味着，从应用程序的角度来看，    协议的功能基本没有变化。为了实现这一点，所有请求和响应语义被保留，虽然传达的语法那些语义已经改变了。

因此，HTTP/1.1 中的语义和内容 [[RFC7231]](https://tools.ietf.org/html/rfc7231)，条件请求 [[RFC7232]](https://tools.ietf.org/html/rfc7232)，范围请求 [[RFC7233]](https://tools.ietf.org/html/rfc7233)，缓存 [[RFC7234]](https://tools.ietf.org/html/rfc7234) 和认证 [[RFC7235]](https://tools.ietf.org/html/rfc7235) 的规范和要求同样适用于 HTTP/2。HTTP/1.1 消息语法和路由 [[RFC7230]](https://tools.ietf.org/html/rfc7230) 的选定部分（例如 HTTP 和 HTTPS URI 方案）也适用于HTTP/2，但此协议的语义表达式在下文中定义。

## 一. HTTP Request/Response Exchange

客户端使用先前未使用的流标识符在新流上发送 HTTP 请求 ([第 5.1.1 节](https://tools.ietf.org/html/rfc7540#section-5.1.1))。服务器在与请求相同的流上发送 HTTP 响应。

HTTP消息（请求或响应）包括：

1. 仅用于响应，零个或多个 HEADERS 帧（每个帧后跟零个或多个 CONTINUATION 帧）包含信息（1xx）HTTP 响应的消息头（参见 [[RFC7230] 第 3.2 节](https://tools.ietf.org/html/rfc7230#section-3.2)和[[RFC7231] 第 6.2 节](https://tools.ietf.org/html/rfc7231#section-6.2)），
2. 一个包含消息头的 HEADERS 帧（后跟零个或多个 CONTINUATION 帧）（参见 [[RFC7230] 第 3.2 节](https://tools.ietf.org/html/rfc7230#section-3.2)）
3. 包含有效载荷主体 payload body 的零个或多个 DATA 帧（参见 [[RFC7230] 第 3.3 节](https://tools.ietf.org/html/rfc7230#section-3.3))
4. 可选地，一个 HEADERS 帧，然后是零或更多 CONTINUATION 帧包含 trailer-part（如果有）（请参阅 [[RFC7230] 第 4.1.2 节](https://tools.ietf.org/html/rfc7230#section-4.1.2)）。

序列中的最后一帧带有 END\_STREAM 标志，注意带有 END\_STREAM 标志的 HEADERS 帧后面可以跟随带有任何 header 块 header block 剩余部分的 CONTINUATION 帧。其他帧（来自任何流）不得出现在 HEADERS 帧和可能跟随的任何 CONTINUATION 帧之间。

HTTP/2 使用 DATA 帧来携带消息有效载荷。[[RFC7230] 第 4.1 节](https://tools.ietf.org/html/rfc7230#section-4.1) 中定义的“分块”传输编码**禁止**在 HTTP/2 中使用。

尾随 header 字段在 header 块 header block 中携带，该 header 块也终止该流。这样的 header 块是以 HEADERS 帧开始的序列，后跟零个或多个 CONTINUATION 帧，其中 HEADERS 帧带有 END\_STREAM 标志。第一个不终止 stream 流的头块不是 HTTP 请求或响应的一部分。


HEADERS 帧(和相关的 CONTINUATION 帧)只能出现在 stream 流的开头或结尾。在收到最终(非信息)状态代码后，接收到未设置 END\_STREAM 标志的 HEADERS 帧的端点必须将相应的请求或响应视为格式错误([第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6))。

HTTP 请求/响应交换完全消耗单个 stream 流。请求以 HEADERS 帧开始，将 stream 流置于“打开”状态。请求以带有 END\_STREAM 的帧结束，这使得流成为客户端的“半关闭（本地）”和服务器的“半关闭（远程）”。响应以 HEADERS 帧开始，以带有 END\_STREAM 的帧结束，该帧将流转换为“关闭”状态。

在服务器发送，或者客户端接收，在设置了 END\_STREAM 标志的帧（包括完成 header block 头块所需的任何 CONTINUATION 帧）之后，HTTP 响应就算完成了。如果响应不依赖于还没有发送和接收的请求的任何部分，则服务器可以在客户端发送整个请求之前发送完整响应。当客户端发送一个完整的响应(例如，带有 END\_STREAM 标志的帧)之后，通过发送带有 NO\_ERROR 错误码的 RST\_STREAM 来中止一个没有错误的请求的传输，这种情况如果发生了，服务端可能可以发送一个请求。客户端不得因收到此类 RST\_STREAM 而丢弃响应，但客户可以出于其他原因自行决定是否丢弃响应。


### 1. Upgrading from HTTP/2

HTTP/2 删除了对 101（交换协议）信息状态代码的支持（[[RFC7231]，第 6.2.2 节](https://tools.ietf.org/html/rfc7231#section-6.2.2)）。101（交换协议）的语义不适用于多路复用协议。替代协议能够使用与 HTTP/2 相同机制去协商其使用（参见 [第 3 节](https://tools.ietf.org/html/rfc7540#section-3) ）。


### 2. HTTP Header Fields

HTTP 头字段通过一系列的键值对的方式来携带信息。有关已注册 HTTP headers 的列表，请参阅 <https://www.iana.org/assignments/message-headers> 上维护的“消息头字段”已注册列表。与 HTTP/1.x 中一样， header 字段名称是 ASCII 字符串，以不区分大小写的方式进行比较。但是，在 HTTP/2 编码之前，必须将 Header 头字段名称转换为小写。包含大写 header 字段名称的请求或响应必须被视为格式错误（[第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6)）。

#### (一) Pseudo-Header Fields

虽然 HTTP/1.x 使用消息起始行（参见 [[RFC7230]，第 3.1 节](https://tools.ietf.org/html/rfc7230#section-3.1)）来传达目标 URI，请求方法和响应的状态代码，但 HTTP/2 使用特殊的伪头字段 ':' 字符（ASCII 0x3a）开头来达到相同的目的。伪头字段不是 HTTP 头字段。端点绝不能生成本文档中没有定义的伪头字段。

伪头字段仅在定义它们的上下文中有效。为 requests 请求定义的伪头字段决不能出现在 responses 响应中；为 responses 响应定义的伪头字段绝不能出现在 requests 请求中。伪头字段不得出现在 trailers 中。端点必须将包含未定义或无效伪头字段的请求或响应视为格式错误（[第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6)）。

所有伪头字段必须出现在头块中，并在常规头字段之前。任何请求或者响应中包含有出现在头块中的伪头字段，并且这些伪头字段在常规字段之后才出现的，这些都应该被视为格式错误（[第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6)）。


#### (二) Connection-Specific Header Fields

HTTP/2 不会使用连接头字段去标识特定的连接头字段。在这个协议中，特定的连接元数据是通过其他的方法进行传输的。端点禁止生成一条包含有特定连接头字段的消息。任何包含有特定连接头字段的消息就会被视为格式错误（[第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6)）。

唯一例外的字段是 TE 头字段，该字段会出现在 HTTP/2 的请求中。它不能包含 trailers 以外的任何值。这意味着将 HTTP/1.x 消息转换为 HTTP/2 的网络中间件将需要删除 Connection 头字段提及到的任何头字段以及 Connection 头字段本身。这些网络中间件还应该删除其他特定于连接的头字段，例如 Keep-Alive，Proxy-Connection，Transfer-Encoding 和 Upgrade，即使它们不是由 Connection 头字段指定的。

**注意：HTTP/2 故意不支持升级到其他协议。[第 3 节](https://tools.ietf.org/html/rfc7540#section-3)中描述的握手方法被认为足以被用来协商替代协议**。

#### (三) Request Pseudo-Header Fields

下面的这些伪头字段是定义在 HTTP/2 的请求中的：

- ":method" 伪头字段包含了 HTTP 方法 ([[RFC7231], Section 4](https://tools.ietf.org/html/rfc7231#section-4))。
- ":scheme" 伪头字段包含了目标 URI 的一部分 scheme ([[RFC3986],第 3.1 节](https://tools.ietf.org/html/rfc3986#section-3.1))。":scheme" 不限于 "http" 和 "https" scheme 的 URI。代理或网关可以对非 HTTP scheme 转换请求，从而使得 HTTP 与非 HTTP 服务进行交互。
- ":authority" 伪头字段包含目标 URI 的部分权限([[RFC3986],第 3.2 节](https://tools.ietf.org/html/rfc3986#section-3.2))。权限不得包含 "http" 或 "https" scheme URI 的已弃用 "userinfo" 子组件。为了确保可以准确地再现 HTTP/1.1请求行，当从具有源或星号形式的请求目标的 HTTP/1.1 请求进行转换时，必须省略该伪头字段（[参见[RFC7230]，第 5.3 节](https://tools.ietf.org/html/rfc7230#section-5.3)）。直接生成 HTTP/2 请求的客户端应该使用 ":authority" 伪头字段而不是 Host 头字段。将 HTTP/2 请求转换为 HTTP/1.1 的网络中间件必须通过复制 ":authority" 伪头字段的值来创建 Host 头字段（如果请求中不存在的话）。
- ":path" 伪头字段包含目标 URI 的路径和查询部分。绝对路径和可选的 "?" 字符后的查询部分。（见 [[RFC3986]的第 3.3 和 3.4 节](https://tools.ietf.org/html/rfc3986)），星号形式的请求包括 ":path" 伪头字段的值 '\*'。对于 "http" 或 "https" 的 URI，此伪头字段不能为空；不包含路径组件的 "http" 或 "https" 的 URI 必须包含值 "/"。此规则的例外是对不包含路径组件的 "http" 或 "https" 的 URI 的 OPTIONS 请求；这些必须包含一个 ":path" 伪头字段，其值为 '\*'（参见 [[RFC7230]，第 5.3.4 节](https://tools.ietf.org/html/rfc7230#section-5.3.4)）。


除非是 CONNECT 请求，否则所有 HTTP/2 请求必须包含 ":method"，":scheme" 和 ":path" 伪头字段的一个有效值（[第 8.3 节](https://tools.ietf.org/html/rfc7540#section-8.3)）。省略强制伪头字段的 HTTP 请求格式不正确（[第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6)）。HTTP/2 没有定义携带 HTTP/1.1 请求行中包含的版本标识符的方法。

#### (四) Response Pseudo-Header Fields

对于 HTTP/2 响应，定义了一个带有 HTTP 状态代码字段的 ":status" 伪头字段（参见[[RFC7231]，第 6 节](https://tools.ietf.org/html/rfc7231#section-6)）。这个伪头字段必须包含在所有响应中; 否则，响应的格式不正确（[第 8.1.2.6 节](https://tools.ietf.org/html/rfc7540#section-8.1.2.6)）。HTTP/2 没有定义携带 HTTP/1.1 状态行中包含的版本或原因短语的方法。



#### (五) Compressing the Cookie Header Field

Cookie 头字段 [COOKIE] 使用分号（“;”）来分隔 cookie 对（或 "crumbs"）。此头字段不遵循 HTTP 中的列表构造规则（参见[[RFC7230]，第 3.2.2 节](https://tools.ietf.org/html/rfc7230#section-3.2.2)），这可防止 cookie 对 被分成不同的 名称-值 对。这会显着降低压缩效率，因为只更新了单个 cookie 对。

为了提高压缩效率，可以将 Cookie 头字段拆分为单独的头字段，每个字段都有一个或多个 cookie 对。如果在解压缩后有多个 Cookie 头字段，则必须在传递到非 HTTP/2 上下文之前使用两个八位字节分隔符 0x3B，0x20（ASCII 字符串“;”）将这些字节连接成单个八位字节字符串，例如一个 HTTP/1.1 连接，或一个通用的 HTTP 服务器应用程序。

因此，以下两个 Cookie 头字段列表在语义上是等效的。

cookie：a = b;c = d;e = f

cookie：a = b
cookie：c = d
cookie：e = f



#### (六) Malformed Requests and Responses

格式错误的请求或响应是 HTTP/2 帧的其他有效序列，但由于存在一些无关的帧，禁止的头字段，缺少必需的头字段或包含大写头字段名称而导致无效的。

包括有效 payload 负载主体的请求或响应可以包括内容长度报头字段。如果内容长度 header 字段的值不等于形成 body 的 DATA 帧有效负载长度的总和，则请求或响应也会视为格式错误。如 [[RFC7230]第 3.3.2 节](https://tools.ietf.org/html/rfc7230#section-3.3.2) 中所述，被定义为没有有效负载的 response 响应，可以具有非零的内容长度报头字段，即使 DATA 帧中不包含任何内容。

处理 HTTP 请求或响应的网络中间件（即，任何不充当隧道的网络中间件）都不得转发格式错误的请求或响应。检测到的格式错误的请求或响应必须被视为 PROTOCOL\_ERROR 类型的流错误（[第 5.4.2 节](https://tools.ietf.org/html/rfc7540#section-5.4.2)）。对于格式错误的请求，服务器可以在关闭或重置流之前发送 HTTP 响应。客户不得接受格式错误的 response 响应。请注意，这些要求旨在防止针对 HTTP 的多种类型的常见攻击；它们是故意严格的，因为宽松会引起一些漏洞。


### 3. Examples

本节展示了 HTTP/1.1 请求和响应，并提供等效 HTTP/2 请求和响应的说明。HTTP GET 请求包括请求头字段而没有有效负载主体，因此作为单个 HEADERS 帧发送，然后是零个或多个 CONTINUATION 帧，其中包含序列化的请求头字段块。下面的 HEADERS 帧同时设置了 END\_HEADERS 和 END\_STREAM 标志; 没有发送 CONTINUATION 帧。

```c
     GET /resource HTTP/1.1           HEADERS
     Host: example.org          ==>     + END_STREAM
     Accept: image/jpeg                 + END_HEADERS
                                          :method = GET
                                          :scheme = https
                                          :path = /resource
                                          host = example.org
                                          accept = image/jpeg
```

类似地，仅包含响应头字段的响应被作为 HEADERS 帧（再次，后面是零个或多个 CONTINUATION 帧）发送，其中包含序列化的响应头字段块。

```c
     HTTP/1.1 304 Not Modified        HEADERS
     ETag: "xyzzy"              ==>     + END_STREAM
     Expires: Thu, 23 Jan ...           + END_HEADERS
                                          :status = 304
                                          etag = "xyzzy"
                                          expires = Thu, 23 Jan ...
```

包含请求头字段和有效载荷数据的 HTTP POST 请求被作为一个 HEADERS 帧发送，接着是包含请求头字段的零个或多个 CONTINUATION 帧，后跟一个或多个 DATA 帧，最后一个 CONTINUATION（或 HEADERS）帧设置有 END\_HEADERS 标志和设置有 END\_STREAM 标志的最终 DATA 帧。

```c
     POST /resource HTTP/1.1          HEADERS
     Host: example.org          ==>     - END_STREAM
     Content-Type: image/jpeg           - END_HEADERS
     Content-Length: 123                  :method = POST
                                          :path = /resource
     {binary data}                        :scheme = https

                                      CONTINUATION
                                        + END_HEADERS
                                          content-type = image/jpeg
                                          host = example.org
                                          content-length = 123

                                      DATA
                                        + END_STREAM
                                      {binary data}
```

注意，对任何给定头字段有贡献的数据，都可以在头块片段之间扩展。在该示例中将头字段分配给帧仅是用来说明的。

包含 header 字段和有效负载数据的响应被作为 HEADERS 帧发送，后跟零个或多个 CONTINUATION 帧，后跟一个或多个 DATA 帧，序列中的最后一个 DATA 帧设置了 END\_STREAM 标志。


```c
     HTTP/1.1 200 OK                  HEADERS
     Content-Type: image/jpeg   ==>     - END_STREAM
     Content-Length: 123                + END_HEADERS
                                          :status = 200
     {binary data}                        content-type = image/jpeg
                                          content-length = 123

                                      DATA
                                        + END_STREAM
                                      {binary data}
```


使用除 101 之外的 1xx 状态代码的信息响应被作为 HEADERS 帧发送，接着是零个或多个 CONTINUATION 帧。

在请求或响应头块以及所有 DATA 帧都已发送之后，尾随头字段作为头块发送。以 trailers 头块为开始的 HEADERS 帧设置了 END\_STREAM 标志。以下示例包括 100（继续）状态代码，该代码是为响应在 Expect 标头字段中包含 “100-continue” 标记的请求而发送的，以及尾随头字段。

```c
     HTTP/1.1 100 Continue            HEADERS
     Extension-Field: bar       ==>     - END_STREAM
                                        + END_HEADERS
                                          :status = 100
                                          extension-field = bar

     HTTP/1.1 200 OK                  HEADERS
     Content-Type: image/jpeg   ==>     - END_STREAM
     Transfer-Encoding: chunked         + END_HEADERS
     Trailer: Foo                         :status = 200
                                          content-length = 123
     123                                  content-type = image/jpeg
     {binary data}                        trailer = Foo
     0
     Foo: bar                         DATA
                                        - END_STREAM
                                      {binary data}

                                      HEADERS
                                        + END_STREAM
                                        + END_HEADERS
                                          foo = bar
```




### 4. Request Reliability Mechanisms in HTTP/2


在 HTTP/1.1 中，HTTP 客户端在发生错误时无法重试非幂等请求，因为无法确定错误的本质。某些服务器处理可能在错误发生之前发生，如果请求被重新尝试，则可能导致不良影响。

HTTP/2 提供了两种机制，用于向客户端提供尚未处理请求的保证：

- GOAWAY 帧表示可能已处理的最高流编号。因此，保证对具有较高编号的流的请求可以安全地重试。
- REFUSED\_STREAM 错误代码可以包含在 RST\_STREAM 帧中，以指示在发生任何处理之前，流正在关闭。可以安全地在重置流上发送的任何重试请求。


未处理的请求没有失败; 客户端可以自动重试它们，即使那些具有非幂等方法的客户端也是如此。服务器不能指示尚未处理流，除非它可以保证这一事实。如果流上的帧被传递到任何流的应用层，则 REFUSED\_STREAM 不能用于该流，并且 GOAWAY 帧必须包含一个大于或等于给定流标识符的流标识符。

除了这些机制之外，PING 帧还为客户端提供了一种轻松测试连接的方法。由于某些中间件（例如，网络地址转换器或负载平衡器）会以静默方式丢弃连接绑定，因此保持空闲的连接可能会中断。PING 帧允许客户端在不发送请求的情况下安全地测试连接是否仍处于活动状态。


## 二. Server Push

HTTP/2 允许服务器对先前客户端发起的相关联的请求，预先发送（或“推送”）响应（以及相应的“承诺”请求）到客户端。当服务器知道客户端需要这些响应以便完全处理对原始请求的响应时，Server Push 可能很有用。

客户端请求的时候可以指明禁用服务器推送这一功能，尽管这是针对每一跳单独协商的。SETTINGS\_ENABLE\_PUSH 设置可以设置为 0 ，表示禁用服务器推送。

Promised 的请求必须是可缓存的（参见[[RFC7231]，第 4.2.3 节](https://tools.ietf.org/html/rfc7231#section-4.2.3)），必须是安全的（参见 [[RFC7231]，第 4.2.1 节](https://tools.ietf.org/html/rfc7231#section-4.2.1)），并且不得包含请求体。接收到不可缓存的 promised 请求，这种情况是不安全的，或者指明了存在请求体的客户端必须使用 PROTOCOL\_ERROR 类型的流错误（[第 5.4.2 节](https://tools.ietf.org/html/rfc7540#section-5.4.2)）重置 promised 的流。请注意，如果客户端无法判断新定义的方法是否是安全，则可能导致重置 promised 的流。

如果客户端实现了 HTTP 缓存，则可以缓存可缓存的推送响应（参见[[RFC7234]，第 3 节](https://tools.ietf.org/html/rfc7234#section-3)）。当 promised 流 ID 标识的 stream 流仍然处于打开状态的时候，推送响应可以在源服务器上被成功的验证（例如，如果存在“无缓存”缓存响应指令（[[RFC7234]，第 5.2.2 节](https://tools.ietf.org/html/rfc7234#section-5.2.2)））。不可缓存的推送响应禁止被任何 HTTP 缓存存储。它们可以单独提供给应用程序。

如果服务器是有权限的，那么就需要在 ":authority" 伪头字段中包含一个值（参见[第 10.1 节](https://tools.ietf.org/html/rfc7540#section-10.1)）。客户端必须将没有权限的服务器的 PUSH\_PROMISE 视为 PROTOCOL\_ERROR 类型的流错误（[第 5.4.2 节](https://tools.ietf.org/html/rfc7540#section-5.4.2)）。

中间件可以从服务器接收推送，并选择不将它们转发到客户端。换句话说，如何利用推送的信息取决于该中间件。同样，中间件可能会选择向客户端进行额外的推送，而不会对服务器采取任何操作。

客户端无法推送。因此，服务器必须将 PUSH\_PROMISE 帧的接收视为 PROTOCOL\_ERROR 类型的连接错误（[第 5.4.1 节](https://tools.ietf.org/html/rfc7540#section-5.4.1)）。客户端必须拒绝任何将 SETTINGS\_ENABLE\_PUSH 设置更改为 0 以外的值的尝试，如果遇到这种情况，可以将更改设置的这条消息视为 PROTOCOL\_ERROR 类型的连接错误（[第 5.4.1 节](https://tools.ietf.org/html/rfc7540#section-5.4.1)）。


### 1. Push Requests

服务器推送在语义上等同于响应一个请求的服务器; 但是，在这种情况下，该请求也是由服务器发送，作为一个 PUSH\_PROMISE 帧。

## 三. The CONNECT Method






------------------------------------------------------

Reference：  

[RFC 7540](https://tools.ietf.org/html/rfc7540)

> GitHub Repo：[Halfrost-Field](https://github.com/halfrost/Halfrost-Field)
> 
> Follow: [halfrost · GitHub](https://github.com/halfrost)
>
> Source: []()