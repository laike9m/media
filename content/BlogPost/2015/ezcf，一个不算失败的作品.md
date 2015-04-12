这几个月断断续续地完成了一个名叫 ezcf 的库。可以直接 import 除 Python 之外的文件格式，在我看来，还算有趣。所以原本的期望，是在 GitHub 超过 30 个 star，但现在看来，还是预期过高了。国内能宣传的地方差不多宣传了一遍，在 V2EX 发了帖之后，多少吸引了一些人，但是其他社区基本无效。然而真正让我没想到的是去 HackerNews 发帖居然连一个 star 都没增加。说实话，一个开源项目的好坏不取决于代码怎么样，知道的人多，用得人多就是好，这代表有人认同这个东西是有用的，而代码质量啊功能完善度啊都是可以逐渐改善的。没有人用的开源项目没有存在的价值。好在真正去看了项目的人，有几个对我说，“项目非常有趣”，这多少令人欣慰。

上周在推上@了 importpython，他说“pretty cool，下次放进 newsletter”。然而紧接着的 newsletter 里并没有。如果下次还没有，大概就真的没有了。

我感觉 ezcf 没有达到预期认可度的最大问题是，大多人第一眼都不知道它到底在干嘛。如果是列出几个函数的用法，写上输入输出，大家都很明白，问题是 ezcf 并不是这样起作用的，或者说，没法让人第一眼就知道 `import ezcf` 这句话起了什么作用，以及和之后的 import 语句有什么关系。说实话，一两句话根本说不清楚这个问题。其次是有直接 import 配置文件需求的人并不多。总而言之，没有办法快速展现出项目的有趣之处，这就是问题。

总之先放上地址  
[https://github.com/laike9m/ezcf](https://github.com/laike9m/ezcf)  
如果不写这篇博客，感觉无法释怀啊。顺便把 README 也放上来好了。

# ezcf

[![Build Status](https://travis-ci.org/laike9m/ezcf.svg)](https://travis-ci.org/laike9m/ezcf)
[![Supported Python versions](https://pypip.in/py_versions/ezcf/badge.svg)](https://pypi.python.org/pypi/ezcf/)
[![PyPI version](https://badge.fury.io/py/ezcf.svg)](http://badge.fury.io/py/ezcf)
[![Coverage Status](https://coveralls.io/repos/laike9m/ezcf/badge.svg)](https://coveralls.io/r/laike9m/ezcf)

ezcf stands for **easy configuration**, it allows you to import JSON/YAML/INI/XML
like .py files. It is useful whenever you need to read from these formats,
especially for reading configuration files.

OK, stop talking, show us some code!  

On the left is what you'll normally do, on the right is the ezcf way. Much more elegant isn't it?

![](https://github.com/laike9m/ezcf/raw/master/code_compare.png)

## Install

    pip install ezcf
    
If you run into `error: yaml.h: No such file or directory`, don't worry,
you can still use ezcf without any problem.

## Supported File Types
ezcf supports `JSON`, `YAML`, `INI` and `XML` with extension `json`, `yaml`, `yml`, `ini`, `xml`.

## Sample Usage
ezcf supports all kinds of valid import statements, here's an example:
```
├── subdir
│   ├── __init__.py
│   └── sample_yaml.yaml
├── test_normal.py
└── sample_json.json
```

Various ways to use configurations in `sample_yaml.yaml` and `sample_json.json`:
```python
import ezcf

from subdir.sample_yaml import *
# or
from subdir.sample_yaml import something
# or
import subdir.sample_yaml as sy
print(sy.something)

from sample_json import *
# or
from sample_json import something
# or
import sample_json as sj
print(sj.something)
```
You can assume they're just regular python files.(Currently ezcf only supports files with utf-8 encoding)

What about relative import? Yes, ezcf supports relative import, as long as you use it *correctly*.

Something to note before using ezcf:

1. ezcf is still in developement. If you find any bug, please report
it in issues;
2. Be careful importing YAML which contains multiple documents: if there exists keys with the same name,
only one of them will be loaded. So it's better not to use multiple documents;
3. All values in `.ini` files are kept as it is and loaded as a string;
4. Since XML only allows single root, the whole xml will be loaded as one dict with root's name as variable name;
5. Namespace package is not supported yet, pull requests are welcome.