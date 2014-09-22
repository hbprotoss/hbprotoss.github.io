<!-- 
.. title: multipart/form-data的实现
.. slug: multipartform-datade-shi-xian
.. date: 2013/03/29 09:19:39
.. tags: Python
.. link: 
.. description: 
-->


写之前先吐槽几句：Python社区太懒了，Python3都推出多少年了，那么多第三方库还不port到Python3。不能安于现状啊！  

******************

下面是正题：

最近写微博客户端，上传图片需要multipart/form-data编码图片和参数，由于前面说的原因，没有第三方库可用，所以就自己实现一下。

先贴一下multipart/form-data的RFC文档地址：[点这里](http://www.ietf.org/rfc/rfc2388.txt)

multipart/form-data主要由三部分组成：  

1. HTTP Header。需要添加头"Content-Type: multipart/form-data; boundary=%s"，这个boundary就是分隔符，见第二条。
2. 分隔符boundary。分隔符是一串和正文内容不冲突的字符串，用以分割多个参数。一般都是N个减号+随机字符串，比如"----------当前时间"。  
   正文需要加header：  
   Content-Disposition: form-data; name="%s"，%s为需要传递的变量名。  
   Content-Type: 指定正文MIME类型，默认是纯文本text/plain，未知类型可以填application/octet-stream。  
3. 数据。要注意的是数据的编码，文档上说"7BIT encoding"，ISO-8859-1即可。

下面贴一段上传新浪微博图片的代码：  


	#!/usr/bin/env python3

	import urllib.request
	import urllib.parse
	import urllib.error
	import time
	import json
	import mimetypes

	def _encode_multipart(params_dict):
	    '''
	    Build a multipart/form-data body with generated random boundary.
	    '''
	    boundary = '----------%s' % hex(int(time.time() * 1000))
	    data = []
	    for k, v in params_dict.items():
		data.append('--%s' % boundary)
		if hasattr(v, 'read'):
		    filename = getattr(v, 'name', '')
		    content = v.read()
		    decoded_content = content.decode('ISO-8859-1')
		    data.append('Content-Disposition: form-data; name="%s"; filename="hidden"' % k)
		    data.append('Content-Type: application/octet-stream\r\n')
		    data.append(decoded_content)
		else:
		    data.append('Content-Disposition: form-data; name="%s"\r\n' % k)
		    data.append(v if isinstance(v, str) else v.decode('utf-8'))
	    data.append('--%s--\r\n' % boundary)
	    return '\r\n'.join(data), boundary

	#############################################################
	url = 'https://upload.api.weibo.com/2/statuses/upload.json'
	access_token = 'xxx'
	file_name = 'big.png'
	path = '/home/xxx/Downloads/' + file_name
	status = '1234567'

	params = {
	    'access_token': access_token,
	    'status': urllib.parse.quote(status),
	    'pic': open(path, 'rb')
	}

	coded_params, boundary = _encode_multipart(params)

	#############################################################
	req = urllib.request.Request(url, coded_params.encode('ISO-8859-1'))
	req.add_header('Content-Type', 'multipart/form-data; boundary=%s' % boundary)
	try:
	    resp = urllib.request.urlopen(req)
	    body = resp.read().decode('utf-8')
	    print(body)
	except urllib.error.HTTPError as e:
	    print(e.fp.read())

**其中尤其要注意的是，POST上去的data部分要encode成ISO-8859-1。一开始一直encode成UTF-8，死活报的都是格式错误。**
