---
layout: post
title: 百度推出CDN公共库
tags:
- BAE
- CDN
- web前端
- 百度
---
百度在之前推出了CDN公共库，同时最近google的公共库经常性的出现无法访问的问题，所以对自己站点的公共库文件进行了更新了。

百度CDN最明显的优势就是速度快，下面是测试截图：

![](/files/BAE-CDN.png)

测试结果一目了然，百度在国内的速度明显优于googlecdn，在国内的话完全可以考虑将google cdn 替换成bae的cdn服务

现在百度主要提供了一下的公共库：

* [backbone](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#backbone)
* [Bootstrap](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#Bootstrap)
* [dojo](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#dojo)
* [ext-core](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#ext-core)
* [Highcharts](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#Highcharts)
* [Highstock](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#Highstock)
* [jqMobi](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#jqMobi)
* [jQuery](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#jQuery)
* [jQuerymobile](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#jQuerymobile)
* [jQuerytools](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#jQuerytools)
* [jQueryui](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#jQueryui)
* [JSON](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#JSON)
* [lesscss](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#lesscss)
* [mootools](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#mootools)
* [prototype](http://developer.baidu.com/wiki/index.php?title=docs/cplat/libs#prototype)

<!--more-->

当然BAE的公共库也有几个问题，主要是一下两个问题：
*百度公共库目前还不支持 HTTPS 加密访问，如果有这个需求的话就需要考虑下了
*百度公共库更新不是很即时，部分库的版本还是比较老，并没有提供最新版本的cdn
比如bootstrap还停留在2.0.4，而官方版本已经是2.3.1版本

而最常用的jquery倒是提供了1.9.1的版本

如果对版本要求不严，百度的公共库的确提供了一个很好的解决方案，不管从速度和稳定性方面都不错

当然如果需要其他的cdn资源，或者对当前库的版本不满意，也可以考虑以下几个不错的cdn资源库：

* [http://www.bootstrapcdn.com/](http://www.bootstrapcdn.com/) 专为bootstrap设立的cdn服务，服务一流，值得原则
* [http://cdnjs.com/](http://cdnjs.com/) 号称拥有所有的资源库，同时包括JavaScript, CSS, SWF, images等，而且支持http/https/spdy协议，速度也比google的cdn要快，绝对是js cdn的不二选择。
