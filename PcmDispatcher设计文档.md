---
title: PcmDispatcher设计文档 
tags: 软件,设计
renderNumberedHeading: true
grammar_cjkRuby: true
grammar_sequence: true
grammar_plantuml: true
---

# 项目背景
## 当前Pcm接口存在的问题
1. Pcm回调地址只能配置成一个，无法供多个业务系统同时调用

![直接调用pcm示意图](./attachments/1582188206323.drawio.html)
可以看出基于现有架构，一个PCM部署单元只能服务于一个业务系统

2. Pcm回调接口采用单线程，如果接收回调的服务处理缓慢，会导致回调请求在pcm内部挤压，严重时会导致卡单

# 解决方案
1. 为原始Pcm系统的调用和回调接口封装一个代理层服务——PcmDispatcher，用于被多个调用方调用并可以回调多个调用方，内部采用pcm_call_log来记录每个回调的地址，并基于单号和回调时间进行匹配，匹配后发送回调

![PcmDispatcher架构图](./attachments/1582188810461.drawio.html)

# 关键设计
1. PcmDispatcher整体调用逻辑时序图
```sequence?title=服务调用关系图
业务调用方 -> PcmDispatcher: 调用pcmCall接口
Note left of PcmDispatcher: 生成event对象（状态为1），存入pcm_call_log表，并加入内部请求队列
PcmDispatcher -> 业务调用方: 返回event Id
PcmDispatcher -> Pcm服务: 内部线程池从请求队列消费event，调用Pcm
Pcm服务 -> PcmDispatcher: Pcm返回成功，则event状态为2，失败状态为3，更新db
Pcm服务 -> PcmDispatcher callback: pcm发送回调
Note right of PcmDispatcher callback: 从报文中解析出applyno，从db中查询该单号对应的event，修改状态为4，更新db，发送event到回调队列
PcmDispatcher callback -> Pcm服务: 回复成功收到回调
PcmDispatcher callback -> 业务调用方:线程池从回调队列消费event，发送回调消息，状态为5

业务调用方 -> PcmDispatcher callback: 回复成功接收（状态为6）或失败（状态为7），修改event状态并更新db

```
2. 业务端pcmcall调用流程图
``` plantuml!
@startuml
|业务调用方|
start
-> 调用pcmcall查询征信;
|#AntiqueWhite|PcmDispatcher|
:验证查询报文，生成pcm调用Event对象(内部包含回调地址和查询请求);
:将pcm调用Evnet对象存入pcm查询队列;
:将pcm调用Evnet对象存入pcm_call_log表;
detach;
|进程内Threadpool|
:从pcm查询队列消费event;
:从event中获取查询参数，封装报文请求pcm接口;
|#AntiqueWhite|pcm|
:pcm征信查询接口;
|进程内Threadpool|
:根据返回值修改pcm_call_log表的状态;
stop
@enduml
``` 
3. pcm回调接收逻辑及回调业务方流程图 
``` plantuml!
@startuml
|Pcm|
start
-> 回调Pcm Dispatcher;
|#AntiqueWhite|PcmD Callback Serivce|
:解析回调报文，提取applyno;
if (提取applyno成功) then
  -[#red]-> 成功;
  :根据applyno查询pcm_call_log表，从中反序列化成回调event对象;
  if (查得event对象) then
    -> 查得;
    :修改event对象的状态为接收到pcm回调，并同步更新pcm_call_log表;
    #HotPink:发送回调event对象到回调队列;
    :返回pcm成功接收标识;
    detach
  else
    -[#black,dotted]->
    :记录错误日志;
    :忽略，返回pcm成功接收标识;
    detach
  endif
else
  -[#black,dotted]->
  :记录错误日志;
  :返回pcm成功接收标识;
  detach
endif

|进程内Threadpool|
:从回调队列消费event;
:从event中获取回调参数，封装报文回调业务调用方;
|#AntiqueWhite|业务调用方|
:接收回调服务;
|进程内Threadpool|
if (返回结果) then
    -> 成功;
    #HotPink:修改event对象的状态为回调成功，并同步更新pcm_call_log表;
    
  else
    -> 失败 或 异常;
    -[#black,dotted]->
    #HotPink:修改event对象的状态为回调失败，并同步更新pcm_call_log表;
    
  endif
stop
@enduml
``` 
4. pcm调用状态转换图
``` plantuml!
@startuml
[*] --> State10
State10 : 接收到业务端pcmcall请求
State10 -> State15

State15 :pcmcall事件加入pcm调用请求队列
State15 -> State20

State20 : pcmcall事件被消费，调用Pcm查询接口
State20 -> State30
State20 -> State40

State40 : pcm查询失败
State40 -> State15
State40 --> [*]

State30 : pcm查询成功
State30 -> State50

State50 : 接收到pcm回调
State50 -> State55


State55 : 回调请求加入回调队列
State55 -> State60
State55-> State70

State60 : 回调业务方成功
State60 --> [*]

State70 : 回调业务方失败
State70 -> State55
State70 --> [*]
@enduml
```

