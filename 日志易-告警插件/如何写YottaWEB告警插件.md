![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200817202135.svg)

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
   port: 8080
   token: 8d295552b9a0a1979f2804887e066396
   username: admin
   password: admin@rizhiyi.com
   operator: admin
   ```

2. 接口信息

   Method+URL

   ```
   GET /v1/{token}/{operator}/spl/search
   ```

   参数

   | 类型  | 名称           | 说明                           | 类型 | 默认值 |
   | ----- | -------------- | ------------------------------ | ---- | ------ |
   | Path  | token必填      | token                          |      |        |
   | Path  | operator必填   | 操作者                         |      |        |
   | Query | task_name必填  | 任务名称                       |      |        |
   | Query | category可选   | 自定义的任务类别，默认为search |      |        |
   | Query | time_range必填 | 时间范围xxxxxxx                |      |        |
   | Query | query必填      | 查询语句                       |      |        |
   | Query | page可选       | 从0开始的结果页码              |      |        |
   | Query | size可选       | size数量                       |      |        |

   响应

   | HTTP代码 | 说明                                             | 类型 |
   | -------- | ------------------------------------------------ | ---- |
   | 200      | 非流式搜索结果,同步发送http请求,等待返回对应结果 |      |

#### 拆解说明

1. 鉴权

   token+ username+password+operator可以认为是鉴权信息

2. 接口信息

   从**Method+URL**可以看出，这个接口使用了GET请求，URL为`/v1/{token}/{operator}/spl/search`，其中`token` `operator`为鉴权信息，这个在参数中也有介绍

   在参数的**类型中**可以看到有Path和Query，那么这俩种分别是什么意思呢

   - Path则表示该参数是属于URL里面的

   - Query则是具体告知API网关这个请求到底是干嘛的，在客户提供的API手册中，Query也可以被称为Params，俩者是同一个意思，填写未知都是在URL后面的，格式为

     ```
     ip:port/v1/{token}/{operator}/spl/search/?xxx=&xxx=
     ```

   **BTW：** 除了有Query类型，还有一个是body，俩者的主要区别在于，前者常用语GET请求中，而后者是用在POST请求中

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

### Yottaweb端的插件

#### 结构体

首先呢，前端告警插件是有一套模板的，按照这个走啥问题都没有，不需要动脑，数值这几项结构就大概知道怎么写了，结构体可分为以下几大块

- ONLINE_CONTENT：常量，这个是消息内容模板，也就是这一大块玩意，可留空也可不留空，django语法，一般调试好语句写到插件里面就ok了

  

- META：常量字典，用于定义告警插件需要在前端张啥样

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

  > 除了这些好像还有什么email_group，account_group，带group会展示每个用户分组下面的信息

  最终就是展示成这样的

  

- _render函数：点击预览的时候就会用到它，不用理会，研发都帮你写好了，照抄

- content函数：用来渲染消息内容模板里面的信息，照抄

  需要注意的是，由于我们在meta字典中可能加入了多行输入项，往往消息内容模板是最后一个值，所以template_str获取数组值需要修改下，既`params.get('configs')[x].get('value')`

  ```python
  template_str = params.get('configs')[3].get('value')
  ```

- handle函数：这里就是告警插件执行动作了，相当于一个主函数，yottaweb调用这个函数执行发送告警

  这个函数里面会有俩个参数，params和alert

  params就是meta字典

  alert就是产生的告警信息，`alert['name']`可以获取到告警名称，具体里面会是啥我也不太清楚，你们自行探索把

- xxx函数：这个就是你执行发送告警的具体动作，建议单独一个函数，不要写在handle函数里面

#### 源码示例

```python
# -*- coding: utf-8 -*-
# description: ServerChan微信告警推送
__author__ = 'chenfei'

import json
import logging.config
import sys
import uuid

import requests
from django.template import Context, Template

reload(sys)
sys.setdefaultencoding("utf-8")

UUID = uuid.uuid1()

# 全局定义好怎么写日志
filelog = logging.FileHandler(filename='/data/rizhiyi/logs/yottaweb/alert.log', mode='a', encoding='utf-8')
fmt = logging.Formatter(fmt="[%(asctime)s] [%(levelname)s] %(message)s",
                        datefmt='%Y-%m-%d %H:%M:%S')
filelog.setFormatter(fmt)
logger1 = logging.Logger(name='alert', level=logging.DEBUG)
logger1.addHandler(filelog)

# 模板，这个就是后续定义告警内容的地方，django语法
ONLINE_CONTENT = '''`这是一条测试数据`'''
# 字典，插件在配置那里怎么显示就是获取的这里的内容
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
            "alias": "ServerChan的token http://sc.ftqq.com/",
            "presence": True,
            "value_type": "string",
            "input_candidate": "",
            "default_value": "",
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


# 预览渲染(点击预览会用到这个函数)
def _render(conf_obj, tmpl_str):
    c = Context(conf_obj)
    t = Template(tmpl_str)
    _content = t.render(c)
    return _content


# 将告警信息渲染好后供后续做数据处理
def content(params, alert):
    template_str = params.get('configs')[1].get('value')
    conf_obj = {'alert': alert}
    _content = _render(conf_obj, template_str)
    return _content


