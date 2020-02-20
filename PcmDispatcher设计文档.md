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
1. 为原始Pcm系统的调用和回调接口封装一个代理层服务——PcmDispatcher，用于多个调用方及其回调地址，内部采用pcm_call_log来记录每个回调的地址，并基于单号和回调时间进行匹配，匹配后发送回调

![PcmDispatcher架构图](./attachments/1582188810461.drawio.html)

# PcmDispatcher逻辑时序图
1. dfdfd
打发打发

```sequence
小明->小李: 你好 小李, 最近怎么样?
Note right of 小李: 小李想了想
小李-->小明: 还是老样子
```


# 表结构
# 接口参数
