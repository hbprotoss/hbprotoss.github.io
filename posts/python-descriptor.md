<!-- 
.. link: 
.. description: 
.. tags: Python
.. date: 2013/06/12 12:31:54
.. title: Python descriptor
.. slug: python-descriptor
-->

一次偶然发现，Python的对象竟然可以在运行期动态添加类定义时没有的属性，这又颠覆了我对Python OO机制的理解。Google了一把，顺着`__dict__`属性一路找到descriptor，揭开了隐藏在Python对象之后的内幕。

本文主要记录Python的descriptor机制，以及其在Python对象的属性、方法绑定上的作用。

先从本文的始作俑者，运行期动态添加对象属性开始讲起。

```python
class A:
    def __init__(self):
        self.value = 'value'
    def f(self):
        print('function f')
a = A()
a.attr = 1
print(a.attr)
```

以上代码奇迹般的没有报错，而且还输出了1。这肯定会让写过C++/Java代码的童鞋表示吃惊，Python变量类型动态也就不稀奇了，对象属性还能动态添加的？Python到底在背后做了什么？

## 神奇的\_\_dict\_\_
在`a.attr = 1`前后分别加上一行`print(a.__dict__)`就会得到如下结果:  
```python
{'value': 'value'}  
1  
{'value': 'value', 'attr': 1} 
``` 

显而易见，我们在运行期定义的属性和类定义时定义的属性都被放在了`__dict__`里。

到这里有人可能就有疑问了，Python里的一切不都是对象麽？为什么成员函数`__init__`、`f`不在这个字典里？

看看`A.__dict__`里有什么就明白了：  
```python
{'__dict__': <attribute '__dict__' of 'A' objects>,
 '__doc__': None,
 '__init__': <function __main__.__init__>,
 '__module__': '__main__',
 '__weakref__': <attribute '__weakref__' of 'A' objects>,
 'f': <function __main__.f>}
```

这时才恍然大悟，如果成员变量看做是对象的属性，那么成员函数就应该看成是类的属性，被全部对象共享嘛。

更精确地讲，以`object.attribute`访问一个对象的属性时，属性的搜索顺序为：

1. 对象自身，`object.__dict__`
2. 对象类型，`object.__class__.__dict__`
3. 对象类型的基类，`object.__class__.__bases__`中的所有`__dict__`。注意，当多重继承的情况下有菱形继承的时候，Python会根据MRO确定的顺序进行搜索。关于MRO(Method Resolution Order)是什么有时间专门写一篇文章总结一下。

当以上三个步骤都没有找到要访问的属性的时候Python就只能抛出`AttributeError`异常了。

## Descriptor是什么
讲了这么多，貌似跟descriptor半毛钱关系没有嘛。别急，接着往下看。

```python
class RevealAccess(object):
    def __init__(self, initval=None, name='var'):
        self.val = initval
        self.name = name

    def __get__(self, obj, objtype):
        print 'Retrieving', self.name
        return self.val

    def __set__(self, obj, val):
        print 'Updating' , self.name
        self.val = val
```

来测试一下这个类

```python
>>> class MyClass(object):
    x = RevealAccess(10, 'var "x"')
    y = 5

>>> m = MyClass()
>>> m.x
Retrieving var "x"
10
>>> m.x = 20
Updating var "x"
>>> m.x
Retrieving var "x"
20
>>> m.y
5
```

这个`RevealAccess`的对象就是一个descriptor，其作用就是在存取变量的时候做了一个hook。访问属性`m.x`就是调用`__get__`方法，设置属性值就是调用`__set__`方法。还可以有一个`__delete__`方法，在`del m.x`时被调用。

只要一个类定义了以上三种方法，其对象就是一个descriptor。我们把同时定义`__get__`和`__set__`方法的descriptor叫做**data descriptor**，把只定义`__get__`方法的叫**non-data descriptor**

## Method binding
有了以上两个概念，我们就能讨论Python的方法绑定了。

还记得讨论`__dict__`时的成员函数`f`吗？按照我们的推测，`A.__dict__['f']`应该和`a.f`是一个东西。但是！！！

```python
>>> A.__dict__['f']
<function __main__.f>
>>> a.f
<bound method A.f of <__main__.A object at 0x7f9d69cc5950>>
```

这两个显然不是一个东西，一个是`function`，一个是`bound method`。这是什么情况？淡定，看下面

```python
>>> A.__dict__['f'].__get__(a, A)
<bound method A.f of <__main__.A object at 0x7f9d69cc5950>>
>>> a.f
<bound method A.f of <__main__.A object at 0x7f9d69cc5950>>
```

这下放心了吧:D

其实，类的成员函数就是一个descriptor，在实例化对象a的时候，Python就做了这么一个过程(伪码，详见Objects/funcobject.c)：`a.f = A.__dict__['f'].__get__(a, A)`

纯Python模拟的函数对象就像这样：

```python
class Function(object):
    . . .
    def __get__(self, obj, objtype=None):
        "Simulate func_descr_get() in Objects/funcobject.c"
        return types.MethodType(self, obj, objtype)
```

然后就好理解`staticmethod`和`classmethod`这两个decorator了吧。`staticmethod`无视了传入的第一个self参数，`classmethod`手工加了一个类对象参数进去。它们的纯Python模拟就像下面所示：

```python
class StaticMethod(object):
 "Emulate PyStaticMethod_Type() in Objects/funcobject.c"

 def __init__(self, f):
      self.f = f

 def __get__(self, obj, objtype=None):
      return self.f
      
class ClassMethod(object):
     "Emulate PyClassMethod_Type() in Objects/funcobject.c"

     def __init__(self, f):
          self.f = f

     def __get__(self, obj, klass=None):
          if klass is None:
               klass = type(obj)
          def newfunc(*args):
               return self.f(klass, *args)
          return newfunc
```

-----
研究Python的底层实现是个很有意思的事，至少能让我在使用Python时更加放心：）

全文完

-----

参考资料：

1. How-To Guide for Descriptors: [http://users.rcn.com/python/download/Descriptor.htm](http://users.rcn.com/python/download/Descriptor.htm "")
2. Python Attributes and Methods: [http://www.cafepy.com/article/python_attributes_and_methods/python_attributes_and_methods.html](http://www.cafepy.com/article/python_attributes_and_methods/python_attributes_and_methods.html "")
3. 《Expert Python Programming》,Tarek Ziadé: [http://book.douban.com/subject/3285148/](http://book.douban.com/subject/3285148/ "")