# 微信推送的主函数
def pushWeChat(summary, token, contents):
    url = "https://sc.ftqq.com/{}.send".format(token)
    params = {
        "text": "{}".format(summary),
        "desp": "{}".format(contents)
    }
    # 提交post请求
    response = requests.request("POST", url, verify=False, params=params)
    # 判断提交请求状态，以及是否提交成功
    if response.status_code == 200 and json.loads(response.text).get('errmsg') == "success":
        logger1.info("[%s] alert_name:\"%s\" sent successfully, the params is:\"%s\", the response is:\"%s\"" % (
            UUID, summary, contents, json.dumps(response.text).encode('utf-8')))
    else:
        logger1.error("[%s] alert_name:\"%s\" sent successfully, the params is:\"%s\", the response is:\"%s\"" % (
            UUID, summary, contents, json.dumps(response.text).encode('utf-8')))


# 整个告警插件的执行主函数
def handle(params, alert):
    try:
        # 告警名称
        summary = alert['name']
        # 从前端配置里面拿到对应值，这个就是上面讲的meta字典
        token = params.get('configs')[0].get('value')
        # 获取消息模板
        contents = content(params, alert)
        if token == "":
            logger1.warning("[%s] alert_name:\"%s\", token cannot be empty" % (UUID, alert['name']))
        else:
            # 执行微信推送
            pushWeChat(summary, token, contents)
    except Exception as e:
        logger1.error("[%s] alert_name:\"%s\",got exception:\"%s\"" % (UUID, alert['name'], e))
        raise
```

### Manager端的插件

> 下期继续，先占位

#### 结构体

#### 源码示例

```python
# -*- coding: utf-8 -*-
# manager端的ServerChan方糖微信推送
__author__ = 'chen.fei'

import datetime
import json
import logging.config
import sys
import uuid
from datetime import datetime

import requests
from common.plugin_util import convert_config

UUID = uuid.uuid1()

reload(sys)
sys.setdefaultencoding("utf-8")

filelog = logging.FileHandler(filename='/data/rizhiyi/logs/yottaweb/manager_alert.log', mode='a', encoding='utf-8')
fmt = logging.Formatter(fmt="[%(asctime)s] [%(levelname)s] [rizhiyi_manager]  %(message)s", datefmt='%Y-%m-%d %H:%M:%S')
filelog.setFormatter(fmt)
logger1 = logging.Logger(name='alert', level=logging.DEBUG)
logger1.addHandler(filelog)


META = {
    "name": "ServerChan微信推送",
    "version": 1,
    "alias": u"微信推送",
    "configs": [
        {
            "name": "token",
            "alias": u"ServerChan的Token_03",
            "value_type": "string",
            "default_value": "",
            "style": {
                "rows": 1,
                "columns": 15
            }
        }
    ],
    "param_configs": [
        {
            "name": "token",
            "alias": u"ServerChan的Token_04",
            "value_type": "string",
            "default_value": "",
            "style": {
                "rows": 1,
                "columns": 15
            }
        }
    ]
}


# 梳理出完整的一条告警信息
def gen_content(alert):
    try:
        module = alert.get("Module", "")
        type = alert.get("Type", "")
        ip = alert.get("Ip", "")
        alarm_time = alert.get("Time", "")
        if alarm_time == "":
            alarm_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        detail = alert.get("Detail", "")
        recovery = alert.get("Recover", "")
        if recovery == "True" or recovery == True:
            recovery = u"恢复"
        else:
            recovery = u""

        if module != "" and ip != "":
            content = u"告警类型:%s,告警模块:%s,告警IP:%s,告警时间:%s,告警详情:%s" % (type, module, ip, alarm_time, detail)
        elif ip != "":
            content = u"告警类型:%s,告警IP:%s,告警时间:%s,告警详情:%s" % (type, ip, alarm_time, detail)
        else:
            content = u"告警类型:%s,告警时间:%s,告警详情:%s" % (type, alarm_time, detail)
        return content
    except Exception as e:
        logger1.exception("Fail to gen syslog content :%s" % (str(e)))
        return ""


# 微信推送主函数
def pushWeChat(token, alert_content):
    url = "https://sc.ftqq.com/{}.send".format(token)
    params = {
        "text": "小朋友，服务挂了",
        "desp": "{}".format(alert_content)
    }
    response = requests.request("POST", url, verify=False, params=params)
    if response.status_code == 200 and json.loads(response.text).get('errmsg') == "success":
        logger1.info("[%s] alert sent successfully, the params is:\"%s\", the response is:\"%s\"" % (
            UUID, alert_content, json.dumps(response.text).encode('utf-8')))
    else:
        logger1.error("[%s] alert sent successfully, the params is:\"%s\", the response is:\"%s\"" % (
            UUID, alert_content, json.dumps(response.text).encode('utf-8')))


def handle(meta, alert):
    # 从meta字典中拿到param_configs信息
    param_configs = meta.get("param_configs")
    if param_configs is None:
        logger1.error("{} No param_configs in meta param".format(UUID))
        return False

    param_configs = convert_config(param_configs)
    token = param_configs.get("token")
    if token == "":
        logger1.warning("[{}] token connot be empty!".format(UUID))
        return True
    # 拿到告警详情
    alert_content = gen_content(alert)
    if alert_content == "":
        logger1.error("[{}] Fail to gen alert content".format(UUID))
        return False
    # 执行推送
    pushWeChat(token, alert_content)
    return True


# 主函数
if __name__ == '__main__':
    meta = {}
    alarm = {}
    handle(meta, alarm)
```