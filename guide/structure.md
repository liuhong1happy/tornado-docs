# Tornado web应用结构

一个Tornado web应用存在一个或者多个RequestHandler之类，一个Application对象（负责路由到特定Handler），和一个main函数来启动server。

一个最小化的HelloWolrd示例其代码如下：

	from tornado.ioloop import IOLoop
	from tornado.web import RequestHandler, Application, url
	
	class HelloHandler(RequestHandler):
	    def get(self):
	        self.write("Hello, world")
	
	def make_app():
	    return Application([
	        url(r"/", HelloHandler),
	        ])
	
	def main():
	    app = make_app()
	    app.listen(8888)
	    IOLoop.current().start()

## Application对象

Application对象是可靠的全局配置，包括了路由表（映射所有的RequestHandler）。

路由表是一个URLSpec对象的数组，每一个对象都至少包含一个正则表达式和一个handler类。排序事项，第一匹配原则是被使用的。如果正则表达式包含捕获groups，这些groups是路径参数而且将传递给Handler的HTTP方法。如果一个字典传递作为URLSpec的第三个元素，它将提供初始化参数（这些参数将传递给RequestHandler.initialize）。最后，URLSpec有一个名称，这会被RequestHandler.reverse_url所使用。

例如，在如下不完整代码，根路径`/`将映射到MainHandler和 /story/（允许传递数量）将会映射到StoryHandler。数量将被作为字符串传递给StoryHandler.get。

	class MainHandler(RequestHandler):
	    def get(self):
	        self.write('<a href="%s">link to story 1</a>' %
	                   self.reverse_url("story", "1"))
	
	class StoryHandler(RequestHandler):
	    def initialize(self, db):
	        self.db = db
	
	    def get(self, story_id):
	        self.write("this is story %s" % story_id)
	
	app = Application([
	    url(r"/", MainHandler),
	    url(r"/story/([0-9]+)", StoryHandler, dict(db=db), name="story")
	    ])

Application构造函数需要传递许多关键参数，以便是用来自定义应用的行为以及启动可选的特性，查看[`Application.settings`](http://www.tornadoweb.org/en/stable/web.html#tornado.web.Application.settings)已了解更多。

