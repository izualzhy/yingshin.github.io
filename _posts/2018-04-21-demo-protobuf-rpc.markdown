---
title: 如何基于protobuf实现一个极简版的RPC
date: 2018-04-21 22:34:44
tags: [protobuf rpc]
---

基于 protobuf 的 RPC 可以说是五花八门，其中不乏非常优秀的代码例如 brpc, muduo-rpc 等。

protobuf 实现了序列化部分，并且预留了 RPC 接口，但是没有实现网络交互的部分。

本文想介绍下，如何实现基于 protobuf 实现一个极简版的 RPC ，这样有助于我们阅读 RPC 源码。

一次完整的 RPC 通信实际上是有三部分代码共同完成：
1. protobuf 自动生成的代码
2. RPC 框架
3. 用户填充代码

本文假设用户熟悉 protobuf 并且有 RPC 框架的使用经验。首先介绍下 protobuf 自动生成的代码，接着介绍下用户填充代码，然后逐步介绍下极简的 RPC 框架的实现思路，相关代码可以直接跳到文章最后。

<!--more-->

## 1. proto

我们定义了`EchoService`, method 为`Echo`.

```cpp
package echo;

option cc_generic_services = true;

message EchoRequest {
    required string msg = 1;
}

message EchoResponse {
    required string msg = 2;
}

service EchoService {
    rpc Echo(EchoRequest) returns (EchoResponse);
}
```

protoc 自动生成`echo.pb.h echo.pb.cc`两部分代码.
其中`service EchoService`这一句会生成`EchoService EchoService_Stub`两个类，分别是 server 端和 client 端需要关心的。

对 server 端，通过`EchoService::Echo`来处理请求，代码未实现，需要子类来 override.

```cpp
class EchoService : public ::google::protobuf::Service {
  ...
  virtual void Echo(::google::protobuf::RpcController* controller,
                       const ::echo::EchoRequest* request,
                       ::echo::EchoResponse* response,
                       ::google::protobuf::Closure* done);
};

void EchoService::Echo(::google::protobuf::RpcController* controller,
                         const ::echo::EchoRequest*,
                         ::echo::EchoResponse*,
                         ::google::protobuf::Closure* done) {
  //代码未实现
  controller->SetFailed("Method Echo() not implemented.");
  done->Run();
}
```


对 client 端，通过`EchoService_Stub`来发送数据，`EchoService_Stub::Echo`调用了`::google::protobuf::Channel::CallMethod`，但是`Channel`是一个纯虚类，需要 RPC 框架在子类里实现需要的功能。

```cpp
class EchoService_Stub : public EchoService {
  ...
  void Echo(::google::protobuf::RpcController* controller,
                       const ::echo::EchoRequest* request,
                       ::echo::EchoResponse* response,
                       ::google::protobuf::Closure* done);
 private:
    ::google::protobuf::RpcChannel* channel_;
};

void EchoService_Stub::Echo(::google::protobuf::RpcController* controller,
                              const ::echo::EchoRequest* request,
                              ::echo::EchoResponse* response,
                              ::google::protobuf::Closure* done) {
  channel_->CallMethod(descriptor()->method(0),
                       controller, request, response, done);
}
```

## 2. server && client

