之前根据Django自带的缓存写了一个简单的缓存系统，针对views而不是页面，同时支持function based和class based views。因为并没有实际投入使用，所以只是个demo（但是测试过）。所有的代码都在一个文件里，参见 [django_cache_for_views.py](https://gist.github.com/laike9m/cb861faea84f3f4d5630)  
Django缓存系统的介绍可见[官方文档](https://docs.djangoproject.com/en/1.7/topics/cache/)，这里用到的缓存方式是Local-memory caching，也就是说不依赖外部的数据库或者文件，直接把缓存放在内存中。使用方法如下：

### 在`settings.py` 中加入设置

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
        'KEY_PREFIX': 'kratos_server'
    }
}
```
### 使用`utils.py`中定义的函数，需要在外部调用的有3个  
`generate_key`, `get_cachevalue`, `set_cachevalue`     
具体的参数请直接参考代码的注释

### 使用举例

```python
@api_view(['GET'])
def cost_month_detail(request):
    """  
    获取某个成本某月的详细采购记录
    """
    costs = Cost	.objects.all()
    item = request.QUERY_PARAMS.get('item', 'undefined')
    cost_type = COST_TYPE_KEY_MAP.get(item, COST_TYPE_UNDEFINED)
    month = request.QUERY_PARAMS.get('month', None)
    key = generate_key('.' , *[item, month])
    data = get_cachevalue(key)
    if not data:
        req_ym_str = month.split('.')
        req_ym_int = [int(req_ym_str[0]), int(req_ym_str[1])]

        if cost_type and req_ym_int:
            costs = costs.filter( cost_type=cost_type,\
                    purchase_date__year=req_ym_int[0],\
                    purchase_date__month=req_ym_int[1])
        serializer = CostSerializer(costs)
        data = serializer.data
        set_cachevalue(key, data, [models[item]])
    return Response(data)
```

<br />
### 说明
Django的缓存在手动操作的情况下就是key-value的结构，添加、读取、删除，这几个操作Django已经封装得很好，无非就是 `cache.add(key,value)`，`cache.get(key)`，`cache.delete(key)`。关键之处在于，缓存的key要如何设计。我们当然可以对每一项需要缓存的东西事先定好key，比如有个Model叫做 `Car`，那么缓存数据库中全部车辆信息的key就叫做 `car_info`。这么做的问题在于，缓存项多了之后，容易搞混，比如我们希望缓存车辆价格信息，是不是又要弄一个 `car_price_info` 呢。而且在添加/获取缓存项的时候，如果每一个地方都要手动填写key，实在是非常麻烦又易出错。  
要避免这种麻烦，只剩下一个选择，那就是自动生成所需要的 key，并且在读取缓存时，程序也能自己知道该用什么key去找缓存。自动生成key的代码，在 `generate_key` 这个函数里

```python
import inspect

def generate_key(seperator, *args):
    """
    :param seperator: key中的分隔符, 比如'-', '.'
    :param args: 需要添加到key中的参数
    :return: 生成的key
    这个函数用来生成缓存的key, 有两种模式
    1. class based view
    会读取class name和method name
    2. function view
    会读取function name
    最后用传入的seperator把所有东西, 包括args连接起来, 作为key返回
    """
    key = []
    frame_back = inspect.currentframe().f_back
    func_name = frame_back.f_code.co_name   
    key.append(func_name)
    
	if 'self' in frame_back.f_locals:
    	class_name = frame_back.f_locals['self'].__class__.__name__
    	key.append(class_name)
    
    key = seperator.join(key + [str(arg) for arg in args])
    return key
```
这段代码起作用的部分一共就9行，不过我自己都觉得看懂好难啊_(:3」∠)_。其实现在记下来的一个目的就是怕之后看不懂（雾。不如我们先看看效果吧：  
假设有一个 function based view 函数 `fbv`：

```python
def fbv(request):
	...
	key = generate_key('-', 'args1', 'args2')
    ...
```
先不考虑上下文，只关注生成key这件事。这里，生成的key是 `fbv-args1-args2`

再看一个 class based view：  

```python
class cbv(View):
	def get(self, request):
		...
		key = generate_key('-', 'args1', 'args2')
		...
```
这里生成的 key 是 `cbv-get-args1-args2`  

现在，`generate_key`这个函数的效果就很清楚了，说白了就是把所有对生成一个有意义且唯一的 key 有帮助的信息用用户自定的分隔符连接起来，就得到了这个 key。这些信息包括：

1. view 函数名
2. class based view 类名（如果有的话）
3. 用户自己添加的信息  

如果用户懒得添加额外信息，那就直接 `generate_key('-')` 就行了，这样一来，每一个view函数都能够生成一个key，并且这个key具有唯一性，之后进行 `add/get/delete` 操作时，直接使用这个key就好，达成了我们希望的自动化。

那么生成key这件事到底是如何做到？我使用了 [inspect](https://docs.python.org/3/library/inspect.html) 这个模块。之前没有用过它，为了这个任务不得已去看了看，结果发现 inspect 真的是非常强大，能完成好多看似不可能的任务。  
由于这不是一篇讲 inspect 的文章，所以这块就不详细展开，看官方文档即可。要说的是三条语句所完成的事情：

1. `frame_back = inspect.currentframe().f_back`  
	获取当前堆栈帧的上一个堆栈帧，换句话说实际上就是调用generate_key函数的堆栈帧,也即view函数的堆栈帧
2. `func_name = frame_back.f_code.co_name`  
	从堆栈帧里面读取代码信息，找到这段代码的名字，其实就是获取view函数名
3. `if 'self' in frame_back.f_locals:`  
	从frame_back也就是view函数所在的堆栈帧读取local namespace，看看 `self` 是否在里面，用来判断这个view function是不是某个类的方法，即判断是否是class based view
4. `class_name = frame_back.f_locals['self'].__class__.__name__`  
	如果是class based view，那我们就通过 `self.__class__.__name__` 获取类的名字

以上就是缓存系统介绍的第一部分，之后应该还会写一篇，讲一下缓存如何和某个 Model 绑定以及缓存的更新。