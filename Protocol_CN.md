# esn协议

此文档解释了esn网络协议的细节,面向API开发者。  
阅读此文档前,请先了解esn系统的用户级基本信息。  
详见:https://github.com/EasyNotification/esn-daemon/blob/master/README_CN.md

## NetPackage 通信包

`通信包`是esn协议的基础构成，用于携带网络中传输的`数据包`。

| 字段名称 | 字段类型 | 描述 |
|  :----- | :----- | :----- |
| code|int|数据包代码|
|size|int|数据包长度|
|crypto|int|是否加密(1:加密)|
|dataPack|JSON|数据包内容|

## DataPackage 数据包

`数据包`是符合格式规定的，携带有信息的一串JSON数据。  
其中的字段和数据含义由`通信包`中的`code`限定。  

每个`数据包`均包含字段`token(string)`用于标记和识别此数据包。  

以下是各数据包的列表：  
|code|名称|发送者|字段列表|类型列表|描述|
|:---|:---|:----|:------|:------|:---|
|0   |PackTest|both|Integer,Msg,Token|int,string,string|测试连接|
|1   |PackLogin|client|User,Pass,Token|string,string,string|凭用户名和密码登录|
|2   |PackResult|server|Result,Error,Token|string,string,string|回复请求的结果|
|3 |PackPush|client|Target,Time,Title,Content,Token|string,string,string,string,string|推送通知|
|4|PackRequest|client|From,To,Limit,Token|int,int,int,string|请求指定范围的通知|
|5|PackRespNotification|server|Id,Target,Time,Title,Content,Source,Token|int,string,string,string,string,string,string|向客户端发送一个通知|
|6|PackReqPrivList|both|Priv,Token|string,string|请求此账户的权限列表|
|7|PackAccountOperation|client|Oper,Name,Pass,Priv,Kick,Token|string,string,string,string,boolean,string|进行账户操作|
|8|PackReqRSAKey|client|Token|string|请求RSA公钥|
|9|PackRSAPublicKey|server|PuclicKey,Token|string,string|向客户端发送RSA公钥|
|10|PackReqRecent|client|Limit,Token|int,string|获取近期指定数目条通知|
|11|PackCount|client|From,To,Token|int,int,string|统计指定范围内的通知数量|
|12|PackRespCount|server|Amount,Token|int,string|发回统计数量|

以下是每个数据包的简介:

### PackResult (2) 由服务端发送

由服务端返回请求结果。  
任何由客户端发起的请求(推送、拉取请求、新增账户等)在服务端处理后将会首先回复一个`PackResult`数据包告知处理结果或异常信息。若一个请求需要返回数据(如请求RSA公钥、统计通知数量等),其返回数据将在`PackResult`数据包之后发回。  
*注意:PackResult包的Token通常与请求数据包的Token一致,后文的说明中的举例将省略此字段。*

**一个标准的无异常的结果信息:**
```JSON
{
    "Result":"Done",//字符串"Done"
    "Error":"",//空字符串
    "Token":"98965e5647d81efd0df0c691119989df"//与请求数据包的Token一致
}
```

**一个包含异常信息的结果信息:**
```JSON
{
    "Result":"",//空字符串
    "Error":"SampleErrorMessage",//异常的详细信息
    "Token":"98965e5647d81efd0df0c691119989df"//与请求数据包的Token一致
}
```
建议:API在收到包含异常的结果信息时应抛出语言级异常。  

**一个包含`JSON语法错误`信息的结果信息:**
```JSON
{
    "Result":"",//空字符串
    "Error":"JSON syntax err",//JSON语法错误说明
    "Token":"ErrorPackage"//错误信息包
}
```
注意:服务端在发送一个`JSON语法错误`信息包之后会立即关闭此连接。

### PackTest (0) 由服务端或客户端发送

测试该连接的状态、端序等信息。

```JSON
{
    "Integer":1,//任意整型数据
    "Msg":"Hello esnd!",//任意字符串数据
    "Token":"98965e5647d81efd0df0c691119989df"//任意唯一字符串,建议使用MD5算法获取某随机字符串的MD5密文
}
```

