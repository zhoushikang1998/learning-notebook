## gRPC

- 通信协议，可以是基于 tcp 的，也可以是基于 http 的。
- 数据格式，一般是一套序列化 + 反序列化机制。
  - 文本的：WebService，JSON-RPC
  - 二进制的：Protocol Buffer

### RPC 应用场景

- 分布式
- 微服务

### 入门技术

- gRPC 流（双向流）
- 发布订阅模式（pubsub 包）

### 进阶技术

- 证书认证
  - gRPC 建立在 HTTP/2 协议之上，对 TLS 提供了很好的支持。
- Token 认证
- 拦截器（过滤器）
  - 通过开源 go-grpc-middleware 包实现拦截链
- 和 web 服务器共存：
  - http
  - https

### gRPC 和 protobuf 的扩展

- 验证器
  - 通过扩展选项模拟默认值特性
  - go-proto-validators ：基于 Protobuf 的扩展特性实现了功能较为强大的验证器功能。
- REST 接口：
  - grpc-gateway 项目：实现了将 gRPC 服务转为 REST 服务的能力
    - 安装 protoc-gen-grpc-gateway 插件
- Protobuf 本身具有反射功能，可以在运行时获取对象的 Proto 文件。
  - grpcurl 工具

## protobuf



### go 环境

#### 安装 protoc

- 下载链接

https://github.com/google/protobuf/releases

- 压缩包解压后，配置环境变量

#### 安装 protoc-gen-go

- 通过该命令安装

```bash
go get github.com/golang/protobuf/protoc-gen-go
```

- 将 protoc-gen-go.exe 放到 bin 目录下，配置环境变量

### proto3 语法

```protobuf
// 必须指定，否则会使用 proto2 语法
syntax = "proto3";

message SearchRequest {
  string query = 1;		// 分配字段编号
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

### 分配字段编号

-  消息中定义的每个字段都有一个**唯一编号**。
  - 字段编号用于在消息二进制格式中标识字段，同时要求消息一旦使用字段编号就不应该改变。
  - 1 到 15 的字段编号需要用 1 个字节来编码，编码同时包括字段编号和字段类型
    - **应将 1 到 15 的编号用在消息的常用字段上**
  - 不能使用 19000 到 19999

### 保留字段

- 将删除字段的编号设置为保留项 `reserved`。
- 可以保留字段编号或名称，但不能在同一条 `reserved` 语句中同时使用字段编号和名称。

### .proto 文件会生成什么

- 对于 **Java**, 编译器会生成一个 `.java` 文件和每个消息类型对应的类，同时包含一个特定的 `Builder`类用于构建消息实例。
- 对于 **Go**， 编译器会生成带有每种消息类型的特定数据类型的定义在`.pb.go` 文件中。



### 枚举

- 每个枚举的定义必须包含一个映射到 0 的常量作为第一个元素。原因是：
  - 必须有一个 0 值，才可以作为数值类型的默认值。
  - 0 值常量必须作为第一个元素，是为了与 proto2 的语义兼容就是第一个元素作为默认值。
- 将相同的枚举值分配给不同的枚举选项常量可以定义别名。
  - 要定义别名需要将 `allow_alisa` 选项设置为 `true`，否则 protocol 编译器当发现别名定义时会报错。
- 枚举的常量值必须在 32 位整数的范围内。因为枚举值在传输时采用的是 varint 编码，同时负值无效因而不建议使用。

### 消息类型也可作为字段类型

- 将 `Result` 消息类型的定义放在同一个 .proto 文件中同时在 `SearchResponse` 消息中指定一个 `Result` 类型的字段



### 嵌套类型

- 支持任意深度的嵌套
- 使用 `Parent.Type` 语法可以在父级消息类型外重用内部定义消息类型



### 消息类型的更新

- `int32`， `uint32`， `int64`， `uint64`， 和 `bool` 是完全兼容的——意味着可以从这些字段其中的一个更改为另一个而不破坏前后兼容性。若解析出来的数值与相应的类型不匹配，会采用与 C++ 一致的处理方案

### 包

- 可以在 .proto 文件中使用 `package` 指示符来避免 protocol 消息类型间的命名冲突。

### JSON 映射

- proto3 支持 JSON 的规范编码，这使得系统间共享数据变得更加容易。

- json 选项：

  - **省略使用默认值的字段**：默认情况下，在 proto3 的 JSON 输出中省略具有默认值的字段。该实现可以使用选项来覆盖此行为，来在输出中保留默认值字段。

  - **忽略未知字段**：默认情况下，proto3 的 JSON 解析器会拒绝未知字段，同时提供选项来指示在解析时忽略未知字段。

  - **使用 proto 字段名称代替 lowerCamelCase 名称**： 默认情况下，proto3 的 JSON 编码会将字段名称转换为 lowerCamelCase（译著：小驼峰）形式。该实现提供选项可以使用 proto 字段名代替。Proto3 的  JSON 解析器可同时接受 lowerCamelCase 形式 和 proto 字段名称。

  - **枚举值使用整数而不是字符串表示**： 在 JSON 编码中枚举值是使用枚举值名称的。提供了可以使用枚举值数值形式来代替的选项。

    

### 命名规范

- 使用驼峰命名法（首字母大写）命名 message，例子：**SongServerRequest**
- 使用下划线命名字段，栗子：**song_name**
- 使用驼峰命名法（首字母大写）命名枚举类型，使用 “大写_大写” 的方式命名枚举值。
  - 每一个枚举值以分号结尾，而非逗号。
- 使用驼峰命名法（首字母大写）命名 RPC 服务以及其中的 RPC 方法

### Base 128 Varints （编码方法）