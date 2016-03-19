## Requests v0.6.0 源码阅读

### 0X00 前言

前些天跟强哥通电话（对，就是害我入了程序猿的坑的魂淡），他说他回成电读cs博了，导师联系好了，应该问题不大了。

他说啊， 实在不想写一点简单的逻辑，维护几行sb代码。准备回学校搞比较火的机器学习，出来去大公司的研究院打杂。

深以为然。

所以说啊，少年，你如果对现状不满，你就使劲折腾，去寻找自己想要的。

如果没能力，那你凭什么对现状不满。

### 0X01 目标

```
0.6.0 (2011-08-17)
++++++++++++++++++

* New callback hook system
* New persistient sessions object and context manager
* Transparent Dict-cookie handling
* Status code reference object
* Removed Response.cached
* Added Response.request
* All args are kwargs
* Relative redirect support
* HTTPError handling improvements
* Improved https testing
* Bugfixes
```

最近的两个版本的小版本都特别少（v0.4.x, v0.5.x），看起来还是很开心的。

但是这 histroy 不能忍 （ex, Bugfixes），谁猜的到这指什么。

不过作者的commit log 写的还不错，偶尔会卖萌，配合 issue 和 pr 基本都能确定指什么，个别除外。

### 0x02 历史

```
0.5.1 (2011-07-23)
++++++++++++++++++

* International Domain Name Support!
* Access headers without fetching entire body (``read()``)
* Use lists as dicts for parameters
* Add Forced Basic Authentication
* Forced Basic is default authentication type
* ``python-requests.org`` default User-Agent header
* CaseInsensitiveDict lower-case caching
* Response.history bugfix
```

1. 域名支持。
2. 获取头信息无需拿取整个body
3. 使用列表代替字典做参数
4. 增加强制基础验证
5. 增加 User-Agent
6. 对大小写无关字典的改进
7. 修复 Respoonse.history 的bug


```
0.6.0 (2011-08-17)
++++++++++++++++++

* New callback hook system
* New persistient sessions object and context manager
* Transparent Dict-cookie handling
* Status code reference object
* Removed Response.cached
* Added Response.request
* All args are kwargs
* Relative redirect support
* HTTPError handling improvements
* Improved https testing
* Bugfixes
```
1. 新的回调钩子系统；
2. 新的上下文管理； ???
3. 透明的字典形式的cookie处理；
4. 移除了 Response.cached
5. 添加了 Response.request
6. 所有的参数以 kwargs 的形式
7. 支持了 相对重定向；？？？
8. 加强了对 HTTPError 处理
9. 加强了对 https 的测试
10. 修bug -_-|||

### 0x03 源码阅读

#### V0.5.1

```
0.5.1 (2011-07-23)
++++++++++++++++++

* International Domain Name Support!
* Access headers without fetching entire body (``read()``)
* Use lists as dicts for parameters
* Add Forced Basic Authentication
* Forced Basic is default authentication type
* ``python-requests.org`` default User-Agent header
* CaseInsensitiveDict lower-case caching
* Response.history bugfix
```

#### 1. 国际化域名支持

国际化域名即 idna，允许非ASCII 的域名。维基链接：https://zh.wikipedia.org/wiki/%E5%9B%BD%E9%99%85%E5%8C%96%E5%9F%9F%E5%90%8D

相关代码如下

```
parsed_url = list(urlparse(self.url))
parsed_url[1] = parsed_url[1].encode('idna')
self.url = urlunparse(parsed_url)
```

用 urlparse（标准库）将 url 分为元祖,其中索引为1的位置是域名。将他encode成 'idna'，再将他拼起来。

相关测试如下

```
def test_idna(self):
	r = requests.get(u'http://➡.ws/httpbin')
    self.assertEqual(r.url, HTTPBIN_URL)
```
哈哈，这域名够我笑一个月。

##### 2. 请求的时候，并不真的拿取整个 body，直到你 read或resp.content的时候

问题来自于 issue #86

原来的请求返回 resp，直接将 Response.content = resp.read()

但是在实际生产中，可能并不需要直接获取body(设想请求一个1G的文件，你就等着它down下来吧～)。

作者问，“那你可以直接 head 吗？”

提问的人说，“我head 一下， 再判断，再get? 好烦啊，我只想请求一次。 而且有些服务端对 head 的请求支持的不好。”

作者就修复了这个问题。其实这个问题的修复，也不是很难，将 Response.read  = resp.read，闭包传给了返回的函数。

当想直接去 Reponse.content的时候，才去执行 Resposne.read()。

因为urllib2.urlopen 返回的也是一个类似文件的对象。

上代码。