*结果包:*

**PackTest仅有一种结果**

```JSON
{
    "Result":"Done",//字符串"Done"
    "Error":"",//空字符串
}
```

### PackLogin (1) 由客户端发送

以特定账户的名称和密码登录。

```JSON
{
    "User":"root",//用户名称
    "Pass":"changeMe",//该账户的密码
    "Token":"98965e5647d81efd0df0c691119989df"
}
```

*异常结果包:*  

**当前连接不处于`ESTABLISHED`状态,可能已经登录**
```JSON
{
    "Result":"",
    "Error":"Cannot login"
}
```

**验证失败,可能由于密码错误、用户不存在或服务端内部错误**
```JSON
{
    "Result":"",
    "Error":"Login failed:<errorMsg>"
}
```

### PackPush (3) 由客户端发送

推送一个通知。

```JSON
{
    "Target":"root",//目标用户,设置为"_global_"以向全局发送,若要发送给多个用户,请使用半角逗号(,)分隔
	"Time":"2021-10-13,14:36:12",//必须按照此格式设置时间:"yyyy-MM-dd,HH:mm:ss"
	"Title":"Test",//通知标题
	"Content":"Hello root!",//通知内容
	"Token":"98965e5647d81efd0df0c691119989df"
}
```

*异常结果包:*

**未登录**
```JSON
{
    "Result":"",
    "Error":"Not logined"
}
```

**无推送权限**

```JSON
{
    "Result":"",
    "Error":"You do not have push priv"
}
```

**无法存储该通知**
```JSON
{
    "Result":"",
    "Error":"<errMessage>"
}
```

### PackRequest (4) 由客户端发送

请求指定范围的发送给此账户的通知。  

需要了解:一个通知具有其唯一ID。详见:https://github.com/EasyNotification/esn-daemon/blob/master/README_CN.md

```JSON
{
    "From":0,//从此数字的ID开始
    "To":50,//到此数字的ID结束
    "Limit":40,//最大发送此数字条通知,若ID From-To中有大于Limit条发送至该用户的通知,将只返回前Limit条
    "Token":"98965e5647d81efd0df0c691119989df"
}
```
仅返回目标是该用户的通知。  
From和To限制了查找通知的ID范围,Limit限制了最大发送通知量。 
若Limit大于From-To,将返回范围中所有发送给该用户的通知。   
服务端收到此请求后,将立即返回结果包,再发送符合条件的通知数据。  

*异常结果包:*

**未登录**
```JSON
{
    "Result":"",
    "Error":"Not logined"
}
```

**无拉取权限**

当前版本中,这个异常结果将不会发生,因为服务端默认所有用户都具有拉取权限

```JSON
{
    "Result":"",
    "Error":"You do not have pull priv"
}
```

### PackRespNotification (5) 由服务端发送

服务端向客户端发送一个通知。  
在连接建立后,任何时候都有可能会有来自其他客户端的通知被发送到该客户端。

```JSON
{
    "Id":3,//通知唯一ID
    "Target":"root,Bob",//目标用户列表
    "Time":"2021-10-13,10:12:02",//被推送的时间
    "Title":"TestTitle",//通知标题
    "Content":"Hello client!",//通知内容
    "Source":"Alice",//来源,由服务端自动设置
    "Token":"98965e5647d81efd0df0c691119989df"//若此通知是非实时的由本客户端请求拉取的,Token将与PackRequest包的Token一致。若此通知是实时的由另外一个客户端推送给该客户端的,此Token将与另外一个客户端推送请求的Token一致
}
```

建议:API在单独线程实时监听PackRespNotification包,并允许使用者设置`钩子(回调函数)`以处理通知。

*此数据包由服务端发送,非客户端请求,故无结果包*

### PackReqPrivList (6) 由客户端发送并由服务端回复

请求该账户的权限列表。  

**由客户端发送包含以下信息的数据包:**
```JSON
{
    "Priv":"",//空字符串
    "Token":"98965e5647d81efd0df0c691119989df"
}
```

