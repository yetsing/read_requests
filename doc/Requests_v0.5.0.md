## Requests v0.5.0 源码阅读

### 0X00 前言

原来有个姑娘经常跟我说她导师让她做了某个事情，然后跟我抱怨，“你说，这事有什么意义呢？”

为了安慰她，我说“其实这个世界所有的事情都是没有意义的，你觉得它有，它就有。”

拆轮子是件很耗体力的事情。

但是才到 0.5 版本，我就接触到了 优秀的上下文管理配置写法、单例写法、特殊的字典、hacker标准库内容等等刷三观的东西。这些都是写API 是遇不到的，即使遇到，也会尽可能的用一些 trick 去避免这些。

所以，以上。

### 0X01 目标

```
0.5.0 (2011-06-21)
++++++++++++++++++

* PATCH Support
* Support for Proxies
* HTTPBin Test Suite
* Redirect Fixes
* settings.verbose stream writing
* Querystrings for all methods
* URLErrors (Connection Refused, Timeout, Invalid URLs) are treated as explicity raised
  ``r.requests.get('hwe://blah'); r.raise_for_status()``

```

不得不说，作者还是很拼的。一个月更一个大版本...

> 哪有什么天才，不过是拿你泡妹子的时间写代码罢了。		——佚名某码神

### 0X02 历史更改

版本更新历史。

```
0.4.1 (2011-05-22)
++++++++++++++++++

* Improved Redirection Handling
* New 'allow_redirects' param for following non-GET/HEAD Redirects
* Settings module refactoring

```

1. 加强了重定向处理类的处理逻辑，就是上一章里面说的递归调用（status_code 30X）。
2. 新的 allow_redirects 参数， 原来的参数名只是为 redirects。
3. settings 模块的重构，希望这次我能看的明白些。

```
0.5.0 (2011-06-21)
++++++++++++++++++

* PATCH Support
* Support for Proxies
* HTTPBin Test Suite
* Redirect Fixes
* settings.verbose stream writing
* Querystrings for all methods
* URLErrors (Connection Refused, Timeout, Invalid URLs) are treated as explicity raised
  ``r.requests.get('hwe://blah'); r.raise_for_status()``

```

1. patch支持（还记得上一篇中提到了空的 patches.py 文件吗，填坑来了）。
2. 代理支持。
3. httpbin 测试集合。
4. 重定向bug修复，果然这个是个坑，这两个版本一直针对这个反复优化。
5. settings 相关的改动，翻译无能...
6. 给所有的方法加入 querystring。
7. URLErrors 会当作异常报出。

### 0X03 源码阅读

### v0.4.1 

```
0.4.1 (2011-05-22)
++++++++++++++++++

* Improved Redirection Handling
* New 'allow_redirects' param for following non-GET/HEAD Redirects
* Settings module refactoring

```

#### 1. 处理 Redirection Handling

因为采用递归去处理请求，即如果判断，请求得到的是重定向，则继续请求。

所以在urllib 那一层，不希望urllib来处理重定向，跟之前的方法一样，hack了urllib2.HTTPRedirectHandler.http_error_30X 几个函数。

代码如下。

```
class HTTPRedirectHandler(urllib2.HTTPRedirectHandler):

    def http_error_301(self, req, fp, code, msg, headers):
        pass

    http_error_302 = http_error_303 = http_error_307 = http_error_301
```	

#### 2. 新的允许重定向参数

原来的 direct 改成了 allow_redirects 。

#### 3. Settings 模块重构

追着 commit log 看了一下，这个版本关于 settings 的改动来自于把settings 写成了一个单例，并且实例化加到变量中，还对 time_out 参数做了缓存。

前前一篇也讲到了对单例模式的应用，用在了权限管理系统中，当你第一次请求，加了auth并且通过，下次请求同样的url 就不需要再带权限了。

两者做法大致相同，但是在全局实例化一个对象。

但是 settings 用到了上下文管理，还是很有趣的。


### v0.5.0 

```
0.5.0 (2011-06-21)
++++++++++++++++++

* PATCH Support
* Support for Proxies
* HTTPBin Test Suite
* Redirect Fixes
* settings.verbose stream writing
* Querystrings for all methods
* URLErrors (Connection Refused, Timeout, Invalid URLs) are treated as explicity raised
  ``r.requests.get('hwe://blah'); r.raise_for_status()``

```

#### 1. PATCH 支持

好吧，刚 google 回来。patch 是 http method中的一种，我之前不知道。

https://tools.ietf.org/html/rfc5789

大意是 post是新建一个数据（资源）,PUT是对资源的替换，而有时候部分替换，所以加了 PATCH。

