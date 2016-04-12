<!-- 
.. title: XPath备忘
.. slug: xpathbei-wang
.. date: 2016-04-12 15:46:09 UTC+08:00
.. tags: Python, Crawler, Scrapy
.. category: 
.. link: 
.. description: 
.. type: text
-->

最近写了几个scrapy的爬虫程序，里面用到了xpath，写个日志记录一下用法。

# XPath是什么

XPath是一种用于xml、html等结构化文档中寻址定位特定元素等描述性语言

# XPath主要功能

下面以如下测试文档为例进行说明：

```html
<html>
	<body>
		<contents id="content">
		<para><a href="one.html" class="normal-link">One</a></para>
		<para><a href="two.html" class="normal-link">Two</a></para>
		<para><a href="three.html" class="ex-link">Three</a></para>
		</contents>
	</body>
</html>

```

## 精确路径寻址

指通过精确制定的路径取得元素。例如

和unix文件系统概念一致，有如下几种定位方式：

* 绝对路径，`/html/body/contents/para`能查找出文档中的三个para元素
* 相对路径，在`/html/body`路径下，`/contents/para`同样也能查找出这三个para元素
* 父级路径，`.`表示当前路径，`..`表示当前路径的父级路径

## 模糊路径寻址

不需要指定绝对路径或根据当前路径确定的相对路径，只需要指定某个子结构，就能查找出所有符合这个子结构的元素。如

* `//contents/para`在任何路径下，都能查找到**整个文档**下的这三个para元素
* `.//contents/para`能在当前路径下，查找到子节点中任何符合`contents/para`结构的元素

## 节点属性匹配

格式：元素[@属性="xxx"]

* `a[@class="normal-link"]`能查找出两个有带normal-link class的a链
* `para[a/@class="ex-link"]`能查找出一级子元素中有带ex-link class的a链的para元素，这里就是`<para><a href="three.html" class="ex-link">Three</a></para>`

## 属性选择

查找某个元素中的特定属性值，如：`a[@class="ex-link"]/@href`能读取第三个a链的href值

## 内置函数

* node()，返回任意种类的节点。比如和内置关键字`child`组合成`/html/body/contents/child::node()`，可以选择所有的para节点
* text()，返回节点中包含的文本。`/html/body/contents/para/a[@class="ex-link"]/text()`返回Three。特别的，和模糊路径寻址配合，如`/html/body/contents//text()`，能返回contents下的`One Two Three`字符串