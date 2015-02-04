# ProtoBuf懒读取  

**我所遇到需要用到ProtoBuf懒读取的两种情况**  
1 采用纯Protobuf进行通信的proxy在中转数据的时候，只需要读取包中的某几个字段，比如中转数据的方式（平均分发，随机分发，按key取余分发），发送数据的server地址（后端worker进程处理后可能需要直接给请求服务的server回包），这个时候需要懒读取需要的字段，而不去全部解析整个Protobuf，导致性能下降      
2 采用Protobuf做数据存储协议描述的，游戏数据采用Protobuf描述，存储到通用的nosql的key-value结构中或者mysql的blob数据中，当不采用内存池缓存玩家全部数据的方式，而采用CacheSvr机制（CacheSvr缓存Protobuf原始二进制数据），GameSvr（无状态）在每次逻辑时都去CacheSvr读写数据，可能每次逻辑都只要其中的某几个字段的时候需要懒读取  

**在项目中，我们采用纯Protobuf协议，中间不含额外的其他二进制协议  
第一种情况我们需要采用Protobuf懒读取处理   
第二种情况我们采用GameSvr缓存玩家全部数据方式（GameSvr是用状态的），所以还不需要懒读取**  

![](https://github.com/zfengzhen/Blog/blob/master/img/protobuf懒读取_proxy.jpg)   

第一种情况举例：  
```  
message Head {
	optional int32 cmd = 1;
    optional int32 ret = 2;
    optional uint64 seq = 3;
    optional int32 route_type = 4;
    optional int32 route_key = 5;
    optional string source_ip = 6;
}

message Msg {
    optional Head head = 1;
    optioanl Body body = 2;
}  
```   

对于纯Protobuf协议，proxy只需要message Head中的相应字段，而对于Body字段不需要解析，透明转发就好。  
但对于Protobuf所提供的API， msg.ParseFromString()等解析函数，都会解析整个对象，把二进制中所有数据填充到Protobuf的Msg对象中，填充的过程对于repeated采用了::google::protobuf::RepeatedPtrField<>采用new分配内存，对于没有定义的字段存入::google::protobuf::UnknownFieldSet，UnknownFieldSet采用vector以及string做存储。  

```c++  
class LIBPROTOBUF_EXPORT UnknownFieldSet {
  ……
  std::vector<UnknownField>* fields_;
  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(UnknownFieldSet);
};

// Represents one field in an UnknownFieldSet.
class LIBPROTOBUF_EXPORT UnknownField {
 ……
 public:
  enum Type {
    TYPE_VARINT,
    TYPE_FIXED32,
    TYPE_FIXED64,
    TYPE_LENGTH_DELIMITED,
    TYPE_GROUP
  };
  uint32 number_;
  uint32 type_;

  union LengthDelimited {
    string* string_value_;
  };

  union {
    uint64 varint_;
    uint32 fixed32_;
    uint64 fixed64_;
    mutable union LengthDelimited length_delimited_;
    UnknownFieldSet* group_;
  };
};  
···  

查阅一些懒解析的方法， 有采用重新定义一份Protobuf描述文件的办法进行：  
···
message LazyMsg {
    optional Head head = 1;
}  
```
   
这种方法把Msg序列化的二进制数据按LazyMsg进行解析，Body的数据会按UnkonwnFieldSet进行处理，会通过相应的API调用string以及vector存入UnkonwnFieldSet，此时虽然不需要具体去解析Body里面的各种嵌套的Message了，效率确实有所提高，但是还是会把数据当成一段二进制存储到UnkonwnFieldSet中，会调用sring以及vector，底层调用new进行内存分配，而且得重新定义一份Protobuf描述文件，增加出错成本。  

查阅Protobuf的源文件中，发现有能够解析序列化后的二进制数据相应的类和API。  
protobuf/src/google/protobuf/io/coded_stream.h  

```c++
class LIBPROTOBUF_EXPORT CodedInputStream {
  ……
}
```

protobuf/src/google/protobuf/wire_format_lite.h  
```c++
class LIBPROTOBUF_EXPORT WireFormatLite {
 public:
  ……
  static bool SkipField(io::CodedInputStream* input, uint32 tag); 
  template<typename MessageType>
  static inline bool ReadMessageNoVirtual(input, MessageType* value);
  ……
}
```

通过这两个类可以写一个只读取某个tag值的工具类ProtobufReader，只读取你想要的字段，而且性能非常好。 
我已经写了一个，参见[ProtobufReader]( https://github.com/zfengzhen/FullTest/blob/master/src/protobuf_test/protobuf_reader.h)   
写了个测试用例，结果如下：  
![](https://github.com/zfengzhen/Blog/blob/master/img/protobuf懒读取_测试结果.jpg)   
