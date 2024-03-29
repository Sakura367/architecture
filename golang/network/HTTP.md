## HTTP与RPC
- RPC：Remote Produce Call远程过程调用，类似的还有RMI（Remote Methods Invoke 远程方法调用，是JAVA中的概念，是JAVA十三大技术之一）。自定义数据格式，基于原生TCP通信，速度快，效率高。早期的webservice，现在热门的dubbo，都是RPC的典型
RPC的框架：webservie(cxf)、dubbo
RMI的框架：hessian
- Http：http其实是一种网络传输协议，基于TCP，规定了数据传输的格式。现在客户端浏览器与服务端通信基本都是采用Http协议。也可以用来进行远程服务调用。缺点是消息封装臃肿。
现在热门的Rest风格，就可以通过http协议来实现。
http的实现技术：HttpClient

相同点：底层通讯都是基于socket，都可以实现远程调用，都可以实现服务调用服务

不同点：
RPC：框架有：dubbo、cxf、（RMI远程方法调用）Hessian
当使用RPC框架实现服务间调用的时候，要求服务提供方和服务消费方 都必须使用统一的RPC框架，要么都dubbo，要么都cxf
跨操作系统在同一编程语言内使用
优势：调用快、处理快

http：框架有：httpClient
当使用http进行服务间调用的时候，无需关注服务提供方使用的编程语言，也无需关注服务消费方使用的编程语言，服务提供方只需要提供restful风格的接口，服务消费方，按照restful的原则，请求服务，即可
跨系统跨编程语言的远程调用框架
优势：通用性强

**区别**
1. HTTP和RPC都属于远程调用；
2. HTTP是一种国际语言，大家都用的协议，而RPC往往是指与框架相关的定制化协议，及上层的IDL；（框架有什么：gRPC、brpc、thrift、srpc等；IDL有什么：protobuf、thrift等；）所以，如果我们更习惯用HTTP，那么只是因为它更通用；
3. 如果我们更习惯用RPC，那么只是因为框架使得我们操作IDL更方便；我们既可以用HTTP client访问RPC server，也可以用RPC client发出HTTP请求；

## HTTP/1.0、HTTP/1.1、HTTP/2
```
浏览器请求 url -> 解析域名 -> 建立 HTTP 连接 -> 服务器处理文件 -> 返回数据 -> 浏览器解析、渲染文件
```
每次请求都需要建立一次 HTTP 连接，也就是我们常说的3次握手4次挥手，这个过程在一次请求过程中占用了相当长的时间，而且逻辑上是非必需的

**HTTP/1.1**
提供了 Keep-Alive，允许我们建立一次 HTTP 连接，来返回多次请求数据。
- HTTP 1.1 基于串行文件传输数据，因此这些请求必须是有序的，所以实际上我们只是节省了建立连接的时间，而获取数据的时间并没有减少
- 最大并发数问题，假设我们在 Apache 中设置了最大并发数 300，而因为浏览器本身的限制，最大请求数为 6，那么服务器能承载的最高并发数是 50

**HTTP/2**
- HTTP/2 引入二进制数据帧和流的概念，其中帧对数据进行顺序标识，这样浏览器收到数据之后，就可以按照序列对数据进行合并，而不会出现合并后数据错乱的情况。因为有了序列，服务器就可以并行的传输数据。

- HTTP/2 对同一域名下所有请求都是基于流，也就是说同一域名不管访问多少文件，也只建立一路连接。同样Apache的最大连接数为300，因为有了这个新特性，最大的并发就可以提升到300，比原来提升了6倍。

- 多路复用
多路复用GRPC使用HTTP/2作为应用层的传输协议，HTTP/2会复用底层的TCP连接。
每一次RPC调用会产生一个新的Stream，每个Stream包含多个Frame，Frame是HTTP/2里面最小的数据传输单位。同时每个Stream有唯一的ID标识，如果是客户端创建的则ID是奇数，服务端创建的ID则是偶数。如果一条连接上的ID使用完了，Client会新建一条连接，Server也会给Client发送一个GOAWAY Frame强制让Client新建一条连接。一条GRPC连接允许并发的发送和接收多个Stream，而控制的参数便是MaxConcurrentStreams，Golang的服务端默认是100。

