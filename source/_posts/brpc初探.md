---
title: brpc初探
date: 2020-08-29 10:51:07
urlname: introduct_to_brpc_tmp
tags:
---

最近组里准备把我们的在线召回服务由proxygen http迁移到brpc调用，我负责调研下brpc相关内容。虽然早已久仰brpc的大名，但并没有特意了解过。本文以brpc提供的`echo_c++`例程入手学习，算是初探brpc的文字记录。
<!--- more --->

# 1. protobuf service
brpc基于[protobuf service](https://developers.google.com/protocol-buffers/docs/overview#services)的RPC接口实现RPC服务。
RPC的请求（request）和回复（response）定义为protobuf的message，RPC服务的接口都由protobuf的service定义。protobuf编译器protoc根据protobuf service中选择的语言（`option cc_generic_services = true`）生成rpc服务的接口和stub存根。下面以`service EchoService`为例进行说明。
```protobuf
syntax="proto2";
package echo;

# 告诉protoc要生成C++ Service基类，如果是java或python，则应分别修改为java_generic_services和py_generic_services
option cc_generic_services = true;

message EchoRequest {
    required string message = 1;
};

message EchoResponse {
    required string message = 1;
};

service EchoService {
    rpc Echo (EchoRequest) returns (EchoResponse);
}
```

## 1.1 接口Interface
使用protoc编译这个`.proto`，将生成同名的`EchoService`类表示这个RPC服务。对service中定义的每个rpc方法（如`rpc Echo (EchoRequest) returns (EchoResponse)`），生成的`EchoService`类中都由一个对应的同名虚函数（如`Echo`）。
这些生成的方法是虚函数，但不是纯虚函数。默认实现只是调用`controller->SetFailed()`并带有一条错误信息，指出该方法未实现，然后调用`done`回调。在实现自己的服务时，你必须子类化（subclass）生成的服务并适当实现它的方法。
```c++
class EchoService : public ::google::protobuf::Service {
 protected:
  // This class should be treated as an abstract interface.
  inline EchoService() {};
 public:
  virtual ~EchoService();

  typedef EchoService_Stub Stub;

  static const ::google::protobuf::ServiceDescriptor* descriptor();

  virtual void Echo(::google::protobuf::RpcController* controller,
                       const ::echo::EchoRequest* request,
                       ::echo::EchoResponse* response,
                       ::google::protobuf::Closure* done);

  // implements Service ----------------------------------------------

  const ::google::protobuf::ServiceDescriptor* GetDescriptor();
  void CallMethod(const ::google::protobuf::MethodDescriptor* method,
                  ::google::protobuf::RpcController* controller,
                  const ::google::protobuf::Message* request,
                  ::google::protobuf::Message* response,
                  ::google::protobuf::Closure* done);
  const ::google::protobuf::Message& GetRequestPrototype(
    const ::google::protobuf::MethodDescriptor* method) const;
  const ::google::protobuf::Message& GetResponsePrototype(
    const ::google::protobuf::MethodDescriptor* method) const;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(EchoService);
};
```

## 1.2 存根Stub
protoc也为中每个服务接口生成了一个Stub实现，客户端使用该Stub实现向实现了该service的服务器发送请求。对于上面的`service EchoService`，Stub实现被定义为`class EchoService_Stub`。
```C++
class EchoService_Stub : public EchoService {
 public:
  EchoService_Stub(::google::protobuf::RpcChannel* channel);
  EchoService_Stub(::google::protobuf::RpcChannel* channel,
                   ::google::protobuf::Service::ChannelOwnership ownership);
  ~EchoService_Stub();

  inline ::google::protobuf::RpcChannel* channel() { return channel_; }

  // implements EchoService ------------------------------------------

  void Echo(::google::protobuf::RpcController* controller,
                       const ::echo::EchoRequest* request,
                       ::echo::EchoResponse* response,
                       ::google::protobuf::Closure* done);
 private:
  ::google::protobuf::RpcChannel* channel_;
  bool owns_channel_;
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(EchoService_Stub);
};
```
Stub已将protobuf service中的每个方法实现为围绕channel（around the channel）的一个wrapper。调用Stub的其中一个方法就是简单地调用`channel->CallMethod()`。Stub将所有调用转发到`RpcChannel`，其又是一个抽象接口，你必须根据自己的RPC系统具体实现。换而言之，生成的Stub提供了一个类型安全的接口，用于进行基于protobuf的RPC调用，而无需将你锁定到任何特定的RPC实现中。

Stub实现是生成的service类的子类，而server在实现自己的rpc服务时，也必须子类化生成的service类。也就是说rpc的client实现（class EchoService_Stub）以及rpc server实现（class EchoService的子类）都是生成的类（class EchoService）的子类。
综上，`class EchoService`由server继承实现，是rpc对应的实现。`class EchoServcie_Stub`由client调用，向实现了`class EchoService`的server发送请求。protobuf service不包括RPC的实现。但是，它包含了将生成的service类（EchoService接口、EchoServer_Stub类）连接到你选择的任意RPC实现所需的所有工具。我们只需要提供`RpcChannel`和`RpcController`的实现。


# 2. client
如前所述，pb service不包括RPC的实现。但是，它包含了将生成的service类（EchoService接口、EchoServer_Stub类）连接到你选择的任意RPC实现所需的所有工具。我们只需要提供`RpcChannel`和`RpcController`的实现。但是如果client要实现异步访问，我们还需要提供`Closure`的实现。
我们来看下rpc client中`RpcChannel`和`RpcController`的实现，以及`Closure`的用法。

## 2.1 `brpc::Channel`
`brpc::Channel`代表了与一台或多台服务器的交互通道（communication line）。


## 2.2 `brpc::Controller`
`brpc::Controller`调解（mediates）单个方法调用。controller的主要目的是提供一种方法来操纵（manipulate）每个RPC调用的设置，并找出有关RPC级别的错误。

## 2.3 `google::protobuf::closure`
`google::protobuf::closure`是回调的抽象接口。异步调用RPC时，必须提供一个闭包（closure）以便在过程（procedure）完成时调用。
```c++
class LIBPROTOBUF_EXPORT Closure {
 public:
  Closure() {}
  virtual ~Closure();

  virtual void Run() = 0;

 private:
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(Closure);
};
```
由于`google::protobuf::closure::Run()`是纯虚函数，因此可以将继承并实现该接口的子类对象可以作为异步访问时的回调对象。更简单的，是使用`NewCallback()`函数，自动构造一个使用一组特定参数来调用特定函数或方法的闭包。

### 2.3.1 `google::protobuf::NewCallback()`
目前`goole::protobuf::NewCallback()`函数支持绑定零、一或两个参数。使用`NewCallback()`创建的回调在执行时会自动删除。`NewCallback()`创建的回调仅且只能被调用一次（is to be called exactly once），其通常是用于RPC回调。
需要注意，`NewCallback()`对于参数类型比较敏感。通常，为参数绑定提供的值必须与回调函数接受的类型完全匹配（exactly match）。并且，参数不能是引用。但是，正确类型的指针可以正常工作。

### 2.3.2 `brpc::NewCallback()`
由于protobuf 3把NewCallback设置为私有，r32035后brpc把NewCallback独立于`src/brpc/callback.h`并增加了一些重载。`brpc::NewCallback()`最多支持10个参数，类型要求同`google::protobuf::NewCallback()`。

# 3. server



