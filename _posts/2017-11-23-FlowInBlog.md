---
layout: post
title: 让你的GithubPage个人博客支持流程图
date: 2017-11-23
categories: blog
tags: [Github,Flowchart,blog]
description: 让你的GithubPage个人博客支持流程图
---

### GithubPage、Jekyll
jekyll+githubPage是一个非常好用的搭建个人博客的平台，通过jekyll来搭建博客系统，染后将博客系统发布到github的远程仓库,这样人们就可以在互联网上访问我们的博客了。

jekyll博文编写采用MarkDown，MarkDown是的一种轻量级标记语言，共功能强大、简单易用、上手很快。Jekyll对MarkDown有很好的支持。

如何在Github搭建jekyll博客，如何使用MarkDown并不是本文的目的，毕竟网上有很多详细的教程，我也是按照网上交互完成部署的，本文主要是纪录我自己在使用过程中遇到的一个问题及其解决方案。

### MarkDown中的流程图

很多MarkDown编辑器对流程图都有很好的支持，在Markdown中可以像这样绘制流程图:
```Text

    ```flow
    st=>start: 用户登陆
    op=>operation: 登陆操作
    cond=>condition: 登陆成功 Yes or No?
    e=>end: 进入后台
    st->op->cond
    cond(yes)->e
    cond(no)->op
    ```
```

这段类似于的代码的文本最终会渲染成这样：

```flow
st=>start: 用户登陆
op=>operation: 登陆操作
cond=>condition: 登陆成功 Yes or No?
e=>end: 进入后台
st->op->cond
cond(yes)->e
cond(no)->op
```

同样的我们也可以用文本来描述时序图：
```Text
    ```sequence
    Andrew->China: Says Hello
    Note right of China: China thinks\nabout it 
    China-->Andrew: How are you? 
    Andrew->>China: I am good thanks!
    ```
```

```sequence
Andrew->China: Says Hello 
Note right of China: China thinks\nabout it 
China-->Andrew: How are you? 
Andrew->>China: I am good thanks!
```

它们在预览时会将MarkDown文件解析成HTML，文件中的MarkDown标记会被解析成HTML标签。jekyll则是发布时会把MarkDown文件生成对应的HTML文件。

我个人常用的编辑器是VS Code，自带基本的MarkDown解析，当然如果需要对MarkDown有更多更全面的支持需要安装扩展插件，不少插件还是很好用的。不过不久前，发现了一个问题：如果文章中使用了流程图代码块，发布后并没有渲染成功。在网上查了不少资料，最后的结论是：jekyll、GithubPage不直接支持
MarkDown流程图等，不支持的原因大概是因为需要引入第三方的框架，毕竟目前大部分对MarkDown流程图的支持是基于“Flowchart.js”、“js-sequence-diagrams”、“mermaid”等等这样的前端框架基础之上。

前面说MarkDown标记会被转成HTML标签，流程图标记也不例外。像上面流程图代码块在Jekyll中会被转成以下元素：
```
    <pre><code class="language-flow">
    st=>start: 用户登陆
    op=>operation: 登陆操作
    cond=>condition: 登陆成功 Yes or No?
    e=>end: 进入后台
    st->op->cond
    cond(yes)->e
    cond(no)->op
    </code></pre>
```
Jekyll会把两个“```”之间的内容处理为格式文本块，同时指定它的样式，一般MarkDown工具都会有内置的一套样式，用来支持大多数格式文本块，尤其是代码块。这也是很多MarkDown工具支持对不同语言有不同关键字高亮，不同语言的高亮又不同配色的原因。

那么如何让描述流程图的文本块渲染成真正的流程图呢？


### 前端绘图框架在Jekyll中的使用

既然MarkDown本身对流程图的渲染使用的是前端框架，那我们来看一下在网页中我们如何使用文本来展示流程图呢？

以[js-sequence-diagrams](https://github.com/bramp/js-sequence-diagrams)为例，它的github主页介绍的使用方法如下:

```Text
	 ```
    <div class="diagram">A->B: Message</div>
    <script>
    var options = {theme: 'hand'};
    $(".diagram").sequenceDiagram(options);
    </script>
    ```
```

例子中div里的内容就是描述时序图的文本，然后通过js代码拿到样式为“diagram”的div，通过调用js-sequence-diagrams框架的方法完成渲染。这个框架使用起来非常简单了。

回到我们jekyll生成的html代码中，不同于上面例子的代码，jekyll生成的html片段是使用pre标签包裹的，样式类名是：”language-sequence“，其中”language-“是固定的前缀，后面是什么取决你在”```“后写的是什么。我们是否可以通过拿到样式名为“language-sequence”的标签调用框架方法来完成渲染呢？

在你的jekyll博客目录有一个layout文件夹，里面有一个post.html文件，这个文件是你的每篇博文的一个模板，发布时，post目录下的每篇博文(MarkDown文件)会以post.html为模板生成对应的html文件，所以我们可以在post.html中接入这些前端框架让我们的每篇文章都自动支持流程图、时序图等等。

在post.html中添加如下代码：

    <script src="/leung-c.github.io/js/jquery.min.js"></script>
    <script src="/leung-c.github.io/js/webfont.js"></script>
    <script src="/leung-c.github.io/js/snap.svg-min.js"></script>
    <script src="/leung-c.github.io/js/underscore-min.js"></script>
    <script src="/leung-c.github.io/js/sequence-diagram.min.js"></script>

    <script>
        $(".language-sequence").sequenceDiagram({theme: 'simple'});
    </script>

前面全部都是引入官方指定依赖的js库，最后一个是引入sequence-diagram.min.js自身。后面的代码块中我按照官方的例子：获取language-sequence样式类的标签，然后调用框架渲染。修改后发布一下Jekyll，就可以看到效果。这个方案确实是可行的。

对其他框架的接入大同小异，只是可能没有js-sequence-diagrams这么傻瓜氏的使用而已，比如flowchart.js，它的官方用例是这样的:

```Text
		```
    	<div id="diagram"></div>
    	<script>
        	var diagram = flowchart.parse("the code definition");//此处为流程描述代码
        	diagram.drawSVG('diagram');
    	</script>
    	<div>
    	```
```

好像不支持直接使用标签来调用它的方法完成渲染，这也意味着我们不能这么写：$(".language-flow").drawSVG();它的例子中是有一个div，和language-sequence中可以直接在div中写流程代码不同，这个div使用来承载流程图的。所以我需要拿到样式为：language-flow的标签里的内容即流程代码，创建一个新标签来完成流程图的渲染，具体代码如下:

```Text
	```
	<script>
	    function flow(name,f){
	        var chart = flowchart.parse(f);
	        chart.drawSVG(name);
	    }
	    window.onload = function (){
	        var cd = document.getElementsByClassName("language-flow");
	        for (var i = 0; i < cd.length; i++) {
	            var t = cd[i].textContent;
	            var canvas = "canvas" + i;
	            cd[i].innerHTML = "<div id=\"" + canvas + "\"></div>"
	            flow(canvas, t);
	        }
	    }
	</script>
	```
```

上面的代码即可使你的jekyll博客支持流程图，同样的方式可以让我们扩展更多的其他好用的前端框架，如mermaid等。

