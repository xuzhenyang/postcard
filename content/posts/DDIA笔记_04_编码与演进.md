---
title: "DDIA 笔记 | 04 编码与演进"
date: 2022-02-19
draft: true
---

## 前言

近期在研读《Designing Data-Intensive Application》，这本书介绍了分布式系统的方方面面，也包含很多 trade-off 以及实现方式。如果仅是泛泛通读一遍，而不去深究咀嚼，多少会觉得有点可惜。于是就有了这个笔记系列，针对书中的每一个章节，结合自己的理解来写一篇笔记。

多花点时间，也会有更多收获吧。

---

![](http://ddia.vonng.com/img/ch2.png)

世界在变，需求在变，数据在变，数据结构也自然有可能发生变化。

第 2 章讲述了各种数据模型，其中，关系型数据的 schema 虽然可以更新，但是在一个时间点上，只能存在一种结构，而 NoSQL 则没有约束，可以同时存放不同结构的新老数据。

数据结构或者 schema 发生了变化，那代码通常也是要跟着调整的。

异想天开一下，如果“pia”得一声，系统所运行的代码在一瞬间就完成了更新，那程序员这个职业可真是幸福。然而事实上，大多数的 app 没办法迅速完成更新，服务端有几十、上百台机器，需要一点一点进行滚动发布，客户端则要看用户的脸色，总有一些“版本更新钉子户”，所以在一个时间点上，新老版本的代码、新老版本的数据会在同时存在。

因此，为了保证系统丝滑运行，程序员们就得时时刻刻考虑：

**向后兼容**：新代码可以读旧代码写入的数据  
**向前兼容**：旧代码可以读新代码写入的数据

下面就来了解一下，数据都有哪些编码格式，以及它们是如何保证兼容性的。

## 数据的编码格式

在内存中，数据可以是对象、列表、数组、哈希表、树，等等，有这么多的结构种类，主要是因为有不同的应用场景，以及为了读取效率、cpu 利用率做了针对性的优化。

如果想把内存中的数据写入文件或者网络中，就需要把数据编码成自包含的字节序列（例如 JSON 文件）。

这两种形式之间需要做翻译，从内存到字节序列叫 encoding（或叫 serialization、marshalling），反过来就是 decoding（或叫 parsing、deserialization，unmarshalling）。

### 语言特定的编码格式

许多编程语言有内建的编码方式，例如 Java 的 java.io.Serializable，Python 的 pickle，还有很多第三方库。

虽然使用起来很方便，但是会存在以下问题：

- 与语言强绑定，架构上比较难做异构。
- 可能会有安全性问题。decoding 就是将类进行实例化，如果攻击者能让应用解码任意的字节序列，也就能实例化任意的类，就可以远程执行代码。
- 在这些库中，数据版本控制往往是事后才考虑的，可能会忽略兼容性。
- 效率低下。例如 Java 的序列化，性能就很糟糕。

因此，除非临时或是简单场景，尽量别采用语言内置的编码吧。

### JSON，XML，以及二进制变体

如果需要跨语言对数据进行编码，第一个想到的应该是 JSON，或者 XML。XML 看起来太复杂、啰嗦，JSON 的流行正是因为它比 XML 简单，而且作为 JavaScript 的子集，浏览器内置就支持。还有一种流行的与语言无关的格式 CSV，当然功能也相对较弱。

JSON、XML、CSV 都是文本格式，是人类可读的，也会遇到一些问题：

- 对于数值（number）的处理含糊不清。XML 和 CSV 不能区分数值和字符串；JSON 能区分数值和字符串，但是不区分整型和浮点数，而且也不能指定精度。
- JSON 和 XML 支持 Unicode 字符串（人类可读的），但是不支持二进制数据（不带字符编码的字节序列）。有一种 hacky 的做法，就是把二进制数据编码为 Base64 文本，但是会徒增 1/3 的数据大小。
- JSON 和 XML 本身虽然支持 schema，但实现起来也很复杂，很多基于 JSON 的工具甚至不支持 schema。
- CSV 不支持 schema，因此如果改动了结构，就需要手动处理。

当然，这些小问题从总体而言，瑕不掩瑜，JSON 这类简单的数据交换格式很可能会继续流行下去，但是相比较于二进制格式，还是太占空间了。一旦数据量达到 TB 级别，对于数据格式的选型就会产生巨大的影响。

因此，出现了非常多二进制版本的 JSON（例如 Message Pack、BSON、BJSON）和 XML（例如 WBXML、Fast Infoset）。

书中重点介绍了 Thrift，Protocol Buffers，Avro 这三种二进制编码库，具体细节就不在此赘述，简单来看下 Protocol Buffers。

```protobuf
message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

以上是使用 Protocol Buffers 定义的 schema，其自带一个代码生成工具，基于 schema 来生成各种编程语言实现的类，在代码里就可以进行编码或者解码。

可以看到，每个字段都有一个数值 tag（1，2，3）和字段类型（string，int64），这个数值 tag 会与数据关联，更改字段名不会有什么影响，但是如果改了数值 tag，数据就无效了。

Protocol Buffers 是这样来保证**向前兼容**的：如果新增了一个字段，并且给新增的字段一个新的数值 tag，那么旧的代码（不知道新增的数值 tag）在读取到新代码写入的数据时，可以直接忽略这个字段；而**向后兼容**呢？每个字段都有一个唯一的数值 tag，新的代码总是可以读取旧的数据的，因为这个数值 tag 的含义不会变，唯一要注意的是，新增的字段不能设置为 required，因为旧的代码里没有这个新字段。所以为了保证向后兼容，除了第一版的 schema，后续新增的字段必须是 optional 或有默认值。

而如果要改变字段的数据类型，虽然可行，但是会有风险，例如数据值会丢失精度或被截断。

Thrift 和 Protocol Buffers 都是基于 schema 来生成代码，这在 Java、C++ 等静态类型语言中很有用，但是对于动态类型语言（如 JavaScript、Ruby 或 Python）就没什么作用，而 Arvo 则可以支持生成动态的 schema。

我们可以看到，尽管 JSON、XML 和 CSV 等文本数据格式非常普遍，但是基于 schema 的二进制编码也是一个可选项。它们有以下优点：

- 比各种二进制 JSON 的变体更加紧凑，因为可以省略字段名称
- 相比较于手工维护文档，schema 是更有价值的（人写的文档并不可靠），因为解码必须用到
- 维护一个 schema 的数据库，就允许你在部署之前检查前后的兼容性
- 对于静态类型语言，通过 schema 生成代码，可以在编译时进行类型检查

## 数据流的形式

刚才讨论了，将数据传递到不共享内存的另一个进程时，需要进行编码转换。

接下来，我们接着来看一些数据在不同进程中流转的常见场景。

### 数据库中的数据流

在数据库中，读写数据也要对数据进行编码/解码，而在同一时刻，会有几个不同的进程同时访问数据库，这些进程可能运行最新的代码，也可能运行旧的代码。所以在数据库的读写上，必须考虑前后兼容问题。

假设你在 schema 里新增了一个字段，然后用新的代码写入这个字段，存进数据库中，随后，旧的代码读取了这条记录，并更新存入了数据库里，如果不注意的话，数据可能会丢失哦。理想的结果，是旧代码能保持新的字段不变，即使不能被解码。

![](http://ddia.vonng.com/img/fig4-7.png)

还有一种情况，数据的生命周期超过了代码的生命周期，这时候就需要进行数据的归档，或是迁移。可以考虑使用 Avro 来动态生成 schema，或是将归档的数据编码为面向分析的列式存储。

### 服务中的数据流：REST 和 RPC

现在流行的 SOA、微服务架构，是将服务拆分成独立可部署、可演进的部分，来降低变更和维护成本。这也就意味着新老版本的客户端、服务端会在同一时间运行，也必须要考虑客户端、服务端使用的数据编码在不同版本的 API 间的兼容性。

#### Web 服务
当服务使用 HTTP 作为底层通讯协议时，可称之为 Web 服务，现在最流行的两种 Web 服务实践是 REST 和 SOAP。

REST 并不是一个协议，而是一种基于 HTTP 原则的设计哲学，强调简单的数据格式，使用 URL 来标识资源，并使用 HTTP 的功能来控制缓存、鉴权和内容类型协商。和 SOAP 相比，REST 越来越流行，基于 REST 原则设计的 API 称为 Restful。

SOAP 是基于 XML 的协议，虽然常常用于 HTTP，但其目的是独立于 HTTP，并避免大多数 HTTP 的功能，有许多庞大、复杂的标准，很大程度上又依赖于工具支持，逐渐淡出了大家的视野。

#### RPC
因为网络的不可靠性，RPC 看起来难以完美实现远端过程调用这一美好愿景，但是它不会消失。上面提到的一些编码基础上，有不少 RPC 框架，例如 Thrift 和 Avro 就支持 RPC，gRPC 使用了 Protocol Buffers 的 RPC 实现。

使用二进制编码格式的自定义 RPC 协议可以有更好的性能，而 RESTful API 也有一些二进制编码不具备的优点，例如方便调试，能被主流的编程语言和平台所支持，还有完善的工具生态（缓存，代理，负载均衡，监控，测试工具等）。

所以，REST 常用于对外的公共 API，而 RPC 框架的重点在于同一组织下的服务，通常也在同一个数据中心内。

### 消息传递中的数据流

异步消息传递系统，介于数据库和 RPC 之间。与 RPC 类似的是，客户端的请求（也就是消息）以低延迟传递到另一个进程；而与数据库类似的是，并不通过网络直接发送数据，而是通过消息队列（或叫消息代理， broker）的中介来临时存储消息。

消息代理通常不强制数据模型，因为消息本身只是包含一些元数据的字节序列，所以你可以使用任意的编码格式。如果编码支持向前、向后兼容，你就可以灵活修改和部署。

---

## Reference

《Designing Data-Intensive Application》
