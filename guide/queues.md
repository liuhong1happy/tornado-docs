# 队列示例---一个并发网络爬虫

Tornado的tornado.queues模块实现了一个异步的生产者和消费者模式的协同程序，类似于Python标准库实现的queue模块。

一个协同程序，yield方式调用Queue.get的值做短暂的停顿。如果队列中超出了最大数量的容量，协同程序yield方式调用Queue.put做短暂停顿，知道有可容纳的空间。

一个Queue维持一定量未完成的任务，这个数量是开始的时候是零。put会增加数量，task_done会较少其数量。

本网络爬虫示例，队列开始容纳base_url。当一个worker抓取页面的链接然后放所有新的链接到队列中，然后调用task_done来削减这个数量。最终，worker抓取的页面的所有URL最后都会被处理，有一些没有完成的就留在了队列中。直到worker调用task_done削减数量到零。主协同程序，等待join，将不会停止或者将会结束。

	import HTMLParser
	import time
	import urlparse
	from datetime import timedelta
	
	from tornado import httpclient, gen, ioloop, queues
	
	base_url = 'http://www.tornadoweb.org/en/stable/'
	concurrency = 10
	
	
	@gen.coroutine
	def get_links_from_url(url):
	    """Download the page at `url` and parse it for links.
	
	    Returned links have had the fragment after `#` removed, and have been made
	    absolute so, e.g. the URL 'gen.html#tornado.gen.coroutine' becomes
	    'http://www.tornadoweb.org/en/stable/gen.html'.
	    """
	    try:
	        response = yield httpclient.AsyncHTTPClient().fetch(url)
	        print('fetched %s' % url)
	        urls = [urlparse.urljoin(url, remove_fragment(new_url))
	                for new_url in get_links(response.body)]
	    except Exception as e:
	        print('Exception: %s %s' % (e, url))
	        raise gen.Return([])
	
	    raise gen.Return(urls)
	
	
	def remove_fragment(url):
	    scheme, netloc, url, params, query, fragment = urlparse.urlparse(url)
	    return urlparse.urlunparse((scheme, netloc, url, params, query, ''))
	
	
	def get_links(html):
	    class URLSeeker(HTMLParser.HTMLParser):
	        def __init__(self):
	            HTMLParser.HTMLParser.__init__(self)
	            self.urls = []
	
	        def handle_starttag(self, tag, attrs):
	            href = dict(attrs).get('href')
	            if href and tag == 'a':
	                self.urls.append(href)
	
	    url_seeker = URLSeeker()
	    url_seeker.feed(html)
	    return url_seeker.urls
	
	
	@gen.coroutine
	def main():
	    q = queues.Queue()
	    start = time.time()
	    fetching, fetched = set(), set()
	
	    @gen.coroutine
	    def fetch_url():
	        current_url = yield q.get()
	        try:
	            if current_url in fetching:
	                return
	
	            print('fetching %s' % current_url)
	            fetching.add(current_url)
	            urls = yield get_links_from_url(current_url)
	            fetched.add(current_url)
	
	            for new_url in urls:
	                # Only follow links beneath the base URL
	                if new_url.startswith(base_url):
	                    yield q.put(new_url)
	
	        finally:
	            q.task_done()
	
	    @gen.coroutine
	    def worker():
	        while True:
	            yield fetch_url()
	
	    q.put(base_url)
	
	    # Start workers, then wait for the work queue to be empty.
	    for _ in range(concurrency):
	        worker()
	    yield q.join(timeout=timedelta(seconds=300))
	    assert fetching == fetched
	    print('Done in %d seconds, fetched %s URLs.' % (
	        time.time() - start, len(fetched)))
	
	
	if __name__ == '__main__':
	    import logging
	    logging.basicConfig()
	    io_loop = ioloop.IOLoop.current()
	    io_loop.run_sync(main)