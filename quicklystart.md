# 快速开始

![images/tornado.png](images/tornado.png)

[Tornado](http://www.tornadoweb.org/)是一个Python web框架和异步网络库，起初在[FriendFeed](http://friendfeed.com/)上开发。通过使用非阻塞网络I/O，tornado可以支撑上万的开放链接，能支持长连接，WebSockets和其它要求长实时链接的应用。

## 相关链接

- 下载4.2.1版本：[tornado-4.2.1.tar.gz](https://pypi.python.org/packages/source/t/tornado/tornado-4.2.1.tar.gz)([版本说明](http://www.tornadoweb.org/en/stable/releases.html))
- 源代码([github](https://github.com/tornadoweb/tornado))
- 邮件列表：[讨论](http://groups.google.com/group/python-tornado)或者[公告](http://groups.google.com/group/python-tornado-announce)
- [Stack Overflow](http://stackoverflow.com/questions/tagged/tornado)
- [Wiki](https://github.com/tornadoweb/tornado/wiki/Links)

## Hello, world

这里是一个简单的“Hello, world”示例web应用。

	import tornado.ioloop
	import tornado.web
	
	class MainHandler(tornado.web.RequestHandler):
	    def get(self):
	        self.write("Hello, world")
	
	application = tornado.web.Application([
	    (r"/", MainHandler),
	])
	
	if __name__ == "__main__":
	    application.listen(8888)
	    tornado.ioloop.IOLoop.current().start()

这个例子没有使用任何的Tornado的异步特性，了解详情可以参看[Simple Chat Room](https://github.com/tornadoweb/tornado/tree/stable/demos/chat)。

## 安装

**自动安装**

	pip install tornado

Tornado在PyPI中有列举，可以通过使用pip或者easy_install。注意源代码中包含了示例应用可能不会出现在这种安装方式的源码中，因此你可以通过拷贝源码的手动安装的方式安装tornado。

**手动安装**：下载[tornado-4.2.1.tar.gz](https://pypi.python.org/packages/source/t/tornado/tornado-4.2.1.tar.gz)

	tar xvzf tornado-4.2.1.tar.gz
	cd tornado-4.2.1
	python setup.py build
	sudo python setup.py install

Tornado的源代码是放在[Github](https://github.com/tornadoweb/tornado)上的。

**前提条件**：Tornado运行在Python 2.6，2.7，3.2，3.3，和3.4。所有的版本都依赖于[certifi](https://pypi.python.org/pypi/certifi)，在Python 2中这还依赖于[backports.ssl_match_hostname](https://pypi.python.org/pypi/backports.ssl_match_hostname)。这些当你使用pip或者easy_install安装tornado时会自动安装。某些Tornado特性将要求下来可选的库：

- [unitest2](https://pypi.python.org/pypi/unittest2)是用来在Python2.6上运行Tornado测试单元组件的（最新的Python版本不再需要）。
- [concurrent.futures](https://pypi.python.org/pypi/futures)是被推荐在Tornado的线程池并可以开启[`ThreadedResolver`](http://www.tornadoweb.org/en/stable/netutil.html#tornado.netutil.ThreadedResolver)用法。这仅仅在Python2中需要；Python 3已经包括了这个标准库。
- [pycurl](http://pycurl.sourceforge.net/)是在`tornado.curl_httpclient`中可选使用的。这要求Libcurl版本7.18.2或者更高;推荐使用版本7.21.1或者更高。
- [Twisted](http://www.twistedmatrix.com/)伴随[`tornado.platform.twisted`](http://www.tornadoweb.org/en/stable/twisted.html#module-tornado.platform.twisted)被使用。
- [pycares](https://pypi.python.org/pypi/pycares)是当线程不适用的情况下一种可选的非阻塞DNS解决方案。
- [Monotime](https://pypi.python.org/pypi/Monotime)添加对monotonic clock的支持（**译者注：monotonic clock字面意思是单调时钟，其含义是机器启动后的时间，这个时间是递增的**），当环境中时间频繁被调整时提供了一个可靠性。

**开发、部署环境**：Tornado运行在所有类Unix的平台上，因为运行在Linux（带有`epoll`）和BSD（带有`kqueue`）能得到最好性能和可伸缩性是被推荐的生产部署环境（尽管Mac OS X是派生自BSD也支持kqueue,其网络性能是有瓶颈的，因此仅作为开发环境使用）。Tornado也将运行在Windows，因为其配置文件不被官方支持，仅推荐作为开发环境使用。