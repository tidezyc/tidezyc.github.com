---
layout: post
title: Kindeditor实现代码插入和修改
category : 前端
tags: kindeditor web web前端
published: true
---
除了CK，Kindeditor还是比较好用的在线编辑器，不过kd有个严重的问题就是插入代码之后再修改会有转码问题，最近研究了下之后，找到了很好的解决方案，分享如下：

在编辑器插入上传的时候进行二次编码转义：（来自[@红薯](http://my.oschina.net/javayou) 的解决方案）：

``` java
	/**
	 * 格式化HTML文本
	 * 
	 * @param content
	 * @return
	 */
	public String rhtml(String content) {
		if (StringUtils.isBlank(content))
			return content;
		String html = content;
		html = StringUtils.replace(html, "&amp;", "&amp;amp;");
		html = StringUtils.replace(html, "&lt;", "&amp;lt;");
		html = StringUtils.replace(html, "&gt;", "&amp;gt;");
		return html;
	}
```

2、经过的二次转义的内容在修改编辑的时候就不会存在问题了，但在显示的时候我们需要进行一次反转义工作：

``` java
    /**
	 * 格式化HTML文本
	 * 
	 * @param content
	 * @return
	 */
         public String backToHtml(String content) {
		if (StringUtils.isBlank(content))
			return content;
		String html = content;
		html = StringUtils.replace(html, "&amp;lt;", "&lt;");
		html = StringUtils.replace(html, "&amp;gt;", "&gt;");
		html = StringUtils.replace(html, "&amp;amp;", "&amp;");
		return html;
	}
```

这里要注意的就是三个replace方法的先后顺序不能乱，即要保证所有元素都只执行了一个反转义；

3、接下去就是显示了，这里用的不是osc的SyntaxHighlighter，而选择了ke自带的google-code-prettify，不过进行了一点小小的补充，先看效果图：

![](http://bcs.duapp.com/thinkjet/img%2Fkindeditor.png)

要实现以上效果，要做一点小的修改：

1、在kindeditor编辑器下的plugins/code目录下修改core.js文件以实现标号效果

``` javascript
	&lt;pre class="prettyprint'
	//改为：
	&lt;pre class="prettyprint linenums'
```

2、当然少不了导入prettify.css和prettify.js两个包

3、最后在boby里加上onload="prettyPrint()"就可以啦，当然jquery里也行，只要保证加载完执行就ok啦

4、为实现以上效果也是对prettify.css进行了一点小的修改，这就不细说啦



` public void static `

``` bash
# Fedora 19,18,17,16,15,14,13,12 & cd /etc/yum.repos.d/ & wget http://download.virtualbox.org/virtualbox/rpm/fedora/virtualbox.repo
```

<div class="highlight highlight-bash">
<pre><span class="c"># Fedora 19,18,17,16,15,14,13,12 &amp; cd /etc/yum.repos.d/ &amp; wget http://download.virtualbox.org/virtualbox/rpm/fedora/virtualbox.repo</span>
</pre>
</div>
