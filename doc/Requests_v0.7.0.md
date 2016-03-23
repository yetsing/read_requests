## Requests v0.7.0 源码阅读

### 0X00 前言

今天娱乐了好久。

当知道自己一天能够做多少事情后，娱乐一天的愧疚感就会越剧烈 。

### 0X01 目标

今天开始看 Requests 的 v0.7.0 版本，这个版本内容太多了，肯定一天搞不完。

```
0.7.0 (2011-10-22)
++++++++++++++++++

* Sessions are now the primary interface.
* Deprecated InvalidMethodException.
* PATCH fix.
* New config system (no more global settings).
```
十个月底 0.7.0 版本，平均一个多月迭代一个版本。

一个团队保持每个月的版本迭代都不容易，一个人就更别说了。

值得一提的是，AUTHORS 里面已经有 36个人了，给所有提过patch 和 suggesstion的人。

其中有一个中国人(我猜)。

```
- 潘旭 (Xu Pan)
```

### 0x02 历史
```
0.6.1 (2011-08-20)
++++++++++++++++++

* Enhanced status codes experience ``\o/``
* Set a maximum number of redirects (``settings.max_redirects``)
* Full Unicode URL support
* Support for protocol-less redirects.
* Allow for arbitrary request types.
* Bugfixes
```

1. 加强了状态码的体验；
2. 加了设置最大重定向次数；
3. 支持 unicode 编码的 URL
4. 允许任意的请求类型
5. 修bug

```
0.6.2 (2011-10-09)
++++++++++++++++++

* GET/HEAD obeys allow_redirects=False.

```

1. GET/HEAD 请求也服从 allow_redirects=False

```
0.6.3 (2011-10-13)
++++++++++++++++++

* Beautiful ``requests.async`` module, for making async requests w/ gevent.

```
1.优雅的 async 模块，为了用 gevent 进行异步请求。

```
0.6.4 (2011-10-13)
++++++++++++++++++

* Automatic decoding of unicode, based on HTTP Headers.
* New ``decode_unicode`` setting.
* Removal of ``r.read/close`` methods.
* New ``r.faw`` interface for advanced response usage.*
* Automatic expansion of parameterized headers.

```

1. 自动给 unicode 解码，基于HTTP 请求头。
2. 新的 decode_unicode 设置。
3. 移除 response.read 和 response.clode 方法。
4. 新的 response.raw 接口，用来高级的response使用。
5. 自动扩展请求头参数。

```
0.6.5 (2011-10-18)
++++++++++++++++++

* Offline (fast) test suite.
* Session dictionary argument merging.
```

1. 离线测试集合。
2. session 字典参数合并。

```
0.6.6 (2011-10-19)
++++++++++++++++++

* Session parameter bugfix (params merging).
```

1. Session 参数bug修复（特指参数merge 功能）。

```
0.7.0 (2011-10-22)
++++++++++++++++++

* Sessions are now the primary interface.
* Deprecated InvalidMethodException.
* PATCH fix.
* New config system (no more global settings).

```

1. Session 目前为主要的接口。
2. 废弃 InvalidMethodException 异常
3. Patch 修复。
4. 新的配置系统（不再是全局配置）

### 0X03 源码阅读

```
0.6.1 (2011-08-20)
++++++++++++++++++

* Enhanced status codes experience ``\o/``
* Set a maximum number of redirects (``settings.max_redirects``)
* Full Unicode URL support
* Support for protocol-less redirects.
* Allow for arbitrary request types.
* Bugfixes
```



#### 1. 状态码

取消代码中的硬编码。ex

之前

```
REDIRECT_STATI = (301, 302, 303, 307)
```

现在

```
REDIRECT_STATI = (codes.moved, codes.found, codes.other, codes.temporary_moved)
```

#### 2. 设置最大重定向次数

也是取消之前代码的硬编码。ex

之前

```
if not len(history) < 30:
```
现在

```
if not len(history) < settings.max_redirects:
```

max_redirects 和之前的 timeout 一样，可以由用户自己设置。

#### 3. 支持 unicode 编码的 URL

我记得在之前的版本已经支持了，并没有看到对这个方面的更改。


#### 4. 允许任意的请求类型

这个 history 应该叫做，不允许更改请求方式

这个commit 去掉了以下代码

```
_METHODS = ('GET', 'HEAD', 'PUT', 'POST', 'DELETE', 'PATCH')

def __setattr__(self, name, value):
    if (name == 'method') and (value):
        if not value in self._METHODS:
            raise InvalidMethod()

    object.__setattr__(self, name, value)

```

method 在初始化 Request 的时候就确定了，并且不允许之后的更改。

