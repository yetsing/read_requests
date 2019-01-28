## Requests v0.9.0 源码阅读
### 0x00 前言

过年前的阅读，感觉自己棒棒哒 :)

### 0x01 源码阅读

#### v0.8.1

```
0.8.1 (2011-11-15)
++++++++++++++++++

* URL Request path fix
* Proxy fix.
* Timeouts fix.
```

1. 修复请求 path 

[issue#265](https://github.com/requests/requests/issues/265)

之前的在发送请求时，请求中的请求行会是绝对 URL ，而不是 path ，像下面这样

`GET http://www.rankingz.com HTTP/1.1`

作者给 Request 类添加了一个方法 path_url ，用于返回 URL 中的 path 。

2. 修复代理

   如代码所示

```python
                ...
                # 重定向中的代码
                request = Request(
                    url=url,
                    headers=headers,
                    files=self.files,
                    method=method,
                    params=self.session.params,
                    auth=self._auth,
                    cookies=cookies,
                    redirect=True,
                    config=self.config,
                    timeout=self.timeout,
                    _poolmanager=self._poolmanager,
                    proxies = self.proxies,
                )
        ...
        # send() 中的代码
        if proxy:
            # conn = poolmanager.proxy_from_url(url)
            conn = poolmanager.proxy_from_url(proxy)
```

之前的情况会导致代理没有作用，重定向时需要传递代理

3. 修复限制时间

重定向时忘记将 timeouts 参数传进去，这次加上了

这个重定向应该是至今为止问题最多的了😂

#### v0.8.2

```
0.8.2 (2011-11-19)
++++++++++++++++++

* New unicode decoding system, based on overridable `Response.encoding`.
* Proper URL slash-quote handling.
* Cookies with ``[``, ``]``, and ``_`` allowed.
```

1.基于重载的 `Response.encoding` 新 unicode 解码系统

2.URL path 中的  '/'  的正确处理

```python
def requote_path(path):
    """Re-quote the given URL path component.

    This function passes the given path through an unquote/quote cycle to
    ensure that it is fully and consistenty quoted.
    """
    parts = path.split("/")
    parts = (urllib.quote(urllib.unquote(part), safe="") for part in parts)
    return "/".join(parts)
```

这个函数将 '/' 去除后，再进行编码，最后加上 '/' ，主要是为了避免 '/' 被编码

3.cookies 中允许包含 [ 、  ]   、 _ 符号

