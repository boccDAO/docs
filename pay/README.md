- ccpay 测试服务器地址为 **http://bcapay.com/api**.

# 获取 ETH 收款地址

获取 ETH 收款地址, 打入该地址的任何 ETH 都会立即计入你的余额.

**python 代码**

```py
import base64
import datetime
import hashlib
import hmac

import requests

c_server = "http://bcapay.com:8000"
c_name = 'bcatest' # 替换为你的账户名
c_secret = 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' # 替换为你的 secret


def b64(s):
    return base64.b64encode(s.encode()).decode()


def httpdate_rfc1123(dt=None):
    dt = dt or datetime.datetime.utcnow()
    return datetime.datetime.utcnow().strftime('%a, %d %b %Y %H:%M:%S GMT')


def sign(username, password, method, uri, date, content_md5=None):
    signarr = [method, uri, date]
    if content_md5:
        signarr.append(content_md5)
    signstr = '&'.join(signarr)
    signstr = base64.b64encode(
        hmac.new(password.encode(), signstr.encode(), digestmod=hashlib.sha1).digest()
    ).decode()
    return 'BOCC %s:%s' % (username, signstr)


class Request:
    def __init__(self, method, uri):
        self.method = method
        self.uri = uri
        self.params = {}
        self.headers = {}
        self.json = {}

    def sign(self):
        date = httpdate_rfc1123()
        signstr = sign(c_name, c_secret, self.method, self.uri, date)
        self.headers['Date'] = date
        self.headers['Authorization'] = signstr

    def request(self):
        return requests.request(
            self.method,
            c_server + self.uri,
            params=self.params or None,
            headers=self.headers or None,
            json=self.json or None,
        )


# 获取 ETH 收款地址
req = Request('GET', '/v1/address')
req.params['type'] = 'ETH'
req.params['uid'] = 1 # 使用 uid 区分你的客户
req.sign()
resp = req.request()
print(resp.json())

# {'code': 200, 'message': 'ok', 'data': {'address': '0x9221cac9a6a8fbe19c643fa36d0345f8e6b1dbb6'}, 'type': 'ETH'}
```

现在已经拿到了 ETH 地址 `0x9221cac9a6a8fbe19c643fa36d0345f8e6b1dbb6`, 将它展示给你的客户, 打入该地址的任何 ETH 都会立即计入你的余额.

# 接受回调

使用 metamask 以太坊钱包(或任何其他钱包), 发送一笔打入 `0x9221cac9a6a8fbe19c643fa36d0345f8e6b1dbb6`(注意替换为你实际取得的 ETH 地址)的交易, 等待交易确认, 前往商家后台查看交易是否到账, 或者使用预先配置的回调服务器接收如下 POST 消息:

```json
{
    "address":"0x3faf55a45b2704ec7dae62d46c8e6edcd6a545a8",
    "amount":"0.3",
    "height":3621154,
    "tx":"0xf11f070c55d2131c04622519ae6671375220175b81b380480030f1abb8a91598",
    "uid":1
}
```

# 查询账户余额

检查你账号的余额

```py
req = Request('GET', '/v1/balance')
req.sign()
resp = req.request()
print(resp.json())

# {"code":200,"message":"ok","data":{"account_id":133,"eth":"0.3"}}
```

# 签名算法

签名所需的认证信息 Authorization 放在 Header 中.

签名计算方法:

```
Authorization: BOCC <username>:<Signature>

<Signature> = Base64 (HMAC-SHA1 (<Secret>,
<Method>&
<URI>&
<Date>&
<Content-MD5>
))
```

举例:
```
POST host HTTP/1.1
User-Agent: Go-http-client/1.1
Content-Length: 170
Authorization: BOCC root:btui0ztJ9Qi7FTbCeklNHhz8bF0=
Content-MD5: f3089c1b0efba4c23c01017bedb2baa9
Content-Type: application/json
Date: Tue, 10 Jul 2018 12:36:48 GMT
Accept-Encoding: gzip

{"address":"0x64ff867048064db76f2987445cc8909267855ec8","amount":"0.5","height":3609645,"tx":"0x02891559e02be4e1e0f1b37f40d9891733f3819ec7661c63f2caa666f2da184e","uid":2}
```

1. 首先计算 http body `{"address":"0x64ff867048064db76f2987445cc8909267855ec8","amount":"0.5","height":3609645,"tx":"0x02891559e02be4e1e0f1b37f40d9891733f3819ec7661c63f2caa666f2da184e","uid":2}` 的 MD5 为 `f3089c1b0efba4c23c01017bedb2baa9`
2. 拼接待签名字符串 `POST&/&Tue, 10 Jul 2018 12:36:48 GMT&f3089c1b0efba4c23c01017bedb2baa9`
3. 通过密钥 `ZJTqPMgkDBoNmZgpJeWy` 计算待签名字符串 HMAC-SHA1
4. base64 编码签名结果得到 `btui0ztJ9Qi7FTbCeklNHhz8bF0=`
5. 设置 HTTP 请求头 `Content-MD5`, `Date` 与 `Authorization`.
