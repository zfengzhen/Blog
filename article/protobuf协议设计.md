# Protobuf协议设计
**作者: fergus (zfengzhen@gmail.com)**    

## 采用extension  
cs_msg.proto   

```
// 二进制格式
// _________________________________
// |____|__________________________|
//   |               |           
// 数据包            |
// 整个长度          |
// (uint16_t)        |
//             Msg序列化后的二进制字符串
//
// 1 根据数据包大小, 解包Msg
// 2 根据head中的cmd字段, 解包extension
// 3 其中head中的cmd字段为extension的tag值
//   比如ProtoCs::kLoginReqFieldNumber
//
// 优点:
// 1 定义了extension的tag可以用于具体的extension的获取
//   不用再定义一套<cmd, 具体Body结构体>对应的键值对
// 2 各个模块定义分离, 用法上通过extension获取具体字段
//
// 使用方法:
// 1 msg->HasExtension(ProtoSs::conn_data)
// 2 msg->ClearExtension(ProtoCs::login_req)
// 3 const ProtoCs::LoginReq& login_req = msg->GetExtension(ProtoCs::login_req)
// 4 ProtoCs::LoginReq* login_req = msg->MutableExtension(ProtoCs::login_req)
// 
// 不足:
// 1 脚本语言(Python, Lua)中无法获取ProtoCs::kLoginReqFieldNumber
// 2 protobuf的ExtensionRegistry采用hash_map实现, 虽然google实现的hash_map性能
//   很高, 但是比不使用extension的性能稍低, 网上有同学测试不带extension的会快2.64倍 

package ProtoCs;

message MsgHead {
    optional int32 cmd = 1;
    optional int32 ret = 2;
    optional uint64 seq = 3;
}

message Msg {
    optional MsgHead head = 1;
    extensions 50 to 10000; 
}
```   

cs_role.proto  

```
import "cs_msg.proto";

package ProtoCs;

enum CsRoleProtoRet {
    option allow_alias = true;
    RET_LOGIN_OK = 0;
    RET_LOGIN_FAILED = -1;
    RET_LOGIN_GAMESVR_FULL = -2;
}

message LoginReq {
    optional bytes account = 1;
    optional bytes password = 2;
}

message LoginRes {
    optional bool numb = 1;
}

extend Msg {
    optional LoginReq login_req = 300;
    optional LoginRes login_res = 301;
}
```

## 不采用extension, 业务逻辑包不用MsgBody嵌套
cs_msg.proto   

```
// 二进制格式
// ______________________________________________
// |____|____|__MsgHead__|______业务逻辑包______|
//   |     \                    
// 数据包   \        
// 整个长度  \       
// (uint16_t)  \      
//           MsgHead包长度
//
// 1 根据MsgHead大小, 解包MsgHead
// 2 根据head中的cmd字段以及业务逻辑包大小, 解包具体业务逻辑包
//
// 优点:
// 1 将MsgHead和业务逻辑包分离, 没有用具体的一个Msg去再包装一层,
//   接入server可能要用到MsgHead的一些字段去做路由或者鉴权,
//   这样就不用把整个Msg解析出来, 提高性能
// 2 业务逻辑包并没有通过一个类似于MsgBody去包装, 
//   这样的话各个模块定义依然分离, 而且减少了一层包装, 提高性能
// 3 简单的使用, 脚本语言(Python, Lua)都支持这些基本功能
// 4 键值对也可以采用字段tag, <kLoginReqFieldNumber, message LoginReq>
//
// 不足:
// 1 业务请求包只能通过char*指针进行传参

package ProtoCs;

message MsgHead {
    optional int32 cmd = 1;
    optional int32 ret = 2;
    optional uint64 seq = 3;
}
```   

cs_role.proto  

```
package ProtoCs;

enum CsRoleProtoRet {
    option allow_alias = true;
    RET_LOGIN_OK = 0;
    RET_LOGIN_FAILED = -1;
    RET_LOGIN_GAMESVR_FULL = -2;
}

message LoginReq {
    optional bytes account = 1;
    optional bytes password = 2;
}

message LoginRes {
    optional bool numb = 1;
}
```  

## 业务逻辑包不采用MsgBody统一封装时遇到的麻烦处理 2013.08.26  

不采用MsgBody统一封装时遇到的麻烦, 在封装消息处理时, 参数的传递非常让人讨厌.   
比如:  
有一个函数负责MsgHead包头处理, 根据包头里面的cmd查找具体处理函数, 这个时候如果没有MsgBody的封装时, 只能传入字符串指针, 等处理处理函数直接序列化到该指针指向的缓冲区, 还得提供缓冲区的大小; 如果通过MsgBody进行包装的话, 在整体处理函数的时候, 建立一个MsgBody的实例, 传入到具体的处理函数, 函数返回时, 整体处理函数进行组包, 传递给发包的函数, 整体一气呵成. 对于不封装MsgBody的方式, 只是多了一层message的嵌套开销, 换来的是整体代码的阅读性更强, 以及在看协议的时候, MsgBody集中了各个具体业务逻辑, 对于有的所有业务逻辑更加清晰.   


```
// 二进制格式
// ______________________________________________
// |____|____|__MsgHead__|________MsgBody_______|
//   |     \                    
// 数据包   \        
// 整个长度  \       
// (uint16_t)  \      
//           MsgHead包长度
//
// 1 根据MsgHead大小, 解包MsgHead
// 2 根据head中的cmd字段以及MsgBody的大小, 解包具体业务逻辑包
//
// 优点:
// 1 将MsgHead和MsgBody分离, 没有用具体的一个Msg去再包装一层,
//   接入server可能要用到MsgHead的一些字段去做路由或者鉴权,
//   这样就不用把整个Msg解析出来, 提高性能
// 2 简单的使用, 脚本语言(Python, Lua)都支持这些基本功能
// 3 将CS, SS的MsgHead和MsgBody的分开定义,  
//   不过度的把SS协议字段暴露给CS, 交给CONNSVR去转换
// 4 键值对也可以采用字段tag, <kLoginReqFieldNumber, message LoginReq>
//
```

cs_msg_head.proto  

```
package ProtoCs;

message MsgHead {
    optional int32 cmd = 1;
    optional int32 ret = 2;
    optional uint64 seq = 3;
}
```   
cs_msg_body.proto

```
import "cs_role.proto"

package ProtoCs;

message MsgBody {
    optional LoginReq login_req = 11; 
    optional LoginRes login_res = 12; 
}
```


cs_role.proto  

```
package ProtoCs;

enum CsRoleProtoRet {
    option allow_alias = true;
    RET_LOGIN_OK = 0;
    RET_LOGIN_FAILED = -1;
    RET_LOGIN_GAMESVR_FULL = -2;
}

message LoginReq {
    optional bytes account = 1;
    optional bytes password = 2;
}

message LoginRes {
    optional bool numb = 1;
}
```  

