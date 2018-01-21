[Requests][1] is one of the, if not the most well-known Python third-party library for Python programmers. With its simple API and high performance, people tend to use requests rather than urllib2 in standard library for HTTP requests. However, people who use requests every day may not know the internals, and today I want to explain two concepts: `pool_connections` and `pool_maxsize`.

Let's start with `Session`:

```python
import requests

s = requests.Session()
s.get('https://www.google.com')
```
It's pretty simple. You probably know requests' `Session` persists cookie. Cool. But do you know `Session` has a [`mount`][3] method?
> `mount(prefix, adapter)`  
Registers a connection adapter to a prefix.  
Adapters are sorted in descending order by key length.

No? Well, in fact you've already used this method when you [initialize a `Session` object][4]:

```python
class Session(SessionRedirectMixin):
    
    def __init__(self):
        ...
        # Default connection adapters.
        self.adapters = OrderedDict()
        self.mount('https://', HTTPAdapter())
        self.mount('http://', HTTPAdapter())
```
Now comes the interesting part. If you've read Ian Cordasco's article [Retries in Requests][5], you should know that `HTTPAdapter` can be used to provide retry functionality. But what is an `HTTPAdapter` really? Quoted from [doc][6]:

>`class requests.adapters.HTTPAdapter(pool_connections=10, pool_maxsize=10, max_retries=0, pool_block=False)`

>The built-in HTTP Adapter for urllib3.

>Provides a general-case interface for Requests sessions to contact HTTP and HTTPS urls by implementing the Transport Adapter interface. This class will usually be created by the Session class under the covers.

>Parameters:	
* `pool_connections` – The number of urllib3 connection pools to cache.
* `pool_maxsize` – The maximum number of connections to save in the pool.
* `max_retries(int)` – The maximum number of retries each connection should attempt. Note, this applies only to failed DNS lookups, socket connections and connection timeouts, never to requests where data has made it to the server. By default, Requests does not retry failed connections. If you need granular control over the conditions under which we retry a request, import urllib3’s Retry class and pass that instead.
* `pool_block` – Whether the connection pool should block for connections.
Usage:

```python
>>> import requests
>>> s = requests.Session()
>>> a = requests.adapters.HTTPAdapter(max_retries=3)
>>> s.mount('http://', a)
```

If the above documentation confuses you, here's my explanation: what HTTP Adapter does is simply **providing different configurations for different requests according to target url**. Remember the code above?

```python
self.mount('https://', HTTPAdapter())
self.mount('http://', HTTPAdapter())
```
It creates two `HTTPAdapter` objects with the default argument `pool_connections=10, pool_maxsize=10, max_retries=0, pool_block=False`, and mount to `https://` and `http://` respectively, which means configuration of the first `HTTPAdapter()` will be used if you try to send a request to `http://xxx`, and the second `HTTPAdapter()` for requests to `https://xxx`. Though in this case two configurations are the same, requests to `http` and `https` are still handled separately. We'll see what it means later.

As I said, the main purpose of this article is to explain `pool_connections` and `pool_maxsize`.

First let's look at `pool_connections`. Yesterday I raised a [question][7] on stackoverflow cause I'm not sure if my understanding is correct, the answer eliminates my uncertainty. HTTP, as we all know, is based on TCP protocol. An HTTP connection is also a TCP connection, which is  identified by a tuple of **five** values:

```
(<protocol>, <src addr>, <src port>, <dest addr>, <dest port>)
```
Say you've established an HTTP/TCP connection with `www.example.com`, assume the server supports `Keep-Alive`, next time you send request to `www.example.com/a` or `www.example.com/b`, you could just use the same connection cause none of the five values change. In fact, [requests' Session automatically does this for you][8] and will reuse connections as long as it can.

The question is, what determines if you can reuse old connection or not? Yes, `pool_connections`! 
> pool_connections – The number of urllib3 connection pools to cache.

I know, I know, I don't want to brought so many terminologies either, this is the last one, I promise. For easy understanding, **one connection pool corresponds to one host**, that's what it is.

Here's an example(unrelated lines are ignored):

