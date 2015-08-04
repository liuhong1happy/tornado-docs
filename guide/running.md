#Tornado 的运行和部署

鉴于Tornado支持它自己的HTTP服务器，因此它的运行和部署相较于基于其他Python web框架的应用而言会有些许不同．你需要写一个main方法来取代wsgi容器的配置工作．

```Python
def main():
    app = make_app()
    app.listen(8888)
    IOLoop.current().start()

if __name__ == '__main__':
    main()
```

配置你的操作系统或者进程管理器来启动这个服务器．请注意务必增加每个进程允许打开的最大文件数量（这样可以避免＂Too many open files＂错误的发生）．可以通过`ulimit`命令来增加限额（例如设置成50000个），更改 /etc/security/limits.conf 文件或者在你的 supervisord 配置中设置 minfds 选项．


#进程和端口

因为Python GIL（全局解释器锁）的限制，请务必运行多个Python进程来榨取多核CPU机器的资源．通常来讲，有几个核心就运行几个进程．


Tornado包含了一个内置的模式用来一次性开启多个进程．这需要你稍微改造一下标准的main函数：

```Python
def main():
    app = make_app()
    server = tornado.httpserver.HTTPServer(app)
    server.bind(8888)
    server.start(0)  # forks one process per cpu
    IOLoop.current().start()
```


这是开启多进程并且让它们共享一个端口的最简单的方法，尽管这仍旧会有一些限制．第一点，每一个子进程都会有一个自己的IOLoop，所以在分支前（开启子进程前），避免与全局IOLoop交互（哪怕是间接的交互）就显得尤为重要．第二点，在这种模式下要实现免停机更新（zero-downtime updates）变得非常困难．最后一点，由于所有的进程共享一个端口，单独监听某一个进程就几乎成了不可能完成的任务了．

此外还有另外一种更加复杂一些的部署方式，这种方式建议单独开启每个进程，并且对每个独立进程进行监听．supervisord　的　“process groups”　（监听组）功能可以很方便的对其进行实现．当每个进程使用不同端口的时候，我们就需要使用 HAProxy 或者 nginx 来充当外部的负载均衡服务器为访问者推荐独立的访问地址．

#运行在在负载均衡的后端


当运行在负载均衡后端时，建议向HTTP服务器的构造器传递`xheader=True`（译者注：http1.1和http1.0的常用标准首部或非标准首部里都没有这么个元素，估计是Tornado HTTPServer自己搞的用来开关代理服务器首部识别的阀值）．如此一来，Tornado就会明白需要使用像X-Real-IP这样的HTTP首部来判别用户IP地址，而不是使用IP Group里跟踪到的均衡器的地址．


下面是一个nginx准系统的配置文件，此文件跟我们用在FriendFeed上的极为相似．它假设nginx和Tornado运行在同一台机器上，并且有四个Tornado服务分别运行在8000-8003端口上．

```
user nginx;
worker_processes 1;

error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
}

http {
    # Enumerate all the Tornado servers here
    upstream frontends {
        server 127.0.0.1:8000;
        server 127.0.0.1:8001;
        server 127.0.0.1:8002;
        server 127.0.0.1:8003;
    }

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;

    keepalive_timeout 65;
    proxy_read_timeout 200;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    gzip on;
    gzip_min_length 1000;
    gzip_proxied any;
    gzip_types text/plain text/html text/css text/xml
               application/x-javascript application/xml
               application/atom+xml text/javascript;

    # Only retry if there was a communication error, not a timeout
    # on the Tornado server (to avoid propagating "queries of death"
    # to all frontends)
    proxy_next_upstream error;

    server {
        listen 80;

        # Allow file uploads
        client_max_body_size 50M;

        location ^~ /static/ {
            root /var/www;
            if ($query_string) {
                expires max;
            }
        }
        location = /favicon.ico {
            rewrite (.*) /static/favicon.ico;
        }
        location = /robots.txt {
            rewrite (.*) /static/robots.txt;
        }

        location / {
            proxy_pass_header Server;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_pass http://frontends;
        }
    }
}
```

#静态文件和积极的文件缓存

你可以在你的应用代码中通过指定静态文件路径来使用Tornado服务静态文件：

```Python
settings = {
    "static_path": os.path.join(os.path.dirname(__file__), "static"),
    "cookie_secret": "__TODO:_GENERATE_YOUR_OWN_RANDOM_VALUE_HERE__",
    "login_url": "/login",
    "xsrf_cookies": True,
}
application = tornado.web.Application([
    (r"/", MainHandler),
    (r"/login", LoginHandler),
    (r"/(apple-touch-icon\.png)", tornado.web.StaticFileHandler,
     dict(path=settings['static_path'])),
], **settings)
```

