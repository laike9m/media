做 PPT 太无聊了，突然想到可以统计一下这个东西，于是就做了一下  
现在基本上所有国外大公司和国内部分公司都在 GitHub 上开源了一部分代码。统计一下这些代码的语言使用情况，多少可以反映公司内部对语言的偏好。很多公司流行的项目都是单独建一个 repo的，没办法统计，所以这里统计大家就随便看看吧。使用了 GitHub 的 API，只有不到四十行代码，所以直接贴在这里了，复制下来装个 requests 就可以直接运行

```python
# coding: utf-8

"""
统计大公司github上的organization 中repo 的语言使用情况
"""

import requests
from collections import defaultdict
from os.path import join
from pprint import pprint


class GetLangStat():

    api_url = "https://api.github.com/orgs"
    ORGANIZATIONS = (
        'Microsoft', 'aws', 'google', 'twitter', 'facebook',
        'alibaba'
    )
    stats = {org: defaultdict(int) for org in ORGANIZATIONS}

    @classmethod
    def get_one_org_repos(cls, org):
        print(org)
        url = join(cls.api_url, org, 'repos')
        r = requests.get(url)
        for repo in r.json():
            cls.stats[org][repo['language']] += 1

    @classmethod
    def get_all_org_repos(cls):
        for org in cls.ORGANIZATIONS:
            cls.get_one_org_repos(org)
        pprint(cls.stats)


if __name__ == '__main__':
    GetLangStat.get_all_org_repos()
```

统计的公司包括 MS，amazon，google, twitter, facebook, 阿里。其中 amazon 似乎只开源了 aws 相关的代码，不过也算进来了。本来想找百度和腾讯的，结果发现百度没有一个统一的 organization，都是按产品散着的，腾讯则基本没有开源代码。。。

下面是统计结果，每个公司只取前五名

阿里   
 
| Language   | repo count |
|------------|------------|
| Java       | 13         |
| C          | 8          |
| C++        | 2          |
| JavaScript | 2          |
| Perl       | 2          |

google  

| Language   | repo count |
|------------|------------|
| JavaScript | 9          |
| C++        | 4          |
| Ruby       | 4          |
| Python     | 3          |
| Java       | 3          |

twitter  

| Language                   | repo count |
|----------------------------|------------|
| Scala                      | 14         |
| Ruby                       | 9          |
| Java                       | 3          |
| Python,CSS,JavaScrit,Shell | 1          |

facebook  

| Language                | repo count |
|-------------------------|------------|
| Java                    | 8          |
| C++                     | 5          |
| PHP                     | 4          |
| C                       | 3          |
| Python, Js, Objective-C | 2          |

aws  

| Language    | repo count |
|-------------|------------|
| Java        | 8          |
| Ruby        | 6          |
| PHP         | 4          |
| JavaScript  | 3          |
| Objective-C | 3          |

咳咳，最后是大微软，说实话我也不确定要不要把微软算成互联网公司。。。

| Language    | repo count |
|-------------|------------|
| C#          | 29         |
| C++         | 1          |

从这个非常不靠谱的统计来看，Java 果然还是最流行的语言啊。。。不会 Java 感觉压力真的好大 orz