## 初识API

### 组成部分

一个完整的API请求包含了以下几部分

- IP+端口

- URL

  跟在IP屁股后面的那串，通常是API网关通常是根据这个URL来判断请求那个接口的

- method

  请求方式，常见的就是GET或者POST

- params(query)/body

  请求体，这个地方就是具体告知API网关，我请求这个接口具体要干嘛，一般来讲，GET是用params传参，而POST是用body传参

- header

  一般API请求涉及到验证信息都会在header头里面

### 如何看懂API文档

大概知道了一个API请求都包含了哪些部分后，我们来看下如何从API手册中获取到这些信息，下面就以咱们日志易API手册中的【7.1.1】非流式搜索来介绍下

#### 基本信息

1. 鉴权

   ```
   ip: 192.168.1.188
   port: 8090
   username: admin
   password: admin@rizhiyi.com
   ```
   
2. 接口信息

   Method+URL

   ```
   GET /search/sheets/
   ```

   参数

   | 类型             | 名称          | 说明                                      |
   | ---------------- | ------------- | ----------------------------------------- |
   | Query Parameters | queryfilters  | 搜索语句的过滤语句                        |
   | Query Parameters | filter_field  | 过滤字段                                  |
   | Query Parameters | query         | 搜索的语句                                |
   | Query Parameters | time_range    | 搜索时间范围                              |
   | Query Parameters | page          | 请求分页的页数,默认为0                    |
   | Query Parameters | size          | 分页每页返回数量，默认值为20              |
   | Request Headers  | Content-Type  | 数据传输格式, 支持  -- 数据传输格式, 支持 |
   | Request Headers  | Authorization | 用户基本鉴权信息, 包含 username和password |

   响应

   | HTTP代码 | 说明                                             | 类型 |
   | -------- | ------------------------------------------------ | ---- |
   | 200      | 非流式搜索结果,同步发送http请求,等待返回对应结果 |      |

#### 拆解说明

1. 鉴权

   通常相关鉴权信息是放在Request Headers中的Authorization中，其内容格式为：username:password 组合后进行base64编码

2. 接口信息

   从**Method+URL**可以看出，这个接口使用了GET请求，URL为`/search/sheets/`，这个在参数中也有介绍

   在参数的**类型中**可以看到有Query Parameters和Request Headers，那么这俩种分别是什么意思呢

   - Request Headers

     为请求头，服务端根据请求头进行处理并返回数据，通常包含传输过去的内容为什么类型以及包含鉴权信息

   - Query
   
     则是具体告知API网关这个请求到底是干嘛的，在客户提供的API手册中，Query也可以被称为Params，俩者是同一个意思，填写未知都是在URL后面的，格式为
     
     ```
     ip:8090/api/v2/search/sheets/?xxx=&xxx=
     ```
   
   **BTW：** 除了有Query类型，还有一个是body，俩者的主要区别在于，前者常用语GET请求中，而后者是用在POST请求中，Body通常为json格式

### 如何调试API接口

写代码都需要一个好的IDE，更何况API调试呢，在这里就统一介绍下大名鼎鼎的postman，这是下载入口：[点我](https://www.postman.com/downloads/)

下面就来介绍下它的界面

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200817201627.png)

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200817201647.png)

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200817201708.png)

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200817201720.png)

当所有信息都填好后就可以点击【send】发送信息了，从底部的status【xxx】里面可以看到是否成功，一般200表明成功了，同时会返回接口数据

最后在这块地方可以导出我们调试好的接口信息，提供了各种语言方式

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200817201744.png)

## 写告警插件

#### 结构体

首先呢，前端告警插件是有一套模板的，按照这个走啥问题都没有，不需要动脑，数值这几项结构就大概知道怎么写了，结构体可分为以下几大块

- re_logger常量

  用以定义生成的日志文件名，抄

  ```python
  re_logger = logging.getLogger("LuZhengFuturesSMS.log")
  ```

- ONLINE_CONTENT：常量

  这个是消息内容模板，也就是这一大块玩意，可留空也可不留空，django语法，一般调试好语句写到插件里面就ok了，可参考(自动渲染spl统计结果)

  ```django
  [日志平台]{%if alert.is_alert_recovery %}[告警恢复]{% endif %}{{alert.name}}
  触发时间: {{ alert.send_time|date:"Y-n-d:H:i:s" }}
  时间范围: {{ alert.strategy.trigger.start_time|date:"Y-n-d:H:i:s" }}到{{ alert.strategy.trigger.end_time|date:"Y-n-d:H:i:s" }}
  {% for result_row in alert.result.hits %}{% for k in alert.result.columns %}{% for rk, rv in result_row.items %}{% if rk == k.name %}{{k.name}}:{{rv}}{% if forloop.last %}{% else %};{% endif %}{% endif %}{% endfor %}{% endfor %}{% endfor %}
  ```

