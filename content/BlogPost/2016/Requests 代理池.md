Requests 本身不提供代理池，然而爬数据又要用，所以只能自己搞。其实还挺简单的。我也不知道为什么这么有用的 feature 一直没有被加入。

```python
import requests


class Client:

    def __init__(self):
        self._session = requests.Session()
        self.proxies = None

    def set_proxy_pool(self, proxies, auth=None, https=True):
        """Randomly choose a proxy for every GET/POST request        
        :param proxies: list of proxies, like ["ip1:port1", "ip2:port2"]
        :param auth: if proxy needs auth
        :param https: default is True, pass False if you don't need https proxy
        """
        from random import choice

        if https:
            self.proxies = [{'http': 'http://' + p, 'https': 'https://' + p} for p in proxies]
        else:
            self.proxies = [{'http': 'http://' + p} for p in proxies]

        def get_with_random_proxy(url, **kwargs):
            proxy = choice(self.proxies)
            kwargs['proxies'] = proxy
            if auth:
                kwargs['auth'] = auth
            return self._session.original_get(url, **kwargs)

        def post_with_random_proxy(url, *args, **kwargs):
            proxy = choice(self.proxies)
            kwargs['proxies'] = proxy
            if auth:
                kwargs['auth'] = auth
            return self._session.original_post(url, *args, **kwargs)

        self._session.original_get = self._session.get
        self._session.get = get_with_random_proxy
        self._session.original_post = self._session.post
        self._session.post = post_with_random_proxy

    def remove_proxy_pool(self):
        self.proxies = None
        self._session.get = self._session.original_get
        self._session.post = self._session.original_post
        del self._session.original_get
        del self._session.original_post

    # You can define whatever operations using self._session
```

替换掉 `Session` 原本的 `get` 和 `post` 方法就行了，不会有什么副作用。`class Client` 并不必需，直接操作 `Session` 是一样的。  

可以用 [httpbin](https://httpbin.org/) 来做验证

```python
def test_proxy():
    # visit http://cn-proxy.com/ to get available proxies if test failed
    proxy_ips = ['112.25.41.136', '180.97.29.57']
    client = Client()
    client.set_proxy_pool(proxy_ips)
    for _ in range(5):
        result = client._session.get('http://httpbin.org/ip').json()
        assert result['origin'] in proxy_ips
        result = client._session.post('http://httpbin.org/post',
                                      data={'m':'1'}).json()
        assert result['form'] == {'m': '1'}
        print(result['origin'])
        assert result['origin'] in proxy_ips

    client.remove_proxy_pool()
    client.set_proxy_pool(proxy_ips, https=False)
    for _ in range(5):
        result = client._session.get('http://httpbin.org/ip').json()
        print(result['origin'])
        assert result['origin'] in proxy_ips
```