#### 5. bugfix
略

### v0.6.2

```
0.6.2 (2011-10-09)
++++++++++++++++++

* GET/HEAD obeys allow_redirects=False.

```

#### 1. GET/HEAD 请求也可以 allow_redirects=False

问题来源于 pr #142

kennethreitz 解释说，在RFC2616中，allow_redirects 是只针对除了 GET/HEAD的请求的。

> The allow_redirects parameter is only for PUSH/POST/&c requests, to allow the behavior described in RFC 2616:

>If the 301 status code is received in response to a request other than GET or HEAD, the user agent MUST NOT automatically redirect the request unless it can be confirmed by the user, since this might change the conditions under which the request was issued.

所以在之前的代码中，关于是否进行递归重定向，有以下的判断逻辑

```
 while (
                ('location' in r.headers) and
                ((self.method in ('GET', 'HEAD')) or
                (r.status_code is codes.see_other) or
                (self.allow_redirects))
            ):
```

即HEAD 和 GET的请求，是默认重定向的。

有个人跳出来给了个 pr ,说我要调试，我就不想他重定向了。

又有一个人说，


>Good though, kennethreitz! Please, apply this pull request, it will make world better ;-)

作者就合并了，逻辑改成了

```
while (
    ('location' in r.headers) and
    ((r.status_code is codes.see_other) or (self.allow_redirects))
):
```
有时候看这些 pr 真是热血。

It will make world better.

### v0.6.3

```
0.6.3 (2011-10-13)
++++++++++++++++++

* Beautiful ``requests.async`` module, for making async requests w/ gevent.

```
#### 1.优雅的 async 模块，为了用 gevent 进行异步请求。

没有找到相关测试，但是在 issue 里面找到了作者的说明。

先上作者测试的效果

```
# -*- coding: utf-8 -*-

import requests
from requests import async

urls = [
    'http://www.readability.com',
    'http://tablib.org',
    'http://httpbin.org',
    'http://python-requests.org',
    'http://kennethreitz.com'
]

urls = urls + urls

## Standard (14.125s)
for url in urls:
     r = requests.get(url)
     r.content
     
## Standard + Keep-Alive (6.390s)
with requests.session() as s:
    for url in urls:
         r = requests.get(url)
         r.content

##Standard + Gevent (1.589s)
>>> rs = [async.get(u) for u in urls]
>>> async.map(rs)
[<Response [200]>, <Response [200]>, ...]

##Gevent + Keep-Alive (1.132s)
>>> rs = [async.get(u) for u in urls]
>>> async.map(rs, keep_alive=True)
[<Response [200]>, <Response [200]>, ...]
```

直接一个O(1) 一个O(n)。

Pure awesomesauce !!!!

关于 gevent 协程的使用，我丢到番外篇去吧（其实是我没太看懂，怕说错了，玷污这个代码）

### v0.6.4

```
0.6.4 (2011-10-13)
++++++++++++++++++

* Automatic decoding of unicode, based on HTTP Headers.
* New ``decode_unicode`` setting.
* Removal of ``r.read/close`` methods.
* New ``r.faw`` interface for advanced response usage.*
* Automatic expansion of parameterized headers.

```

#### 1. 自动给 unicode 解码，基于HTTP 请求头。

跟第四点有关，返回的内容自动 decode_unicode。

#### 2. 新的 decode_unicode 设置。

请求回来的 body可以自动 decode, 在 settings类中可以设置，默认为True,跟第四点有关。

#### 3. 移除 response.read 和 response.close 方法。

这个 pass了。跟下面那个有关

#### 4. 新的 response.raw 接口，用来高级的response使用。

Response.raw 是

```
#: File-like object representation of response (for advanced usage).
```

来自于 pr #150

实现了，不用将所有的Response 家在到内存中（因为可能很大）。

#### 5. 自动扩展请求头参数。

加入函数 heander_expand(headers):

函数 docstring 如下

```
"""Returns an HTTP Header value string from a dictionary.

    Example expansion::

        {'text/x-dvi': {'q': '.8', 'mxb': '100000', 'mxt': '5.0'}, 'text/x-c': {}}
        # Accept: text/x-dvi; q=.8; mxb=100000; mxt=5.0, text/x-c

        (('text/x-dvi', {'q': '.8', 'mxb': '100000', 'mxt': '5.0'}), ('text/x-c', {}))
        # Accept: text/x-dvi; q=.8; mxb=100000; mxt=5.0, text/x-c
"""    
```

一个将字典形式的数据自动转换成请求头的函数。

### v0.6.5

