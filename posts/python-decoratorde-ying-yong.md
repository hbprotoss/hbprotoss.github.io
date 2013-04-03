<!-- 
.. title: Python decorator的应用
.. slug: python-decoratorde-ying-yong
.. date: 2013/04/03 21:23:39
.. tags: 
.. link: 
.. description: 
-->


Decorator是23种设计模式之一，提出的目的是为了在不改变现有代码的前提下，通过在头部或尾部添加代码来扩展功能。  
Python语言内建支持decorator模式。由于语法不是本文重点，对语法不熟悉的童鞋可以参考以下几篇文章：  

1. Stackoverflow上的神文：[Understanding Python decorators](http://stackoverflow.com/a/1594484/837187)
2. [[Python学习]decorator的使用](http://blog.donews.com/limodou/archive/2004/12/19/207521.aspx)
3. [PEP318](http://www.python.org/dev/peps/pep-0318/)


下面说一下最近在写SNS客户端插件时Python decorator的应用。  
插件的方法完成的任务是：  

+ 根据OAuth协议，设置好参数，访问HTTP API
+ 读取服务器返回的json
+ 解析json

异常可能来自访问API过程中的网络问题，以及解析json发现服务器返回了错误代码（比如API访问过于频繁，access token有问题等）。在写头几个API的时候做法很自然，用try-except块包裹可能出异常的代码，将各种标准库的异常或者服务器返回的错误信息封装成自己的exception，raise一下告诉主程序有异常需要处理。  
但是写了几个方法后发现，每个方法的异常处理流程几乎一模一样，特别是封装标准库的异常，唯一不同的就是解析服务器返回的错误信息。伪码如下：  

	def method(xxx):
	    try:
	        f = urllib.request.urlopen(url, data, headers)
	        raw_data = f.read().decode('utf-8')
	        rtn = json.loads(raw_data)
	    except Exception1 as e:
	        log.error(e)
	        raise CustomException1()
	    except Exception2 as e:
	        log.error(e)
	        raise CustomException2()
            xxx
	    else:
	        return rtn

出于懒的原因（说得好听点，出于代码复用的目的:D)，得想个办法把这些重复性的工作提取出来。  
仔细观察可以发现，这些方法在except处理完异常之后都没有额外的工作需要进行。换句话说，可以认为这些代码都位于核心处理代码之后，这是自然就想到了decorator模式，将异常处理放在decorator里，再decorate一下各个方法就可以了。  
Decorator如下：  

	def sinaMethod(func):
	    def func_wrapper(*args, **kwargs):
	        try:
	            raw_rtn = func(*args, **kwargs)
	        except urllib.error.HTTPError as e:
	            if e.fp:
	                # Sina app error
	                error_msg = json.loads(e.fp.read().decode('utf-8'))
	                error_code = str(error_msg['error_code'])
	                if error_code in exception_dict:
	                    raise exception_dict[error_code](error_msg['error'])
	                else:
	                    raise weiUnknownError(str(e))
	            else:
	                # Network error
	                raise weiNetworkError(str(e))
	        else:
	            return raw_rtn
	    return func_wrapper

其作用就是，尝试调用被装饰函数，如果顺利的话直接返回结果。如果有HTTPError异常发生，则解析之，并根据相应情况封装成自定义的异常抛出。  
这里使得代码比较简短的一个关键就是这个exception_dict，里面维护了新浪错误代码和需要抛出的异常类之间的映射。根据返回的error_code查找这个字典，如果有则说明这是我们需要关心的异常，那么就用错误消息构造异常对象抛出。如果没有，就不是需要关心的异常，以weiUnknownError异常抛出。  
由于新浪在出错时除了返回HTTP报头，还有一段附加data，所以如果e.fp为None则说明不是应用层的异常，可能是网络层及以下出了问题，抛出weiNetworkError。  

使用就很简单了，在被装饰的函数前一行加`@sinaMethod`就可以了。  
就这么简单~~  

**************************

PS：附带说一句，编程语言对设计模式在语法上的支持会大大简化程序猿的编码量。如果是C++/Java，甚至C要用decorator模式就得协商老长一段了~~^_^
