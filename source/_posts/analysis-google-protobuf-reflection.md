---
title: 解析Google Protocol Buffer消息类型的自动反射原理
id: 236
categories:
  - 网络编程
date: 2015-03-14 21:46:10
tags:
---

Google Protocol Buffer（简称protobuf）是Google内部混合语言数据标准，protobuf是一种紧凑的可扩展的二进制消息格式，适合做网络数据传输、数据存储的消息格式。

protobuf作为网络传输的消息格式，有两个问题需要解决：

*   消息长度。protobuf打包的数据没有自带长度信息，需要应用程序自己在发送和接收消息的时候做切分。
*   消息类型。protobuf打包的数据没有自带消息类型，需要由发送方把类型信息传给接收方，接收方再根据收到的消息类型创建具体的Message对象。
    第一个问题很好解决，可以在protobuf消息的前面附加一个固定长度的HeaderLen即可。

第二个问题可能很多人会想到在HeaderLen和protobuf消息之间再加上一个消息类型数据段，由接收方根据不同的类型创建相应的Message对象。但其实protobuf已经自带了消息类型反射的功能。

## 自动反射

Google Protocol Buffer具有reflection的功能，可以根据type name创建相应的类型的Message对象。（下图引用自[陈硕的blog](http://blog.csdn.net/solstice/article/details/6300108)）

[![](/images/blog-analysis_google_protobuf_reflection.png)](/images/blog-analysis_google_protobuf_reflection.png)

protobuf自动反射功能的关键所在就是Descriptor了，每个Message对象对应一个相应Descriptor。尽管创建Mesasge对象的过程中没有直接调用Descriptor，但是它却在其中起到了关键桥梁的作用。<!--more-->

```c++
#include <google/protobuf/descriptor.h>
#include <google/protobuf/message.h>
#include <string>

google::protobuf::Message* createMessage(const std::string& type_name)
{
    google::protobuf::Message* message = NULL;
    const google::protobuf::Descriptor* descriptor =
        google::protobuf::DescriptorPool::generated_pool()->FindMessageTypeByName(type_name);
    if (descriptor)
    {
        const google::protobuf::Message* prototype =
            google::protobuf::MessageFactory::generated_factory()->GetPrototype(descriptor);
        if (prototype)
        {
            message = prototype->New();
        }
    }
    return message;
}
```
这样使用type_name传参调用DescriptorPool的FindMessageTypeByName拿到一个const类型的Descriptor，然后再通过Descriptor的GetPrototype获得const类型的Message，最后经由const Message的New即可得到一个可自由读写的Message对象。但是需要注意的是这个对象是动态创建的，调用者在使用完毕后必须动态释放掉它，所以推荐使用shared_ptr管理该对象。

## 原理

知道了如何使用protobuf进行消息类型的自动反射后，下面就来讲讲它是如何实现的。

首先我们在源码中找DescriptorPool::FindMessageTypeByName的实现：
```c++
const Descriptor* DescriptorPool::FindMessageTypeByName(
    const string& name) const {
  Symbol result = tables_->FindByNameHelper(this, name);
  return (result.type == Symbol::MESSAGE) ? result.descriptor : NULL;
}
```

发现其中调用了tables_的FindByNameHelper函数，关于table_可以在DescriptorPool定义中找到

```c++
// This class contains a lot of hash maps with complicated types that
// we'd like to keep out of the header.
class Tables;
scoped_ptr<Tables> tables_;
```
table_是一个数据表项的集合（在目前手上的protobuf 2.6版本中是std::map和std::set的集合，其中的HashMap也是由std::map和std::set实现的），这样基本就可以推断出是在这个数据表项集合中通过type name查找得到相应类型的Descriptor。

这个时候就有个疑问了，这个数据表项集合中的内容是什么时候生成的？

先来看一个简单的protobuf文件，rpc.proto
```
package blink;

enum MessageType
{
    REQUEST = 1;
    RESPONSE = 2;
    ERROR = 3;  // not used
}

enum ErrorCode
{
    NO_ERROR = 0;
    WRONG_PROTO = 1;
    NO_SERVICE = 2;
    NO_METHOD = 3;
    INVALID_REQUEST = 4;
    INVALID_RESPONSE = 5;
    TIMEOUT = 6;
}

message RpcMessage
{
    required MessageType type = 1;
    required fixed64 id = 2;

    optional string service = 3;
    optional string method = 4;
    optional bytes request = 5;
    optional bytes response = 6;
    optional ErrorCode error = 7;
}
```
在使用protobuf编译器生成得到的rpc.pb.cc文件中可以看到这样一个函数
```c++
void protobuf_AddDesc_rpc_2eproto() {
  static bool already_here = false;
  if (already_here) return;
  already_here = true;
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  ::google::protobuf::DescriptorPool::InternalAddGeneratedFile(
    "\n\trpc.proto\022\005blink\"\237\001\n\nRpcMessage\022 \n\004typ"
    "e\030\001 \002(\0162\022.blink.MessageType\022\n\n\002id\030\002 \002(\006\022"
    "\017\n\007service\030\003 \001(\t\022\016\n\006method\030\004 \001(\t\022\017\n\007requ"
    "est\030\005 \001(\014\022\020\n\010response\030\006 \001"
    "(\014\022\037\n\005error\030\007 \001"
    "(\0162\020.blink.ErrorCode*3\n\013MessageType\022\013\n\007R"
    "EQUEST\020\001\022\014\n\010RESPONSE\020\002\022\t\n\005ERROR\020\003*\201\001\n\tEr"
    "rorCode\022\014\n\010NO_ERROR\020\000\022\017\n\013WRONG_PROTO\020\001\022\016"
    "\n\nNO_SERVICE\020\002\022\r\n\tNO_METHOD\020\003\022\023\n\017INVALID"
    "_REQUEST\020\004\022\024\n\020INVALID_RESPONSE\020\005\022\013\n\007TIME"
    "OUT\020\006", 365);
  ::google::protobuf::MessageFactory::InternalRegisterGeneratedFile(
    "rpc.proto", &protobuf_RegisterTypes);
  RpcMessage::default_instance_ = new RpcMessage();
  RpcMessage::default_instance_->InitAsDefaultInstance();
  ::google::protobuf::internal::OnShutdown(&protobuf_ShutdownFile_rpc_2eproto);
}
```
可以看到这个函数将上述.proto文件的元信息传入DescriptorPool::InternalAddGeneratedFile了。
```c++
void DescriptorPool::InternalAddGeneratedFile(
    const void* encoded_file_descriptor, int size) {
  InitGeneratedPoolOnce();
  GOOGLE_CHECK(generated_database_->Add(encoded_file_descriptor, size));
}
```
进而又传入了generated_database_的Add函数中去了，在源码中找到generated_database_的定义
```c++
EncodedDescriptorDatabase* generated_database_ = NULL;

inline void InitGeneratedPoolOnce() {
  ::google::protobuf::GoogleOnceInit(&generated_pool_init_, &InitGeneratedPool);
}

static void InitGeneratedPool() {
  generated_database_ = new EncodedDescriptorDatabase;
  generated_pool_ = new DescriptorPool(generated_database_);

  internal::OnShutdown(&DeleteGeneratedPool);
}
```
这样在之前InternalAddGeneratedFile中就初始化了generated_database_，接着找到generated_database_的类型EncodedDescriptorDatabase及其add的定义。
```c++
class LIBPROTOBUF_EXPORT EncodedDescriptorDatabase : public DescriptorDatabase {
 public:
  // ...
  bool Add(const void* encoded_file_descriptor, int size);
  // ...
 private:
  SimpleDescriptorDatabase::DescriptorIndex<pair<const void*, int> > index_;
};

bool EncodedDescriptorDatabase::Add(
    const void* encoded_file_descriptor, int size) {
  FileDescriptorProto file;
  if (file.ParseFromArray(encoded_file_descriptor, size)) {
    return index_.AddFile(file, make_pair(encoded_file_descriptor, size));
  } else {
    GOOGLE_LOG(ERROR) << "Invalid file descriptor data passed to "
                  "EncodedDescriptorDatabase::Add().";
    return false;
  }
}
```
之前的.proto元数据传入了index_成员的AddFile函数中，而index_的类型DescriptorIndex<pair<const void*, int> >的定义是:
```c++
template <typename Value>
  class DescriptorIndex {
   public:
    // Helpers to recursively add particular descriptors and all their contents
    // to the index.
    bool AddFile(const FileDescriptorProto& file,
                 Value value);
    // ...

   private:
    map<string, Value> by_name_;
    map<string, Value> by_symbol_;
    map<pair<string, int>, Value> by_extension_;
};
```
果然，DescriptorIndex实际上是一些数据表项集合的包装，也就是说之前的.proto元数据存入了数据表项集合中。

也就是说在protobuf生成的rpc.pb.cc文件中的protobuf_AddDesc_rpc_2eproto函数可以将.proto的元数据存入数据集合中。

接着，我们可以在rpc.pb.cc中发现下面一段代码：
```c++
struct StaticDescriptorInitializer_rpc_2eproto {
  StaticDescriptorInitializer_rpc_2eproto() {
    protobuf_AddDesc_rpc_2eproto();
  }
} static_descriptor_initializer_rpc_2eproto_;
```
在全局对象static_descriptor_initializer_rpc_2eproto_的构造函数中会调用protobuf_AddDesc_rpc_2eproto函数。我们知道C++全局对象的初始化是在进入main函数之前进行的，也就是说在程序一开始的时候就已经将.proto文件中的元信息导入到了数据表项集合中去了，当然前提是该程序必须链接了rpc.pb.cc。

如此，protobuf的自动反射就可以正常的运转了。

## 参考文献

1.  Google Developers. [Protocol Buffers](https://developers.google.com/protocol-buffers/)
2.  陈硕. [一种自动反射消息类型的 Google Protobuf 网络传输方案, 2011](http://blog.csdn.net/solstice/article/details/6300108)
