---
layout: post
title:"记一次 chrome 调试"
data: 2016-04-22 23:11:41 -0001
permalink: "debug"
---

前几天发现chrome打开本来便利的后台系统，点击菜单时总是会把内容载入到全局页面，没有了navbar和左侧菜单，调试极其不方便，但是用搜狗浏览器打开确实正常，旁边几个人也不知道怎么回事，只是一直强调他们打开没有问题,不禁觉得很奇怪。下班之后决定一探究竟。

后台系统的框架大体如下，menuBar通过jquery插件ztree读取数据库菜单配置生成。

{% highlight html%}
<div class="navbar"></div>
<div class="menubar"></div>
<div class="content">
    <iframe name="mainFrame"></iframe>
</div>
{% endhighlight %}

通过chrome开发者工具比较了搜狗浏览器和chrome浏览器的菜单元素，发现二级菜单的链接元素，chrome浏览器中__target__属性丢失了，导致页面信息无法定位载入到iframe中。最外层的原因算是找到了，属性丢失有点然人摸不着头脑，因此有必要搞清楚菜单加载的过程。

菜单的层级的层级记录在数据库中，包括层级、url、名称等信息。ztree读取后台传递的json数据后根据菜单信息组装菜单元素。一级菜单首次加载后直接写入到页面，二级菜单信息展开菜单时才真正生成，生成的html格式如下：

{% highlight html %}
<li class="menubar li"><a onclick="" href="menu_url" target="mainFrame" style class="link">二级菜单</a></li>
{% endhighlight %}

ztree的主要工作是把数据组装成类似上述的html片段，jquery最终把html转换成dom节点，然后插入到事件触发的元素—-即一级菜单之中。通过调试代码可以看到执行appendChild之前，<a>标签中包含的dom片段target属性仍然存在，但是执行完b.appendChild(a)之后，从conlose中可以看到a中target已经不存在了。因此基本确定了target丢失的边界点。

{% highlight javascript %}
    //a 为生成的dom片段 b为一级菜单元素
    var b = wa(this,a);
    b.appendChild(a);
{% endhighlight %}

我首先怀疑，是不是最新的chrome浏览器在插入dom片段时对于<a>的处理有问题，（这个时候我对浏览器的能力变得有点不太信任，被IE的怪异模式蹂躏的后遗症~~）
于是我本地写了html页面，通过JS生成上述的dom片段，插入到html中，没有发现异常，target属性没有丢失。

再次我在问题页面，打开开发者工具，通过console解释器直接创建<a>标签，添加target属性，获取content元素，然后把<a>元素append到content中，神奇的事情又发生了，target属性又消失了，创建其他元素添加target属性不会出现。我猜测当前的执行环境对<a>元素执行特殊的操作，首先怀疑的就是bootStrap了..然而搜索代码没有看到特殊处理的片段。

于是我考虑把网页代码资源全部存储到本地，通过逐步去除依赖的Js文件来定位相关的Js，然而本地打开页面，加载全部Js没有发现任何问题.

看来问题还是得在原始的环境中排查.但是appendChild之后，执行的流程就交给浏览器内核了，无法看到后续的执行过程，调试到这个地方就结束了。考虑js性能工具监控工具Profile能否监控到appendChild之后的js动作呢，试了一下,console执行的JS的来源全部定位到VM中，对当前问题分析没什么帮助。

还有什么办法能看到非调试域中的JS执行情况呢，append之后dom重新绘制的过程肯定触发了一些事件。突然想到以前试验效果都不太好的Break on Attributes Modification，通过console创建包含target属性的<a>标签，打上Attibute断点，不得不说一句Chrome真心强大，在console视图中直接可以打Dom断点！！
然后执行appendChild动作，敲下enter按键，终于捕获到了！！

>>kill-evil.js

holly shit,原来是插件。

>Prevent printing, selection spying, unwanted tabs, etc.

{% highlight javascript %}

function absolve(elt) {
    elt.oncontextmenu = null;
    elt.onselectstart = null;
    elt.onmousedown = null;
    elt.oncut = null;
    elt.oncopy = null;
    elt.onpaste = null;
    if (elt.nodeName === 'A' && window.location === top.location)
        elt.removeAttribute('target');
}
{% endhighlight %}

处理起来真是粗暴...

看到停留在方法上的断点蓝条，我满意的关上显示屏，终于可以下班了..

回去的路上，复盘了一个多少小时的排查过程，首先还是庆幸没有在路径不清晰的时候把完全归结到浏览器，而是尝试确定问题的边界，更没有换个浏览器了事，毕竟还是chrome好用，无可替代。
再者，这个处理问题的时间还是偏长，第一部浏览器对比的时候只想到了内核版本差异可能造成差异，没有想到运行环境差异，搜狗可是一个插件都没装哪，而这个差异其实比前者要更大。
另外，对浏览器的调试技术还是很不熟悉，break on dom这个其实是很早的功能，但是之前几次分析其他网站Js的时候用的不是很理想，于是我基本直接放弃了这条路径，其实这个才是正常的捕获途径。这一次经历也让我对此有了更深的认识。
处理问题时其实比记录要焦躁一点，google完全看不到相关信息，很容易回到怀疑浏览器然后放弃的轨道。但是，最终还是解决了。后面用起来就顺手多了。