# 异步和非阻塞IO

实时的web特性要求为每一个用户提供长链接。在传统的同步web server，这意味着要要为每一个用户提供一个线程，当然每个线程的资源都是昂贵的。

为了减小并发链接的开销，Tornado使用一个单线程的事件循环机制。这意味着所有的应用代码都应该把目标确定为异步和非阻塞，因为仅有这样的操作是有效的。

术语异步和非阻塞非常的亲密，常常可以交换使用，而且它们更擅长于相同的事情。

## 阻塞

一个功能阻塞是它要等待某些返回。一个功能可能阻塞有许多种原因：网络IO，磁盘IO，线程锁等等。事实上，每种功能阻塞，至少有一点点，它是运行的并且会占用CPU资源。

功能可能在某些方面阻塞，也可能在另一些方面不阻塞。例如，tornado.httpclient在默认的结构性阻塞是由于DNS解析而不是其它网络操作。在Tornado上下文，我们通常会讨论关于在网络IO上下文的阻塞，因为所有种类的阻塞都是最小化的。

## 异步

一个异步功能在它结束之前放回，通常造成某些网络放生在后台，在触发某些特定动作执行。异步接口有很多特点：

- Callback参数
- 返回占位符(Future, Promise, Deferred)
- 传递给一个队列
- Callback注册(例如 POSIX signals)

不管使用哪种接口，异步功能默认不同程度的影响它的调用者；这里没有办法让异步能传递给它的回调者以同步的方式。

## 实例

这是一个简单的同步功能:
	
	from tornado.httpclient import HTTPClient
	
	def synchronous_fetch(url):
	    http_client = HTTPClient()
	    response = http_client.fetch(url)
	    return response.body

这里相同的功能重写成异步带callback参数的方式：

	from tornado.httpclient import AsyncHTTPClient
	
	def asynchronous_fetch(url, callback):
	    http_client = AsyncHTTPClient()
	    def handle_response(response):
	        callback(response.body)
	    http_client.fetch(url, callback=handle_response)

然后，我们使用`Future`代替callback：
	
	from tornado.concurrent import Future
	
	def async_fetch_future(url):
	    http_client = AsyncHTTPClient()
	    my_future = Future()
	    fetch_future = http_client.fetch(url)
	    fetch_future.add_done_callback(
	        lambda f: my_future.set_result(f.result()))
	    return my_future

实际Future的写法是相当的复杂，但是Futures在Tornado中推荐使用，因为他们很多优点。

错误处理始终存在，当`Future.result`方法抛出exception ，Futures会让他们都使用协同程序。

协同程序将在接下来的章节中提到。这里协同程序的实例功能，这越来越像同步写法。

	from tornado import gen
	
	@gen.coroutine
	def fetch_coroutine(url):
	    http_client = AsyncHTTPClient()
	    response = yield http_client.fetch(url)
	    raise gen.Return(response.body)

raise gen.Return(response.body)这一句是Python2（Python3.2以下）手工执行的，在其里边的生成器不会返回值。为了克服这样的困难，Tornado协同程序抛出一个特别的类型的Exception叫做Return。协同程序捕获到这个Exception并且允许它返回值。Python3.3及其更高版本，return response.body则可以实现相同的结果。