```python
s = requests.Session()
s.mount('https://', HTTPAdapter(pool_connections=1))
s.get('https://www.baidu.com')
s.get('https://www.zhihu.com')
s.get('https://www.baidu.com')

"""output
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.baidu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 None
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.zhihu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 2621
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.baidu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 None
"""
```
`HTTPAdapter(pool_connections=1)` is mounted to `https://`, which means only one connection pool persists at a time. After calling `s.get('https://www.baidu.com')`, the cached connection pool is `connectionpool('https://www.baidu.com')`. Now `s.get('https://www.zhihu.com')` came, and the session found that it cannot use the previously cached connection because it's not the same host(one connection pool corresponds to one host, remember?). Therefore the session had to create a new connection pool, or connection if you would like. Since `pool_connections=1`, session cannot hold two connection pools at the same time, thus it abandoned the old one which is `connectionpool('https://www.baidu.com')` and kept the new one which is `connectionpool('https://www.zhihu.com')`. Next `get` is the same. This is why we see three `Starting new HTTPS connection` in log.

What if we set `pool_connections` to 2:

```python
s = requests.Session()
s.mount('https://', HTTPAdapter(pool_connections=2))
s.get('https://www.baidu.com')
s.get('https://www.zhihu.com')
s.get('https://www.baidu.com')
"""output
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.baidu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 None
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.zhihu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 2623
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 None
"""
```
Great, now we only created connection twice and saved one connection establishing time.

Finally, `pool_maxsize`.

First and foremost, you should be caring about `pool_maxsize` only if you use `Session` in a **multithreaded** environment, like making concurrent requests from multiple threads using the **same** `Session`.

Actually, `pool_maxsize` is an argument for initializing urllib3's [`HTTPConnectionPool`][9], which is exactly the connection pool we mentioned above.
`HTTPConnectionPool` is a container for a collection of connections to a specific host, and `pool_maxsize` is the number of connections to save that can be reused. If you're running your code in one thread, it's neither possible nor needed to create multiple connections to the same host, cause requests library is blocking, so that HTTP request are always sent one after another.

Things are different if there are multiple threads.

```python
def thread_get(url):
    s.get(url)

s = requests.Session()
s.mount('https://', HTTPAdapter(pool_connections=1, pool_maxsize=2))
t1 = Thread(target=thread_get, args=('https://www.zhihu.com',))
t2 = Thread(target=thread_get, args=('https://www.zhihu.com/question/36612174',))
t1.start();t2.start()
t1.join();t2.join()
t3 = Thread(target=thread_get, args=('https://www.zhihu.com/question/39420364',))
t4 = Thread(target=thread_get, args=('https://www.zhihu.com/question/21362402',))
t3.start();t4.start()
"""output
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.zhihu.com
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (2): www.zhihu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/36612174 HTTP/1.1" 200 21906
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 2606
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/21362402 HTTP/1.1" 200 57556
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/39420364 HTTP/1.1" 200 28739
"""
```
See? It established two connections for the same host `www.zhihu.com`, like I said, this can only happen in a multithreaded environment.
In this case, we create a connectionpool with `pool_maxsize=2`, and there're no more than two connections at the same time, so it's enough.
We can see that requests from `t3` and `t4` did not create new connections, they reused the old ones.

What if there's not enough size?

```python
s = requests.Session()
s.mount('https://', HTTPAdapter(pool_connections=1, pool_maxsize=1))
t1 = Thread(target=thread_get, args=('https://www.zhihu.com',))
t2 = Thread(target=thread_get, args=('https://www.zhihu.com/question/36612174',))
t1.start()
t2.start()
t1.join();t2.join()
t3 = Thread(target=thread_get, args=('https://www.zhihu.com/question/39420364',))
t4 = Thread(target=thread_get, args=('https://www.zhihu.com/question/21362402',))
t3.start();t4.start()
t3.join();t4.join()
"""output
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.zhihu.com
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (2): www.zhihu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/36612174 HTTP/1.1" 200 21906
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 2606
WARNING:requests.packages.urllib3.connectionpool:Connection pool is full, discarding connection: www.zhihu.com
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (3): www.zhihu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/39420364 HTTP/1.1" 200 28739
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/21362402 HTTP/1.1" 200 57556
WARNING:requests.packages.urllib3.connectionpool:Connection pool is full, discarding connection: www.zhihu.com
"""
```
Now, `pool_maxsize=1`，warning came as expected:

```
Connection pool is full, discarding connection: www.zhihu.com
```
We can also noticed that since only one connection can be saved in this pool, a new connection is created again for `t3` or `t4`. Obviously this is very inefficient. That's why in urllib3's documentation it says:
> If you’re planning on using such a pool in a multithreaded environment, you should set the maxsize of the pool to a higher number, such as the number of threads.

Last but not least, `HTTPAdapter` instances mounted to different prefixes are **independent**.

