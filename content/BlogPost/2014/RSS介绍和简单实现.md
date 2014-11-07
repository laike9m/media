现在用RSS的人已经很少了，估计网民中不超过5%，这些人中大部分从事IT行业，小部分则是上个时代遗留下来的网民。

RSS 是一种xml规范, 最新版本是2.01。一个网站提供RSS， 本质上就是把网站希望推送给读者的内容按照 RSS 规范写入一个xml, 并且在一个页面（比如/blog/rss）提供这个 xml 的访问。然后 feedburner 或者别的 rss 工具会读取这个页面，检查更新，提供推送。

RSS标准格式如下，下面只列出需要关注的标签

```xml
<title>Lift Off News</title>
<link>http://liftoff.msfc.nasa.gov/</link>
<description>Liftoff to Space Exploration.</description>
<item>
    <title>Star City</title>
    <link>http://liftoff.msfc.nasa.gov/news/2003/news-starcity.asp</link>
    <description>
        How do Americans get ready to work with Russians aboard the
        International Space Station? They take a crash course in culture, language
        and protocol at Russia's Star City.
    </description>
</item>
<item>
    another item
</item>
```

rss 有总的 title, link, description, 有一堆 item, 每一个 item 自己也有 title, link, description 属性，description 就是正文, 例如一篇博客文章的内容。有这些内容就够了。

那么如何给自己的网站加上 rss 呢？很简单，只要照着这个格式生成一个页面就好了。这里还是以 Django 和我自己的博客为例。我的博客的 rss 地址是 [http://www.laike9m.com/blog/rss/](http://www.laike9m.com/blog/rss/)

虽说是自己实现，但是 Django 已经把大部分事情都做好了。  
首先我们要用 Django 提供的 `Feed` 类派生出一个子类

```python
from .models import BlogPost
from django.contrib.syndication.views import Feed

exclude_posts = ("about", "projects", "talks")

class BlogPostFeed(Feed):
     title = "laike9m's blog"
     link = "/blog/rss"
     description = "Update on laike9m blog's articles."
 
     def items(self):
          return BlogPost.objects.exclude(title__in=exclude_posts).filter(category="programming")[:5]

     def item_title(self, item):
          return item.title

     def item_link(self, item):
          return item.get_absolute_url()

     def item_description(self, item):
          html = item.display_html()
          import re
          html = re.sub(r"/media/", r"http://laike9m.com/media/", html)
          return html

     def item_pubdate(self, item):
          return item.pub_date
```

这个子类可能有两种写法，这里只看这一种。需要提供的方法有 `items`, `item_title`, `item_link`, `item_description`，`item_pubdate` 并非必须。可以看到这几个方法对应了之前列出的 rss 元素，所以这个类干的事情，就是告诉 Django 要如何填充 rss 中必须的元素，比如说构造 `<title></title>` 的时候，就通过 `get_title` 这个方法，获取到 `item.title`，而每个 `item` 是在 `items` 中定义的，可以看到，`item` 实际上就是 BlogPost 这个 Model 的实例，也就是一篇文章。  
这里我作了一些筛选，没有把所有文章都放到 rss 里面，这句话

```python
BlogPost.objects.exclude(title__in=exclude_posts).filter(category="programming")[:5]
```

做的事情是把标题属于 `exclude_posts` 的文章排除，这些文章是我不想放进 rss 的，有 "about", "projects", "talks" 这三篇（因为它们并不是真正的文章，只是以和文章同样的形式保存用来构成博客的一些页面的），然后又通过 `filter` 选出那些属于编程类的文章。其实派生类的 `items` 方法不一定要返回 Model Objects，但是这么做的确有一些好处，这里的筛选功能就是因为使用了 Model Objects 而变得非常简单。

然后只要在 `urls.py` 里面加上下面这句话就行了

```python
url('^rss/$', BlogPostFeed())
```

通常 `url()` 的第二个参数是 views 里的某个函数，但是这里我们只要把刚才写的 Feed 类的派生的填进来就好了。

这就是为博客提供 rss 所要做的全部工作。Django 大大简化了步骤，不过使用其它框架，亦或是自己从 html 级开始写出 rss 页面，原理是一样的。简单来讲就是：写一个页面，填上 rss 必须的元素，仅此而已。

如果使用 feedburner 来提供 rss 服务，去注册一个账号，然后把自己的 rss 页面地址填进去，就OK了

![feed](/media/content/BlogPost/images/add_feed.jpg)