- META常量字典

  用于定义告警插件需要在前端张啥样

  这里就拿个栗子来讲把，具体的描述就参考监控告警手册【5.2.1】
  
  ```json
  META = {
      # 插件名称
      "name": "ServerChan微信告警推送",
      # 版本号
      "version": 1,
      "alias": "ServerChan微信告警推送",
      # 这就是前端展示的配置信息
      "configs": [
          {
              "name": "token",
    					# 展示这个输入框的说明
              "alias": "ServerChan的token http://sc.ftqq.com/",
    					# 下面这些默认填写
              "presence": True,
              "value_type": "string",
              "input_candidate": "",
    					# 输入框默认展示值
              "default_value": "",
    					# 默认就好
              "style": {
                  "rows": 1,
                  "cols": 15
              }
          },
          {
              "name": "content",
              "alias": "消息内容模板",
              "presence": True,
              "value_type": "string",
              "default_value": "{}".format(ONLINE_CONTENT),
              "style": {
                  "rows": 2,
                  "cols": 30
              }
          },
      ]
}
  ```

  这里要强调下，config数组下面一个json串就是一个框框显示，同时我们还有个`input_type`输入方式类型，这个key值有多个

  - email: 含义是用户信息中的电子邮箱，其输入交互会带有下拉选项提示，提示内容分为用户

    分组和用户。

  - phone: 含义是用户信息中的电话号码，其输入交互会带有下拉选项提示，提示内容分为用户

    分组和用户。

  - drop_down：下拉菜单，只能单选，取值是拿的`input_candidate`里面的数据

  - drop_down_multiple：下拉菜单，可多，取值是拿的`input_candidate`里面的数据

  > 除了这些好像还有什么email_group，account_group，带group会展示每个用户分组下面的信息，涉及分组需要自行处理分组获取相关信息

  最终就是展示成这样的

- _render函数

  点击预览的时候就会用到它，不用理会，研发都帮你写好了，照抄

  ```python
  def _render(conf_obj, tmpl_str):
      c = Context(conf_obj)
      t = Template(tmpl_str)
      _content = t.render(c)
      return _content
  ```

- content函数

  用来渲染消息内容模板里面的信息，照抄

  需要注意的是，由于我们在meta字典中可能加入了多行输入项，往往消息内容模板是最后一个值，所以template_str获取数组值需要修改下，既`params.get('configs')[x].get('value')`

  ```python
  template_str = params.get('configs')[3].get('value')
  ```

  参考

  ```python
  def content(template_str, alert):
      conf_obj = {'alert': alert}
      template_str = params.get('configs')[3].get('value')
      _content = _render(conf_obj, template_str)
      return _content
  ```

- handle函数：

  这里就是告警插件执行动作了，相当于一个主函数，yottaweb调用这个函数执行发送告警

  这个函数里面会有俩个参数，params和alert

  params就是meta字典

  alert就是产生的告警信息，`alert['name']`可以获取到告警名称，具体里面会是啥我也不太清楚，你们自行探索把

  参考

  ```python
  def handle(params, alert):
      try:
          # 从前端告警方式输入框获取内容
          phones = params.get('configs')[0].get('value')
          msg = params.get('configs')[1].get('value')
          # 调用具体推送告警函数
          xxx(xx)
      except Exception as e:
          log_and_reply(logging.ERROR, "告警:{}, 发送异常:{}".format(alert['name'], str(e)))
  ```

  

- set_logger函数

  全局定义写日志，照抄

  ```python
  def set_logger(reset_logger):
      global logger
      logger = reset_logger
  ```

- log_and_reply函数

  即在日志文件输出，也在点击预览时输出内容，是一个日志格式定义，照抄

  ```python
  def log_and_reply(log_level, comment):
      global reply_content
      reply_content = ""
      log_content = {
          logging.FATAL: re_logger.fatal,
          logging.ERROR: re_logger.error,
          logging.WARNING: re_logger.warning,
          logging.INFO: re_logger.info,
          logging.DEBUG: re_logger.debug
      }
      log_content.get(log_level)(comment)
      reply_content = '%s%s%s' % (reply_content, "\n", comment)
  ```

- execute_reply函数

  点击【预览】按钮调用的函数，照抄

  ```python
  def execute_reply(params, alert):
      re_logger.info("reply_content start")
      handle(params, alert)
      re_logger.info("reply_content: %s" % reply_content)
      return reply_content
  ```

- xxx函数

  这个就是你执行发送告警的具体动作，建议单独一个函数，不要写在handle函数里面

#### 源码示例

