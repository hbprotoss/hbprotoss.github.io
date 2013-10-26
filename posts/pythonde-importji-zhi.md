<!-- 
.. title: python的import机制
.. slug: pythonde-importji-zhi
.. date: 2013/03/18 19:11:42
.. tags: Python
.. link: 
.. description: 
-->


同一个模块以相同的模块名被导入时，模块内的代码只执行一遍（模块名必须相同）。  
例：  

>import/  
>    test.py  
>    mo/  
>        A.py  

****************************************************************************

	# A.py
	a = 1
	print(a)
	a += 1

****************************************************************************

	# test.py
	#!/usr/bin/env python3
	import os
	import sys
	import mo.A
	import mo.A
	path = os.path.abspath('./mo')
	print(path)
	sys.path.append(path)
	import A

****************************************************************************
test.py中，第4行执行后输出1。第5行不输出，因为mo.A已经被导入当前namespace。  

但是第9行，虽然都是同一个物理文件A.py，但是被导入时的名字不同（mo.A和A），所以还是会输出1。mo.A和A作为两个独立的module对象存在当前namespace中。  

如果要把模块中的变量当全局变量来使用的话，一定要从项目顶部通过“.”一层一层import。  