# 实体类图

``` plantuml!
@startuml
class BaseEvent {
  long Id
  String applyno
  LocalDateTime createTime
  LocalDateTime updateTime
  Status Status
  String callbackUrl
  String trackingId
  int priority
}

class PcmCallEvent {
  int callPcmRetry
  String busiReqBody
  String pcmCallRequestXml
  String pcmCallResponseXml
  LocalDateTime pcmCallTime
  BusiPcmCallInfo pcmCallInfo (非持久化属性)
}

class PcmCallBackEvent {
  int callbackBusiRetry
  String pcmCallbackRequestXml
  String pcmCallbackResponseXml
  String busiCallbackBody
  LocalDateTime pcmCallbackTime
  BusiCallBackInfo busiCallBackInfo (非持久化属性)
}

class BusiCallBackInfo{
  String application_number
  String[] roles 
}

class BusiPcmCallInfo{
  String application_number
  String application_date
  String request_date
  String capoperator
  String operator
  String queryreason
  PeopleInfo[] infos 
}

class PeopleInfo{
  String applicant_nme
  String id_card_typ
  String id_card_nbr
  String application_role
  String applicant_sex
  String date_of_birth
}

BusiPcmCallInfo "1" o-- "many" PeopleInfo
PcmCallEvent <|-- BaseEvent
PcmCallEvent "1" *-- "1" BusiPcmCallInfo

PcmCallBackEvent <|-- BaseEvent
PcmCallBackEvent "1" *-- "1" BusiCallBackInfo

enum Status {
  State10
  State15
  State20
  State30
  State40
  State50
  State55
  State60
  State70
}
@enduml
```

# 表结构

| table       		 	| field      | type     | null | default |
| ------------ | ---------- | -------- | ---- | ------- |
| pcm_call_log | id         		| BIGINT   | no   |         |
| 			   |	 applyno         		| char   | no   |  ''       |
|              | createTime 		  | timestamp | no   |    0     |
|              | updateTime 		 | timestamp | no |    0     |
|              |  status|   int  		 |  no  |  0     |
|              |  callbackUrl  	   |  char(255)        |  no    |  ''      |
|              |    trackingId        | char(255)         | no     |  ''       |
|              |  priority         |  int        |    no  |    50     |
|              |    busiReqBody        |   VARCHAR       |   no   |  ''       |
|              |   pcmCallRequestXml         |    VARCHAR      |   no    |   ''      |
|              |   pcmCallResponseXml         |    VARCHAR      |   no    |   ''      |
|              |   pcmCallTime         |    timestamp      |   no    |   0      |
|              |    callbackBusiRetry        |  int        |    no  |   0      |
|              |    pcmCallbackRequestXml        |  VARCHAR        |    no  |   ''      |
|              |    pcmCallbackResponseXml        |  VARCHAR        |    no  |   ''      |
|              |    pcmCallbackTime        |  timestamp        |    no  |   0      |
|              |    busiCallbackBody        |  VARCHAR        |    no  |   ''      |


# 接口参数
* pcmCall接口：
	
	* url: /pcmCall
	* method: POST
	* 请求头/请求体/响应体如下：
~~~?title=request header
	Content-Type: application/json
~~~
```json?title=request body
{
    "tracking_id": "123123123123",
    "callback_url": "http://10.50.1.1/xxx",
	"priority" : 50,
	"pcmInfo":{
    "application_number": "GW-A401735000",
    "application_date": "2018-03-24 14:06:57",
    "request_date": "2018-09-17 19:15:57",
    "capoperator": "0000447K",
    "operator": "xdsp",
    "queryreason": "02",
    "infos": [
       {
        "applicant_nme": "赵六六",
        "id_card_typ": "0",
        "id_card_nbr": "123123123123123123",
        "application_role": "A",
        "applicant_sex": "M",
        "date_of_birth": "1982-06-28"
      }
    ]
  }
}
``` 
```json?title=response body
{
    "errorCode" :  0,
	"errorMsg" : "xxxx",
	"rawErrorMsg" : "ccccc",
	"data": {
		"eventId": "123123123"
	}
	
}
```

* pcmCallback接口：
	
	* url:  callback_url的值
	* method: POST
	* 请求头/请求体/响应体如下：
~~~?title=request header
	Content-Type: application/json
~~~
```json?title=request body(发送的回调报文)
{
    "eventId": "123123123",
    "tracking_id": "123123123123",
	"pcmResult":{
    "application_number": "GW-A401735000",
    "application_role": [
		"A", "B", "C"
	],
    
  }
}
``` 
```json?title=response body
{
    "errorCode" :  0,
	"errorMsg" : "xxxx",
	"rawErrorMsg" : "ccccc",	
}
```