有过 RPC 使用经验的话，都了解 server 端代码类似于这样(参考[brpc echo_c++ server.cpp](https://github.com/brpc/brpc/blob/master/example/echo_c%2B%2B/server.cpp))

```cpp
//override Echo method
class MyEchoService : public echo::EchoService {
public:
  virtual void Echo(::google::protobuf::RpcController* /* controller */,
                       const ::echo::EchoRequest* request,
                       ::echo::EchoResponse* response,
                       ::google::protobuf::Closure* done) {
      std::cout << request->msg() << std::endl;
      response->set_msg(
              std::string("I have received '") + request->msg() + std::string("'"));
      done->Run();
  }
};//MyEchoService

int main() {
    MyServer my_server;
    MyEchoService echo_service;
    my_server.add(&echo_service);
    my_server.start("127.0.0.1", 6688);

    return 0;
}
```
只要定义子类 service 实现 method 方法，再把 service 加到 server 里就可以了。

而 client 基本这么实现(参考[brpc echo_c++ client.cpp](https://github.com/brpc/brpc/blob/master/example/echo_c%2B%2B/client.cpp))

```cpp
int main() {
    MyChannel channel;
    channel.init("127.0.0.1", 6688);

    echo::EchoRequest request;
    echo::EchoResponse response;
    request.set_msg("hello, myrpc.");

    echo::EchoService_Stub stub(&channel);
    MyController cntl;
    stub.Echo(&cntl, &request, &response, NULL);
    std::cout << "resp:" << response.msg() << std::endl;

    return 0;
}
```

这样的用法看起来很自然，但是仔细想想背后的实现，肯定会有很多疑问：

1. 为什么 server 端只需要实现`MyEchoService::Echo`函数，client端只需要调用`EchoService_Stub::Echo`就能发送和接收对应格式的数据？中间的调用流程是怎么样子的？
2. 如果 server 端接收多种 pb 数据（例如还有一个 method `rpc Post(DeepLinkReq) returns (DeepLinkResp);`），那么怎么区分接收到的是哪个格式？
3. 区分之后，又如何构造出对应的对象来？例如`MyEchoService::Echo`参数里的`EchoRequest EchoResponse`，因为 rpc 框架并不清楚这些具体类和函数的存在，框架并不清楚具体类的名字，也不清楚 method 名字，却要能够构造对象并调用这个函数？

可以推测答案在`MyServer MyChannel MyController`里，接下来我们逐步分析下。

## 3. 处理流程

考虑下 server 端的处理流程

1. 从对端接收数据
2. 通过标识机制判断如何反序列化到 request 数据类型
3. 生成对应的 response 数据类型
4. 调用对应的 service-method ，填充 response 数据
5. 序列化 response
6. 发送数据回对端

具体讲下上一节提到的接口设计的问题，体现在2 3 4步骤里，还是上面 Echo 的例子，因为 RPC 框架并不能提前知道`EchoService::Echo`这个函数，怎么调用这个函数呢？

`google/protobuf/service.h`里`::google::protobuf::Service`的源码如下：

```cpp
class LIBPROTOBUF_EXPORT Service {
      virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller,
                          const Message* request,
                          Message* response,
                          Closure* done) = 0;
};//Service
```

Service 是一个纯虚类，`CallMethod = 0`，`EchoService`实现如下

```cpp
void EchoService::CallMethod(const ::google::protobuf::MethodDescriptor* method,
                             ::google::protobuf::RpcController* controller,
                             const ::google::protobuf::Message* request,
                             ::google::protobuf::Message* response,
                             ::google::protobuf::Closure* done) {
  GOOGLE_DCHECK_EQ(method->service(), EchoService_descriptor_);
  switch(method->index()) {
    case 0:
      Echo(controller,
             ::google::protobuf::down_cast<const ::echo::EchoRequest*>(request),
             ::google::protobuf::down_cast< ::echo::EchoResponse*>(response),
             done);
      break;
    default:
      GOOGLE_LOG(FATAL) << "Bad method index; this should never happen.";
      break;
  }
}
```

可以看到这里会有一次数据转化`down_cast`，因此框架可以通过调用`::google::protobuf::ServiceCallMethod`函数来调用`Echo`，数据统一为`Message*`格式，这样就可以解决框架的接口问题了。

再考虑下 client 端处理流程。

`EchoService_Stub::Echo`的实现里:

```cpp
  channel_->CallMethod(descriptor()->method(0),
                       controller, request, response, done);
```

因此先看下`::google::protobuf::RpcChannel`的实现:

```cpp
// Abstract interface for an RPC channel.  An RpcChannel represents a
// communication line to a Service which can be used to call that Service's
// methods.  The Service may be running on another machine.  Normally, you
// should not call an RpcChannel directly, but instead construct a stub Service
// wrapping it.  Example:
//   RpcChannel* channel = new MyRpcChannel("remotehost.example.com:1234");
//   MyService* service = new MyService::Stub(channel);
//   service->MyMethod(request, &response, callback);
class LIBPROTOBUF_EXPORT RpcChannel {
 public:
  inline RpcChannel() {}
  virtual ~RpcChannel();

  // Call the given method of the remote service.  The signature of this
  // procedure looks the same as Service::CallMethod(), but the requirements
  // are less strict in one important way:  the request and response objects
  // need not be of any specific class as long as their descriptors are
  // method->input_type() and method->output_type().
  virtual void CallMethod(const MethodDescriptor* method,
                          RpcController* controller,
                          const Message* request,
                          Message* response,
                          Closure* done) = 0;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(RpcChannel);
};
```

pb 的注释非常清晰，channel 可以理解为一个通道，连接了 rpc 服务的两端，本质上也是通过 socket 通信的。

但是`RpcChannel`也是一个纯虚类，`CallMethod = 0`。

因此我们需要实现一个子类，基类为`RpcChannel`，并且实现`CallMethod`方法，应该实现两个功能：

1. 序列化 request ，发送到对端，同时需要标识机制使得对端知道如何解析(schema)和处理(method)这类数据。
2. 接收对端数据，反序列化到 response

此外还有`RpcController`，也是一个纯虚类，是一个辅助类，用于获取RPC结果，对端IP等。

## 4. 标识机制

上一节提到的所谓标识机制，就是当 client 发送一段数据流到 server ，server 能够知道这段 buffer 对应的数据格式，应该如何处理，对应的返回数据格式是什么样的。

最简单暴力的方式就是在每组数据里都标识下是什么格式的，返回值希望是什么格式的，这样一定能解决问题。

但是 pb 里明显不用这样，因为 server/client 使用相同（或者兼容）的 proto，只要标识下数据类型名就可以了。不过遇到相同类型的 method 也会有问题，例如

```cpp
service EchoService {
    rpc Echo(EchoRequest) returns (EchoResponse);
    rpc AnotherEcho(EchoRequest) returns (EchoResponse)
}
```

因此可以使用 service 和 method 名字，通过 proto 就可以知道 request/response 类型了。

因此，结论是：**我们在每次数据传递里加上`service method`名字就可以了。**

pb 里有很多 xxxDescriptor 的类，`service method`也不例外。例如`GetDescriptor`可以获取`ServiceDescriptor`.

```cpp
class LIBPROTOBUF_EXPORT Service {
  ...

  // Get the ServiceDescriptor describing this service and its methods.
  virtual const ServiceDescriptor* GetDescriptor() = 0;
};//Service
```

通过`ServiceDescriptor`就可以获取对应的`name`及`MethodDescriptor`.

```cpp
class LIBPROTOBUF_EXPORT ServiceDescriptor {
 public:
  // The name of the service, not including its containing scope.
  const string& name() const;
  ...
  // The number of methods this service defines.
  int method_count() const;
  // Gets a MethodDescriptor by index, where 0 <= index < method_count().
  // These are returned in the order they were defined in the .proto file.
  const MethodDescriptor* method(int index) const;
};//ServiceDescriptor
```

而`MethodDecriptor`可以获取对应的`name`及从属的`ServiceDescriptor`

```cpp
class LIBPROTOBUF_EXPORT MethodDescriptor {
 public:
  // Name of this method, not including containing scope.
  const string& name() const;
  ...
  // Gets the service to which this method belongs.  Never NULL.
  const ServiceDescriptor* service() const;
};//MethodDescriptor
```

因此：
1. server 端传入一个`::google::protobuf::Service`时，我们可以记录 service name 及所有的 method name.
2. client 端调用`virtual void CallMethod(const MethodDescriptor* method...`时，也可以获取到 method name 及对应的 service name.

这样，就可以知道发送的数据类型了。

## 5. 构造参数

前面还提到的一个问题，是如何构造具体参数的问题。实现 RPC 框架时，肯定是不知道`EchoRequest EchoResponse`类名的，但是通过`::google::protobuf::Service`的接口可以构造出对应的对象来

```cpp
  //   const MethodDescriptor* method =
  //     service->GetDescriptor()->FindMethodByName("Foo");
  //   Message* request  = stub->GetRequestPrototype (method)->New();
  //   Message* response = stub->GetResponsePrototype(method)->New();
  //   request->ParseFromString(input);
  //   service->CallMethod(method, *request, response, callback);
  virtual const Message& GetRequestPrototype(
    const MethodDescriptor* method) const = 0;
  virtual const Message& GetResponsePrototype(
    const MethodDescriptor* method) const = 0;
```

而`Message`通过`New`可以构造出对应的对象

```cpp
class LIBPROTOBUF_EXPORT Message : public MessageLite {
 public:
  inline Message() {}
  virtual ~Message();

  // Basic Operations ------------------------------------------------

  // Construct a new instance of the same type.  Ownership is passed to the
  // caller.  (This is also defined in MessageLite, but is defined again here
  // for return-type covariance.)
  virtual Message* New() const = 0;
  ...
```

这样，我们就可以得到`Service::Method`需要的对象了。

## 6. Server/Channel/Controller子类实现

前面已经介绍了基本思路，本节介绍下具体的实现部分。

### 6.1. RpcMeta

`RpcMeta`用于解决传递 service-name method-name 的问题，定义如下

```cpp
package myrpc;

message RpcMeta {
    optional string service_name = 1;
    optional string method_name = 2;
    optional int32 data_size = 3;
}
```

其中`data_size`表示接下来要传输的数据大小，例如`EchoRequest`对象的大小。

同时我们还需要一个`int`来表示`RpcMeta`的大小，因此我们来看下`Channel`的实现

### 6.2. Channel

```cpp
//继承自RpcChannel，实现数据发送和接收
class MyChannel : public ::google::protobuf::RpcChannel {
public:
    //init传入ip:port，网络交互使用boost.asio
    void init(const std::string& ip, const int port) {
        _io = boost::make_shared<boost::asio::io_service>();
        _sock = boost::make_shared<boost::asio::ip::tcp::socket>(*_io);
        boost::asio::ip::tcp::endpoint ep(
                boost::asio::ip::address::from_string(ip), port);
        _sock->connect(ep);
    }

    //EchoService_Stub::Echo会调用Channel::CallMethod
    //其中第一个参数MethodDescriptor* method，可以获取service-name method-name
    virtual void CallMethod(const ::google::protobuf::MethodDescriptor* method,
            ::google::protobuf::RpcController* /* controller */,
            const ::google::protobuf::Message* request,
            ::google::protobuf::Message* response,
            ::google::protobuf::Closure*) {
        //request数据序列化
        std::string serialzied_data = request->SerializeAsString();

        //获取service-name method-name，填充到rpc_meta
        myrpc::RpcMeta rpc_meta;
        rpc_meta.set_service_name(method->service()->name());
        rpc_meta.set_method_name(method->name());
        rpc_meta.set_data_size(serialzied_data.size());

        //rpc_meta序列化
        std::string serialzied_str = rpc_meta.SerializeAsString();

        //获取rpc_meta序列化数据大小，填充到数据头部，占用4个字节
        int serialzied_size = serialzied_str.size();
        serialzied_str.insert(0, std::string((const char*)&serialzied_size, sizeof(int)));
        //尾部追加request序列化后的数据
        serialzied_str += serialzied_data;

        //发送全部数据:
        //|rpc_meta大小（定长4字节)|rpc_meta序列化数据（不定长）|request序列化数据（不定长）|
        _sock->send(boost::asio::buffer(serialzied_str));

        //接收4个字节：序列化的resp数据大小
        char resp_data_size[sizeof(int)];
        _sock->receive(boost::asio::buffer(resp_data_size));

        //接收N个字节：N=序列化的resp数据大小
        int resp_data_len = *(int*)resp_data_size;
        std::vector<char> resp_data(resp_data_len, 0);
        _sock->receive(boost::asio::buffer(resp_data));

        //反序列化到resp
        response->ParseFromString(std::string(&resp_data[0], resp_data.size()));
    }

private:
    boost::shared_ptr<boost::asio::io_service> _io;
    boost::shared_ptr<boost::asio::ip::tcp::socket> _sock;
};//MyChannel
```

通过实现`Channel::CallMethod`方法，我们就可以在调用子类方法，例如`EchoService_Stub::Echo`时自动实现数据的发送/接收、序列化/反序列化了。

### 6.3 Server

`Server`的实现会复杂一点，因为可能注册多个`Service::Method`，当接收到 client 端的数据，解析`RpcMeta`得到`service-name method-name`后，需要找到对应的`Service::Method`，注册时就需要记录这部分信息。

因此，我们先看下`add`方法的实现：

```cpp
class MyServer {
public:
    void add(::google::protobuf::Service* service) {
        ServiceInfo service_info;
        service_info.service = service;
        service_info.sd = service->GetDescriptor();
        for (int i = 0; i < service_info.sd->method_count(); ++i) {
            service_info.mds[service_info.sd->method(i)->name()] = service_info.sd->method(i);
        }

        _services[service_info.sd->name()] = service_info;
    }
    ...
private:
    struct ServiceInfo{
        ::google::protobuf::Service* service;
        const ::google::protobuf::ServiceDescriptor* sd;
        std::map<std::string, const ::google::protobuf::MethodDescriptor*> mds;
    };//ServiceInfo

    //service_name -> {Service*, ServiceDescriptor*, MethodDescriptor* []}
    std::map<std::string, ServiceInfo> _services;
```

我在实现里，`_services`记录了 service 及对应的`ServiceDescriptor MethodDescriptor`。而`ServiceDescritpr::FindMethodByName`方法可以查找 method ，因此不记录`method_name`也可以。不过出于性能考虑，我觉得还可以记录更多，例如 req/resp 数据类型等。

注册 service 后，就可以启动 server 监听端口和接收数据了

```cpp
//监听ip:port，接收数据
void MyServer::start(const std::string& ip, const int port) {
    boost::asio::io_service io;
    boost::asio::ip::tcp::acceptor acceptor(
            io,
            boost::asio::ip::tcp::endpoint(
                boost::asio::ip::address::from_string(ip),
                port));

    while (true) {
        auto sock = boost::make_shared<boost::asio::ip::tcp::socket>(io);
        acceptor.accept(*sock);

        std::cout << "recv from client:"
            << sock->remote_endpoint().address()
            << std::endl;

        //接收4个字节：rpc_meta长度
        char meta_size[sizeof(int)];
        sock->receive(boost::asio::buffer(meta_size));

        int meta_len = *(int*)(meta_size);

        //接收rpc_meta数据
        std::vector<char> meta_data(meta_len, 0);
        sock->receive(boost::asio::buffer(meta_data));

        myrpc::RpcMeta meta;
        meta.ParseFromString(std::string(&meta_data[0], meta_data.size()));

        //接收req数据
        std::vector<char> data(meta.data_size(), 0);
        sock->receive(boost::asio::buffer(data));

        //数据处理
        dispatch_msg(
                meta.service_name(),
                meta.method_name(),
                std::string(&data[0], data.size()),
                sock);
    }
}
```

`start`启动一个循环，解析`RpcMeta`数据并接收 request 数据，之后交给 dispatch_msg 处理。

```cpp
void MyServer::dispatch_msg(
        const std::string& service_name,
        const std::string& method_name,
        const std::string& serialzied_data,
        const boost::shared_ptr<boost::asio::ip::tcp::socket>& sock) {
    //根据service_name method_name查找对应的注册的Service*
    auto service = _services[service_name].service;
    auto md = _services[service_name].mds[method_name];

    std::cout << "recv service_name:" << service_name << std::endl;
    std::cout << "recv method_name:" << method_name << std::endl;
    std::cout << "recv type:" << md->input_type()->name() << std::endl;
    std::cout << "resp type:" << md->output_type()->name() << std::endl;

    //根据Service*生成req resp对象
    auto recv_msg = service->GetRequestPrototype(md).New();
    recv_msg->ParseFromString(serialzied_data);
    auto resp_msg = service->GetResponsePrototype(md).New();

    MyController controller;
    auto done = ::google::protobuf::NewCallback(
            this,
            &MyServer::on_resp_msg_filled,
            recv_msg,
            resp_msg,
            sock);
    //调用Service::Method（即用户实现的子类方法）
    service->CallMethod(md, &controller, recv_msg, resp_msg, done);
```

用户填充`resp_msg`后，会调用`done`指定的回调函数（也就是我们在 `MyEchoService::Echo` 代码里对应的`done->Run()`这一句）。

在用户填充数据后，`on_resp_msg_filled`用于完成序列化及发送的工作。

```cpp
void MyServer::on_resp_msg_filled(
        ::google::protobuf::Message* recv_msg,
        ::google::protobuf::Message* resp_msg,
        const boost::shared_ptr<boost::asio::ip::tcp::socket> sock) {
    //avoid mem leak
    boost::scoped_ptr<::google::protobuf::Message> recv_msg_guard(recv_msg);
    boost::scoped_ptr<::google::protobuf::Message> resp_msg_guard(resp_msg);

    std::string resp_str;
    pack_message(resp_msg, &resp_str);

    sock->send(boost::asio::buffer(resp_str));
}
```

`pack_message`用于打包数据，其实就是在序列化数据前插入4字节长度数据

```cpp
    void pack_message(
            const ::google::protobuf::Message* msg,
            std::string* serialized_data) {
        int serialized_size = msg->ByteSize();
        serialized_data->assign(
                    (const char*)&serialized_size,
                    sizeof(serialized_size));
        msg->AppendToString(serialized_data);
    }
```

程序输出如下

```
$ ./client
resp:I have received 'hello, myrpc.'
```

```
$ ./server
recv from client:127.0.0.1
recv service_name:EchoService
recv method_name:Echo
recv type:EchoRequest
resp type:EchoResponse
hello, myrpc.
```

完整代码，打包放在了[Tiny-Tools](https://github.com/yingshin/Tiny-Tools/tree/master/protobuf-rpc-demo)，使用 cmake 编译，注意指定 protobuf boost 库的路径。
