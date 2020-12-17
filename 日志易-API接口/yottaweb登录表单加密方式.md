![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812194156.png)

## 引言

> 在讲这边文章前先提出这4个问题，明明我们有API接口，为什么还要搞这玩意

1. 为什么要整明白加密方式
   虽然在3.0 3.2期间已经提供了很多API供使用，但难免还是有部分接口未对外开放，需要使用的时候却用不了，比如：下载报表，获取heka采集配置，获取仪表盘spl等，但是这些yottaweb自身就能做到，不偷来用岂不是浪费了
2. 知道了表单加密有什么用
   了解了加密后，就相当去你获取到了一个密钥，你可以随意使用yottaweb的任何接口，做各种操作，本质上和爬虫没区别，就是这么为所欲为
3. 表单加密的原理是什么
   见下文
4. 如何用还原表单加密
   见下文

## yottaweb表单加密逻辑

### 登录接口

一个登陆操作无外乎就是一个表单提交的过程，在这个过程中将获取到的账号密码等信息通过接口或者其他方式推送到服务端做校验，在校验完成后，用户浏览器和服务端都会生成一份cookie数据，用于下次用户在登录平台做校验，免去重复登录

而我们要利用yottaweb的接口，恰好就需要获取到这个cookie数据，而yottaweb本身有session超时的配置，会导致一段时间后cookie失效，从而要求重新登录，而作为爬虫，显然是要自动化，而不是失效后再去找

这个时候我们就可以利用模拟登录的方式，直接生成一个新的cookie，那么就可以愉快的玩耍了

对于web前端来说，登录的操作通常采用的方式就AJAX请求，向服务端发出请求，这个时候就可以借助我们Chrome大法做调试了

在登录窗口F12打开浏览器控制台，进入【network】-->【XHR】，然后执行下登录操作，看看前端发了什么请求

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812205959.png)
从请求URL来看，登录的动作也是有接口可用，同时body中带了 `data:xxxxx`的数据，很明显这就是我们的用户名密码等信息加密后的数据了

从接口返回结果我们可以看到这些信息
![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812211229.png)
```json
{
    "location":"/search/",
    "password_notify_time":"7d",
    "password_update_time":1592292584000,
    "status":"1",
    "user_id":1
}
```

### 找出加密的代码

还是那个登录请求，打开`initiator`，看下发出这个请求都涉及到哪些文件
![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812211844.png)
在这地方看起来有俩个文件，一个是`passwordReset`，一个是`index`的，看起来前者不像，我们先去查`index.xxx.chuk.js`，鼠标挪到这个文件上可以看到这个js的路径是`static/dist/login`目录下
知道路径后我们去到【source】打开这个文件，`{}`可以格式化js文件
![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812212718.png)
然后搜索下`/auth/login`，同时要是`post`请求的
![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812212905.png)
到这里我们看到了提交请求的函数，data参数值是一个传参，这里我们打个断点看下，然后点击登录
![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812213059.png)
看到这个，很明显找对了，但还是无法看到是怎么生成的，从结果来看是有函数将生成的结果传入进来，那我们用chrome的`step out of curret function`看下是谁生成的
![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812213231.png)

------

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812213439.png)
到了这，那就一目了然了，这货的步骤是
1. 获取 用户账号 密码 otacode imagecode
   ```js
                       var c = {
                           username: o,
                           password: Object(C.a)(n, l, r),
                           otacode: s,
                           imgcode: _
                       }
   ```
2. 密码这块调用了一个函数，传入三个参数进行了一次加密
3. 最后调用`convertBase64(JSON.stringify(c))`生成加密后的数据

------

> 知道逻辑后我们一个一个拆解

1. 首先我们可以看到原始表单数据是酱紫的
   ```json
   {username: "admin", password: "admin@rizhiyi.com", otacode: "", imgcode: ""}
   ```
2. 然后密码这块调用一个函数，传入三个参数
   ```
   n = admin@rizhiyi.com
   l = 1qaz0plm2wsxhgyi
   r = w7sdncj09olxnbxs
   ```
3. 要看这个函数是什么，我们在这个地方打个断点，使用`setp into next function`，看下是什么函数
    ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812215315.png)
    ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812215411.png)
    这个就是对`n l r`参数加密的地方了，有些熟悉，是个aes对称加密，加密文本是我们的密码`n`，加密模式为`CBC`，偏移量IV为`r`，加密密码为`l`，最后`toString`转换成字符串
4. 最后我们在`convertBase64(JSON.stringify(c))`打个断点看下这个函数是干嘛的
    ![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20200812220634.png)
    到这我们就可以看到，这个函数先将我们的json表单数据做了一次`base64`编码
        ```json
        {"username":"admin","password":"BeKjgMBd21Vc90mBNtLFyu2c4RkEWnE0UV53q0RIioE=","otacode":"","imgcode":""}
        ```
    然后对编码后的字符串进行切割拼接，最后生成body中data参数值

## python还原表单加密

从上面的断点调试中我么你知道了data参数值的生成逻辑，那么我们就用python还原下前端js的加密过程

```python
# -*- coding: utf-8 -*-

import requests
import json
import logging
import sys, os
import uuid
import base64
import pyaes

reload(sys)
sys.setdefaultencoding('utf8')

fmt = logging.Formatter(fmt="%(asctime)s %(levelname)s %(message)s", datefmt='%Y-%m-%d %H:%M:%S')
sh = logging.StreamHandler()  # 往屏幕上输出
sh.setFormatter(fmt)  # 设置屏幕上显示的格式
logger1 = logging.Logger(name='alert', level=logging.DEBUG)
logger1.addHandler(sh)


YottaWeb = "192.168.1.188"
YottaWebUserName = "admin"
YottaWebPasswd = "xxxxxxx"
SECRET_IV = 'w7sdncj09olxnbxs'
SECRET_KEY = '1qaz0plm2wsxhgyi'

def get_cookie(version):
    """
    返回cookies，每次请求需要带上cookies
    :return cookies:
    """
    global user_info
    url = "http://{}/api/v0/auth/login".format(YottaWeb)

    if version == "3.0":
        # 3.0 yottaweb没对密码做aes加密，直接就base64编码后截取字符拼接
        user_info = """{"username":"%s","password":"%s","otacode":"","imgcode":""}""" % (YottaWebUserName, YottaWebPasswd)
    elif version == "3.2" or version == "3.1":
        # 由于3.2表单做了aes对称加密，所以要换用新的方式做加密，这里用到了pyaes库做下简单的加密
        encrypter = pyaes.Encrypter(pyaes.AESModeOfOperationCBC(SECRET_KEY, SECRET_IV))
        passwd = encrypter.feed(YottaWebPasswd)
        passwd = encrypter.feed()
        user_info = """{"username":"%s","password":"%s","otacode":"","imgcode":""}""" % (YottaWebUserName, base64.b64encode(passwd))
    base_data = base64.b64encode(user_info)
    req_data = "data=" + base_data[-3:] + base_data[2:len(base_data) - 3] + base_data[0:2]

    payload = '{}'.format(req_data)
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded'
    }
    try:
        response = requests.request("POST", url, headers=headers, data=payload)
        return "sessionid=" + response.cookies["sessionid"]
    except Exception as e:
        logger1.exception("{} get_cookie 获取cookie失败，抛出如下错误:{}".format(UUID, e))
        
 
if __name__ == '__main__':
    get_cookie("3.1")

```

