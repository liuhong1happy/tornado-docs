# 协同程序

协同程序是Tornado推荐使用的写异步代码的最好方式。协同程序通过Python的yield表达式延迟和恢复执行，替换掉链式callback的调用方式。

协同程序几乎类似于同步代码，且仅仅开销一个线程。他们使得并发变得更加容易，通过减少传递上下文时的数量。

示例代码：

	from tornado import gen
	
	@gen.coroutine
	def fetch_coroutine(url):
	    http_client = AsyncHTTPClient()
	    response = yield http_client.fetch(url)
	    # In Python versions prior to 3.3, returning a value from
	    # a generator is not allowed and you must use
	    #   raise gen.Return(response.body)
	    # instead.
	    return response.body

## 原理

包含yield的表达式的函数是一个生成器，所有生成器都是异步的；当他们返回一个生成器对象时才调用，而不是运行完成就调用。@gen.coroutine装饰器是和yield表达式生成器进行沟通的，通过带协同程序的调用函数返回Future。

这里有一个协同程序装饰器的内部循环简化的版本：

	# Simplified inner loop of tornado.gen.Runner
	def run(self):
	    # send(x) makes the current yield return x.
	    # It returns when the next yield is reached
	    future = self.gen.send(self.next)
	    def callback(f):
	        self.next = f.result()
	        self.run()
	    future.add_done_callback(callback)

装饰器从生成器接收Future，等待Future结束，然后取消Future并发送结果作为yield表达式的值。许多异步代码接触过Future类，而是立即传递Future返回给异步函数通过一个yield表达式。

## 协同程序模式

#### 结合callback

为了使用callback实现异步代码而不是Future，需要调用Task。这件添加一个callback参数为你返回的Future，当你调用yield表达式时。

	@gen.coroutine
	def call_task():
	    # Note that there are no parens on some_function.
	    # This will be translated by Task into
	    #   some_function(other_args, callback=callback)
	    yield gen.Task(some_function, other_args)

####　调用阻塞函数

最简单调用阻塞函数的方式是使用一个ThreadPoolExecutor，这将返回Future来兼容协同程序：

	thread_pool = ThreadPoolExecutor(4)
	
	@gen.coroutine
	def call_blocking():
	    yield thread_pool.submit(blocking_func, args)

#### 并行调用

协同程序装饰器能认识数组或者字典对象中各自的Future，并同时等待所有的Futures。

	@gen.coroutine
	def parallel_fetch(url1, url2):
	    resp1, resp2 = yield [http_client.fetch(url1),
	                          http_client.fetch(url2)]
	
	@gen.coroutine
	def parallel_fetch_many(urls):
	    responses = yield [http_client.fetch(url) for url in urls]
	    # responses is a list of HTTPResponses in the same order
	
	@gen.coroutine
	def parallel_fetch_dict(urls):
	    responses = yield {url: http_client.fetch(url)
	                        for url in urls}
	    # responses is a dict {url: HTTPResponse}

#### 交叉存取

有时，保存Future替代一直获取yield的值是非常有用的，因此你可以启动另外的操作持续等待：

	@gen.coroutine
	def get(self):
	    fetch_future = self.fetch_next_chunk()
	    while True:
	        chunk = yield fetch_future
	        if chunk is None: break
	        self.write(chunk)
	        fetch_future = self.fetch_next_chunk()
	        yield self.flush()

####　循环

循环使用coroutines是机智的，当python中没法yield时，for或者while循环中捕获yield的值。

	import motor
	db = motor.MotorClient().test
	
	@gen.coroutine
	def loop_example(collection):
	    cursor = db.collection.find()
	    while (yield cursor.fetch_next):
	        doc = cursor.next_object()