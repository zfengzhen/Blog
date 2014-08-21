# Protobuf协议设计

## 采用extension  
cs_msg.proto   

```
// 二进制格式
// _________________________________
// |____|__________________________|
//   |               |           
// 数据包            |
// 整个长度          |
// (4个字节)         |
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

## 不采用extension
cs_msg.proto   

```
// 二进制格式
// ______________________________________________
// |____|____|__MsgHead__|______业务逻辑包______|
//   |     \                    
// 数据包   \        
// 整个长度  \       
// (4个字节)  \      
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
//
// 不足:
// 1 需要维护一套<cmd, 具体业务逻辑包>的键值对

package ProtoCs;

message MsgHead {
    optional int32 cmd = 1;
    optional int32 ret = 2;
    optional uint64 seq = 3;
}
```   

cs_role.proto  

```
import "cs_msg.proto";

package ProtoCs;

enum CsRoleProtoRet {
    option allow_alias = true;
    // 快速注册
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