```python
# -*- coding: utf-8 -*-
"""
-------------------------------------------------
   File Name   :     LuZhengFuturesSMS.py
   Description :     鲁证期货yottaweb短信告警
   Company     :     AlphaBrock
   Author      :     jcciam@outlook.com
   Date        :     2022/12/6 14:44
-------------------------------------------------
"""
import json
import logging
import hashlib
import requests
import uuid
from django.template import Context, Template

# 短信网关的IP或者域名
SMSUrl = ""
# 短信接口调用账户
SMSAccount = ""
# 短信接口调用账户密码
SMSPasswd = ""
# 短信签名
SMSSign = ""
# HTTP前置代理
proxies = {
   'http': 'http://proxy.example.com:8080',
   'https': 'http://proxy.example.com:8090',
}

re_logger = logging.getLogger("LuZhengFuturesSMS.log")

ONLINE_CONTENT = '''[日志平台]{%if alert.is_alert_recovery %}[告警恢复]{% endif %}{{alert.name}}
触发时间: {{ alert.send_time|date:"Y-n-d:H:i:s" }}
时间范围: {{ alert.strategy.trigger.start_time|date:"Y-n-d:H:i:s" }}到{{ alert.strategy.trigger.end_time|date:"Y-n-d:H:i:s" }}
{% for result_row in alert.result.hits %}{% for k in alert.result.columns %}{% for rk, rv in result_row.items %}{% if rk == k.name %}{{k.name}}:{{rv}}{% if forloop.last %}{% else %};{% endif %}{% endif %}{% endfor %}{% endfor %}{% endfor %}'''

META = {
    "name": "LuZhengFuturesSMS",
    "version": 2,
    "alias": "鲁证期货短信推送",
    "configs": [
        {
            "name": "receiver",
            "alias": "接收者",
            "placeholder": "手机号, 以英文逗号分割, 如:123,456",
            "presence": True,
            "value_type": "string",
            "input_type": "phone",
            "default_value": "",
            "style": {
                "rows": 1,
                "cols": 20
            }
        },
        {
            "name": "content",
            "alias": u"消息内容模板",
            "presence": True,
            "value_type": "string",
            "default_value": "{}".format(ONLINE_CONTENT),
            "style": {
                "rows": 10,
                "cols": 40
            }
        }
    ]
}


def _render(conf_obj, tmpl_str):
    c = Context(conf_obj)
    t = Template(tmpl_str)
    _content = t.render(c)
    return _content


def content(template_str, alert):
    conf_obj = {'alert': alert}
    _content = _render(conf_obj, template_str)
    return _content


def handle(params, alert):
    try:
        phones = params.get('configs')[0].get('value')
        msg = params.get('configs')[1].get('value')
        sendText = content(msg, alert)
        logger.info("告警:{}, 接收人:{}, 发送内容:{}".format(alert['name'], phones, sendText))
        if phones == "":
            log_and_reply(logging.ERROR, "告警:{}, 手机号不能为空".format(alert['name']))
            return
        if len(phones) < 11:
            log_and_reply(logging.ERROR, "告警:{}, 手机号长度小于11位".format(alert['name']))
            return

        if len(phones) > 11 and "," not in phones:
            log_and_reply(logging.ERROR, "告警:{}, 手机号格式输入有误, 以英文逗号分割, 如:123,456".format(alert['name']))
            return

        url = "http://{}/json/sms/Submit".format(SMSUrl)
        headers = {
            "Content-Type": "application/json"
        }
        body = {
            "account": SMSAccount,
            "password": hashlib.md5(SMSPasswd.encode(encoding='UTF-8')).hexdigest(),
            "msgid": str(uuid.uuid1()).replace("-", ""),
            "phones": phones,
            "content": sendText,
            "sign": SMSSign,
            "subcode": "",
            "sendtime": ""
        }
        logger.debug("告警:{}, body:{}".format(alert['name'], json.dumps(body, ensure_ascii=False)))
        response = requests.post(url, json=body, headers=headers, verify=False, timeout=30)
        log_and_reply(logging.INFO, "告警:{}, 发送结果:{}".format(alert['name'], response.text))
    except Exception as e:
        log_and_reply(logging.ERROR, "告警:{}, 发送异常:{}".format(alert['name'], str(e)))


def set_logger(reset_logger):
    global logger
    logger = reset_logger


def log_and_reply(log_level, comment):
    global reply_content
    reply_content = ""
    log_content = {
        logging.FATAL: re_logger.fatal,
        logging.ERROR: re_logger.error,
        logging.WARNING: re_logger.warning,
        logging.INFO: re_logger.info,
        logging.DEBUG: re_logger.debug
    }
    log_content.get(log_level)(comment)
    reply_content = '%s%s%s' % (reply_content, "\n", comment)


def execute_reply(params, alert):
    re_logger.info("reply_content start")
    handle(params, alert)
    re_logger.info("reply_content: %s" % reply_content)
    return reply_content
```





