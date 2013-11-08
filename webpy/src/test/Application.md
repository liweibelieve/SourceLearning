## 分模块测试

### application.py

对 application.py 的测试，调用命令：

    python test/application.py


程序入口依然会调用 `webtest.main()`

```python
if __name__ == '__main__':
    webtest.main()
```

再仔细观察 `webtest.main` 调用的 `web.test` 中的 `main`函数

```python
def main(suite=None):
    if not suite:
        main_module = __import__('__main__')
        # allow command line switches
        args = [a for a in sys.argv[1:] if not a.startswith('-')]
        suite = module_suite(main_module, args or None)

    result = runTests(suite)
    sys.exit(not result.wasSuccessful())
```

回顾一下 `module_suite`函数，

```python
def module_suite(module, classnames=None):
    """Makes a suite from a module."""
    if classnames:
        return unittest.TestLoader().loadTestsFromNames(classnames, module)
    elif hasattr(module, 'suite'):
        return module.suite()
    else:
        return unittest.TestLoader().loadTestsFromModule(module)
```

与 `alltests.py` 不同，`application.py`中并没有 `suite`，
所以直接进入第三分支。后面的流程就和之前讲的后半部分一致了。

这里，我们主要集中于 `application.py`中的具体内容。

* 1. test_reloader(self)

```python
def test_reloader(self):
    write('foo.py', data % dict(classname='a', output='a'))
    import foo
    app = foo.app
    
    self.assertEquals(app.request('/').data, 'a')
    
    # test class change
    time.sleep(1)
    write('foo.py', data % dict(classname='a', output='b'))
    self.assertEquals(app.request('/').data, 'b')

    # test urls change
    time.sleep(1)
    write('foo.py', data % dict(classname='c', output='c'))
    self.assertEquals(app.request('/').data, 'c')
```

总的来说，这个过程会生成一个 foo.py 文件

```python
import web

urls = ("/", "a")
app = web.application(urls, globals(), autoreload=True)

class a:
    def GET(self):
        return "a"

```

这是一个典型的 web 服务端应用程序，表示对 `/` 发起 `GET`
请求时，会调用 `class a` 中的 `GET` 函数，测试就是看看
web.application 是否可以正常完成任务，即是否可以正确返回"a"

下面详细看代码。

首先使用 `write` 生成了一个 `foo.py` 程序:

```python
        write('foo.py', data % dict(classname='a', output='a'))
```

write 源代码:

```python
def write(filename, data):
    f = open(filename, 'w')
    f.write(data)
    f.close()
```

data 定义:
```python
data = """
import web

urls = ("/", "%(classname)s")
app = web.application(urls, globals(), autoreload=True)

class %(classname)s:
    def GET(self):
        return "%(output)s"

"""
```

`data` 相当于一个小型 web 程序的模板，类名和返回值由
一个 `dict` 指定，生成一个字符串，由 `write` 生成文件。 

下面是类别和返回值为 `a` 时的 `foo.py`

```python
import web

urls = ("/", "a")
app = web.application(urls, globals(), autoreload=True)

class a:
    def GET(self):
        return "a"
```

测试的方式采用 `TestCase` 中的 `assertEquals` 函数，比较
实际值与预测值。

```python
import foo
app = foo.app
self.assertEquals(app.request('/').data, 'a')
```

`app.request('/')` 会得到一个Storage类型的值：

    <Storage {'status': '200 OK', 'headers': {}, 'header_items': [], 'data': 'a'}>

其中的 `data` 就是 `foo.py` 中 `GET` 返回的值。 

我对这个 `app.request('/')` 是比较困惑的。以 `foo.py` 为例，
之前写程序时，一般是有一个这样的程序：

```python
import web

urls = ("/", "a")
app = web.application(urls, globals(), autoreload=True)

class a:
    def GET(self):
        return "a"

if __name__ == "__main__":
    app.run()
```

然后在浏览器中请求 `0.0.0.0:8080/` 。
而在 `request` 之前，也没看到 `run` 啊，怎么就能 `request` 回
数据呢，而且通过上述代码运行后，程序会一直运行直到手动关闭，
而 `request` 的方式则是测试完后，整个程序也结束了。