```
def __getattr__(self, name):
    """Read and returns the full stream when accessing to :attr: `content`"""
    if name == 'content':
        if self._content is not None:
            return self._content
        self._content = self.read()
        if self.headers.get('content-encoding', '') == 'gzip':
            try:
                self._content = zlib.decompress(self._content, 16+zlib.MAX_WBITS)
            except zlib.error:
                pass
        return self._content

```

#### 3. 用列表代替字典做参数

问题来自 issue #83

之前是不支持多参数的，因为存放的数据参数为字典。现在改为 tuple 的字典

按照 issue 里面的例子说明下

```
requests.get('http://localhost:7999?ck=1&ck=2&ck=3')
```

之前的处理逻辑，参数会变为

```
{'ck': 3}
```
现在改为了 

```
[('ck', ), ('ck', 2), ('ck', 3)]
```

代码就不贴了，算了修了一个 bug 吧。


#### 4. 增加强制基础验证

问题来自 issue #88

问题是有一个用户请求 github 的API，操作自己的私有仓库，带了auth，但是并不能获得数据。

作者回到
> You aren't doing anything wrong. The way Basic Auth works is the client requests the url w/o auth, the server challenges it (403), and the client sends the auth. GitHub is sending a not found 404 for security reasons, which isn't standard protocol.

翻译一下，你的代码没问题，只是呢，验证的工作方式是客户端先去请求url，如果返回403(Forbidden)，则用头信息里面的auth 去验证，但是github为了安全考虑，返回的是404，所以就跪了，当然啦这不是标准的协议定义的。

真神奇，今天才知道返回404 还能保证安全，因为403，被人会知道url没有错，这里有东西，但是404别人就不知道了。666

代码改动的地方，请求默认改成了 HTTPForcedBasicAuthHandler。这个类也是对 HTTPBasicAuthHandler 的继承，看一下具体的实现逻辑。

```
def _http_error_auth_reqed(self, authreq, host, req, headers):

    authreq = headers.get(authreq, None)

    if self.retried > 5:
        # retry sending the username:password 5 times before failing.
        raise urllib2.HTTPError(req.get_full_url(), 401, "basic auth failed",
                        headers, None)
    else:
        self.retried += 1

    if authreq:

        mo = self.rx.search(authreq)

        if mo:
            scheme, quote, realm = mo.groups()

            if scheme.lower() == 'basic':
                response = self.retry_http_basic_auth(host, req, realm)

                if response and response.code not in (401, 404):
                    self.retried = 0
                return response
    else:
        response = self.retry_http_basic_auth(host, req, 'Realm')

        if response and response.code not in (401, 404):
            self.retried = 0
        return response
```

这里面还是挺复杂的，有涉及到 www-authenticate 的东西，就说之前HTTP 1.0版本的验证信息（账号密码）是带在请求头里面的，这会有安全问题。如果请求头里面有 www-authenticate，判断逻辑是有不同的。

rfc 的文档贴在这里吧。

https://tools.ietf.org/html/rfc2617

#### User-Agent

用 requests 发送请求的，默认加入 User-Agent头信息：

```
settings.base_headers = {'User-Agent': 'python-requests.org'}
```

#### 小写字母 get

对 CaseInsensitiveDict 的改进。包括允许删除操作， get 如果没有元素，返回None，而不是 IndexError 。

没什么好说：）

#### v0.6.0

```
0.6.0 (2011-08-17)
++++++++++++++++++

* New callback hook system
* New persistient sessions object and context manager
* Transparent Dict-cookie handling
* Status code reference object
* Removed Response.cached
* Added Response.request
* All args are kwargs
* Relative redirect support
* HTTPError handling improvements
* Improved https testing
* Bugfixes
```

#### 1. 新的回调钩子系统

钩子系统还是很有趣的， 可以在请求前，请求后，返回response 之前做一些你想做的事情。

看实现

```
    r = Request(**args)

    # Pre-request hook.
    r = dispatch_hook('pre_request', hooks, r)

    # Send the HTTP Request.
    r.send()

    # Post-request hook.
    r = dispatch_hook('post_request', hooks, r)

    # Response manipulation hook.
    r.response = dispatch_hook('response', hooks, r.response)

    return r.response
```

如图，在请求的不同阶段会运行 dispatch_hook 这个函数，来看下这个函数

```
def dispatch_hook(key, hooks, hook_data):
    """Dipatches a hook dictionary on a given peice of data."""

    hooks = hooks or dict()

    if key in hooks:
        try:
            return hooks.get(key).__call__(hook_data) or hook_data

        except Exception, why:
            warnings.warn(str(why))

    return hook_data
```

key 用来标记请求发生的时间，hook 是传进来的字典参数， hook_data 为回调函数名

如果key 存在于hook，调用hook[key] 函数。

官网文档有个例子说明使用方法。

http://docs.python-requests.org/zh_CN/latest/user/advanced.html#id8


