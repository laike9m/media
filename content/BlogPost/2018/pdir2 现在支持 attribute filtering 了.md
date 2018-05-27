最近应用户[要求](https://github.com/laike9m/pdir2/issues/37)给 [pdir2](https://github.com/laike9m/pdir2) 添加了一个功能，在 0.3.0 中发布。

现在你可以更精确地获取所需属性了：  
`properties`: Find properties/variables defined in the inspected object.

`methods`: Find methods/functions defined in the inspected object.

`public`: Find public attributes.

`own`: Find attributes that are not inherited from parent classes.

比如这个例子（完整文档参考 [wiki](https://github.com/laike9m/pdir2/wiki/Attribute-Filtering)）

```
class Base(object):
    base_class_variable = 1

    def __init__(self):
        self.base_instance_variable = 2

    def base_method(self):
        pass


class DerivedClass(Base):
    derived_class_variable = 1

    def __init__(self):
        self.derived_instance_variable = 2
        super(DerivedClass, self).__init__()

    def derived_method(self):
        pass


inst = DerivedClass()

pdir(inst).properties  # 'base_class_variable', 'base_instance_variable',
                       # 'derived_class_variable', 'derived_instance_variable', '__class__',
                       # '__dict__', '__doc__', '__module__', '__weakref__'

pdir(inst).methods  # '__subclasshook__', '__delattr__', '__dir__', '__getattribute__',
                    # '__setattr__', '__init_subclass__', 'base_method',
                    # 'derived_method', '__format__', '__hash__', '__init__', '__new__',
                    # '__repr__', '__sizeof__', '__str__', '__reduce__', '__reduce_ex__',
                    # '__eq__', '__ge__', '__gt__', '__le__', '__lt__', '__ne__'

pdir(inst).public  # 'base_method', 'derived_method', 'base_class_variable',
                   # 'base_instance_variable', 'derived_class_variable',
                   # 'derived_instance_variable'

pdir(inst).own  # 'derived_method', '__init__', 'base_instance_variable',
                # 'derived_class_variable', 'derived_instance_variable', '__doc__',
                # '__module__'

pdir(inst).public.own.properties  # 'base_instance_variable','derived_class_variable',
                                  # 'derived_instance_variable'
pdir(inst).own.properties.public  # ditto
pdir(inst).properties.public.own  # ditto

pdir(inst).public.own.properties.search('derived_inst')  # 'derived_instance_variable'
```

支持 Method Chaining，支持和 `search()` 方法一起使用。大概就这样。