原来写 RESTful API 的时候，我都直接用 PUT 的。

```
	Several applications extending the Hypertext Transfer Protocol (HTTP)
   require a feature to do partial resource modification.  The existing
   HTTP PUT method only allows a complete replacement of a document.
   This proposal adds a new HTTP method, PATCH, to modify an existing
   HTTP resource.

```
这是摘要。


#### 2. 支持代理

issue #66

pull request #54

调用了 urllib2 里面的 ProxyHandlers

```
 if self.proxies:
     _handlers.append(urllib2.ProxyHandler(self.proxies))
```


#### 3. httpbin	测试集合

即然说到了对测试优化，就贴一下关于post的测试吧。

```
   def test_POSTBIN_GET_POST_FILES(self):
        bin = requests.post('http://www.postbin.org/')
        self.assertEqual(bin.status_code, 302)

        post_url = bin.headers['location']
        post = requests.post(post_url, data={'some': 'data'})
        self.assertEqual(post.status_code, 201)

        post2 = requests.post(post_url, files={'some': open('test_requests.py')})
        self.assertEqual(post2.status_code, 201)

        post3 = requests.post(post_url, data='[{"some": "json"}]')
        self.assertEqual(post.status_code, 201)
```
1. 测试是否返回重定向状态码；
2. 是否能从 Response 的 headers 拿取 location,并正常post, 返回201；
3. 文件上传测试；
4. 字符串格式 data 上传测试；

```

    def test_POSTBIN_GET_POST_FILES_WITH_PARAMS(self):
        bin = requests.post('http://www.postbin.org/')
        self.assertEqual(bin.status_code, 302)

        post_url = bin.headers['location']

        post2 = requests.post(post_url, files={'some': open('test_requests.py')}, data={'some': 'data'})
        self.assertEqual(post2.status_code, 201)
```
1. post文件的同时，post data，验证是否正常；

```
    def test_POSTBIN_GET_POST_FILES_WITH_HEADERS(self):
        bin = requests.post('http://www.postbin.org/')
        self.assertEqual(bin.status_code, 302)

        post_url = bin.headers['location']

        post2 = requests.post(post_url, files={'some': open('test_requests.py')},
        headers = {'User-Agent': 'requests-tests'})

        self.assertEqual(post2.status_code, 201)

```
1. post 请求加请求头更改，是否正常；

#### 4. 重定向 bug 修复

重定向时，有时候在 headers['location'] 中没有带域名，只有路径（浏览器会支持重定向到单前域名）。

为了修复这个问题，加了几行代码。

```
	url = r.headers['location']

	# Facilitate for non-RFC2616-compliant 'location' headers
	# (e.g. '/path/to/resource' instead of 'http://domain.tld/path/to/resource')
	if not urlparse(url).netloc:
    	parent_url_components = urlparse(self.url)
    	url = '%s://%s/%s' % (parent_url_components.scheme, parent
```

#### 5. settings 的改动, verbose 支持



verbose 支持来自作者自己提的一个 issue(#61)。

实现的功能如下

```
>>> with requests.settings(verbose=sys.stderr):
...    r = requests.get('http://httpbin.org')
...
GET http://httpbin.org
```

代码

```
# Logging
if settings.verbose:
    settings.verbose.write('%s   %s   %s\n' % (
        datetime.now().isoformat(), self.method, self.url
    ))
```

突然发现了实现一个上下文之后好好用啊！

#### 6. 给 method 加入 querystring

```
def test_nonurlencoded_post_querystring(self):
    r = requests.post(httpbin('post'), params='fooaowpeuf')
    self.assertEquals(r.status_code, 200)
    self.assertEquals(r.headers['content-type'], 'application/json')
    self.assertEquals(r.url, httpbin('post?fooaowpeuf'))
    rbody = json.loads(r.content)
    self.assertEquals(rbody.get('form'), {}) # No form supplied
    self.assertEquals(rbody.get('data'), '')


def test_urlencoded_post_query_and_data(self):
    r = requests.post(httpbin('post'), params=dict(test='fooaowpeuf'),
                      data=dict(test2="foobar"))
    self.assertEquals(r.status_code, 200)
    self.assertEquals(r.headers['content-type'], 'application/json')
    self.assertEquals(r.url, httpbin('post?test=fooaowpeuf'))
    rbody = json.loads(r.content)
    self.assertEquals(rbody.get('form'), dict(test2='foobar'))
    self.assertEquals(rbody.get('data'), '')
```

这两个测试跟 history 上这一点有关。

但是并没有明白， querystring 和这些断言有什么联系。

猜测是能返回经过拼接的url(也就是 querystring)。

### 0X04 后记

略。



