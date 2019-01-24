## Requests v0.8.0 源码阅读

### 0X00 前言
wangshunping 的源码阅读已经看完了，接下来的版本就是我自己的征程了:)

### 0X01 目标

### 0x02 历史
```
0.7.1 (2011-10-23)
++++++++++++++++++

* Move away from urllib2 authentication handling.
* Fully Remove AuthManager, AuthObject, &c.
* New tuple-based auth system with handler callbacks.
```
1.移除 urllib2 的认证处理
2.完全移除 AuthManager, AuthObject 等
3.新的基于元组的认证回调函数
```
0.7.2 (2011-10-23)
++++++++++++++++++

* PATCH Fix.
```
修复 PATCH
```
0.7.3 (2011-10-23)
++++++++++++++++++

* Digest Auth fix.
```
修复摘要认证
不传输明文密码，而是用密码和其它的一些东西生成摘要，将这个摘要发给服务器验证
```
0.7.4 (2011-10-26)
++++++++++++++++++

* Sesion Hooks fix.
```
修复 Session 的钩子
```
0.7.5 (2011-11-04)
++++++++++++++++++

* Response.content = None if there was an invalid repsonse.
* Redirection auth handling.
```
1.如果得到了无效的响应，让 Response.content = None
2.重定向认证处理
```
0.7.6 (2011-11-07)
++++++++++++++++++

* Digest authentication bugfix (attach query data to path)
```
修复摘要认证 bug (将 query 加在 path 后面）
```
0.8.0 (2011-11-13)
++++++++++++++++++

* Keep-alive support!
* Complete removal of Urllib2
* Complete removal of Poster
* Complete removal of CookieJars
* New ConnectionError raising
* Safe_mode for error catching
* prefetch parameter for request methods
* OPTION method
* Async pool size throttling
* File uploads send real names
```
1.支持长连接
2.完全移除 Urllib2
3.完全移除 Poster
4.完全移除 CookieJars
5.新的异常类 ConnectionError
6.错误捕捉的安全方式
7.增加 prefetch 参数
8.增加 OPTION 方法
9.异步池的大小的限制
10.文件上传发送正确的名字

### 0X03 源码阅读
#### v0.7.1
```
0.7.1 (2011-10-23)
++++++++++++++++++

* Move away from urllib2 authentication handling.
* Fully Remove AuthManager, AuthObject, &c.
* New tuple-based auth system with handler callbacks.
```
1.移除 urllib2 的认证处理
作者抛弃了 urllib2 的认证，手动实现两种认证：Basic 和 Digest
```python
# auth.py
def http_basic(r, username, password):
    """Attaches HTTP Basic Authentication to the given Request object.
    Arguments should be considered non-positional.

    """
    ...

def http_digest(r, username, password):
    """Attaches HTTP Digest Authentication to the given Request object.
    Arguments should be considered non-positional.
    """
    ...

def dispatch(t):
    """Given an auth tuple, return an expanded version."""
    ...

# models.py
Request(object):
    def __init__(self, ...):
        ...
        #: Authentication tuple to attach to :class:`Request <Request>`.
        self.auth = auth_dispatch(auth)
        ...
    def send(self, anyway=False):
        ...
        if self.auth:
            auth_func, auth_args = self.auth
            r = auth_func(self, *auth_args)
            self.__dict__.update(r.__dict__)
        ...
```
根据用户的 auth 的不同返回不同的认证函数，发送请求前调用认证函数，更新请求
作者为什么要抛弃 urllib2 ，不晓得 (┬＿┬)
2.完全移除 AuthManager, AuthObject 等
同上
在 issues212 中，作者说
"Luckily, auth manager will be removed soon. It's my least favorite part of the codebase.I was so happy to see it die in a fire :)"
3.新的基于元组的认证回调函数
这个就是第一点中展示的那个 dispatch 函数，简单点说，就是 auth 参数是一个 tuple
根据参数的不同返回不同的认证回调函数和认证数据
详细代码如下
```python
def dispatch(t):
    """Given an auth tuple, return an expanded version."""

    if not t:
        return t
    else:
        t = list(t)

    # Make sure they're passing in something.
    assert len(t) >= 2

    # If only two items are passed in, assume HTTPBasic.
    if (len(t) == 2):
        t.insert(0, 'basic')

    # Allow built-in string referenced auths.
    if isinstance(t[0], basestring):
        if t[0] in ('basic', 'forced_basic'):
            t[0] = http_basic
        elif t[0] in ('digest',):
            t[0] = http_digest

    # Return a custom callable.
    return (t[0], tuple(t[1:]))
```
还有一点就是，请求钩子的执行从 Session 类中移到了 Request 类
#### v0.7.2
```
0.7.2 (2011-10-23)
++++++++++++++++++

* PATCH Fix.
```
修复 PATCH
这个没什么好说的，自己看
```python
def patch(url, data='', **kwargs):
    # 注释这一行是 v0.7.1 的
    # return request('patch', url,  data='', **kwargs)
    return request('patch', url,  data=data, **kwargs)
```
想要不写出 bug ，真难 (*/ω＼*)
#### v0.7.3
```
0.7.3 (2011-10-23)
++++++++++++++++++

* Digest Auth fix.
```
修复摘要认证
删掉了 http_digest 函数中的几行 print 语句
`oops, debugging prints`
作者你真可爱(๑•ᴗ•๑)
#### v0.7.4
```
0.7.4 (2011-10-26)
++++++++++++++++++

* Sesion Hooks fix.
```
修复 Session 的钩子
如下
```python
Session(object):
    ...
    def request(self, ...):
        for attr in self.__attrs__:
            session_val = getattr(self, attr, None)
            local_val = args.get(attr)

            args[attr] = merge_kwargs(local_val, session_val)
        # args = dispatch_hook('args', hooks, args)
        # 采用更新后的 hooks 属性
        args = dispatch_hook('args', args['hooks'], args)
```
还有一个 python2.5 的语法错误(issue#208)
应该是默认参数和可变参数(\*args) 引起的歧义
python3.6 参数的顺序：必选参数、默认参数、可变参数、命名关键字参数和关键字参数
```python
def patched(f):
    """Patches a given API function to not send."""

    def wrapped(*args, **kwargs):
        # return f(*args, return_response=False, **kwargs)

        kwargs['return_response'] = False

        return f(*args, **kwargs)

    return wrapped
```
v0.7.5
```
0.7.5 (2011-11-04)
++++++++++++++++++

* Response.content = None if there was an invalid repsonse.
* Redirection auth handling.
```
1.如果得到了无效的响应，让 Response.content = None
```python
Response(object):
    ...
    @property
    def content(self):
        ...
        # self._content = self.raw.read()
        try:
            self._content = self.raw.read()
        except AttributeError:
            return None
        ...
```
当返回了无效的响应时， self.raw=None ,就会抛出 AttributeError 
对于用户来说，可能会以为自己程序出错
因此，响应无效时，返回 None 给用户，更 pythonic
详情请看 issue#236
2.重定向认证处理
由于重新发起请求，认证方式和所需的一些参数可能发生改变
因此，重定向时需要传入用户的 auth 参数
self.\_auth 保存用户的 auth 参数
self.auth 保存处理过后的
#### v0.7.6
```
0.7.6 (2011-11-07)
++++++++++++++++++

* Digest authentication bugfix (attach query data to path)
```
修复摘要认证 bug (将 query 加在 path 后面）
如代码所示
```python
def http_digest(r, username, password):
    """Attaches HTTP Digest Authentication to the given Request object.
    Arguments should be considered non-positional.
    """

    def handle_401(r):
        """Takes the given response and tries digest-auth, if needed."""

        s_auth = r.headers.get('www-authenticate', '')

        if 'digest' in s_auth.lower():
            ...
            # path = urlparse(r.request.url).path
            p_parsed = urlparse(r.request.url)
            path = p_parsed.path + p_parsed.query
```
#### v0.8.0
```
0.8.0 (2011-11-13)
++++++++++++++++++

* Keep-alive support!
* Complete removal of Urllib2
* Complete removal of Poster
* Complete removal of CookieJars
* New ConnectionError raising
* Safe_mode for error catching
* prefetch parameter for request methods
* OPTION method
* Async pool size throttling
* File uploads send real names
```
1.支持长连接
```python
# session.py
class Session(object):
    """A Requests session."""

    __attrs__ = [
        'headers', 'cookies', 'auth', 'timeout', 'proxies', 'hooks',
        'params', 'config']


    def __init__(self, ...):
        ...
        self.poolmanager = PoolManager(
            num_pools=self.config.get('pool_connections'),
            maxsize=self.config.get('pool_maxsize')
        )
        ...


# models.py
class Request(object):
    ...
    def send(self, ...):
        ...
        if proxy:
            conn = poolmanager.proxy_from_url(url)
        else:
            # Check to see if keep_alive is allowed.
            if self.config.get('keep_alive'):
                conn = self._poolmanager.connection_from_url(url)
            else:
                conn = connectionpool.connection_from_url(url)
        ...
```
使用 urllib3 模块实现
如果请求需要保持长连接，就在连接池(self.\_poolmanager)中创建连接
这样下次继续向同一 URL 发请求时，就直接在连接池中取得连接
2.完全移除 Urllib2
3.完全移除 Poster
4.完全移除 CookieJars
Poster 是作者之前用的轮子
这三点一起说，作者使用了 urllib3 模块，之前的都被抛弃了
喜新厌旧╭(╯^╰)╮
5.新的异常类 ConnectionError
6.错误捕捉的安全方式
默认配置中增加了一项 `defaults['safe_mode'] = False`
作用如下
```python
Request(object):
    ...
    def send(self, ...):
        ...
            try:
                # Send the request.

                r = conn.urlopen(
                    method=self.method,
                    url=url,
                    body=body,
                    headers=self.headers,
                    redirect=False,
                    assert_same_host=False,
                    preload_content=prefetch,
                    decode_content=False,
                    retries=self.config.get('max_retries', 0),
                    timeout=self.timeout,
                )


            except MaxRetryError, e:
                if not self.config.get('safe_mode', False):
                    raise ConnectionError(e)
                else:
                    r = None

            except (_SSLError, _HTTPError), e:
                if not self.config.get('safe_mode', False):
                    raise Timeout('Request timed out.')
        ...
```
如果设置为 True ，发送请求过程中的异常都会被忽略
7.增加 prefetch 参数
prefetch=False 的话，响应中的内容先不读取，直到我调用 Response.content 时再读取
8.增加 OPTION 方法
9.异步池大小的限制
从提交历史来看，这个应该是指这段代码
```python
def map(requests, prefetch=True, size=None):
    """Concurrently converts a list of Requests to Responses.

    :param requests: a collection of Request objects.
    :param prefetch: If False, the content will not be downloaded immediately.
    :param size: Specifies the number of requests to make at a time. If None, no throttling occurs.
    """

    # jobs = [gevent.spawn(send, r) for r in requests]
    # gevent.joinall(jobs)
    if size:
        pool = Pool(size)
        pool.map(send, requests)
        pool.join()
    else:
        jobs = [gevent.spawn(send, r) for r in requests]
        gevent.joinall(jobs)

    ...
```
`size: Specifies the number of requests to make at a time. If None, no throttling occurs.`
看到作者的这段注释，我终于明白是什么意思了
10.文件上传发送正确的名字
这个我猜应该就是那个新增的 guess_filename 函数

### 0X04 后记
这个版本更新直接把用的轮子都换了，看着是真心费劲
今天就这样了
明天再重新梳理一遍
