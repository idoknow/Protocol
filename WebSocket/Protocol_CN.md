# esn webSocket协议

基于WebSocket技术的esn通信协议  
此文档是基于WebSocket实现的esn通信协议标准,esn-daemon将支持此协议

## 概述

此协议与基于Socket的通信协议大体相似,具有相同的概念和连接流程。  
以下将介绍不同之处。

## NetPackage通信包

`通信包`是esn的websocket协议的基础构成,用于携带网络中传输的`数据包`  
一个`通信包`即为一串JSON数据,此数据包含以下字段:
| 字段名称 | 字段类型 | 描述 |
|  :----- | :----- | :----- |
|Code|int|数据包代码|
|Crypto|int|是否加密(1:加密)|
|DataPack|string|数据包内容JSON的字符串|

e.g. 此通信包包含了一个编号为0的数据包(PackTest)
```JSON
{
    "Code":0,
    "Crypto":0,
    "DataPack":"{
        \"Integer\":1,
        \"Msg\":\"Hello WebSocket!\",
        \"Token\":\"98965e5647d81efd0df0c691119989df\"
    }"
}
```

## DataPackage数据包

与基于Socket的协议相同，查阅Socket/API_CN.md