所以，下一部，想比较一下 `application.run` 和 `application.request` 的不同。

我们只看关键部分，即返回的值是如何被设值的。

在 `web.application.request` 中:

```python
def request(self, localpart='/', method='GET', data=None,
            host="0.0.0.0:8080", headers=None, https=False, **kw):

    ...

response = web.storage()
def start_response(status, headers):
    response.status = status
    response.headers = dict(headers)
    response.header_items = headers
response.data = "".join(self.wsgifunc()(env, start_response))
return response
```

上述代码中 `self.wsgifunc()(env, start_response)` 比较另人困惑，
我还以为是调用函数的新方式呢，然后看了一下 `wsgifunc` 的代码，
它会返回一个函数`wsgi`，`wsgi`以 `(env, start_response)` 为参数。
在 `wsgi` 内部，又会调用 `handle_with_processors`, `handle_with_processors` 会再调用 `handle`

```python
    def handle(self):
        fn, args = self._match(self.mapping, web.ctx.path)
        return self._delegate(fn, self.fvars, args)
```

测试了一下，`self._match()` 会得到类名, `self._delegate` 会
得到返回的字符串，即 `GET`的返回值。

进入 `self._delegate`, 会最终调用一个关键函数 `handle_class`:

```python
def handle_class(cls):
    meth = web.ctx.method
    if meth == 'HEAD' and not hasattr(cls, meth):
        meth = 'GET'
    if not hasattr(cls, meth):
        raise web.nomethod(cls)
    tocall = getattr(cls(), meth)
    return tocall(\*args)
```

参数`cls`值为`foo.a`, `meth` 会得到方法名 `GET`, 然后
`tocall` 会得到函数 `a.GET`, 至此，我们终于得以调用，
`GET`函数，得到了返回的字符串。

从整个过程可以看出，没有启动服务器的代码，只是不断地调用
函数，最终来到 `GET` 函数。


再看看 `web.application.run`:

```python
def run(self, *middleware):
    return wsgi.runwsgi(self.wsgifunc(\*middleware))
```

接着，我们来到 `wsgi.runwsgi`:

```python
def runwsgi(func):
    """
    Runs a WSGI-compatible `func` using FCGI, SCGI, or a simple web server,
    as appropriate based on context and `sys.argv`.
    """
    
    if os.environ.has_key('SERVER_SOFTWARE'): # cgi
        os.environ['FCGI_FORCE_CGI'] = 'Y'

    if (os.environ.has_key('PHP_FCGI_CHILDREN') #lighttpd fastcgi
      or os.environ.has_key('SERVER_SOFTWARE')):
        return runfcgi(func, None)
    
    if 'fcgi' in sys.argv or 'fastcgi' in sys.argv:
        args = sys.argv[1:]
        if 'fastcgi' in args: args.remove('fastcgi')
        elif 'fcgi' in args: args.remove('fcgi')
        if args:
            return runfcgi(func, validaddr(args[0]))
        else:
            return runfcgi(func, None)
    
    if 'scgi' in sys.argv:
        args = sys.argv[1:]
        args.remove('scgi')
        if args:
            return runscgi(func, validaddr(args[0]))
        else:
            return runscgi(func)
    
    
    server_addr = validip(listget(sys.argv, 1, ''))
    if os.environ.has_key('PORT'): # e.g. Heroku
        server_addr = ('0.0.0.0', intget(os.environ['PORT']))
    
    return httpserver.runsimple(func, server_addr)
```

前面是对参数 `sys.argv` 分析，根据参数指定，启动相应服务，
我们的简单web程序没有参数，所以直接来到最后几行。

关键在于最后的 `return`

```python
return httpserver.runsimple(func, server_addr)
```

可以看出，这里启动了一个服务器。

再看看 `httpserver.runsimple`的代码：

