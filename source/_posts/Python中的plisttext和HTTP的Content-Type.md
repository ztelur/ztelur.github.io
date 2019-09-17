---
title: Python中的plisttext和HTTP的Content-Type
tags:
  - http
categories:
  - python
abbrlink: '64e12480'
date: 2016-05-28 16:51:37
---

&emsp;这段时间本人在学习`Android Service`相关的内容，临时需要一个可以提供文件上传和下载功能的服务器，于是上网查找了一个简单服务器的python实现代码，本着温顾一下HTTP协议的想法，于是深入研究了一下其中的代码，发现大家对`SimpleHTTPRequestsHandler`中的`self.headers.plisttext.split("=")[1]`语句的含义不是很理解，于是自己查阅了一下python源码定义和相关HTTP协议文档，理解了这段代码的含义。

#### 源码定义
&emsp;我们先来看一下关于`plisttext`的源码定义。
``` python
#https://svn.python.org/projects/python/branches/alpha100/Lib/mimetools.py
class Message(rfc822.Message):
	def __init__(self, fp):
    ....
		self.typeheader = \
			self.getheader('content-type')
		....
	def parsetype(self):
		str = self.typeheader
		if str == None:
			str = 'text/plain'
		if ';' in str:
			i = string.index(str, ';')
			self.plisttext = str[i:]
			str = str[:i]
		else:
			self.plisttext = ''
		....
```
&emsp;从源码中可以得出，`plisttext`与HTTP头部`content-type`有关，这里我们就要回想一下`content-type`的有关定义了。
&emsp;在[w3c的文档](https://www.w3.org/Protocols/rfc1341/4_Content-Type.html)给出了`content-type`的格式定义,我们可以发现，`content-type`对的值有可选的内容，使用`；`隔开，所以`plisttext`的值就是`parameter`的内容。
```
Content-Type := type "/" subtype *[";" parameter] 

type :=          "application"     / "audio" 
          / "image"           / "message" 
          / "multipart"  / "text" 
          / "video"           / x-token 

x-token := <The two characters "X-" followed, with no 
           intervening white space, by any token> 

subtype := token 

parameter := attribute "=" value 

attribute := token 

value := token / quoted-string 

token := 1*<any CHAR except SPACE, CTLs, or tspecials> 

tspecials :=  "(" / ")" / "<" / ">" / "@"  ; Must be in 
           /  "," / ";" / ":" / "\" / <">  ; quoted-string, 
           /  "/" / "[" / "]" / "?" / "."  ; to use within 
           /  "="                        ; parameter values 
```

#### 使用原理
&emsp;知道了`plisttext`代表的含义，我们再来看一下它在文件上传过程中的作用吧。我们先来看一下它在处理文件上传的`post`请求时的作用吧。
``` python
boundary = self.headers.plisttext.split("=")[1]
remainbytes = int(self.headers['content-length'])
line = self.rfile.readline()
remainbytes -= len(line)
if not boundary in line:
    return (False,"Content NOT begin with boundary")
line = self.rfile.readline()
remainbytes -= len(line)
filename = re.findall(r'Content-Disposition.*name="file"; filename="(.*)"',line)
if not fn:
    return (False,"Can't find out file name")
```
&emsp;我们都知道当通过`html`的`form`来进行文件提交时，浏览器会发送`POST`请求，并且content-type为`multipart/form-data; boundary=----WebKitFormBoundaryqdHXHkzdBEGWWZka`,所以，`plisttext`的值为`boundary=----WebKitFormBoundaryqdHXHkzdBEGWWZka`。`boundary`在`HTTP`的body中会使用到，因为post请求提交了很多类型的数据，所以必须使用`boundary`进行间隔，也就是所谓的[Multipart Content-Type时的body格式](https://www.w3.org/Protocols/rfc1341/7_2_Multipart.html)。详细的body的格式在w3c的文档中有详细的介绍。

&emsp;这里贴一张wireShark截获的tcp包的信息，来帮助大家理解一下这段python代码的原理。通过`form`提交一份文件和一个名为other的字符串。
```
 POST / HTTP/1.1
Host: localhost:8080
Connection: keep-alive
Content-Length: 269353
Cache-Control: max-age=0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Origin: http://localhost:8080
User-Agent: Mozilla/5.0 (X11; Linux i686) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/43.0.2357.81 Chrome/43.0.2357.81 Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryqdHXHkzdBEGWWZka
Referer: http://localhost:8080/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.8,zh-CN;q=0.6,zh;q=0.4

------WebKitFormBoundaryqdHXHkzdBEGWWZka
Content-Disposition: form-data; name="file"; filename="AndroidStudy.png"
Content-Type: image/png
..... //图片内容
------WebKitFormBoundaryqdHXHkzdBEGWWZka
Content-Disposition: form-data; name="other"

ddd
------WebKitFormBoundaryqdHXHkzdBEGWWZka--
```
