---
layout: post
title:  "读书笔记：高性能网站建设"
date:   2014-12-28
tags: tech
author: hoohack
categories: 读书笔记
---

**第一章  减少HTTP请求**

- 使用图片地图：当导航栏包含多张图片时，可以将其合并成一张图片，再通过计算位置触发不同的链接
- CSS sprites：将图标合并，引入一张背景图，通过CSS控制其位置
- 内联图片：将图片编码后再放到data后面。可用PHP的base64_encode对图片文件进行编码。
- 合并脚本和样式文件：理想情况下一个页面一个CSS文件



**第二章  使用CDN(Content Delivery Networks)**

[CDN][1]是指内容分发网络
CDN的做法是指将组建分布到其他服务器上
缺点：

> * 响应时间可能会受到其他网站的影响
> * 无法直接控制组件服务器所带来的特殊麻烦
> * 工作度量会随CDN性能下降所影响

**第三章    为组件添加长的Expires头**

Expires 属性设置页面在失效前被缓存的时间。如果用户在页面失效前返回同一页面，缓存的版本将显示出来。

**第四章  压缩脚本和样式表**

通过压缩文件从而减少HTTP响应大小来减少响应时间

**第五章  将样式表放在顶部**

将样式表放在页面顶部加载可以保证页面基本内容的呈现

**第六章  将脚本文件放在底部**

将脚本放在底部加载可以实现页面的逐步呈现和提高下载的并行度。

> 如果脚本使用document.write向页面中插入内容，就不能将其移动到页面中靠后的位置。但是如果一个脚本可以延迟加载，那么它一定可以移到页面底部加载。

**第七章  避免CSS表达式**

CSS表达式是动态设置CSS属性的强大（但危险）方法。
如使用CSS表达式可以实现隔一个小时切换一次背景颜色：

    background-color: expression((new Date()).getHours()%2?"#FFFFFF": "#000000" );

表达式的问题在于对其进行的求值频率比人们期望的高。它们不只在页面呈现和大小改变时求值，当页面滚动、甚至用户鼠标在页面上移过时都要求值。
如果一定要使用CSS表达式，有两种技术可以避免CSS表达式产生这一问题：
> * 创建一次性表达式
> * 使用事件处理器取代CSS表达式

**第八章  使用外部的Javascript和CSS**

如果你的网站的本质上能够为用户带来高完整缓存率，使用外部文件的收益就更大。如果不大可能产生完整缓存，则内联是更好的选择。
如果你的网站中的每个/很多页面都使用了相同的Javascript和CSS，使用外部文件可以提高这些组件的重用率。

**第九章  减少DNS查找**

Internet是通过IP地址来查找服务器的。由于IP地址很难记忆，通常使用包含主机名的URL来取代它，但当浏览器发送其请求时，IP地址仍然是必需的。这就是[DNS][2](Domain Name System)所处的角色。DNS将主机名映射到IP地址上，就像电话本将人名映射到他们的电话号码一样。当你在浏览器中键入github.com时，连接到浏览器的DNS解析器就会返回服务器的IP地址。

> DNS查找可以被缓存起来以提高性能

**第十章  精简JavaScript**

精简是从代码中移除不必要的字符以减小文件大小，进而改善加载时间的实践。在代码被精简后，所有的注释以及不必要的空白字符(空格、换行和制表符)都将被移除。对于JavaScript而言，这可以改善响应时间效率，因为下载的文件大小减小了。

**第十一章  避免重定向**

重定向用于将用户从一个URL重新路由到另一个URL。
在重定向完毕并且HTML文档下载完毕之前，没有任何东西显示给用户。
重定向会使你的页面变慢

> 给URL的结尾添加斜线"/"

**第十二章  移除重复脚本**

> * 在页面中多次包含相同的脚本会使页面变慢
> * 在IE中，如果脚本没有被缓存，或在重新加载页面时，会产生额外的HTTP请求
> * 在Firefox和IE中，脚本会被多次求值

**第十三章 配置或移除ETag**

实体标签(Entity Tag,[ETag][3])，是Web服务器和浏览器用于确认缓存组件的有效性的一种机制。

**第十四章  使Ajax可缓存**

确保Ajax请求遵守性能指导，尤其应具有长久的Expires头。

  [1]: http://zh.wikipedia.org/zh-cn/%E5%85%A7%E5%AE%B9%E5%82%B3%E9%81%9E%E7%B6%B2%E8%B7%AF
  [2]: http://zh.wikipedia.org/zh/%E5%9F%9F%E5%90%8D%E7%B3%BB%E7%BB%9F
  [3]: http://zh.wikipedia.org/zh-cn/HTTP_ETag
