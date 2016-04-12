<!-- 
.. title: 搭建智能翻墙路由器
.. slug: da-jian-zhi-neng-fan-qiang-lu-you-qi
.. date: 2015-04-16 08:49:13 UTC+08:00
.. tags: OpenWRT, DNS, ShadowSocks, 翻墙
.. category: 
.. link: 
.. description: 
.. type: text
-->

本文着重介绍如何搭建智能翻墙路由器，实现避免DNS污染，并且自动根据是否是国内IP来决定是否翻墙，从而使任何连接路由器的设备无障碍穿墙出去。

思路是，用shadowsocks建立起翻墙代理服务，用shadowsocks的udp relay模式转发DNS请求，解决DNS污染，再配置iptables根据IP决定是否走shadowsocks来翻墙。

# 准备材料
* 能刷OpenWrt的智能路由器。我用的是小米路由器MINI，16MB ROM，128MB DDR2内存，MT7620A处理器，运行毫无压力（真的不是广告。。。）
* OpenWrt。这里选的是[PandoraBox](http://downloads.openwrt.org.cn/PandoraBox/)版
* shadowsocks client，PandoraBox自带
* ChinaDNS，PandoraBox自带
* VPS一台
* shadowsocks server，选用C实现：shadowsocks-libev，go和Python版没有UDP relay功能，不能实现DNS请求转发

# 步骤
## shadowsocks server
shadowsocks-libev的安装参考[http://shadowsocks.org/en/download/servers.html](http://shadowsocks.org/en/download/servers.html)中的“C with libev”一节

安装完成之后将如下配置写入config.json

```
{
	"server":"0.0.0.0",
	"server_port":8025,
	"password":"123456",
	"timeout":300,
	"method":"aes-256-cfb"
}
```

分别指定了服务器的binding address，端口，密码，超时时间和加密方式。按需更改

在远程VPS上启动shadowsocks server: `ss-server -c config.json -u`

注意得加上`-u`选项，enable udprelay mode。用作DNS请求转发，避免DNS污染。

## shadowsocks client
在本地路由器上起shadowsocks client

### ss-redir

ss-redir用于将客户端的原始数据封装成shadowsocks协议内容，转发给server，实现透明转发。

在本地路由器启动ss-redir: `ss-redir -s "your_server_ip" -p "your_server_port" -l "local_service_port" -m "encryption_method" -k "server_password" -f "pid_file_path"`
注意这里不需要`-u`选项，转发的是TCP包，ss-redir也不支持这个选项。

### ss-tunnel

ss-tunnel用于实现本地port forward，和ssh的port forward一样，只是加密方式用了shadowsocks协议，用于在本地起服务，转发DNS请求

在本地路由器启动ss-tunnel: `ss-tunnel -s "your_server_ip" -p "your_server_port" -l "local_service_port" -m "encryption_method" -k "server_password" -L "server_ip:server_port" -f "pid_file_path" -u`

这个`-L`选项理论上可以填国外DNS的IP/PORT，比如`8.8.8.8:53`，我在我的VPS起了一个DNS转发服务，填了自己的IP/PORT，效果应该一样。`-u`是开启udp relay，DNS是UDP包嘛

## ChinaDNS
如果所有DNS请求都走国外DNS server，国内有些网站（如微博）在海外有服务器，就会比较慢。ChinaDNS保证的就是国内域名能解析成国内IP，国外域名解析成国外IP。

在本地路由器启动ChinaDNS: `chinadns -l /etc/chinadns_iplist.txt -c /etc/chinadns_chnroute.txt -d -p "local_dns_port" -s 114.114.114.114,127.0.0.1:8026`

`-s`选项后加以逗号分隔的DNS服务器列表，最好国内、国外各有一个。由于前面通过ss-tunnel在本地起了一个端口转发到我的DNS服务，所以填的是`127.0.0.1:8026`。

ChinaDNS原理是，向所有列表中的DNS server发DNS请求，判断结果可信任条件是：国内DNS解析出的是国内IP，国外DNS解析出的是国外IP，`/etc/chinadns_chnroute.txt`里包含了国内IP网段，`/etc/chinadns_iplist.txt`里包含的是常见的被污染后的DNS解析结果IP。这样只要通过`dnsmasq`之类的把本地DNS请求转发到ChinaDNS的端口上，就能解决DNS污染的问题。

## iptables
前面的准备工作完成之后，只要配置一下iptables，将国内TCP流量直接放行，国外TCP流量转发到ss-redir起的端口就行，判断依据是IP。

在nat表中新建一个SHADOWSOCKS链

```
iptables -t nat -N SHADOWSOCKS
```

远程VPS流量直接放行

```
iptables -t nat -A SHADOWSOCKS -d xxx.xxx.xx.xxx -j RETURN
```

内网流量直接放行

```
iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN
```

国内IP流量直接放行(列表较长，这里不列举了，网上随便搜搜就有)

剩下的TCP流量转发到ss-redir端口

```
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports xxx
```

# 完工
搞定之后，任何连接这个路由器的设备都能实现透明翻墙了。

以上