```python
def runsimple(func, server_address=("0.0.0.0", 8080)):
    """
    Runs [CherryPy][cp] WSGI server hosting WSGI app `func`. 
    The directory `static/` is hosted statically.

    [cp]: http://www.cherrypy.org
    """
    global server
    func = StaticMiddleware(func)
    func = LogMiddleware(func)
    
    server = WSGIServer(server_address, func)

    if server.ssl_adapter:
        print "https://%s:%d/" % server_address
    else:
        print "http://%s:%d/" % server_address

    try:
        server.start()
    except (KeyboardInterrupt, SystemExit):
        server.stop()
        server = None
```

从注释中可以看出，这里使用了 `CherryPy`中的 `WSGI server`,
启动了服务器。

```python
        server.start()
```

我们不再继续深入下去。只是直观地了解一下， `application.run`
和 `application.request` 的不同之处。

从这里我们看出了相当重要的概念 `WSGI`。
这里有几篇对 `WSGI` 作介绍的博客：

* [Wsgi研究](http://blog.kenshinx.me/blog/wsgi-research/) 
* [wsgi初探](http://linluxiang.iteye.com/blog/799163) 
* [Getting Started with WSGI] (http://lucumr.pocoo.org/2007/5/21/getting-started-with-wsgi/) 
* [An Introduction to the Python Web Server Gateway Interface] (http://ivory.idyll.org/articles/wsgi-intro/what-is-wsgi.html) 
* [化整為零的次世代網頁開發標準: WSGI](http://blog.ez2learn.com/2010/01/27/introduction-to-wsgi/) 


这里有对 `WSGI` 的详细介绍:

* [WSGI Tutorial](http://webpython.codepoint.net/wsgi_tutorial) 
* [WSGI org](http://wsgi.readthedocs.org/en/latest/) 
* [PEP 333](http://www.python.org/dev/peps/pep-0333/) 

* 2. test_UppercaseMethods(self)

这个函数源代码是：

```python
def testUppercaseMethods(self):
    urls = ("/", "hello")
    app = web.application(urls, locals())
    class hello:
        def GET(self): return "hello"
        def internal(self): return "secret"
        
    response = app.request('/', method='internal')
    self.assertEquals(response.status, '405 Method Not Allowed')
```

这个函数似乎是在测试 `request` 请求的方法不正确时，`application`
的反应。

我们打印一下 `response` 的值，发现内容是这样的：

    <Storage {'status': '405 Method Not Allowed', 'headers': {'Content-Type': 'text/html', 'Allow': 'GET, HEAD, POST, PUT, DELETE'}, 'header_items': [('Content-Type', 'text/html'), ('Allow', 'GET, HEAD, POST, PUT, DELETE')], 'data': 'None'}>

从这里可以看出允许的方法：

    'Allow': 'GET, HEAD, POST, PUT, DELETE'

`response.status` 最终值为：

    'status': '405 Method Not Allowed',


我们再一次深入 `web.application.request`, 看一看具体实现。

```python
    def request(self, localpart='/', method='GET', data=None,
                host="0.0.0.0:8080", headers=None, https=False, **kw):
        
        ...

        env = dict(
            env, 
            HTTP_HOST=host, 
            REQUEST_METHOD=method, 
            PATH_INFO=path, 
            QUERY_STRING=query, 
            HTTPS=str(https)
            )

        ...

        response = web.storage()
        def start_response(status, headers):
            response.status = status
            response.headers = dict(headers)
            response.header_items = headers

        response.data = "".join(self.wsgifunc()(env, start_response))

        return response
```

我们只保留了关键代码，可以看出，method 首先被设置在了 `env` 中，然后，
通过 

```python
    self.wsgifunc()(env, start_response))
```

`start_response` 被 `wsgifunc()` 返回的函数调用，`response` 的值被设置。 如果阅读了前面推荐的几篇关于 `WSGI` 的文章，对这里的 `self.wsgifunc()` 和 `start_response`应该不会感觉到陌生。

好，我们继续深入 `web.application.wsgifunc()`

```python
    def wsgifunc(self, *middleware):
        
        ...

        def wsgi(env, start_resp):
            # clear threadlocal to avoid inteference of previous requests
            self._cleanup()

            self.load(env)
            try:
                # allow uppercase methods only
                if web.ctx.method.upper() != web.ctx.method:
                    raise web.nomethod()

                result = self.handle_with_processors()
                if is_generator(result):
                    result = peep(result)
                else:
                    result = [result]
            except web.HTTPError, e:
                result = [e.data]


            result = web.safestr(iter(result))

            status, headers = web.ctx.status, web.ctx.headers
            start_resp(status, headers)
            
            def cleanup():
                self._cleanup()
                yield '' # force this function to be a generator

            return itertools.chain(result, cleanup())

        for m in middleware: 
            wsgi = m(wsgi)

        return wsgi
```

从上述代码可以看出，返回的函数就是 `wsgi`, `wsgi` 在返回前，可能
会使用 `middleware` 进行一层一层的包装。不过这里，我们只看 `wsgi`
的代码就可以了。

关键的两行在：

```python
status, headers = web.ctx.status, web.ctx.headers
start_resp(status, headers)
```

这里，`web.application.requests.start_response` 函数 被调用，并设置好
`status` 和 `headers`，二者的值从 `web.ctx` 中取得。所以，下一部，我们
关心 `web.ctx` 值的来源。因为 `method` 的信息在 `env` 中，所以 
`web.ctx` 的值一定会受 `env` 的影响，所以在 `wsgi` 中要找到 `env`是如何
被使用的。

```python
self.load(env)
try:
    # allow uppercase methods only
    if web.ctx.method.upper() != web.ctx.method:
        raise web.nomethod()
```

我们猜想 `self.load(env)` 对 `web.ctx` 进行了设置，然后后面的 `try`
对 `web.ctx` 检查，这里对 `upper`的检查，让我们想到了，原来一开始的
`testUppercaseMethods` 是在测试 `method` 名字是不是全是大写。。。

于是在 `raise web.nomethod()` 前加一个 `print` ，测试一下是不是执行到
了这里，发现还真是。那么我们再深入 `load` ，看看如何根据 `env` 设置
`web.ctx`。

```python
def load(self, env):
    """Initializes ctx using env."""
    ctx = web.ctx
    ctx.clear()
    ctx.status = '200 OK'

    ...

    ctx.method = env.get('REQUEST_METHOD')

    ...
```

`load` 函数会根据 `env` 设置 `web.ctx` ，但是只是简单赋值，没有对
`method` 方法的检查。于是，我明白了，原来是在 `raise web.nomethod()`中对
`status` 进行了设置。

在 `raise` 之前打印 `web.ctx.status` 的值，果然是 `200 OK`。那我们再进入
`web.nomethod` 看看吧。

这个函数在 `webapi.py` 中，进入文件一看，果然是对许多状态的定义。
找到 `nomethod` ，发现指向 `NoMethod`，下面看 `NoMethod`:

```python
class NoMethod(HTTPError):
    """A `405 Method Not Allowed` error."""
    def __init__(self, cls=None):
        status = '405 Method Not Allowed'
        headers = {}
        headers['Content-Type'] = 'text/html'
        
        methods = ['GET', 'HEAD', 'POST', 'PUT', 'DELETE']
        if cls:
            methods = [method for method in methods if hasattr(cls, method)]

        headers['Allow'] = ', '.join(methods)
        data = None
        HTTPError.__init__(self, status, headers, data)
```

发现还真是对错误号 `405` 的处理。 设置好status, 最终的处理
发给了 `HTTPError`，所以我们再进入 `HTTPError`。

```python
class HTTPError(Exception):
    def __init__(self, status, headers={}, data=""):
        ctx.status = status
        for k, v in headers.items():
            header(k, v)
        self.data = data
        Exception.__init__(self, status)
```

这里终于发现了对 `ctx.status` 的设置。这就是在这里我们知道了下面这两
行关键代码的最终来源。

```python
status, headers = web.ctx.status, web.ctx.headers
start_resp(status, headers)
```

前者设置 `status` 和 `headers`，后者调用 `requests` 函数中的 `start_response` 对 `response` 的状态进行了设置。最后返回给最一开始 `testUppercaseMethods` 中的 `response`。