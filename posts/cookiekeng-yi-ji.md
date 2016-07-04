<!-- 
.. title: Cookie坑一记
.. slug: cookiekeng-yi-ji
.. date: 2016-07-04 15:23:48 UTC+08:00
.. tags: Cookie
.. category: 
.. link: 
.. description: 
.. type: text
-->

关于Cookie的domain属性，[RFC6265](https://tools.ietf.org/html/rfc6265#section-4.1.2.3)上是这么说的：

1. 如果domain属性缺失，那么该cookie只能对**当前host**可见。比如，xx.com下有个cookie，`Set-Cookie`头没有带domain属性，那么这个cookie只对xx.com下的页面可见，a.xx.com等子域名不可见
2. .xx.com这种domain其实是不合法的，只不过浏览器在见到这种域名的时候，需要自动把最前面的.忽略掉，把xx.com作为cookie的domain，即xx.com及其子域名都可见