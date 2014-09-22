<!-- 
.. title: 实现简单线程池
.. slug: shi-xian-jian-dan-xian-cheng-chi
.. date: 2013/03/18 18:28:37
.. tags: Python, Qt, 线程池
.. link: 
.. description: 
-->


前段时间在写代码的时候（用了Qt）出现了似乎跟线程池有关的bug，在不确定是否是Qt线程池库QThreadPool的bug的情况下，自己实现了一个简单的线程池以确定到底是我的问题还是QThreadPool是否真有bug。（剧透下，最后证明不是QThreadPool有问题，可能是信号在线程之间传递的时候有问题，这个bug至今未修复= =）  

下面记录下我这个线程池的实现思路。  
主要有两个类，Thread、Manager。ThreadPool类是整体的包装。  

Thread，继承自QThread，用来执行线程代码，并且作为线程对象存在。  
Manager，当前线程活动情况的管理类，检查线程池内空闲线程的情况，并管理新添加的任务。  

## **Manager**  
Manager内部有一个维护空闲线程对象的集合idle_thread，考虑到需要在O(1)的时间内添加或删除指定元素，用了python内置的set。set内部使用哈希实现的，所以时间效率基本能控制在O(1)左右。  
还有一个已添加任务的队列queue。时间效率上，入队和出队都要求O(1)的时间，但是最先想到的python内置的list对象是数组和链表的结合，入队时间是O(1)，出队时间却是O(n)（参见[这个链接](http://blog.csdn.net/digimon/article/details/7875781)），所以用了collections.deque，一个双向链表，能实现入队和出队都是O(1)的时间。另外，由于我不需要优先级的概念，所以线性表足矣。以后如果要考虑优先级的话会改成堆。  
另外就是一个idle_thread的互斥量mutex。因为set对象是线程不安全的，所以在添加和删除元素的时候必须有互斥量保护。而collections.deque文档上说是线程安全的，所以就不需要互斥量之类的机制了。  

Manager的接口有：  
hasIdleThread，返回线程池内是否还有空闲线程。  
setThreadIdle，将某个线程加入到空闲线程池中。  
startTask，先将task入队，然后看是否有空闲线程，如果有则在这个空闲线程上执行，没有的话就留在队列里等待下次调度。  

## **Thread**  
在run方法内，通过Manager取出一个task执行，循环直到再没有新的task为止，之后跳出循环，将自己加入到空闲线程集合内。  
其实这样并不能保证线程一直存活，还是会有销毁和重建线程的动作，而且某些极端情况下还会很频繁。但是我当时的情况是要执行的任务会一次性全部加入到队列里，而下一次同样的添加动作会相隔比较长的时间，所以问题不大。更重要的是，当时只是为了确定QThreadPool是否有bug，这方面没有考虑地太多。  

下面是具体代码（遵循GPLv3协议）  

	#!/usr/bin/env python3

	import collections
	from PyQt4.QtCore import *
	from app import logger

	log = logger.getLogger(__name__)

	class Thread(QThread):
	    def __init__(self, manager, name):
		super(Thread, self).__init__()
		self.manager = manager
		self.name = name
		
	    def __str__(self):
		return self.name
		
	    def run(self):
		while True:
		    try:
			log.debug(self.name + str(self.manager.queue))
			task = self.manager.dequeueTask()
			task.run()
			if task.autoDelete():
			    del task
		    except IndexError:
			# There is no task
			log.debug(self.name + 'No task, %s quiting' % self)
			break
		    except Exception as e:
			print(e)
			
		self.manager.setThreadIdle(self)

	class Manager:
	    def __init__(self, max_thread_count):
		self.max_thread_count = max_thread_count
		
		self.threads = [Thread(self, 'thread_%d' % i) for i in range(max_thread_count)]
		
		self.idle_threads = set(self.threads)
		self.idle_threads_mutex = QMutex()
		
		self.queue = collections.deque()
		
	    def hasIdleThread(self):
		return len(self.idle_threads) != 0
	    
	    def setThreadIdle(self, thread):
		self.idle_threads_mutex.lock()
		self.idle_threads.add(thread)
		log.debug('%s becomes idle' % thread)
		self.idle_threads_mutex.unlock()
	    
	    def startTask(self, task):
		self.enqueueTask(task)
		log.debug('%s enqueued' % task)
		
		self.idle_threads_mutex.lock()
		try:
		    thread = self.idle_threads.pop()
		    thread.start()
		    log.debug('%s started' % task)
		except KeyError:
		    # No idle thread
		    log.debug('No idle thread')
		    pass
		except Exception as e:
		    print(e)
		self.idle_threads_mutex.unlock()
	    
	    def enqueueTask(self, task):
		self.queue.append(task)
	    
	    def dequeueTask(self):
		return self.queue.popleft()

	class ThreadPool:
	    def __init__(self, max_thread_count=2):
		self.manager = Manager(max_thread_count)
		
	    def start(self, task):
		if not isinstance(task, QRunnable):
		    raise TypeError('Task must be QRunnable object.')
		self.manager.startTask(task)
