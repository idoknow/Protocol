# API实现模式

此文档解释了esn-api-java的实现方法,并定义esn-api的实现细节。

1. 请先阅读esn-daemon的[用户级基本信息说明](https://github.com/EasyNotification/esn-daemon/blob/master/README_CN.md),此文档介绍了esn的一些基本概念。  
2. 阅读[关于协议的说明](https://github.com/EasyNotification/Protocol/blob/master/Protocol_CN.md).  

## 摘要

实现一个esn-api,需要使用网络、多线程同步、异常处理相关的技术。此文档将会介绍esn-api的实现模式标准,包括连接登录、网络数据处理、操作接口实现等内容。  
**注意:此文档中专有名词的语境是Java**

### 命名

请按以下格式命名api仓库:
esn-api-\<language>

### API工作流程

连接到指定服务端口->发送识别码->发送账户和密码登录->启动新线程以处理收到的`通信包`->接受上层调用  

重要逻辑:API在连接到服务并成功登录后,需要启动新线程,用于阻塞接收来自服务端的所有通信包,这些包可能包含连接断开通知(数据包类型:PackResult)、请求JSON语法错误信息(PackResult)、来自其他客户端推送的实时通知(PackRespNotification)等。  
收到这些包后,请从JSON数据中获取此包的`Token`,并将Token作为键存入`等待列表`或者存入`接收列表`。  
以上所声明两个列表均为HashMap,其键类型均为`string(字符串)`用于通过Token索引通信包(其值类型)。    


## 初始化流程

### 数据储存

1. 新建一个对象`ESNSession`(或者结构、句柄等变量类型),此对象中提供了可被上层调用的接口(push pull)等方法(函数)。
2. 在此对象中声明一个HashMap(哈希表):`waitPacks`作为`等待列表`,以`string(字符串)`类型作为其键类型,以`string(字符串)`或其他表示通信包的类型作为其值类型,这个HashMap记录了API正在等待的具有特定Token的通信包。
3. 在此对象中声明一个HashMap(哈希表):`recvPacks`作为`接收列表`,其属性与`等待列表`一致,这个HashMap记录了未被等待的但是被接收到的通信包。
4. 实现一个`随机Token产生函数`,产生的Token要求为string类型,且保证在任意时间间隔内任意地点任意esn-api取得的Token均是唯一的。  
*建议算法:*  
4.1. 定义一个string类型数据`str=<当前时间戳>+":<编程语言名称>:"+<相当规模的随机数>`,例如:`1634299562:java:21377`  
4.2. 对变量`str`进行MD5运算,获得密文作为Token

### 建立连接

注意:通信过程中数据均为大端序,请注意查询编程语言默认的网络数据端序。

1. 使用基于TCP连接的`Socket(套接字)`连接到指定的服务端口(通常表示为\<Host>:\<Port>)  
2. 连接成功后,向服务端发送一个Integer类型数字:119812525 以声明此连接由esn-api发起,遵循esn协议。
3. 以阻塞的方法接收一个来自服务端的Integer类型数字`ProtocolVersion`,这个数字描述了服务端的协议版本,每个服务端发行版对应一个唯一数字,目前的`ProtocolVersion`在1000-1999之间(闭区间)。目前认为一旦收到不在此区间内的协议版本号,应该立即断开连接并抛出语言级异常.如果长时间未收到协议版本号,请检查网络环境和端序问题。

### 登录

1. 创建一个`DataPackage数据包`类型为PackLogin,该JSON包含PackLogin的所有字段并包含随机Token。例见[Protocol_CN.md](Protocol_CN.md)中的PackLogin数据包说明。
2. 获取该数据包的JSON字符串的字节量`size`(请使用UTF8编码)。
3. 向Socket依次写入三个Integer类型数据`code`(该数据包编号,详见[Protocol_CN.md](Protocol_CN.md))、`size`(上一步获取的size)、`encrypted`(常数0,目前不支持加密功能)。
4. 接下来向Socket逐字节写入数据包JSON的数据。
5. 写入完毕,即开始接收通信包,通信包同样以三个Integer类型整数开始(code,size,encrypted)。
6. 逐字节接收数据包JSON数据,字节量由前一步size确定,读取完成后将字节数组转为string字符串类型。
7. 若数据包JSON中的`Result`字段值为`Done`,则登录成功,否则登录失败,详细见[Protocol_CN.md](Protocol_CN.md)。

### 读取此账户权限

### 启动通信包接收线程

1. 新建一个线程以循环进行以下操作,该线程要求在断连时能被关闭并释放资源。
2. 从对端获取一个通信包(具体方法参照[Protocol_CN.md](Protocol_CN.md)),并从JSON中获取`Token`字段的值Token。
3. 检查`waitPacks`等待列表的键列表中是否包含此Token。若不包含,请将此数据包以Token作为键存入`recvPacks`。若列表中包含此Token,则意味着有线程正在等待该数据包,需要通过一些方法通知此线程。  
*举例:Java中的实现方式*  
3.1. 等待此数据包的线程在`waitPacks`的一个值上等待,待接收器接收到数据后调用该值的notify方法使等待中的线程继续获取数据包并继续运行。
4. 无需进行第三步操作的情况:若收到Token为`LogoutPackage`的PackResult数据包,请通知上层调用方:该连接已被服务端主动断开,然后结束该线程并释放相关资源。若收到任何编号为5的数据包(`PackRespNotification`),即为收到新的通知(由本客户端请求的,或由其他客户端实时发送的),请通知上层调用方:收到新的通知。

## 操作接口实现

登录成功后,该连接可以进行推送通知、拉取通知、计数通知、添加账户、删除账户等操作,API需为上层调用方提供这些函数接口。

### 推送通知函数

命名:pushNotification  
参数:target(string),title(string),content(string)  
实现:
1. 向服务端发送一个包装好的含有`PackPush`的具有随机Token的通信包
2. 检查recvPacks中是否有含有该Token的PackResult数据包,若没有,则将Token登记到waitPacks中,并在其值上等待,直到被唤醒后,从recvPacks中根据Token读取PackResult数据包。
3. 检查PackResult中的Result字段和Error字段,若Error字段不为空字符串,则应抛出语言级异常。

### 统计通知数量函数

统计通知ID在某一范围内的发送给当前账户的通知数量

命名:countNotification
参数:from(int),to(int)
返回:统计数量(int)
实现
1. 参照以上函数的流程发送一个`PackCount`包
2. 检查PackResult,若Error字段为空字符串,则继续按照以上流程获取一个PackRespCount包
3. 返回该包中的统计数量

### 拉取指定范围通知函数

命名:requestNotification  
参数:from(int),to(int),limit(int)  
实现:
1. 参照以上函数的流程发送一个`PackRequest`包
2. 检查PackResult

注意:此操作应仅发送一个请求,客户端收到服务端回复的通知应该由接收器处理。

### 拉去近期通知函数

命名:requestRecent
参数:limit(int)
实现:
1. 参照以上函数的流程发送一个`PackReqRecent`包
2. 检查PackResult包

注意:此操作应仅发送一个请求,客户端收到服务端回复的通知应该由接收器处理。

### 添加账户函数

命名:addAccount  
参数:name(string),password(string),privilege(string)  
实现:
1. 参照以上函数的流程发送一个`PackAccountOperation`包,Oper字段设置为`add`,注意:新增的账户可以具有以下权限(push推送权限 pull拉取权限 account账户操作权限),多个权限间请用空格分隔
2. 检查PackResult

注意:不可创建名称重复的账户

### 删除账户函数

命名:removeAccount
参数:name(string),kick(boolean)
实现:
1. 参照以上函数的流程发送一个`PackAccountOperation`包,Oper字段设置为`remove`
2. 检查PackResult

注意:若Kick字段为true,任何以该账户登录的客户端连接将会立即被断开。

## 参考

[esn-api-java](https://github.com/EasyNotification/esn-api-java)