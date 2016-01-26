---
layout: post
title:  "朋友圈图片查看"
date:   2016-01-26 23:36:26
categories: programming
tags: programming
author: "ScorpiusZ"
---

今天朋友圈加了一个新功能花钱查看模糊的图片    
同事们很兴奋的要__hack__一下    
本来我以为会有ssl 加密的    
后来发现其实是http 的    
于是乎上步骤  

##1. 运行一Proxy(代理)

将下面的代码保存为 __proxy.rb__ 

```ruby
#!/usr/bin/env ruby
# run a local http proxy
require 'rubygems'
require 'webrick'
require 'webrick/httpproxy'
p = WEBrick::HTTPProxyServer.new :Port => ARGV.first
trap("INT") { p.shutdown }
p.start
```

```ruby
#运行proxy server
ruby proxy.rb 8008
```

##2. 设置手机代理

点击叹号       
![] (https://raw.githubusercontent.com/ScorpiusZ/ScorpiusZ.github.io/master/assets/images/wifi-1.png)

设置代理地址端口为刚才启动代理设置的端口(__8008__)      
![] (https://raw.githubusercontent.com/ScorpiusZ/ScorpiusZ.github.io/master/assets/images/wifi-2.png)

然后清空微信的缓存 打开朋友圈就会看到这样的    
![] (https://raw.githubusercontent.com/ScorpiusZ/ScorpiusZ.github.io/master/assets/images/log.png)

其中红色划线的就是没有加模糊效果的原图地址


##3. 最后别忘了去Wifi 设置里把Proxy 的设置取消
祝大家__Happy Hacking__