这项设置会将所有以　/static/　为URL路径的请求定位到特定的静态目录内（本例为"static"目录）．例如： http://localhost:8888/static/foo.png　请求会定位到特定的静态目录内的 foo.png 文件．

在以上设置中，我们在Tornado中明确将apple-touch-icon.png文件配置到了StaticFileHandler的跟路径位置，尽管他的物理位置位于"static"目录中．（正则表达式中的捕获组用来告诉StaticFileHandler请求文件的文件名；重访捕获组会将方法参数传递给handlers．）你可以对sitemap.xml文件做与apple-touch-icon.png同样的事来将它的访问路径定位在网站根目录上．当然，你也可以避免在你的HTML代码中使用 `<link />` 标记来伪造apple-touch-icon.png跟路径．
译者注：StaticFileHandler为Tornado内置类，用来实例化静态对象．

为了达到提升性能的目的，通常来说让浏览器积极地缓存静态文件是一个好主意，这样浏览器就不需要发送　If-Modified-Since 或者 Etag 这种阻塞页面渲染的请求．Tornado支持这种开箱即用的静态内容版本．

想要利用这一特性，请在模板中使用`static_url`方法，而不是直接将静态文件地址键入到其中．

```Html
<html>
   <head>
      <title>FriendFeed - {{ _("Home") }}</title>
   </head>
   <body>
     <div><img src="{{ static_url("images/logo.png") }}"/></div>
   </body>
 </html>
```


`static_url()`函数会将其参数转换成　URI　中的相对路径，如　/static/images/logo.png?v=aae54．`v`参数是 logo.png 的哈希值，它的意义在于让Tornado服务器将缓存首部发送给用户浏览器，使得浏览器无限期缓存其内容．

由于`v`参数的值是基于文件内容的，如果你更新了文件并且重启了服务器，服务器就会发送一个新的`v`值给用户浏览器，这样浏览器就会自动的获取新文件．如果文件内容没有发生变化，浏览器就会继续使用已经缓存的本地拷贝，而不会向服务器检查文件更新，这样就可以显著提高渲染性能．

在生产环境中，或许你想要将静态文件服务交给如 nginx 这种优化能力更佳的静态文件服务器．我们可以配置大多数web服务器来识别出static_url()的version标签，从而设置缓存头部．一下是我们用在FriendFeed中的nginx配置的相关部分．

```
location /static/ {
    root /var/friendfeed/static;
    if ($query_string) {
        expires max;
    }
 }
```

#调试模式和自动加载

如果你在Application的构造器中设置了`debug=True`的话，应用就会在调试／开发模式下运行．在此模式下，会有几个非常方便的特性用于开发过程．（这些特性也可以通过单独的标记进行设置，如果`debug=True`和这些特性的独立标记两者同时存在，那么独立标记优先权更大）


1. `autoreload=True`: 应用会监测其源文件中的代码变动，如果有变动发生，应用会重新启动．这将会大大减少开发过程中的手工重启次数．尽管如此，某些失败情况（例如在引用时的语法错误）仍会导致服务器宕机，这种情况调试模式也无法挽回．
2. `compiled_template_cache=False`: 模板将不会被缓存．
3. `static_hash_cache=False`: 静态文件的哈希值（用于static_url函数）将不会被缓存．（译者注：官档解释为，如为False，静态文件URL每次都会被重新计算．换成人话就是静态文件不被缓存．）
4. `serve_traceback=True`: 如果RequestHandler中有异常未被捕获，那么一个包含堆栈追踪信息的错误页就会生成．


自动重载模式与HTTP服务器的多进程模式互不兼容．如果你正在使用自动重载模式，请勿在 HTTPServer.start 中设置除了`1`之外的任何其他值（或者调用tornado.process.fork_processes）．

调试模式的自动重载特性可以利用`tornado.autoreload`，当作一个独立的模块来使用．两者亦可以结合使用以针对语法错误检查提供额外的鲁棒性：在应用内设置`autoreload=True`来发现运行时代码的变更，并且使用`python -m tornado.autoreload myserver.py`来启动应用，捕捉启动时的语法和其他错误．

重载行为会失去Python解释器命令行的参数（例如 -u）．因为重载行为使用 sys.executable 和 sys.argv 重新执行Python命令．另外，如果更改这些参数的话将会引起重载行为的异常．

在某些平台上（例如Windows和Mac OSX 10.6版本之前），进程不会就地更新(updated “in-place”)，所以当代码变更被检测到的时候，旧的服务器程序会推出，然后新的服务器程序启动．这会对一部分集成开发环境产生困扰．


原文地址：http://www.tornadoweb.org/en/stable/guide/running.html  
翻译：[Mr Ping](http://mr-ping.com)