```
0.6.5 (2011-10-18)
++++++++++++++++++

* Offline (fast) test suite.
* Session dictionary argument merging.
```


#### 1. 离线测试集合。

新增加了不需要网络就可以进行get 等测试的函数。

实现是用到了作者造的另外一个轮子 envoy 。



#### 2. session 字典参数合并。

添加 merge_kwargs 函数，当配置中存在该配置时，会更新成新的配置。

这个版本中这个功能有问题，带个小版本有修复，直接丢到下个版本讲好咯。

### v0.6.6

```
0.6.6 (2011-10-19)
++++++++++++++++++

* Session parameter bugfix (params merging).
```

#### 1. Session 参数bug修复（特指参数merge 功能）。

上测试

```
def test_session_persistent_params(self):

    params = {'a': 'a_test'}

    s = Session()
    s.params = params

    # Make 2 requests from Session object, should send header both times
    r1 = s.get(httpbin('get'))
    assert params['a'] in r1.content


    params2 = {'b': 'b_test'}

    r2 = s.get(httpbin('get'), params=params2)
    assert params['a'] in r2.content
    assert params2['b'] in r2.content


    params3 = {'b': 'b_test', 'a': None, 'c': 'c_test'}

    r3 = s.get(httpbin('get'), params=params3)

    assert not params['a'] in r3.content
    assert params3['b'] in r3.content
    assert params3['c'] in r3.content
```
如上面测试所示。

1. 可以给某个测试设置参数，添加到Session中；
2. 可以更新其参数；
3. 更新的时候设置为 None,则该设置被删除。

### v0.7.0

函数的时候逻辑也很简单，更新的跟之前的进行对比 ：）


```
0.7.0 (2011-10-22)
++++++++++++++++++

* Sessions are now the primary interface.
* Deprecated InvalidMethodException.
* PATCH fix.
* New config system (no more global settings).

```

#### 1. Session 目前为主要的接口

现在所有 method 都去调用 session 提供的接口，直接是两套逻辑（两套很类似的逻辑，除了需要比较params）。

现在的 requests.get(xxxx) 代码为：

```
def get(url, **kwargs):
    kwargs.setdefault('allow_redirects', True)
    return request('GET', url, **kwargs)

def request(method, url,
    params=None,
    data=None,
    headers=None,
    cookies=None,
    files=None,
    auth=None,
    timeout=None,
    allow_redirects=False,
    proxies=None,
    hooks=None,
    return_response=True,
    config=None):

    s = session()
    return s.request(
        method, url, params, data, headers, cookies, files, auth,
        timeout, allow_redirects, proxies, hooks, return_response,
        config
    )
```

即直接调用 session() 的逻辑进行处理。

之前为：

```
def request(method, url,
    params=None, data=None, headers=None, cookies=None, files=None, auth=None,
    timeout=None, allow_redirects=False, proxies=None, hooks=None, return_response=True):

    ....

    # Arguments manipulation hook.
    args = dispatch_hook('args', hooks, args)

    r = Request(**args)

    # Pre-request hook.
    r = dispatch_hook('pre_request', hooks, r)

    # Don't send if asked nicely.
    if not return_response:
        return r

    # Send the HTTP Request.
    r.send()

    # Post-request hook.
    r = dispatch_hook('post_request', hooks, r)

    # Response manipulation hook.
    r.response = dispatch_hook('response', hooks, r.response)

    return r.response
```

各种参数的配置，各种钩子。Session 中也有类似的这样的一整套。

#### 2. 废弃 InvalidMethodException 异常

删除这种异常的代码。

#### 3. Patch 修复

因为回滚之类的操作出现的一个bug, issue #160

#### 4. 新的配置系统（不再是全局配置）

之前我巴拉巴拉说半天的上下文形式的配置，这个版本直接干掉了，直接改成了一个默认配置文件。

心好累。

估计大部分都不太喜欢这样的用法？

改成了这种形式的默认配置文件。

```

    """
    requests.defaults
    ~~~~~~~~~~~~~~~~~

    This module provides the Requests configuration defaults.
    """

    from . import __version__

    defaults = dict()


    defaults['base_headers'] = {
        'User-Agent': 'python-requests/%s' % __version__,
        'Accept-Encoding': ', '.join([ 'identity', 'deflate', 'compress', 'gzip' ]),
    }

....

```

### 0X04 后记

感谢所有真正意义上的 coder。 

在互联网行业工作一年有余，认识一些真正意义上的coder，他们单纯不做作，目的只是 make the world better。

行就是行，不行就是不行，怎样就是怎样，让我倍感美好。

最近不是流行一句话叫，少一些套路，多一点真诚。