- 超时重连
我们在通过调用Dial或者DialContext函数创建连接时，默认只是返回ClientConn结构体指针，同时会启动一个Goroutine异步的去建立连接。如果想要等连接建立完再返回，可以指定grpc.WithBlock()传入Options来实现。超时机制很简单，在调用的时候传入一个timeout的context就可以了。重连机制通过启动一个Goroutine异步的去建立连接实现的，可以避免服务器因为连接空闲时间过长关闭连接、服务器重启等造成的客户端连接失效问题。也就是说通过GRPC的重连机制可以完美的解决连接池设计原则中的空闲连接的超时与保活问题。

#### HTTP长连接与TCP长连接
TCP的keepalive是侧重在保持客户端和服务端的连接，一方会不定期发送心跳包给另一方，没有断掉一方的定时发送几次心跳包。如果间隔发送几次，对方都返回的是RST，而不是ACK，那么就释放当前连接。

HTTP的keep-alive一般我们都会带上中间的横杠，普通的HTTP连接是客户端连接上服务端，然后结束请求后，由客户端或者服务端进行http连接的关闭。下次再发送请求的时候，客户端再发起一个连接，传送数据，关闭连接。这么个流程反复。但是一旦客户端发送connection: keep-alive头给服务端，且服务端也接受这个keep-alive的话，这个连接就可以复用了。一个HTTP处理完之后，另外一个HTTP数据包也直接从这个连接发送。减少新建和断开TCP连接的消耗。

二者的作用简单来说：

HTTP协议的keep-alive意图在于短时间内连接复用，希望可以短时间内在同一个连接上进行多次请求/响应。

TCP的KeepAlive机制意图在于保活、心跳、检测连接错误。当一个TCP连接两端长时间没有数据传输时(通常默认配置是2小时)，发送keepalive探针，探测链接是否存活。

总之，记住HTTP的Keep-Alive和TCP的KeepAlive不是一回事。

TCP的keepalive是在ESTABLISHED状态的时候，双方如何检测连接的可用性。而HTTP的keep-alive说的是如何避免进行重复的TCP三次握手和四次挥手的环节。

## HTTPS
### 通讯过程
1. 客户端请求建立SSL连接
2. 服务端返回证书信息（服务端公钥）
3. 客户端SSL/TLS校验证书
4. 客户端生成会话秘钥（对称秘钥），用公钥加密发送给服务端
5. 服务端用私钥解密得到会话秘钥
6. 客户端、服务端通讯用会话秘钥加密
```
生成会话秘钥是非对称加密
通话信息使用会话秘钥进行对称加密
```

### 客户端验证数字证书（服务端公钥）
1. 客户端预装CA机构公钥
2. 用CA机构公钥解密数字签名得 S
3. 对数字证书hash得值 T
4. 对比T、S值

### 加密算法
```
对称加密，如 AES
基本原理：将明文分成 N 个组，然后使用密钥对各个组进行加密，形成各自的密文，最后把所有的分组密文进行合并，形成最终的密文。
优点：算法公开、计算量小、加密速度快、加密效率高
缺点：双方都使用同样密钥，安全性得不到保证
```
```
非对称加密，如 RSA
基本原理：同时生成两把密钥：私钥和公钥，私钥隐秘保存，公钥可以下发给信任客户端
私钥加密，持有私钥或公钥才可以解密
公钥加密，持有私钥才可解密
优点：安全，难以破解
缺点：算法比较耗时
```
```
不可逆加密，如 MD5，SHA
基本原理：加密过程中不需要使用密钥，输入明文后由系统直接经过加密算法处理成密文，这种加密后的数据是无法被解密的，无法根据密文推算出明文。
```

### HTTP与HTTPS
- 端口不同：Http是80，Https443
- 安全性：http是超文本传输协议，信息是明文传输，https则是通过SSL加密处理的传输协议，更加安全。
- 是否付费：https需要拿到CA证书，需要付费
- 连接方式：http和https使用的是完全不同的连接方式（HTTP的连接很简单，是无状态的；HTTPS 协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全。）