```python
s = requests.Session()
s.mount('https://', HTTPAdapter(pool_connections=1, pool_maxsize=2))
s.mount('https://baidu.com', HTTPAdapter(pool_connections=1, pool_maxsize=1))
t1 = Thread(target=thread_get, args=('https://www.zhihu.com',))
t2 =Thread(target=thread_get, args=('https://www.zhihu.com/question/36612174',))
t1.start();t2.start()
t1.join();t2.join()
t3 = Thread(target=thread_get, args=('https://www.zhihu.com/question/39420364',))
t4 = Thread(target=thread_get, args=('https://www.zhihu.com/question/21362402',))
t3.start();t4.start()
t3.join();t4.join()
"""output
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (1): www.zhihu.com
INFO:requests.packages.urllib3.connectionpool:Starting new HTTPS connection (2): www.zhihu.com
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/36612174 HTTP/1.1" 200 21906
DEBUG:requests.packages.urllib3.connectionpool:"GET / HTTP/1.1" 200 2623
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/39420364 HTTP/1.1" 200 28739
DEBUG:requests.packages.urllib3.connectionpool:"GET /question/21362402 HTTP/1.1" 200 57669
"""
```
The above code is easy to understand so I'm not gonna explain.

I guess that's all. Hope this article help you understand requests better. BTW I created a gist [here][11] which contains all of the testing code used in this article. Just download and play with it :)

## Appendix
1. For https, requests uses urllib3's [HTTPSConnectionPool][10], but it's pretty much the same as HTTPConnectionPool so I didn't differeniate them in this article.
2. `Session`'s `mount` method ensures the longest prefix gets matched first. Its implementation is pretty interesting so I posted it here.

    ```python
def mount(self, prefix, adapter):
    """Registers a connection adapter to a prefix.
    Adapters are sorted in descending order by key length."""
    self.adapters[prefix] = adapter
    keys_to_move = [k for k in self.adapters if len(k) < len(prefix)]
    for key in keys_to_move:
        self.adapters[key] = self.adapters.pop(key)
    ```
Note that `self.adapters` is an `OrderedDict`.
3. A more advanced config option `pool_block`  
Notice that when creating an adapter, we didn't introduce the last parameter `pool_block`
    ```
    HTTPAdapter(pool_connections=10, pool_maxsize=10, max_retries=0, pool_block=False)
    ```
Setting it to `True` is equivalent to setting `block=True` when creating a [urllib3.connectionpool.HTTPConnectionPool][12], the effect is this, quoted from urllib3's doc:  
    >If set to True, no more than maxsize connections will be used at a time. When no free connections are available, the call will block until a connection has been released. This is a useful side effect for particular multithreaded situations where one does not want to use more than maxsize connections per host to prevent flooding.

    But actually, requests doesn't set a timeout([https://github.com/shazow/urllib3/issues/1101](https://github.com/shazow/urllib3/issues/1101)), which means that if there's no free connection, an `EmptyPoolError` will be raised immediately.

4. 刚发现，我就说为什么白天晕乎乎的，原来是吃了白加黑的黑片\_(:3」∠)\_

[1]: http://docs.python-requests.org/en/latest/
[2]: http://docs.python-requests.org/en/latest/user/advanced/#session-objects
[3]: http://docs.python-requests.org/en/latest/api/#requests.Session.mount
[4]: https://github.com/kennethreitz/requests/blob/master/requests/sessions.py#L340-L341
[5]: http://www.coglib.com/~icordasc/blog/2014/12/retries-in-requests.html
[6]: http://docs.python-requests.org/en/latest/api/#requests.adapters.HTTPAdapter
[7]: http://stackoverflow.com/questions/34837026/whats-the-meaning-of-pool-connections-in-requests-adapters-httpadapter
[8]: http://docs.python-requests.org/en/latest/user/advanced/#keep-alive
[9]: http://urllib3.readthedocs.org/en/latest/pools.html#module-urllib3.connectionpool
[10]: http://urllib3.readthedocs.org/en/latest/pools.html#urllib3.connectionpool.HTTPSConnectionPool
[11]: https://gist.github.com/laike9m/ead19c65a416c7022c00
[12]: https://urllib3.readthedocs.io/en/1.4/pools.html#urllib3.connectionpool.HTTPConnectionPool
[13]: https://github.com/shazow/urllib3/blob/1191719a9ce99e3b87f280f92ffede99929bc9dd/urllib3/connectionpool.py#L236