**服务端收到请求后使用同样类型的数据包返回权限信息**

```JSON
{
    "Priv":"push pull account",//该账户所具有的权限由空格分开
    "Token":"98965e5647d81efd0df0c691119989df-1"//Token为请求包的Token+"-1"
}
```

注意:客户端需要先接收结果数据包`PackResult`再接收权限数据包`PackReqPrivList`。

*异常结果包:*

**未登录**
```JSON
{
    "Result":"",
    "Error":"Not logined"
}
```

### PackAccountOperation (7) 由客户端发送

进行账户操作。  
在`Oper`字段指定操作`add`或`remove`以添加或删除账户。

**添加账户**
```JSON
{
    "Oper":"add",//指定add操作
    "Name":"Alice",//新账户名
    "Pass":"alivePassword",//新账户的密码
    "Priv":"push pull account",//新账户的权限(push,pull,account以空格分隔)
    "Kick":"false",//不重要字段,填false或true
    "Token":"98965e5647d81efd0df0c691119989df"
}
```

**删除账户**
```JSON
{
    "Oper":"remove",//指定remove操作
    "Name":"Alice",//要删除的账户名
    "Pass":"",//不重要字段
    "Priv":"",//不重要字段
    "Kick":"true",//是否将已使用此用户身份登录的客户端下线(true或false)
    "Token":"98965e5647d81efd0df0c691119989df"
}
```

*异常结果包:*

**未登录**
```JSON
{
    "Result":"",
    "Error":"Not logined"
}
```

**无账户修改权限**

```JSON
{
    "Result":"",
    "Error":"You do not have account operation priv"
}
```
**无法修改账户**

以下情况会发生此错误:
1. 尝试操作root账户
2. 账户名不符合正则表达式:`^[0-9a-zA-Z_]{1,}$`
3. 尝试添加的账户与某一已存在账户重名
4. 尝试删除不存在的账户
5. `Oper`字段的值不是`add`或`remove`

```JSON
{
    "Result":"",
    "Error":"<errMessage>"
}
```

### PackReqRSAKey (8) 由客户端发送

请求生成RSA公钥。  
注意:服务端生成的公钥在某些语言(Java等)上不可用,目前已禁用通信包加密。

### PackRSAPublicKey (9) 由服务端发送

服务端向客户端返回RSA公钥。  
注意:服务端生成的公钥在某些语言(Java等)上不可用,目前已禁用通信包加密。

### PackReqRecent (10) 由客户端发送

请求近期的发送给该账户的通知。

```JSON
{
    "Limit":40,//指定请求的近期通知数量最大值
    "Token":"98965e5647d81efd0df0c691119989df"
}
```

*异常结果包:*

**未登录**
```JSON
{
    "Result":"",
    "Error":"Not logined"
}
```

**无拉取权限**

当前版本中,这个异常结果将不会发生,因为服务端默认所有用户都具有拉取权限

```JSON
{
    "Result":"",
    "Error":"You do not have pull priv"
}
```

### PackCount (11) 由客户端发送

统计某一ID范围内发送给该账户的通知数量。

```JSON
{
    "From":0,//最小ID
    "To":50,//最大ID,如果想统计所有范围的发送给该账户的数量,请将此值设为0
    "Token":"98965e5647d81efd0df0c691119989df"
}
```
注意:服务端只会统计发送给该账户的通知数量。

**服务端将回复结果包和统计数据包(见下一数据包详情)**

*异常结果包:*

**未登录**
```JSON
{
    "Result":"",
    "Error":"Not logined"
}
```

**无拉取权限**

当前版本中,这个异常结果将不会发生,因为服务端默认所有用户都具有拉取权限

```JSON
{
    "Result":"",
    "Error":"You do not have pull priv"
}
```

### PackRespCount (12) 由服务端发送

服务端回复通知数量统计信息。

```JSON
{
    "Amount":41,//数量统计结果
    "Token":"98965e5647d81efd0df0c691119989df-1"//Token为请求包的Token+"-1"
}
```
