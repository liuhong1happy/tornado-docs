# 概述

[Tornado](http://www.tornadoweb.org/)是一个Python web框架和异步网络库，起初在[FriendFeed](http://friendfeed.com/)上开发。通过使用非阻塞网络I/O，tornado可以支撑上万的开放链接，能支持长连接，WebSockets和其它要求长实时链接的应用。

Tornado可以大致分成4个部分：

- Web框架 (包括创建web应用的RequestHandler类, 和许多支持的类)。
- HTTP客户端和服务端实现 ([`HTTPServer`](http://www.tornadoweb.org/en/stable/httpserver.html#tornado.httpserver.HTTPServer) 和 [`AsyncHTTPClient`](http://www.tornadoweb.org/en/stable/httpclient.html#tornado.httpclient.AsyncHTTPClient))。
- 异步网络库 ([`IOLoop`](http://www.tornadoweb.org/en/stable/ioloop.html#tornado.ioloop.IOLoop) 和 [`IOStream`](http://www.tornadoweb.org/en/stable/iostream.html#tornado.iostream.IOStream)), 为HTTP部分提供构建模块，也可以被用来实现其它协议。
- 协同程序库([`tornado.gen`](http://www.tornadoweb.org/en/stable/gen.html#module-tornado.gen)) 这将允许异步代码不用再写成链式回调的方式。

Tornado web框架和 HTTP server 一起为[WSGI](http://www.python.org/dev/peps/pep-3333/)提供一个全栈式选择。 使用Tornado web框架在WSGI容器 ([`WSGIAdapter`](http://www.tornadoweb.org/en/stable/wsgi.html#tornado.wsgi.WSGIAdapter)), 或者是使用Tornado HTTP server 作为另外的WSGI框架([`WSGIContainer`](http://www.tornadoweb.org/en/stable/wsgi.html#tornado.wsgi.WSGIContainer)), 这样的结合方式是有局限性的；为了充分利用Tornado，你应该联合使用Tornado的web框架和HTTP server。