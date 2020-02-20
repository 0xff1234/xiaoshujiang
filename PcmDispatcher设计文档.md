---
title: PcmDispatcher设计文档 
tags: 软件,设计
renderNumberedHeading: true
grammar_cjkRuby: true
grammar_sequence: true
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

# PcmDispatcher逻辑时序图
1.
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


# 表结构
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
    "callback_url": "http://10.50.1.1/xxx",
	"pcmInfo":{
    "application_number": "GW-A401735000",
    "application_date": "2018-03-24 14:06:57",
    "request_date": "2018-09-17 19:15:57",
    "capoperator": "0000447K",
    "operator": "xdsp",
    "queryreason": "02",
    "infos": {
      "info": {
        "applicant_nme": "赵六六",
        "id_card_typ": "0",
        "id_card_nbr": "123123123123123123",
        "application_role": "A",
        "applicant_sex": "M",
        "date_of_birth": "1982-06-28"
      }
    }
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