#### 2.  新的 session 参数
```
def test_session_HTTPS_200_OK_GET(self):

    s = Session()
    r = s.get(httpsbin('/'))
    self.assertEqual(r.status_code, 200)
```

测试正常可用。

```
def test_session_persistent_headers(self):

    heads = {'User-agent': 'Mozilla/5.0'}

    s = Session()
    s.headers = heads
    # Make 2 requests from Session object, should send header both times
    r1 = s.get(httpbin('user-agent'))

    assert heads['User-agent'] in r1.content
    r2 = s.get(httpbin('user-agent'))

    assert heads['User-agent'] in r2.content
    self.assertEqual(r2.status_code, 200)
```

验证重复请求，User-agent 始终存在

看了下写法，然后我就跪着敲这些源码阅读了...

简单来说就是装饰器加上下文

上代码

```
class Session(object):
    """A Requests session."""

    __attrs__ = ['headers', 'cookies', 'auth', 'timeout', 'proxies', 'hooks']


    def __init__(self, **kwargs):

        # Set up a CookieJar to be used by default
        self.cookies = cookielib.FileCookieJar()

        # Map args from kwargs to instance-local variables
        map(lambda k, v: (k in self.__attrs__) and setattr(self, k, v),
                kwargs.iterkeys(), kwargs.itervalues())

        # Map and wrap requests.api methods
        self._map_api_methods()

```

第一个注释： 设置 cookies

第二个注释： 设置 所有 headers, cookies 等内容

第三个注释： 从requests.api中加载所有的 method (get,post等内容)，并给他们加上了装饰器

接着看下第三个注释的调用函数的代码

```
def _map_api_methods(self):
    """Reads each available method from requests.api and decorates
    them with a wrapper, which inserts any instance-local attributes
    (from __attrs__) that have been set, combining them with **kwargs.
    """

    def pass_args(func):
        def wrapper_func(*args, **kwargs):
            inst_attrs = dict((k, v) for k, v in self.__dict__.iteritems()
                    if k in self.__attrs__)
            # Combine instance-local values with kwargs values, with
            # priority to values in kwargs
            kwargs = dict(inst_attrs.items() + kwargs.items())

            # If a session request has a cookie_dict, inject the
            # values into the existing CookieJar instead.
            if isinstance(kwargs.get('cookies', None), dict):
                kwargs['cookies'] = add_dict_to_cookiejar(
                    inst_attrs['cookies'], kwargs['cookies']
                )

            if kwargs.get('headers', None) and inst_attrs.get('headers', None):
                kwargs['headers'].update(inst_attrs['headers'])

            return func(*args, **kwargs)
        return wrapper_func

    # Map and decorate each function available in requests.api
    map(lambda fn: setattr(self, fn, pass_args(getattr(api, fn))),
            api.__all__)

```

map给所有函数加装饰器，装饰器函数 wrapper_func。

在请求之前，更新存在的配置（所有 headers, cookies 等内容）

囫囵吞枣，见笑了。

#### 3. 透明的 cookie 处理

这应该指response 中可以拿取 cookie	。

详情在 issue #116

#### 4. 移除 Response.cached

没找到 issue 跟这有有关，看了一下是作者自己删的，算是小改动。


#### 5. 添加 Response.request

在返回 Response 中加入 request类

```
self.response.request = self
```

#### 6. 所有的参数变为 kwargs

小改动

原来

```
def get(url,
    params=None, headers=None, cookies=None, auth=None, timeout=None,
    proxies=None):
```

现在

```    
def get(url, **kwargs):

```

#### 7. 相对重定向支持

```
def test_relative_redirect_history(self):

    for service in SERVICES:

        r = requests.get(service('relative-redirect', '3'))
        self.assertEquals(r.status_code, 200)
        self.assertEquals(len(r.history), 3)

```

依然是上次那个随便post的网站提供的功能。

```
In [3]: r = requests.get('http://httpbin.org/relative-redirect/3')

In [4]: r.status_code
Out[4]: 200

In [5]: r.history
Out[5]: [<Response [302]>, <Response [302]>, <Response [302]>]

```

代码方面也是小改动。

#### HTTPError 错误处理加强

在这个版本中加入了状态码和内容，但是并没有用到 ：）

#### https 的测试

测试加入 大量针对 https 没什么好说的，跟之前的 http 差别只是协议。

#### Bugfixes

略。

### 0x04 后记

其实这个源码阅读很多东西希望能抽出来写一点东西。

细致分析下内容的设计，定位番外篇好啦。

番外篇肯定比这种流水账受欢迎。

嗯。



如果有问题，欢迎各种形式 的issue 和 pr

[github v0.6.0](https://github.com/wangshunping/read_requests/blob/master/doc/Requests_v0.6.0.md)


